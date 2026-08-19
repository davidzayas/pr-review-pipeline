---
description: Show the current PR review pipeline status
allowed-tools: Bash(gh:*), Bash(cat:*), Bash(jq:*), Read, Write, Edit
---

# Pipeline Status

Read `${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` for how to find and parse
claim comments. **This command NEVER writes to GitHub** — no comments,
labels, reviews, or merges. It may repair the LOCAL state file only.

1. Read `.claude/pr-pipeline-state.json`. If missing, check for
   `.claude/pr-pipeline-state.last.json` (a completed/cancelled run) and
   report that; otherwise say no pipeline exists.
2. For each PR in the queue, fetch live data
   (`gh pr view <PR> --json state,reviewDecision,headRefOid,statusCheckRollup,title,author,labels`)
   and its canonical claim comment. If live data cannot be fetched, show
   those fields as `unknown` and make no repairs for that PR.
3. Local-only repairs (save the state file after):
   - Live merged/closed facts repair terminal pipeline statuses.
   - Remote claim payload (owner, run_id, generation, timestamps, state)
     replaces cached claim metadata. NEVER copy a claim comment's
     nonterminal pipeline_status into the core status field.
4. Print a compact table:

| # | PR | Title | Author | Pipeline status | Live state | Checks | Claim (owner/gen) | Claim sync | Stale? | Notes |

   `Stale?` = yes / no / unknown per the protocol's staleness rule. Flag
   LOUDLY: any ACTIVE claim past its stale_at (a released claim is never
   stale — likely abandoned run, suggest
   `/pr-review-pipeline:resume` or `/pr-review-pipeline:cancel`), any
   label/comment disagreement (orphaned label, or active claim missing its
   label), any claim owned by a different run_id (superseded), and any
   sync_status of failed/partial.
5. Highlight the PR currently blocking the pipeline (first non-merged,
   non-skipped entry), what it is waiting on, and how to continue.

$ARGUMENTS
