---
description: Resume a paused PR review pipeline from saved state
allowed-tools: Bash(gh:*), Bash(git:*), Bash(sleep:*), Bash(jq:*), Bash(cat:*), Bash(date:*), Bash(python3:*), Bash(uuidgen:*), Read, Write, Edit, Task
---

# Resume PR Review Pipeline

Resume the pipeline saved in `.claude/pr-pipeline-state.json`.

## Claim protocol

This pipeline claims PRs on GitHub so humans do not duplicate reviews. Read
`${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` NOW and follow it exactly for every
claim operation below. Claim operations are best-effort and never block core
processing.

1. Read the state file. If it does not exist, tell the user there is no pipeline to resume and suggest `/pr-review-pipeline:review <pr-numbers>`.
2. Verify `gh auth status` and that the repo matches `state.repo`.
3. Schema check: if `schema_version` is missing or 1, upgrade in place — add
   `schema_version: 2`, `run_id` (generate one), `run_status` ("paused"), and
   an empty claim object (all nulls) per queued PR, then run step 4's
   reconciliation to backfill. If `schema_version` > 2 or the file is invalid
   JSON: STOP with an error before any GitHub access.
4. Reconcile remote → local BEFORE any GitHub write, for the complete queue:
   - Live merged/closed PR facts repair terminal pipeline statuses
     (merged → `merged`; closed unmerged → `skipped_error`).
   - Supersession fence — run this check on each fetched remote claim
     payload BEFORE replacing any cached claim metadata: if any
     outstanding (non-terminal) PR's remote claim has
     `claim_state: "active"` AND a `run_id` different from this state
     file's top-level `run_id` AND a generation ≥ the locally cached
     generation (treat a null or missing locally cached generation as 0,
     so any active foreign claim supersedes a claim this run never
     recorded), mark that local claim `sync_status: "superseded"`, tell
     the user which run owns it, and STOP — this run may not continue or
     write any claims. A RELEASED foreign claim is not a fence violation:
     report it and continue.
   - After the fence passes: each PR's canonical claim comment (protocol:
     Finding the claim comment) replaces cached claim metadata — owner,
     run_id, generation, timestamps, state — and its comment id is cached
     into `claim.comment_id`.
   - Queue order, configuration, and nonterminal execution statuses stay
     as the local file says.
5. Re-stamp every outstanding claim you own — a claim is yours ONLY if its
   post-fence `run_id` equals this state file's top-level `run_id`; never
   edit or release a claim whose `run_id` differs, regardless of its
   generation or state — (updated_at = now, stale_at =
   now + max_wait_min, current pipeline_status; fence first), set
   `run_status: "active"`, and resume the pipeline exactly as defined in
   `/pr-review-pipeline:review` at the appropriate step (same mapping as
   before: paused/waiting/changes_requested → activity check;
   reviewing/pending → Step 1; approved → retry merge). All transition and
   release rules from `/review` apply, including claim refresh and terminal
   releases.

$ARGUMENTS
