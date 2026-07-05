---
name: codex-cli-bg-sessions
description: Drive the `codex` (OpenAI Codex) CLI as a local background worker — spawn it, inspect its progress, monitor its real token usage, resume/steer it, and forcibly kill it if it runs away — without using Codex Cloud. Use this whenever you'd otherwise reach for the `codex` MCP server's `codex`/`codex-reply` tools: both are fully synchronous (the calling turn blocks until Codex's whole turn finishes) with no background, list, kill, or cost-telemetry primitives at all. `codex exec`, backgrounded via the shell yourself, is the workaround. Also use when asked to check on, babysit, or kill a runaway/looping local Codex session.
---

# Codex CLI Background Sessions

## Version this was validated against

Empirically measured against `codex-cli` **0.142.5** (`codex --version`; also visible as `session_meta.payload.cli_version` in on-disk rollout files) as of 2026-07-05 — see `references/methodology.md` for how each finding was derived and what's most likely to have drifted.

Run `codex --version` at the start of a task and keep it in mind; a mismatch isn't itself worth acting on or mentioning up front. Only if something actually misbehaves (a flag errors differently, an event schema field is missing, resume doesn't recall context) is that the moment to note the version gap and offer to re-run the relevant probe in `references/methodology.md` to reconcile this skill.

## Why this exists

The `codex` MCP server exposes exactly two tools: `codex` (start a thread) and `codex-reply` (continue a thread by `threadId`). Both are fully synchronous — the calling tool call blocks until Codex's entire turn (however many internal tool calls it takes) completes, then returns one final result. There is no background flag, no list/status tool, no kill/stop tool, and no cost/token telemetry exposed anywhere in either tool's schema.

**No cloud.** Codex does have a genuinely async, detached execution mode — `codex cloud exec --env <ID> "<prompt>"`, polled via `codex cloud status`/`list` — but it requires a pre-configured cloud environment and creates real remote (billable) state. This skill deliberately does not use it. Everything below is local: a `codex exec` process running on this machine, backgrounded by the calling shell.

## Quick reference

| Need | Command | Notes |
|---|---|---|
| Spawn in the background | `codex exec --json "<prompt>" > out.jsonl 2>err.log & pid=$!` | No native `--bg`; background it yourself. Keep stdout/stderr in separate files — stderr can carry non-JSON log lines that break parsing. |
| Get the session/thread id | Read the first line of `out.jsonl` | `{"type":"thread.started","thread_id":"..."}` — printed immediately, no need to wait on the on-disk rollout file. |
| Inspect progress | Tail `out.jsonl` | Clean JSONL events (`item.started`/`item.completed`/`turn.completed`) — no ANSI mess to strip. |
| Monitor real cost/tokens | Read `turn.completed.usage` in `out.jsonl` | Inline per-turn `input_tokens`/`cached_input_tokens`/`output_tokens`/`reasoning_output_tokens` — **no dedup needed**, unlike Claude Code's transcript. |
| Steer / send a follow-up | `codex exec resume <thread-id> --json "<followup>"` | Continues the same thread. **Verify** the returned `thread_id` matches what you passed — see footgun below. |
| Kill a runaway session | `kill <pid>` (the PID from `$!`) | Direct SIGTERM, no daemon in between. Measured near-instant (~0.003s call, process gone within ~0.03s). |

## Spawning

```
codex exec --json --skip-git-repo-check "<prompt>" > out.jsonl 2> err.log &
pid=$!
```

- `--json` prints one JSON event per line to stdout as the turn progresses (not just a final result).
- There is no built-in backgrounding, session manager, or job list — `$!` (or your own PID bookkeeping) is the only handle you get. Track it yourself.
- Ignore a `WARNING: proceeding, even though we could not create PATH aliases` line if it appears — environment noise, unrelated to the run's success.
- **Approval/trust does not gate `exec`, confirmed by testing.** Ran it against a directory with no `~/.codex/config.toml` project entry at all — created fresh, and trust explicitly declined ("No") when the interactive CLI prompted for it earlier — and the shell command still executed and the turn completed normally in ~11s, no hang, no approval prompt, no different behavior at all versus a `trusted` project. Non-interactive `exec` appears to always auto-execute regardless of project trust level (trust/approval prompts are a TUI/interactive-session concept that doesn't apply here). Declining trust also does **not** persist an explicit `untrusted` entry in `config.toml` the way some other projects have — the directory just has no entry either way. Don't build "might be stuck on approval" into loop-detection logic; it isn't a real failure mode for `exec`.

## Inspecting and cost monitoring

Tail your own redirected stdout file — there's no separate `logs`/`agents` command to reach for, and no ANSI to strip:

- `item.started` / `item.completed` — one pair per tool call or message. A shell command shows up as `{"type":"item.completed","item":{"type":"command_execution","command":"...","exit_code":0,"status":"completed"}}`; the model's reply as `{"type":"item.completed","item":{"type":"agent_message","text":"..."}}`.
- `turn.completed` — carries `usage` directly: `{"input_tokens":...,"cached_input_tokens":...,"output_tokens":...,"reasoning_output_tokens":...}`. This is the whole turn's real usage, once, no duplication — just sum these across turns for a running total. (Contrast with the `claude-code-bg-sessions` skill, where the equivalent Claude Code transcript logs the same turn's usage repeatedly across multiple content-block lines and needs deduping by `message.id`. Codex does not have that problem.)
- A richer on-disk transcript also exists at `~/.codex/sessions/<yyyy>/<mm>/<dd>/rollout-<timestamp>-<thread-id>.jsonl`, using a *different* event envelope (`session_meta`/`event_msg`/`response_item`/`turn_context` — not `thread.started`/`item.*`). It additionally carries `rate_limits` (plan usage %, reset time) inside its `token_count` events. Only bother reading it if you need that extra detail or per-turn `duration_ms`/`time_to_first_token_ms` (found on its `task_complete` events) — your own captured stdout is sufficient for routine cost/progress monitoring. See `references/event-schemas.md` for the full field layout of both formats.

## Resuming / steering

```
codex exec resume <thread-id> --json "<followup>"
```

This genuinely continues the same thread — verified by asking a session to recall a fact from earlier in its own run and getting the correct answer back, with the same `thread_id` echoed in the resumed run's `thread.started` event.

**Footgun, confirmed by testing:** passing a blank or invalid thread id does **not** error. It silently starts a brand-new, unrelated thread (a fresh `thread_id` you didn't ask for) and answers whatever it's asked fluently and wrongly, with no indication anything went sideways. Always check the `thread_id` in the resumed run's `thread.started` event equals the id you intended before trusting its reply — don't assume resume failed loudly if the target id was wrong.

There is no interactive `attach` for this local flow (that exists for `codex resume`/`codex fork`'s *interactive* sessions, not `exec`'s), and no fork-vs-continue distinction to worry about the way Claude Code's `--resume`/`--fork-session` has — `exec resume` always continues the real thread.

## Killing a runaway session

Because there's no session manager, there's also no dedicated stop command — kill the OS process you started directly:

```
kill <pid>
```

Measured: the `kill` call itself returns in a few milliseconds, and the process is gone within roughly another few hundredths of a second — dramatically faster than Claude Code's `claude stop` (~1-1.5s), because there's no daemon/IPC layer to route the request through; you're signaling the process directly. Confirmed genuine mid-turn interruption, not graceful completion: a killed run's stdout has no trailing `turn.completed` event, and its on-disk rollout file stops mid-turn too.

## Practical watchdog recipe

1. `codex exec --json "<task>" > out.jsonl 2> err.log &`, capture `pid=$!`.
2. Read the first line of `out.jsonl` for `thread_id`.
3. Poll `out.jsonl` for new `item.*`/`turn.completed` events; sum `turn.completed.usage` for cumulative tokens, and track wall-clock time since spawn.
4. If token growth or elapsed time blows past what the task should reasonably need (or no new events land for far longer than expected — possible stuck-on-approval), `kill <pid>` — it'll be dead in well under a second.
