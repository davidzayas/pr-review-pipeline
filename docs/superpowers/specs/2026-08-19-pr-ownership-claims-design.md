# PR Ownership Claims — Design

Ideation: gpt-5.6-sol via codex MCP · Facilitation: Claude

Date: 2026-08-19
Status: Approved section-by-section by the maintainer during GPT-led brainstorming.

## Purpose

Make pipeline ownership of queued PRs visible and durable on GitHub — so humans
don't duplicate reviews — without weakening review or merge safety, and with the
abandoned-run case handled first-class.

## Settled decisions (made by the maintainer before/during ideation)

1. Claim ALL queued PRs at queue time, not review time.
2. Release only on terminal states (`merged`/`skipped_error`) — claims persist
   through `waiting` and `paused`.
3. One claim comment per PR, edited in place at every transition,
   self-describing with timestamps and its own staleness deadline derived from
   `--max-wait`, including instructions for humans on when a claim may be
   ignored.
4. A `/cancel` command releases all claims and archives state.
5. No automatic release based on staleness by other runs — stale claims are
   flagged loudly and removed only by explicit human action, `/cancel`, or a
   fresh `/review` reclaiming.
6. All claim operations are best-effort, never blocking the core pipeline.
7. When `/review` finds a non-stale claim from another run, it requires
   explicit takeover confirmation.
8. `/status` may repair local state only (correct the state file to match
   GitHub reality); it never writes to GitHub.

## Measurable success criteria

Every queued PR is claimed before reviewing begins; active conflicts require
takeover confirmation; transitions update one durable comment; paused claims
expose a staleness deadline; terminal states and `/cancel` release claims;
`/status` detects divergence and may repair only local state; claim failures
never stop normal processing.

## Risky assumptions

Maintainers can edit the existing claim comment (holds for anyone with write
access, which pipeline operators need to merge); hidden markers remain
discoverable; timestamps are sufficiently consistent; and generation checks
adequately mitigate — but cannot eliminate — the race window created by
GitHub's non-atomic APIs (no compare-and-swap).

## Selected approach

**A. Split authority with claim generations** (selected over B
"local-authoritative projection" and C "GitHub-authoritative claims").

Local state owns queue order, configuration, and execution progress. GitHub
owns observable PR facts and claim ownership. A stable marker, run ID,
generation, claimant, status, timestamps, and staleness deadline live in the
single claim comment. Reconciliation uses explicit field-level authority;
takeovers increment the generation, preventing an abandoned run from
overwriting a newer owner.

## Architecture

The feature adds a GitHub-visible ownership layer without changing the strict,
locally driven execution pipeline.

- **Local authority:** `.claude/pr-pipeline-state.json` remains authoritative
  for queue order, configuration, and nonterminal execution progress.
- **GitHub authority:** Live PR data is authoritative for merged/closed status;
  the claim comment is authoritative for claim owner, run ID, generation, and
  lease timestamps.
- **Projection:** The `pr-pipeline` label marks an active claim. One durable
  issue comment per PR — identified by a versioned hidden HTML marker — is
  reused across runs and edited rather than recreated.

Each run receives a unique `run_id`. Each takeover increments the PR's
`claim_generation`; resume retains the existing generation. This prevents an
abandoned older run from silently overwriting a newer claim. A non-stale claim
from another run requires explicit takeover confirmation. A stale claim is
loudly disclosed and may be reclaimed by intentionally including the PR in a
fresh `/review`; it is never automatically released.

The comment records the claimant, run ID, generation, pipeline status,
claimed/updated timestamps, `stale_at` derived from `--max-wait`, and human
instructions. All outstanding claims are refreshed on pipeline transitions so
queued PRs do not appear abandoned while earlier PRs are progressing. Entering
`paused` re-stamps every outstanding claim.

Reconciliation follows field-level authority:

- `/review` and `/resume` read remote ownership first, repair local claim
  metadata, then project subsequent local transitions to GitHub.
- `/status` may repair only the local file from live terminal PR facts and
  remote claim metadata; it never writes to GitHub.
- Terminal `merged`/`skipped_error` transitions and `/cancel` edit the comment
  to a released outcome and remove the label. Historical comments remain.

Claim writes are best-effort. The pipeline completes a queue-wide claim attempt
before reviewing the first PR, records synchronization failures, and continues
without weakening ordering, review, or merge safety.

## Components

| Component | Responsibility |
|---|---|
| `commands/review.md` | Rejects overwriting an existing local active run; preflights every queued PR's claim before mutation; consolidates active conflicts into one takeover confirmation; initializes `run_id`; claims all PRs before reviewing; refreshes outstanding claims after every transition. |
| `commands/resume.md` | Reconciles live PR and claim data into local state, verifies generation ownership, re-stamps outstanding claims, then resumes. A newer remote generation fences off the old run. |
| `commands/status.md` | Fetches live PR and claim data, repairs local terminal facts and claim metadata, and displays owner, generation, synchronization, and staleness. It never changes GitHub. |
| `commands/cancel.md` | New command. Releases every active claim matching the local run and generation, records failures, marks the run cancelled, and archives state even when release attempts fail. It never releases a newer owner's claim. |
| `agents/pr-reviewer.md` | Unchanged. It reviews code only and remains unaware of ownership mechanics. |
| `README.md` | Documents `/cancel`, claim visibility, takeover behavior, staleness semantics, and best-effort limitations. |

The state file gains:

- Top-level `schema_version`, `run_id`, and `run_status` (`active`, `paused`,
  `completed`, or `cancelled`).
- Per-PR `claim` data: `comment_id`, `owner`, `run_id`, `generation`,
  `claimed_at`, `updated_at`, `stale_at`, `state` (`active` or `released`),
  `sync_status` (`synced`, `partial`, `failed`, or `superseded`), and
  `last_error`.
- Cancellation metadata at run level; existing per-PR pipeline statuses remain
  unchanged.

The canonical issue comment contains a visible ownership summary plus a
versioned hidden marker and machine-readable payload. Required payload fields:

`repo`, `pr`, `owner`, `run_id`, `generation`, `pipeline_status`,
`claim_state`, `claimed_at`, `updated_at`, `stale_at`, `released_at`, and
`release_reason`.

The visible text tells humans who owns the PR, the current pipeline state, when
the claim becomes stale, and that staleness permits ignoring or explicitly
reclaiming the claim but does not release it automatically.

The generic `pr-pipeline` label represents only an active claim. Release edits
the comment to preserve history and removes the label. Comment lookup uses the
cached ID first, then searches for the versioned marker; an existing canonical
comment is reused across generations.

All command prompts implement the same generation fence: normal refresh or
release may edit a claim only when its remote run ID and generation match local
state. Only a confirmed `/review` takeover may advance the generation.

## Data flow

**`/review` startup**

1. Validate arguments, authentication, repository, and absence of an active
   local run.
2. Fetch every queued PR's live state, canonical claim comment, and label
   before making GitHub changes.
3. Classify claims:
   - Missing or released: create/reuse with the next generation.
   - Stale: warn loudly and reclaim with the next generation.
   - Active from another run: request one consolidated takeover confirmation.
4. If takeover is declined, stop without initializing state or changing claims.
5. Create local state with a new `run_id`, then attempt claims for the entire
   queue. New comments start at generation 1; reused comments increment their
   generation.
6. Record each synchronization result locally. Only after all claim attempts
   finish does strict-order review begin.

**Pipeline transitions**

For every status transition:

1. Persist the core transition locally first.
2. Use one transition timestamp to refresh every outstanding matching claim:
   - `updated_at = transition timestamp`
   - `stale_at = transition timestamp + max_wait`
   - `pipeline_status = current per-PR status`
3. Persist claim synchronization results locally.

For `merged` or `skipped_error`, the affected comment is instead updated to
`claim_state: released`, given a release reason and timestamp, and its label is
removed. Remaining claims are refreshed. Entering `paused` refreshes all
outstanding claims without releasing them.

**`/resume`**

1. Load state and fetch live PR and claim data for the complete queue.
2. Apply remote-to-local reconciliation before any GitHub write:
   - Live merged/closed facts repair terminal pipeline status.
   - Remote claim ownership, generation, and timestamps replace cached claim
     metadata.
   - Queue order, configuration, and nonterminal execution status remain
     locally authoritative.
3. If a newer remote generation owns any outstanding PR, mark that local claim
   `superseded` and stop; the old run cannot overwrite it.
4. Otherwise, re-stamp matching active claims and resume at the first
   nonterminal PR.

**`/status`**

`/status` performs the same read-side reconciliation and saves permitted local
repairs. It never copies a claim comment's nonterminal `pipeline_status` into
core execution state and never writes to GitHub. It then reports ownership,
generation, deadline, staleness, label/comment disagreement, synchronization
failures, and the blocking PR.

**`/cancel`**

1. Read the active state and fetch each outstanding remote claim.
2. Release only claims whose run ID and generation still match local state;
   superseded claims remain untouched.
3. Record every success, mismatch, or failure.
4. Set `run_status: cancelled`, add `cancelled_at`, and archive to
   `.last.json` regardless of best-effort release failures.

The canonical comment — not the label — determines remote ownership. The label
is only a visible active-claim projection. A missing label therefore produces
`partial` synchronization, while a newer canonical comment establishes
supersession.

## Error handling

Claim failures are degraded-mode events; core state durability, ownership
fencing, and existing merge safety remain hard guards.

| Condition | Required behavior |
|---|---|
| Claim comment/label read fails | Record `sync_status: failed`, warn that ownership could not be verified, and continue unclaimed. Do not infer absence or staleness. |
| Comment create/edit or label add/remove fails | Record the operation and error, continue the pipeline, and retry only on the next transition or command invocation. |
| Active foreign claim | Require explicit takeover confirmation. Declining causes no state or GitHub mutations. |
| Stale claim | Warn prominently and reclaim only through the intentional fresh `/review`; never release automatically. |
| Newer or different generation during `/resume` | Mark the local claim `superseded` and stop before GitHub writes. The old run must never overwrite the new owner. |
| Generation changes during a write | Re-fetch before editing and verify after editing. On mismatch, stop further claim writes for that PR and mark it `superseded`. GitHub's lack of compare-and-swap leaves a documented residual race. |
| Missing cached comment | Search by marker. If none exists, create a replacement; an orphaned label is reported but is not treated as ownership authority. |
| Duplicate marked comments | Never create another. Prefer the cached matching ID, otherwise the highest generation and latest update; flag duplicates for manual cleanup. |
| Malformed or mismatched payload | Treat ownership as unknown and non-stale. Initial takeover requires confirmation; an active run stops claim writes for that PR but continues core processing. |
| Missing/invalid `stale_at` | Report staleness as unknown and never auto-classify the claim as stale. |
| Terminal release fails | Preserve the terminal pipeline result, archive the failure, and leave the remote claim to become visibly stale. |
| `/status` cannot fetch live data | Retain cached local values, display live fields as unknown, and make no speculative repairs. |
| `/cancel` lacks GitHub access | Record release failures and archive the cancelled run; cancellation does not claim that remote cleanup succeeded. |

Before every claim mutation, the command compares remote `run_id` and
`generation` with local state. Only a confirmed `/review` takeover may
intentionally replace them. Semantic ownership guards — takeover confirmation
and supersession — are distinct from best-effort API failures and may stop a
run.

Local-state failures are blocking because the file is the execution authority.
Invalid JSON, an unsupported future schema, repository mismatch, or inability
to persist a transition stops or pauses before further core side effects.
Recognized legacy state is upgraded locally; missing claim metadata is
populated through a queue-wide reconciliation pass. If a GitHub side effect
occurred before a state write failed, the next `/resume` must fetch live
reality before retrying it.

Existing core rules remain unchanged: authentication or permission failures
affecting review/merge operations pause the pipeline; required-check failures
never permit approval or merge; branch-behind and conflict handling follow the
baseline workflow. Claim-specific permission failures do not trigger that pause
when core GitHub operations remain available.

## Testing

Testing uses a disposable GitHub sandbox repository and real `gh` operations. A
thin external shell script may provision PR fixtures and assert resulting
comments, labels, and state files, but no test harness ships with the plugin.

Test fixtures should include passing, failing-check, draft, conflicting,
externally closed, and author-updated PRs. Use two maintainer identities for
takeover tests and a restricted identity for permission failures. Short polling
values keep pause scenarios practical.

| Scenario | Required result |
|---|---|
| Three-PR queue startup | All three receive claim attempts before the first review; each has one canonical comment and active label. |
| Normal transitions | Comment IDs remain stable; all outstanding deadlines refresh; strict queue order remains intact. |
| Pause/resume | Claims persist, pause re-stamps them, and resume retains run ID and generation. |
| Stale abandonment | `/status` flags loudly without remote mutation or release; fresh `/review` increments generation. |
| Active conflict declined | No state or GitHub mutation occurs. |
| Active conflict accepted | One confirmation covers the queue; affected generations increment. |
| Old run after takeover | `/resume` marks it superseded and performs no claim writes. |
| Merge or skipped error | Comment records release and the label is removed before advancing. |
| `/cancel` | Matching claims are released, newer generations remain untouched, and state is archived. |
| `/status` reconciliation | External merge/close and remote claim changes repair local state; comment bodies, timestamps, and labels remain byte-for-byte unchanged. |
| Claim permission/API failure | Failure is recorded and visible while core review processing continues. |
| Core auth/permission failure | Pipeline pauses according to existing safety rules. |
| Malformed, missing, or duplicate comments | Conservative ownership handling follows the approved error rules and creates no additional duplicate. |
| Concurrent takeover race | Final remote ownership is authoritative; a losing run detects mismatch on its next fence check. |
| Required checks failing | Approval and merge never occur. |
| Legacy/corrupt state | Recognized legacy state upgrades; corrupt or future-schema state stops without GitHub mutation. |

Capture the command transcript, state JSON, canonical comment body and ID,
labels, PR state, review history, and timestamps before and after each
scenario.

Because LLM execution is nondeterministic, repeat critical safety scenarios —
queue-wide claiming, takeover decline/acceptance, supersession, pause/resume,
cancellation, `/status` non-mutation, and failing required checks — three times
in fresh sessions.

Acceptance requires every scenario to match the specified outcome, no duplicate
canonical comments, no merge over failing required checks, no push to an author
branch, and every injected claim failure to be recorded without altering core
queue progression.

## Notes from facilitation (Claude, codebase checks — accepted by the maintainer)

- Hidden HTML-comment markers in claim comments are invisible in the GitHub UI
  and retrievable via `gh api` — standard bot pattern.
- "Refresh all outstanding claims on every transition" multiplies `gh` calls
  (N comments per transition); acceptable at this queue scale, worth revisiting
  for very large queues.
- If another run takes over even one PR of a queue, `/resume` halts the entire
  old run rather than skipping that PR (conservative; matches the strict-queue
  philosophy).
- Takeover testing requires a second GitHub identity (bot/machine account).
- Self-approval limitation: GitHub forbids approving one's own PR, so the
  pipeline's approve step fails on PRs authored by the operating account —
  unchanged by this feature.

Cross-model review additions (Claude — to be pinned in the implementation
plan):

- The exact versioned marker string must be fixed at planning time (e.g.
  `<!-- pr-review-pipeline:claim:v1 ... -->`) and treated as immutable within
  a schema version.
- Label handling must be idempotent: ensure `pr-pipeline` exists before first
  use (create-if-missing, tolerate already-exists), per the best-effort rules.
