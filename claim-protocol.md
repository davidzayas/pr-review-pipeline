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
