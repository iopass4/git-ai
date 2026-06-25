# Session Event Attribution Recovery Plan

## Problem

Git AI can miss attribution when AI editing hooks were not installed, were
installed after an agent session started, or were installed after AI-generated
changes already existed in the working tree. In those cases normal checkpoints
are absent, so post-commit authorship finalization currently leaves committed
added lines as unknown/untracked even when local transcript metrics show an AI
session was active at the same time the files were modified.

The goal is to add a recovery solver that uses persisted metrics session events
as durable session evidence. It should run after bash mtime recovery and before
adjacent edge recovery, so stronger shell-call evidence wins first and edge
extension only sees lines still unknown after timestamp/session recovery.

## Current System Shape

- `recover_attribution()` in `src/authorship/attribution_recovery.rs` is the
  post-commit recovery entry point.
- The current solver order is:
  1. `bash_mtime`
  2. `edge_extension`
- Recovery only considers unknown lines inside committed hunks.
- Bash recovery uses captured file `mtime`/`ctime` values, a three-second
  window, session note insertion, recovered checkpoint metrics, and solver
  metadata.
- Session transcript ingestion emits metrics event id `5`
  (`MetricEventId::SessionEvent`) with cached DB metadata including
  `event_ts`, `event_kind`, `session_id`, `trace_id`, `tool`,
  `external_session_id`, and `external_tool_use_id`.
- Session-event metric rows may also include `repo_url` in the serialized
  event attributes when stream processing can infer a repository working
  directory.

## Design Principles

- Recover only committed lines that are still unknown. Never overwrite explicit
  AI, known-human, or legacy-human/no-data attestations already present in the
  note.
- Use the same captured file timestamp source as bash recovery so commit-time
  filesystem changes do not distort matching.
- Treat session-event evidence as weaker than bash history. Bash recovery runs
  first and session-event recovery only considers remaining unknown lines.
- Require a session-linked metrics row. A row without an internal session id,
  external session id, or tool cannot create a usable `SessionRecord`.
- Prefer repository-linked candidates. If metrics rows contain a `repo_url`,
  only candidates for the current repository should be selected. If no nearby
  candidate has repo evidence, allow a time-only fallback only when selection is
  unambiguous.
- Keep the three-second window tight and symmetric around file timestamps:
  session-event timestamps must be within plus or minus three seconds of at
  least one captured file timestamp.
- Keep solver metadata explicit enough to audit why attribution was recovered.
- Reuse existing attribution recovery helpers where possible: unknown-line
  calculation, line range compression, session id generation, attestation
  insertion, and recovered checkpoint metrics.

## Data Model

Add a query-focused record to `src/metrics/db.rs`:

```rust
pub(crate) struct SessionEventRecoveryCandidate {
    pub row_id: i64,
    pub event_ts: u32,
    pub session_id: String,
    pub trace_id: Option<String>,
    pub tool: String,
    pub model: Option<String>,
    pub external_session_id: String,
    pub external_tool_use_id: Option<String>,
    pub repo_url: Option<String>,
}
```

`model` may be absent because session events do not consistently carry model
metadata in common attributes. In that case the recovered `AgentId.model`
should be `"unknown"`, matching existing preset fallbacks.

Add a metrics DB query:

```rust
pub(crate) fn session_event_candidates_near_timestamps(
    &self,
    timestamps_ns: &[u128],
    window_ns: u128,
) -> Result<Vec<SessionEventRecoveryCandidate>, GitAiError>
```

Query rows with `event_kind = MetricEventId::SessionEvent` and `event_ts`
within the min/max seconds implied by the nanosecond timestamp windows. Use the
cached metadata columns for session/tool ids and parse only candidate
`event_json` values to recover `repo_url` and optional model from attributes.

## Solver: Session Event Mtime Recovery

Add `recover_session_event_mtime()` in
`src/authorship/attribution_recovery.rs` and call it between
`recover_bash_mtime()` and `recover_adjacent_edges()`.

For each eligible file:

1. Recompute remaining unknown committed lines after bash recovery.
2. Use captured file timestamps when available, otherwise read committed-file
   `mtime`/`ctime` from the working tree using the existing timestamp helper.
3. Query session-event candidates within the three-second window around all
   eligible file timestamps.
4. Score candidates for each file:
   - `same_repo_url`: candidate serialized metrics repo URL exactly matches the
     current repo URL.
   - `same_existing_session`: candidate session id already has any attestation
     in the current commit.
   - `time_only_unambiguous`: no repo-linked candidate exists and exactly one
     session id is present in the matching time window.
5. Select the best candidate by tier, nearest timestamp distance, then newest
   row id. If the only viable evidence is ambiguous time-only evidence from
   multiple session ids, do not recover.
6. Add one attestation for all remaining unknown committed lines in that file:

   `candidate.session_id::generate_trace_id()`

7. Ensure `authorship_log.metadata.sessions[candidate.session_id]` exists with:
   - `agent_id.tool = candidate.tool`
   - `agent_id.id = candidate.external_session_id`
   - `agent_id.model = candidate.model.unwrap_or("unknown")`
   - `human_author = Some(human_author)`
   - `custom_attributes = None`
8. Emit a recovered checkpoint metric using the existing helper pattern:
   - `kind`: `"ai_agent"`
   - `edit_kind`: `"attribution_recovery_session_event"`
   - `checkpoint_type`: `"recovered_session_event_mtime"`
   - timestamp: selected session event timestamp
   - attrs: repo URL, branch, base commit, commit sha, session id, new trace id,
     tool, model, external session id, author id

Recovery metadata JSON should include:

- `solver`: `"session_event_mtime"`
- `file_path`
- `unknown_lines`
- `file_timestamps_ns`
- `selected_metric_row_id`
- `selected_event_ts`
- `selected_session_id`
- `selected_external_session_id`
- `selected_external_tool_use_id`
- `selected_tool`
- `selected_model`
- `selected_repo_url`
- `target_repo_url`
- `distance_ns`
- `window_ns`
- `selection_tier`
- `candidate_count`

## Guardrails

- Do not run when there are no unknown committed lines.
- Do not recover rows with missing `session_id`, `tool`, or
  `external_session_id`.
- Do not recover a file when no file timestamp is available.
- Do not use session-event recovery for rows from `mock_ai`.
- Do not select a time-only candidate when more than one distinct session id is
  nearby.
- Do not let session-event recovery reassign lines already recovered by bash.
- Do not let edge recovery see stale unknown-line state; it must recompute
  remaining unknown lines after session-event recovery.

## Tests First

Add RED tests before implementation.

1. Metrics DB unit: `session_event_candidates_near_timestamps` returns only
   event id `5` rows inside the plus/minus three-second window and excludes
   rows outside it.
2. Metrics DB unit: candidates require session id, tool, and external session
   id; malformed rows are skipped.
3. Metrics DB unit: candidate parsing exposes repo URL, trace id, external
   tool-use id, and falls back to model `None` when absent.
4. Attribution recovery unit: candidate selection prefers same repo URL over a
   closer time-only session.
5. Attribution recovery unit: time-only matching is rejected when multiple
   session ids are equally plausible in the window.
6. Integration: with no checkpoints, a committed file whose mtime is within
   three seconds of a repo-linked session event is attributed to that session.
   Assert committed lines and blame after the commit.
7. Integration: an explicit known-human checkpoint on the same commit remains
   human even when a nearby session event exists.
8. Integration: bash recovery wins over session-event recovery when both are
   nearby; the final note points to the bash session for the recovered lines.
9. Integration: a session event outside the three-second window leaves unknown
   committed lines unattributed.
10. Integration: two nearby time-only sessions without repo URL do not recover
    attribution.

Use the repo harness for all integration tests and assert committed line-level
attribution after every commit.

## Verification

During implementation:

1. Run focused unit tests for `src/metrics/db.rs` and
   `src/authorship/attribution_recovery.rs`.
2. Run the focused integration tests added for session-event recovery.
3. Run `task fmt`.
4. Run `task lint`.
5. Run a broader relevant attribution test set before opening the PR.

Before PR completion, open a draft PR, monitor the Ubuntu CI jobs first, and
address actionable Devin feedback before treating the task as complete.
