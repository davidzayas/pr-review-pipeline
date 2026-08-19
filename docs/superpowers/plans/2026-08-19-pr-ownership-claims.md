# PR Ownership Claims Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add lease-style GitHub-visible ownership claims (label + self-describing comment with generation fencing) to the pr-review-pipeline plugin, plus a `/cancel` release command.

**Architecture:** Split authority per the approved spec ([2026-08-19-pr-ownership-claims-design.md](../specs/2026-08-19-pr-ownership-claims-design.md)): the local state file stays authoritative for queue/config/progress; the per-PR claim comment (hidden versioned payload) is authoritative for ownership; the `pr-pipeline` label is a display-only projection. Shared claim mechanics live in one `claim-protocol.md` that every command reads at runtime.

**Tech Stack:** Claude Code plugin (prompt-markdown commands), GitHub CLI (`gh`), `jq`. No executable code ships; "implementation" = authoring command prompts. No unit-test harness exists — verification is `claude plugin validate` + consistency greps per task, and a scripted sandbox scenario pass at the end.

## Global Constraints

- Claim operations are best-effort: they may NEVER block, fail, or reorder core review/merge processing.
- Generation fence before every claim mutation: edit only when remote `run_id` AND `generation` match local state; only a confirmed `/review` takeover advances the generation.
- `/status` never writes to GitHub; it may repair the LOCAL state file only.
- No automatic release on staleness — ever. Stale claims are flagged loudly; release happens only via terminal transition, `/cancel`, or a fresh `/review` reclaiming.
- Marker string is exactly `pr-review-pipeline:claim:v1` — immutable within schema version 2.
- Label name is exactly `pr-pipeline`; creation is idempotent (create-if-missing, tolerate already-exists).
- The claim comment is edited in place, never deleted or recreated while one exists; release edits it (history preserved) and removes the label.
- State file `schema_version` is `2`. Recognized legacy (v1 / no `schema_version`) state upgrades in place; corrupt or future-schema state stops before any GitHub mutation.
- Existing safety rails unchanged: never merge over failing required checks, never push to author branches, pause on core auth errors (claim-only permission failures do NOT pause).
- All timestamps ISO-8601 UTC (`date -u +%Y-%m-%dT%H:%M:%SZ`).

---

### Task 1: Claim protocol reference file

**Files:**
- Create: `claim-protocol.md` (plugin root)

**Interfaces:**
- Produces: the protocol sections `Payload schema`, `Comment template`, `Finding the claim comment`, `Creating/editing claims`, `Label management`, `Generation fence`, `Staleness`, `Release`, `Timestamps` — referenced by name from every command in Tasks 2–5.
- Produces: claim payload JSON keys (exact): `repo`, `pr`, `owner`, `run_id`, `generation`, `pipeline_status`, `claim_state`, `claimed_at`, `updated_at`, `stale_at`, `released_at`, `release_reason`.
- Produces: state-file claim object keys (exact): `comment_id`, `owner`, `run_id`, `generation`, `claimed_at`, `updated_at`, `stale_at`, `state`, `sync_status`, `last_error`.

- [ ] **Step 1: Write `claim-protocol.md` with exactly this content**

````markdown
# Claim Protocol (v1)

Shared mechanics for PR ownership claims. Commands MUST follow this file
exactly. All claim operations are best-effort: on failure, record
`sync_status`/`last_error` in the state file and continue the core pipeline.

## Identity

- **Marker (immutable):** `pr-review-pipeline:claim:v1`
- **Label (exact name):** `pr-pipeline`
- **Owner:** the authenticated login from `gh api user --jq .login`
- **Run ID:** generate once per run: `run-$(date -u +%Y%m%d-%H%M%S)-$(uuidgen | tr 'A-Z' 'a-z' | cut -c1-8)`

## Timestamps

ISO-8601 UTC. Now: `date -u +%Y-%m-%dT%H:%M:%SZ`.
`stale_at` = transition time + `max_wait_min` minutes. Compute portably:

```bash
python3 -c "from datetime import datetime,timedelta,timezone; print((datetime.now(timezone.utc)+timedelta(minutes=$MAX_WAIT_MIN)).strftime('%Y-%m-%dT%H:%M:%SZ'))"
```

A claim is **stale** when now > its payload `stale_at`. A missing or
unparseable `stale_at` means staleness is UNKNOWN — never treat unknown as
stale.

## Payload schema

One JSON object on a single line inside an HTML comment. Every field is
required; use JSON `null` where a value does not apply yet.

```
<!-- pr-review-pipeline:claim:v1 {"repo":"<owner/repo>","pr":<number>,"owner":"<login>","run_id":"<run-id>","generation":<int>,"pipeline_status":"<per-PR status>","claim_state":"active|released","claimed_at":"<iso>","updated_at":"<iso>","stale_at":"<iso>","released_at":"<iso|null>","release_reason":"<string|null>"} -->
```

## State-file claim object

Each queued PR's state entry carries a `claim` object with exactly these
keys (spec: Components): `comment_id`, `owner`, `run_id`, `generation`,
`claimed_at`, `updated_at`, `stale_at`, `state` (`"active"` or
`"released"` — mirrors the payload's `claim_state`), `sync_status`
(`synced` | `partial` | `failed` | `superseded`), `last_error`
(message or null). Commands update this object on every claim attempt.

## Comment template

The canonical comment body is the payload marker line followed by visible
text. Active claim:

```
<!-- pr-review-pipeline:claim:v1 {...} -->
🤖 **This PR is managed by pr-review-pipeline** (owner: @<owner>, run `<run_id>`, generation <generation>).

- Pipeline status: `<pipeline_status>` — <one-line meaning, e.g. "queued for review", "changes requested, polling for author activity">
- Claimed: <claimed_at> · Last update: <updated_at>
- **Stale after: <stale_at>** — if that time has passed with no update here and no verdict, the run was likely abandoned: this claim may be ignored, and the `pr-pipeline` label removed. It will not release itself.

Please do not review this PR separately while the claim is active — a verdict review will be posted here.
```

Released claim (same comment, edited):

```
<!-- pr-review-pipeline:claim:v1 {...with claim_state:"released"...} -->
🤖 **pr-review-pipeline has released this PR** (reason: <release_reason>, at <released_at>).

This PR is open for normal review activity.
```

## Finding the claim comment

1. If the state file has `claim.comment_id`, fetch it directly:
   `gh api repos/{owner}/{repo}/issues/comments/<comment_id>` — if it still
   contains the marker, it is canonical.
2. Otherwise list and filter:
   `gh api repos/{owner}/{repo}/issues/<PR>/comments --paginate --jq '[.[] | select(.body | contains("pr-review-pipeline:claim:v1"))]'`
3. Zero matches → no claim exists. One match → canonical. Multiple matches →
   NEVER create another; prefer the cached `comment_id` if it is among them,
   else the one with highest `generation` (latest `updated_at` as
   tie-break); warn the user to clean up the duplicates manually.

## Creating / editing claims

- Create: `gh api repos/{owner}/{repo}/issues/<PR>/comments -f body="$BODY"`
  → record returned `.id` as `claim.comment_id`.
- Edit: `gh api -X PATCH repos/{owner}/{repo}/issues/comments/<comment_id> -f body="$BODY"`
- After any edit, re-fetch the comment and confirm the payload `run_id` and
  `generation` are yours; on mismatch mark that PR's claim
  `sync_status: superseded` and stop claim writes for it (core processing
  continues).

## Label management

- Ensure once per run (idempotent, best-effort):
  `gh label create pr-pipeline --description "PR is managed by the pr-review-pipeline plugin" --color 5319e7 2>/dev/null || true`
- Add on active claim: `gh pr edit <PR> --add-label pr-pipeline`
- Remove on release: `gh pr edit <PR> --remove-label pr-pipeline`
- The label is NEVER ownership authority — only the comment payload is. A
  label present with no marked comment is an orphan: report it, do not treat
  the PR as claimed.

## Generation fence

Before EVERY claim mutation (edit or release):

1. Re-fetch the canonical comment and parse its payload.
2. Proceed only if payload `run_id` == local `claim.run_id` AND payload
   `generation` == local `claim.generation`.
3. On mismatch: set local `claim.sync_status = "superseded"`, do not write,
   and never attempt again this run. Core processing continues.

Generations advance ONLY in `/review` claim classification:
- No comment exists → create with `generation: 1`.
- Comment exists with `claim_state: "released"` → reuse it, `generation + 1`.
- Comment exists, active, stale → loud warning, reuse it, `generation + 1`.
- Comment exists, active, NOT stale, other `run_id` → require explicit user
  takeover confirmation before reusing with `generation + 1`; declined =
  stop everything, zero mutations.
- Comment exists, active, unparseable payload → ownership UNKNOWN and
  non-stale: require the same explicit confirmation.

## Release

Terminal transitions (`merged`, `skipped_error`) and `/cancel` release a
claim — run completion is not a separate trigger, since every PR is already
terminal (and thus released) when a run completes. Release means (fence
first): editing the comment to the released
template (`claim_state: "released"`, `released_at` = now, `release_reason`
one of `merged`, `skipped_error`, `cancelled`), then removing the label.
Comment stays forever (history). If release fails: record it, continue —
the claim will go visibly stale on its own.

## Recording sync results

Every claim attempt records into the PR's state entry:
`sync_status`: `synced` (all writes ok) | `partial` (comment ok, label
failed or vice versa) | `failed` (could not read or write) |
`superseded` (fence mismatch) — plus `last_error` (message or null).
````

- [ ] **Step 2: Consistency check**

Run: `grep -c 'pr-review-pipeline:claim:v1' claim-protocol.md`
Expected: 5 (identity + payload + template refs + find recipe). Also `grep -n 'pr-pipeline' claim-protocol.md` — label name appears with exact spelling only.

- [ ] **Step 3: Commit**

```bash
git add claim-protocol.md
git commit -m "feat: add claim protocol reference (marker v1, payload schema, fence rules)"
```

---

### Task 2: `/review` — claim-at-queue-time, transitions, releases

**Files:**
- Modify: `commands/review.md`

**Interfaces:**
- Consumes: every protocol section from Task 1 (referenced by name).
- Produces: state schema v2 (top-level `schema_version: 2`, `run_id`, `run_status`; per-PR `claim` object with Task 1's keys) — relied on by Tasks 3, 4, 5.

- [ ] **Step 1: Add claim-protocol preamble**

In `commands/review.md`, insert after the `# PR Review Pipeline` heading paragraph:

```markdown
## Claim protocol

This pipeline claims PRs on GitHub so humans do not duplicate reviews. Read
`${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` NOW and follow it exactly for every
claim operation below. Claim operations are best-effort and never block core
processing.
```

- [ ] **Step 2: Extend Setup — active-run guard, schema v2, run_id**

Replace Setup item 2's state JSON block and surrounding text with:

```markdown
2. If `.claude/pr-pipeline-state.json` exists with `run_status: "active"` or
   `"paused"`, STOP: tell the user a run already exists (show its queue and
   status) and that they must `/pr-review-pipeline:resume` it or
   `/pr-review-pipeline:cancel` it first. Never overwrite an active run.
3. Generate `run_id` per the protocol. Initialize the state file:

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
```

Keep the existing statuses line; add after it: `Run statuses: active, paused, completed, cancelled.`

- [ ] **Step 3: Add claim preflight + queue-wide claiming (before the pipeline loop)**

Insert a new section between Setup and "Pipeline loop":

```markdown
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
```

- [ ] **Step 4: Add the transition rule**

Append to the "Rules" section of `commands/review.md`:

```markdown
- After EVERY per-PR status transition (persist it to the state file FIRST),
  refresh every outstanding active claim you own with one shared transition
  timestamp: `updated_at` = now, `stale_at` = now + max_wait_min,
  `pipeline_status` = that PR's current status (fence first, per protocol).
- On `merged` or `skipped_error`, RELEASE that PR's claim (protocol: Release,
  reason = the status) before moving on; then refresh the remaining claims.
- Entering `paused` re-stamps every outstanding claim without releasing.
  Set top-level `run_status` accordingly on pause ("paused") and completion
  ("completed").
```

- [ ] **Step 5: Wire releases into Steps 3, 4, 5 and Completion**

Make these three edits in the existing pipeline-loop text:
1. Step 3 (APPROVE), after the merge succeeds: add `4. Release this PR's claim (reason: merged) per the protocol, then refresh remaining claims.` (renumber the existing "Set status merged" item accordingly — release comes after recording `merged`.)
2. Step 1's skip rule: append `— and release its claim (reason: skipped_error) if one was created.`
3. Completion section: append `All claims should already be released by their terminal transitions; verify none remain active (best-effort), set run_status: "completed", then archive the state file as before.`

- [ ] **Step 6: Validate**

Run: `claude plugin validate .` (if unavailable in this environment, run `/plugin validate` in an interactive session — record which was used).
Then: `grep -n 'CLAUDE_PLUGIN_ROOT' commands/review.md` (expect 1 hit), `grep -c 'run_status' commands/review.md` (expect ≥3).

- [ ] **Step 7: Commit**

```bash
git add commands/review.md
git commit -m "feat(review): claim-at-queue-time, generation fence, terminal releases"
```

---

### Task 3: `/resume` — reconciliation and supersession fence

**Files:**
- Modify: `commands/resume.md`

**Interfaces:**
- Consumes: protocol sections (Task 1); state schema v2 (Task 2).

- [ ] **Step 1: Add protocol preamble** — same block as Task 2 Step 1, inserted after the `# Resume PR Review Pipeline` heading.

- [ ] **Step 2: Replace the numbered resume procedure**

Replace the existing numbered list body (keep items 1–2, auth/repo checks) so items 3+ read:

```markdown
3. Schema check: if `schema_version` is missing or 1, upgrade in place — add
   `schema_version: 2`, `run_id` (generate one), `run_status` ("paused"), and
   an empty claim object (all nulls) per queued PR, then run step 4's
   reconciliation to backfill. If `schema_version` > 2 or the file is invalid
   JSON: STOP with an error before any GitHub access.
4. Reconcile remote → local BEFORE any GitHub write, for the complete queue:
   - Live merged/closed PR facts repair terminal pipeline statuses
     (merged → `merged`; closed unmerged → `skipped_error`).
   - Each PR's canonical claim comment (protocol: Finding the claim comment)
     replaces cached claim metadata: owner, run_id, generation, timestamps,
     state.
   - Queue order, configuration, and nonterminal execution statuses stay as
     the local file says.
5. Supersession fence: if any outstanding (non-terminal) PR's remote claim
   carries a different run_id with a generation ≥ the local one, mark that
   local claim `sync_status: "superseded"`, tell the user which run owns it,
   and STOP — this run may not continue or write any claims.
6. Otherwise re-stamp every outstanding claim you own (updated_at = now,
   stale_at = now + max_wait_min, current pipeline_status; fence first), set
   `run_status: "active"`, and resume the pipeline exactly as defined in
   `/pr-review-pipeline:review` at the appropriate step (same mapping as
   before: paused/waiting/changes_requested → activity check;
   reviewing/pending → Step 1; approved → retry merge). All transition and
   release rules from `/review` apply, including claim refresh and terminal
   releases.
```

- [ ] **Step 3: Validate** — `grep -n 'superseded' commands/resume.md` (expect ≥2), `grep -n 'CLAUDE_PLUGIN_ROOT' commands/resume.md` (expect 1).

- [ ] **Step 4: Commit**

```bash
git add commands/resume.md
git commit -m "feat(resume): remote-first reconciliation, supersession fence, claim re-stamp"
```

---

### Task 4: `/status` — read-side reconciliation and claim reporting

**Files:**
- Modify: `commands/status.md`

**Interfaces:**
- Consumes: protocol sections (Task 1); state schema v2 (Task 2).

- [ ] **Step 1: Replace the body of `commands/status.md` (below frontmatter) with**

```markdown
# Pipeline Status

Read `${CLAUDE_PLUGIN_ROOT}/claim-protocol.md` for how to find and parse
claim comments. **This command NEVER writes to GitHub** — no comments,
labels, reviews, or merges. It may repair the LOCAL state file only.

1. Read `.claude/pr-pipeline-state.json`. If missing, check for
   `.claude/pr-pipeline-state.last.json` (a completed/cancelled run) and
   report that; otherwise say no pipeline exists.
2. For each PR in the queue, fetch live data
   (`gh pr view <PR> --json state,reviewDecision,headRefOid,statusCheckRollup,title,author`)
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
   LOUDLY: any claim past its stale_at (likely abandoned run — suggest
   `/pr-review-pipeline:resume` or `/pr-review-pipeline:cancel`), any
   label/comment disagreement (orphaned label, or active claim missing its
   label), any claim owned by a different run_id (superseded), and any
   sync_status of failed/partial.
5. Highlight the PR currently blocking the pipeline (first non-merged,
   non-skipped entry), what it is waiting on, and how to continue.

$ARGUMENTS
```

- [ ] **Step 2: Validate** — `grep -c 'NEVER writes to GitHub' commands/status.md` (expect 1); confirm no `gh pr edit`, `gh pr review`, `gh pr merge`, or `-X PATCH` appears: `grep -nE 'pr edit|pr review|pr merge|PATCH' commands/status.md` (expect no matches).

- [ ] **Step 3: Commit**

```bash
git add commands/status.md
git commit -m "feat(status): claim columns, staleness flags, local-only repair"
```

---

### Task 5: `/cancel` — new release command

**Files:**
- Create: `commands/cancel.md`

**Interfaces:**
- Consumes: protocol sections (Task 1); state schema v2 (Task 2).

- [ ] **Step 1: Write `commands/cancel.md` with exactly this content**

````markdown
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
4. For each queued PR whose claim `state` is `active`: fetch the canonical
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
````

- [ ] **Step 2: Validate** — `claude plugin validate .` (or note fallback); `grep -n 'generation fence' commands/cancel.md` (expect 1).

- [ ] **Step 3: Commit**

```bash
git add commands/cancel.md
git commit -m "feat: add /cancel command — explicit claim release and run archive"
```

---

### Task 6: Documentation and version bump

**Files:**
- Modify: `README.md`
- Modify: `.claude-plugin/plugin.json`

**Interfaces:**
- Consumes: behavior defined in Tasks 1–5 (documentation must match exactly: label `pr-pipeline`, staleness = last update + max-wait, takeover confirmation, `/cancel` semantics, best-effort caveat).

- [ ] **Step 1: README — add `/cancel` to Usage**

After the `### /pr-review-pipeline:resume` block, add:

```markdown
### `/pr-review-pipeline:cancel`

Cancels the current run: asks for confirmation, releases every claim the run
still owns (label removed, claim comment edited to "released"), marks the
run `cancelled`, and archives its state. Claims owned by a newer run are
left untouched. If GitHub is unreachable the run is still cancelled locally
and the remote claims are left to go visibly stale.
```

- [ ] **Step 2: README — add an "Ownership claims" subsection under How it works**

After the numbered how-it-works list, add:

```markdown
### Ownership claims

When a run starts it claims every queued PR on GitHub: a `pr-pipeline` label
plus one pinned-style comment per PR stating who owns it, the current
pipeline status, and — critically — a **stale-after deadline** (last update
+ `--max-wait`). The comment is edited in place at every transition, so an
abandoned run is self-evident: past the deadline with no verdict, the
comment itself says the claim may be ignored and the label removed. Claims
are released when a PR reaches a terminal state (merged / skipped), on
`/cancel`, and never automatically on staleness. If a new `/review` finds a
live claim from another run, it asks for explicit takeover confirmation
before reclaiming (a generation counter in the comment prevents the old run
from overwriting the new owner). All claim operations are best-effort — a
GitHub hiccup never blocks the review pipeline itself.
```

Also update the workflow mermaid diagram: add one node after `NEXT` → claim step is global, so instead add a node between `START` and `NEXT`: `START --> CLAIM["Claim all queued PRs<br/>(label + lease comment)"] --> NEXT`, and change the PAUSE node text to `"Pause — state saved, claims re-stamped.<br/>Continue with /resume or release with /cancel"`. Update the Layout tree to include `claim-protocol.md` and `commands/cancel.md`.

- [ ] **Step 3: Bump plugin version**

In `.claude-plugin/plugin.json`: `"version": "1.0.0"` → `"version": "1.1.0"`.

- [ ] **Step 4: Validate** — re-render the mermaid block (extract to a file, POST to mermaid.ink as done for the original README work, or eyeball with `claude plugin validate .`); `grep -n 'cancel' README.md` (expect ≥4).

- [ ] **Step 5: Commit**

```bash
git add README.md .claude-plugin/plugin.json
git commit -m "docs: document ownership claims and /cancel; bump to 1.1.0"
```

---

### Task 7: Sandbox scenario validation

**Files:**
- Create: `docs/superpowers/plans/2026-08-19-claims-test-results.md` (results log, committed)

**Interfaces:**
- Consumes: everything above; the spec's Testing section scenario table.

No unit harness exists — this validates the prompts end-to-end against a
disposable repo, per the spec's Testing section. Single-identity scenarios
only; the two-identity takeover/race scenarios and restricted-identity
permission scenarios require a second GitHub account and are **deferred as
accepted debt** unless the maintainer provides one (record the decision in
the results log).

- [ ] **Step 1: Provision sandbox**

```bash
gh repo create davidzayas/pr-pipeline-sandbox --private --add-readme --clone
cd pr-pipeline-sandbox
for n in 1 2 3; do
  git checkout -b test-pr-$n main && echo "change $n" > file-$n.txt && git add . && git commit -m "test PR $n" && git push -u origin test-pr-$n
  gh pr create --title "Test PR $n" --body "sandbox" --head test-pr-$n --base main
  git checkout main
done
```

- [ ] **Step 2: Run scenario A — queue startup + normal flow.** In a fresh Claude Code session in the sandbox clone: `/pr-review-pipeline:review 1 2 3 --poll-interval 1 --max-wait 2`. Assert with `gh pr view <n> --json labels` and `gh api repos/davidzayas/pr-pipeline-sandbox/issues/<n>/comments`: every PR got exactly one marked comment + label BEFORE PR 1's verdict was posted; after PR 1 merges its comment says released and label is gone; state file has `schema_version: 2`, `run_id`, per-PR claim objects.

- [ ] **Step 3: Run scenario B — pause, stale flagging, resume.** Let the run pause on a blocked PR (max-wait 2 min). Assert: pause re-stamped comments (`updated_at` ~ pause time). Wait past stale_at, run `/pr-review-pipeline:status`: asserts loud stale flag, and NO GitHub mutation (capture comment `updated_at` before/after — must be byte-identical). Then `/pr-review-pipeline:resume`: run_id and generation unchanged, claims re-stamped.

- [ ] **Step 4: Run scenario C — cancel.** Start a fresh 1-PR run, then `/pr-review-pipeline:cancel`. Assert: confirmation was asked; comment edited to released (reason `cancelled`); label removed; state archived to `.last.json` with `run_status: "cancelled"`.

- [ ] **Step 5: Run scenario D — stale reclaim + legacy state.** (1) With scenario B's abandoned claims still on GitHub and the state file deleted, run a fresh `/review` on the same PRs: assert loud stale warning, reuse of the SAME comment ids, generation incremented. (2) Hand-write a v1-format state file (no `schema_version`, no claims) and run `/resume`: assert in-place upgrade + reconciliation backfill, no duplicate comments created.

- [ ] **Step 6: Repeat the critical trio** (scenario A claiming order, scenario C cancel, scenario B `/status` non-mutation) two more times each in fresh sessions (LLM nondeterminism, per spec). Log pass/fail per iteration.

- [ ] **Step 7: Record results and clean up**

Write `docs/superpowers/plans/2026-08-19-claims-test-results.md`: scenario × iteration table, deviations found, the deferred two-identity scenarios noted as accepted debt. Delete the sandbox repo only with the maintainer's explicit confirmation (`gh repo delete` is destructive — ask first). Commit the results log:

```bash
git add docs/superpowers/plans/2026-08-19-claims-test-results.md
git commit -m "test: sandbox validation results for ownership claims"
```

---

## Self-review notes (completed)

- **Spec coverage:** claim-at-queue-time (T2), transition refresh + stale_at lease (T2), pause re-stamp (T2/T3), takeover confirmation + generation advance (T1/T2), supersession fence (T1/T3/T5), /status local-only repair + reporting (T4), /cancel (T5), best-effort matrix (T1 + per-command text), duplicate/malformed/orphan handling (T1), legacy upgrade (T3), docs (T6), sandbox testing incl. repeat-3× (T7). Two-identity scenarios: explicitly deferred as accepted debt in T7 — the only spec item not fully executed.
- **Placeholder scan:** clean — every step carries its exact content or exact command.
- **Type consistency:** payload keys, state claim keys, label name, and marker string are identical across T1–T5 (single source: T1 Interfaces).
