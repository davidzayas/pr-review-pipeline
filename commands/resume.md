---
description: Resume a paused PR review pipeline from saved state
allowed-tools: Bash(gh:*), Bash(git:*), Bash(sleep:*), Bash(jq:*), Bash(cat:*), Read, Write, Edit, Task
---

# Resume PR Review Pipeline

Resume the pipeline saved in `.claude/pr-pipeline-state.json`.

1. Read the state file. If it does not exist, tell the user there is no pipeline to resume and suggest `/pr-review-pipeline:review <pr-numbers>`.
2. Verify `gh auth status` and that the repo matches `state.repo`.
3. Find the first PR in `queue` whose status is not `merged` or `skipped_error`. That PR is the current head of the pipeline.
4. Re-fetch its current state from GitHub and continue the pipeline exactly as defined in `/pr-review-pipeline:review`, starting at the appropriate step:
   - `paused` / `waiting` / `changes_requested` → check for new activity since `head_sha_at_review`; if there is activity, re-review (Step 1 of the pipeline); otherwise resume polling (Step 5).
   - `reviewing` / `pending` → start review from Step 1.
   - `approved` (but not merged) → retry the merge (Step 3.2).
5. Continue through the remaining queue until complete, following all the same rules (strict order, polling with the saved `poll_interval_min` / `max_wait_min`, state file updates after every transition).

$ARGUMENTS
