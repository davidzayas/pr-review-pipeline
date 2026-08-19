---
description: Run the PR review pipeline over an ordered list of PR numbers
argument-hint: <pr-number> [pr-number...] [--merge-method squash|merge|rebase] [--poll-interval <min>] [--max-wait <min>]
allowed-tools: Bash(gh:*), Bash(git:*), Bash(sleep:*), Bash(jq:*), Bash(cat:*), Bash(mkdir:*), Read, Write, Edit, Glob, Grep, Task
---

# PR Review Pipeline

You are running an automated PR review pipeline. Process the PRs given in `$ARGUMENTS` **strictly in the order provided**. Do not skip ahead: a PR that is not approved blocks the pipeline until it is approved and merged.

## Claim protocol

This pipeline claims PRs on GitHub so humans do not duplicate reviews. Read
`${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` NOW and follow it exactly for every
claim operation below. Claim operations are best-effort and never block core
processing.

## Arguments

`$ARGUMENTS` contains PR numbers in priority order, plus optional flags:

- `--merge-method` — `squash` (default), `merge`, or `rebase`
- `--poll-interval` — minutes between polls while waiting on a blocked PR (default: 5)
- `--max-wait` — total minutes to poll a blocked PR before pausing the pipeline (default: 240)

If no PR numbers were provided, ask the user for them and stop.

## Setup

1. Verify prerequisites: run `gh auth status` and confirm the current directory is a git repo with a GitHub remote (`gh repo view --json nameWithOwner`). If either fails, tell the user what to fix and stop.
2. If `.claude/pr-pipeline-state.json` exists with `run_status: "active"` or
   `"paused"`, STOP: tell the user a run already exists (show its queue and
   status) and that they must `/pr-review-pipeline:resume` it or
   `/pr-review-pipeline:cancel` it first. Never overwrite an active run.
3. Generate `run_id` per the protocol. The state file is NOT created here — it
   is created in step 4 of the Claim preflight below, after claim
   classification (and takeover confirmation, if needed) succeeds. This is
   its schema:

```json
{
  "schema_version": 2,
  "repo": "<owner/repo>",
  "run_id": "<run-id>",
  "run_status": "active",
  "merge_method": "squash",
  "poll_interval_min": 5,
  "max_wait_min": 240,
  "queue": [
    {
      "pr": 123,
      "status": "pending",
      "head_sha_at_review": null,
      "notes": "",
      "claim": {
        "comment_id": null,
        "owner": null,
        "run_id": null,
        "generation": null,
        "claimed_at": null,
        "updated_at": null,
        "stale_at": null,
        "state": null,
        "sync_status": null,
        "last_error": null
      }
    }
  ],
  "started_at": "<iso timestamp>",
  "updated_at": "<iso timestamp>"
}
```

Statuses: `pending`, `reviewing`, `changes_requested`, `waiting`, `approved`, `merged`, `paused`, `skipped_error`.
Run statuses: active, paused, completed, cancelled.

Update this file after **every** status change so `/pr-review-pipeline:resume` and `/pr-review-pipeline:status` work at any time.

## Claim preflight and queue-wide claiming

Do this BEFORE reviewing any PR, and BEFORE creating the state file's claims:

1. Ensure the `pr-pipeline` label exists (protocol: Label management).
2. For EVERY queued PR, fetch live state and find any existing claim comment
   (protocol: Finding the claim comment). Classify per the protocol's
   Generation fence section: missing/released → claim; stale or unparseable →
   claim only per its rules; active foreign non-stale → collect into a
   conflict list.
3. If the conflict list is non-empty, present ONE consolidated takeover
   confirmation listing every conflicted PR, its owner, run_id and stale_at.
   If the user declines: stop entirely — no state file, no GitHub changes.
4. Only after classification (and confirmation, if needed): create the state
   file, then claim every queued PR — create or edit its comment to the
   active template (`pipeline_status` = its queue status, `stale_at` = now +
   max_wait_min) and add the label. Record `comment_id` and sync results per
   the protocol. Failures are recorded and skipped — never block.
5. Begin strict-order review only after every claim attempt has finished.

## Pipeline loop — for each PR in order

### Step 1: Gather PR context

```bash
gh pr view <PR> --json number,title,body,author,state,isDraft,mergeable,mergeStateStatus,headRefName,baseRefName,headRefOid,reviewDecision,statusCheckRollup,files,additions,deletions
gh pr diff <PR>
gh api repos/{owner}/{repo}/pulls/<PR>/comments --paginate   # review comments incl. Copilot
gh api repos/{owner}/{repo}/pulls/<PR>/reviews --paginate
gh pr view <PR> --json comments                               # issue-level comments
```

Skip with status `skipped_error` (and tell the user) if the PR is closed or already merged — and release its claim (reason: skipped_error) if one was created. If it is a draft, treat it as blocked (Step 5) — drafts are not reviewed until marked ready.

### Step 2: Review the code

Use the `pr-reviewer` agent (Task tool, subagent_type `pr-review-pipeline:pr-reviewer`) to perform the review. Give it the PR number, title, body, diff, and all comments. It returns a structured verdict: `APPROVE` or `REQUEST_CHANGES`, with findings.

The review must cover:

- **Correctness**: logic errors, edge cases, error handling
- **Copilot comments**: every unresolved GitHub Copilot review comment must be evaluated. For each one, decide: (a) valid and already fixed, (b) valid and must be fixed → contributes to REQUEST_CHANGES, or (c) not applicable → explain why in the review
- **Security**: injection, secrets, authz/authn issues in changed code
- **Tests**: are changes covered? Do CI checks pass (`statusCheckRollup`)?
- **Failing or pending required checks**: never approve/merge over failing required checks; pending checks mean wait (Step 5)

### Step 3: If verdict is APPROVE

1. Post the approval:

```bash
gh pr review <PR> --approve --body "<review summary: what was checked, disposition of each Copilot comment, why it is safe to merge>"
```

2. Merge with the configured method:

```bash
gh pr merge <PR> --squash --delete-branch   # or --merge / --rebase per config
```

If the merge fails because the branch is behind or has conflicts:
- Behind base: `gh pr update-branch <PR>`, then wait for checks (poll per Step 5 rules) and retry the merge once checks pass.
- Real conflicts: treat as REQUEST_CHANGES with a comment explaining the conflict (Step 4).

3. Set status `merged` and record it.
4. Release this PR's claim (reason: merged) per the protocol, then refresh remaining claims.
5. Move to the next PR.

### Step 4: If verdict is REQUEST_CHANGES

1. Post the review with an @mention of the author so they are notified and clearly assigned:

```bash
gh pr review <PR> --request-changes --body "<body>"
```

The body must contain:
- Opening line: `@<author-login> changes are requested on this PR — please address the items below.`
- A numbered list of required changes. For each: the file and line, what is wrong, and **concretely how to resolve it** (suggested code where practical, using ```suggestion blocks in follow-up line comments if precise)
- The disposition of each Copilot comment (fix required / already addressed / not applicable and why)

2. Also assign the author: `gh pr edit <PR> --add-assignee <author-login>` (ignore failure if not permitted).
3. Record `head_sha_at_review` = current `headRefOid`, set status `changes_requested`, then go to Step 5.

### Step 5: Wait for a blocked PR (polling)

The pipeline **does not advance** past a blocked PR. Poll it:

1. Set status `waiting`. Loop up to `max_wait_min` total:
   - `sleep <poll_interval_min * 60>`
   - Re-fetch: `gh pr view <PR> --json headRefOid,isDraft,state,reviewDecision,statusCheckRollup`
   - **Ready for re-review** when: new commits exist (`headRefOid` != `head_sha_at_review`), or the author replied/resolved the review threads, or a draft became ready. When ready → go back to Step 1 for this same PR (full re-review of the new state).
   - PR closed by author → mark `skipped_error`, inform the user, continue to next PR.
2. If `max_wait_min` elapses with no activity: set status `paused`, save state, and stop. Tell the user exactly this: which PR is blocked, why, and that they can continue later with `/pr-review-pipeline:resume`.

## Completion

When every PR in the queue is `merged` (or `skipped_error`), print a final summary table: PR, title, verdict history, outcome, and any skipped PRs with reasons. Then delete or archive the state file (`.claude/pr-pipeline-state.json` → `.claude/pr-pipeline-state.last.json`). All claims should already be released by their terminal transitions; verify none remain active (best-effort), set run_status: "completed", then archive the state file as before.

## Rules

- Never force-merge, never override branch protections, never approve with failing required checks.
- Never push commits to the PR author's branch — feedback only.
- Keep all GitHub-posted text professional and specific; no filler.
- Update the state file after every transition.
- If `gh` returns an auth/permission error, pause (status `paused`) and tell the user rather than retrying blindly.
- After EVERY per-PR status transition (persist it to the state file FIRST),
  refresh every outstanding active claim you own with one shared transition
  timestamp: `updated_at` = now, `stale_at` = now + max_wait_min,
  `pipeline_status` = that PR's current status (fence first, per protocol).
- On `merged` or `skipped_error`, RELEASE that PR's claim (protocol: Release,
  reason = the status) before moving on; then refresh the remaining claims.
- Entering `paused` re-stamps every outstanding claim without releasing.
  Set top-level `run_status` accordingly on pause ("paused") and completion
  ("completed").

Begin now with the arguments: $ARGUMENTS
