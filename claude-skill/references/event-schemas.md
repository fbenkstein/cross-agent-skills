# Event schemas (observed, `codex-cli` 0.142.5)

Codex has **two distinct JSONL schemas** for what looks like the same data. Don't assume they match.

## 1. `codex exec --json` stdout stream

What you get by capturing the process's own stdout. Flat, simple, one event type per line under `type`:

- `{"type":"thread.started","thread_id":"<uuid>"}` — first line, always. This *is* the session id (same value that later appears in the on-disk rollout filename and in MCP `codex`/`codex-reply`'s `threadId`).
- `{"type":"turn.started"}` — a new turn (one user prompt → one full agent response, possibly with several tool calls inside) began.
- `{"type":"item.started","item":{"id":"item_N","type":"command_execution","command":"...","aggregated_output":"","exit_code":null,"status":"in_progress"}}` — a shell command started.
- `{"type":"item.completed","item":{...same shape, "exit_code":0,"status":"completed"}}` — it finished. Note `command` appears in *both* the started and completed events for the same item — don't count string occurrences across the stream as a proxy for "number of distinct commands run," it double-counts.
- `{"type":"item.completed","item":{"id":"item_N","type":"agent_message","text":"..."}}` — the model's visible reply text for this turn.
- `{"type":"turn.completed","usage":{"input_tokens":N,"cached_input_tokens":N,"output_tokens":N,"reasoning_output_tokens":N}}` — last line of a completed turn. Absent if the process was killed mid-turn.

Stderr is separate and can contain plain-text log lines — always redirect it to its own file, never merge with the `--json` stdout you intend to parse. Observed examples that turned out to be harmless noise, not failures: a recurring `Shell snapshot validation failed: ... syntax error in conditional expression` warning on every run; an occasional `Reading additional input from stdin...` line; and, once, an unrelated `rmcp::transport::worker` fatal about a different configured MCP server (Notion) failing to connect (HTTP 503) — none of these affected the run's own turn completing successfully.

## 2. On-disk rollout transcript

Path: `~/.codex/sessions/<yyyy>/<mm>/<dd>/rollout-<local-timestamp>-<thread-id>.jsonl` — one file per thread, keyed by the same `thread_id`/UUID, written whether the session came from `codex exec`, an interactive session, or the MCP `codex` tool (`session_meta.payload.source` records which).

Line envelope: `{"timestamp": "...", "type": "<envelope-type>", "payload": {...}}`. Envelope `type` values seen:

- `session_meta` — first line. `payload` has `session_id`, `cwd`, `originator`, `cli_version`, `source` (`"mcp"`, `"exec"`, etc.), `model_provider`, `base_instructions`.
- `turn_context` — per-turn config snapshot.
- `response_item` — model/user content, `payload.type` one of `message` / `reasoning` (with `encrypted_content`, not readable).
- `event_msg` — the interesting one for monitoring. `payload.type` includes:
  - `task_started` — `turn_id`, `started_at`, `model_context_window`.
  - `user_message` / `agent_message` — the actual text content of a turn.
  - `token_count` — `payload.info.total_token_usage` (cumulative for the whole session) and `payload.info.last_token_usage` (just this turn) — both already deduplicated, one event per turn, no aggregation bug to work around. Also carries `payload.rate_limits`: `used_percent`/`window_minutes`/`resets_at` for `primary` and `secondary` windows, plus `plan_type` — telemetry Claude Code's transcript doesn't expose at all.
  - `task_complete` — `turn_id`, `last_agent_message`, `completed_at`, `duration_ms`, `time_to_first_token_ms`. Useful for per-turn latency if you need it; `out.jsonl`'s `turn.completed` doesn't carry timing, only usage.

Prefer the stdout stream (schema 1) for live monitoring of a session you're driving yourself — it's simpler and you already have the file handle. Reach for the rollout file (schema 2) only when you need `rate_limits` or per-turn timing, or when inspecting a session you didn't spawn yourself (e.g. one driven through the MCP server).
