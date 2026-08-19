---
description: Cancel the current PR review pipeline run and release all its claims
allowed-tools: Bash(gh:*), Bash(git:*), Bash(jq:*), Bash(cat:*), Bash(date:*), Bash(python3:*), Bash(uuidgen:*), Read, Write, Edit
---

# Cancel PR Review Pipeline

Cancel the run saved in `.claude/pr-pipeline-state.json` and release its
GitHub claims. Read `${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` and follow it
for every claim operation.

1. Read the state file. If it does not exist, tell the user there is nothing
   to cancel. If `run_status` is already `completed` or `cancelled`, say so
   and stop.
2. Confirm with the user: show the queue and each PR's status, and ask
   whether to cancel the run (releasing all its claims). Declined = stop,
   nothing changes.
3. Verify `gh auth status`. If GitHub is unreachable, skip to step 5 —
   cancellation still succeeds locally; say clearly that remote claims were
   NOT released and will go visibly stale on their own.
4. For each queued PR (regardless of its cached claim state — a desynced or
   legacy state file must not hide a live remote claim): fetch the canonical
   claim comment, apply the generation fence (protocol), and release only if
   the remote run_id and generation match local state (reason: `cancelled`).
   A superseded claim (newer owner) is left untouched and marked
   `sync_status: "superseded"`. Record every success, mismatch, or failure
   in the state file; failures never abort the loop.
5. Set `run_status: "cancelled"` and `cancelled_at` = now. Archive the state
   file to `.claude/pr-pipeline-state.last.json` (removing the original)
   REGARDLESS of release failures.
6. Print a summary: per PR — released / superseded (left for its new owner) /
   failed (will go stale; may be cleaned up manually per the claim comment's
   own instructions).

$ARGUMENTS
