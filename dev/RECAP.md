# Directory-based auto-approve PoC — recap

## Goal

Prove out a GitHub Actions pattern: a bot auto-approves PRs that touch only
`dev/`/`int/`, and does nothing for PRs touching `stg/`/`prd/` (leaving those
to normal human review). Modeled on a friend's existing (broken) setup at
another company, to diagnose why his wasn't working and produce a known-good
reference.

## Architecture

- **`RoloTomassi/autoapprove-workflows`** (public) — the reusable workflow
  containing the actual approve/no-op logic.
- **`RoloTomassi/autoapprove-poc`** (public) — the "code repo," with
  `dev/`, `int/`, `stg/`, `prd/` directories and a small stub workflow that
  calls the reusable one.
- **GitHub App** `rolotomassi-autoapprove-bot` (App ID `5017660`) — mints a
  short-lived installation token used to submit the approval. Chosen over a
  machine-user PAT for scoped, non-standing credentials.
- **`ShinjiNakamoto-fd`** — a real second account, standing in for "a human
  reviewer who isn't the PR author." Used deliberately instead of a
  placeholder/fake username, since an invalid CODEOWNERS entry risks being
  silently skipped rather than actually enforced — a fake owner could have
  given a false-positive result.
- Branch protection via a **repository ruleset** on `autoapprove-poc`:
  `required_approving_review_count: 1`, `require_code_owner_review: true`,
  `dismiss_stale_reviews_on_push: true`, `require_last_push_approval: true`.

## Bugs found and fixed

### 1. Caller workflow's permission ceiling too low

**Symptom:** every run failed instantly (`startup_failure`, 0s, no logs).

**Exact error** (recovered by scraping the Actions run page — not exposed via
`gh run view` or the Checks API):

> Invalid workflow file: `.github/workflows/autoapprove.yml (Line: 7, Col: 3)`.
> Error calling workflow `'RoloTomassi/autoapprove-workflows/.github/workflows/autoapprove.yml@main'`.
> The nested job 'autoapprove' is requesting `'pull-requests: write'`, but is
> only allowed `'pull-requests: none'`.

**Cause:** a reusable workflow's job-level `permissions:` can never exceed
what the *calling* workflow is allowed. The stub never declared its own
`permissions:` block, so it inherited the repo's read-only default token
permission.

**Fix:** added an explicit `permissions: { pull-requests: write, contents: read }`
block to the caller (`autoapprove-poc/.github/workflows/autoapprove.yml`).

---

### 2. Blanket `*` CODEOWNERS pattern blocks the bot on every path

This is the bug we set out to reproduce from the friend's setup:
`* @org/team1 @org/team2` in CODEOWNERS, with "Require review from Code
Owners" enabled.

We reproduced this twice. The first time (PR #3) was blocked by *two*
overlapping bugs at once (this one, plus bug #3 below, which we hadn't found
yet) — the API evidence was solid but not fully isolated:

```
reviews:        [{ author: "rolotomassi-autoapprove-bot", state: "APPROVED" }]
reviewDecision: "REVIEW_REQUIRED"
mergeStateStatus: "BLOCKED"
```

After fixing bug #3, we deliberately re-reverted CODEOWNERS to the blanket
`*` pattern (PR #7) and reran the test (PR #8) to get a clean, isolated
capture with only this bug present. Exact text from the PR page:

> rolotomassi-autoapprove-bot[bot] approved these changes
> Awaiting requested review from ShinjiNakamoto-fd
> ShinjiNakamoto-fd is a code owner
> At least 1 approving review is required to merge this pull request.

Notably, the bot's approval this time had **no "read-only permissions"
caveat** — confirming bug #3's fix held, and this block was coming from the
code-owner requirement alone.

**Cause:** `*` matches every file, so `require_code_owner_review` applies to
`dev/`/`int/` too. A GitHub App bot can never be a valid CODEOWNERS entry
(Apps can't be added to Teams and can't be listed individually) — so no
matter how the bot is configured, it can never satisfy a code-owner
requirement it's subject to.

**Fix:** scoped CODEOWNERS to only `/stg/` and `/prd/` (PR #4, and restored
again via PR #9 after the documentation re-test). `dev/`/`int/` paths now
have no assigned owner at all, so the code-owner check doesn't apply to them
— only the bot's own approval is needed there.

**Related quirk found along the way:** the first time we fixed CODEOWNERS,
an *already-open* test PR (predating the fix) stayed blocked with the same
"Awaiting requested review" message, even though the file it touched no
longer matched any CODEOWNERS pattern. GitHub had generated the review
request against the old blanket pattern before the fix landed, and doesn't
retroactively cancel it just because the base branch's CODEOWNERS changed
later. A **fresh PR** cut after the fix never generated that request in the
first place and was unaffected. (The other lever, not needed here:
explicitly removing the stale requested reviewer via the API/UI.)

---

### 3. Read-only access doesn't count toward required reviews — for humans *and* for the bot

**Symptom A (human):** `ShinjiNakamoto-fd` had read-only collaborator access
(enough for GitHub to treat it as a *valid* CODEOWNERS entry) and submitted a
real `APPROVED` review, but:

```
reviewDecision: "REVIEW_REQUIRED"
```

**Fix:** bumped `ShinjiNakamoto-fd` to write access
(`gh api repos/.../collaborators/ShinjiNakamoto-fd -X PUT -f permission=write`).
The *existing* review recounted immediately — no re-approval needed.
`reviewDecision` flipped to `"APPROVED"` within seconds.

**Symptom B (the bot):** even after fixes #1 and #2, a clean `dev/`-only PR
still showed:

> Review required — At least 1 approving review is required by reviewers
> with write access.
>
> rolotomassi-autoapprove-bot[bot] — **Approved these changes with read-only
> permissions**
>
> Merging is blocked — New changes require approval from someone other than
> the last pusher.

despite the App having `Pull requests: Read and write` correctly configured
(confirmed by checking the App's installation settings directly — its
`Contents` permission was set to `No access`, not merely read-only; the
"read-only permissions" wording in GitHub's UI appears to be a simplified
label for "less than write," not a literal readout of the configured level).

**Cause:** GitHub computes an App's "role" for review-counting purposes off
its broader repository access (centered on `Contents`), not the
`Pull requests` scope specifically. The App only had `Metadata: read`
(automatic) and `Pull requests: write` — nothing on `Contents` — so GitHub
treated its overall standing as read-only for this purpose, even though
`Pull requests: write` was enough for it to *submit* a review at all.

**Fix:** granted the App `Contents: Read and write` and re-accepted the
updated installation permissions. Same as with the human case, the
*existing* bot review recounted live — `reviewDecision` flipped to
`"APPROVED"`, `mergeStateStatus` to `"CLEAN"`, no new PR needed.

## Final confirmed behavior

**`dev/`-only PR** — bot approves, review counts, no CODEOWNERS match,
merges cleanly on the bot's approval alone.

**`stg/`-only PR** — bot correctly does nothing. Exact log line from the
`evaluate` step:

```
Decision: approve=false (no whitelist match: stg/README.md)
```

PR stays blocked, requiring `ShinjiNakamoto-fd`'s real review via CODEOWNERS
— exactly the safety property the design is meant to guarantee.

## Checklist for the friend's setup

Given his bot is confirmed to also be a GitHub App (shows as `[bot]` in PR
reviews), both root causes here were plausible independently — but one is
now ruled out:

- [ ] Is `Require review from Code Owners` enabled, and does his CODEOWNERS
      use a blanket `*` (or any pattern matching dev/int paths)? **Most
      likely culprit** — his bug reproduces exactly what we saw in bug #2.
- [x] ~~Does his bot's App installation have `Contents` permission, or only
      `Pull requests`?~~ Checked: his App shows "Read and write access to
      code and pull requests" — `Contents` ("code" in GitHub's App
      permission UI) is already granted. Bug #3 is very likely **not** part
      of his problem; his bot's approval should already count toward the
      required-review number. The blanket CODEOWNERS pattern (bug #2) is
      the much stronger suspect for his setup.
