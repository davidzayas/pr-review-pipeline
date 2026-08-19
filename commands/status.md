---
description: Show the current PR review pipeline status
allowed-tools: Bash(gh:*), Bash(cat:*), Bash(jq:*), Read
---

# Pipeline Status

1. Read `.claude/pr-pipeline-state.json`. If missing, check for `.claude/pr-pipeline-state.last.json` (a completed run) and report that; otherwise say no pipeline exists.
2. For each PR in the queue, fetch live data (`gh pr view <PR> --json state,reviewDecision,headRefOid,statusCheckRollup,title,author`) and print a compact table:

| # | PR | Title | Author | Pipeline status | Live state | Checks | Notes |

3. Highlight the PR currently blocking the pipeline (first non-merged entry), what it is waiting on, and how to continue (`/pr-review-pipeline:resume`).

$ARGUMENTS
