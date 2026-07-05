---
name: claude-code-bg-sessions
description: Drive a `claude` (Claude Code) CLI session as a background worker from outside an interactive terminal — spawn it, inspect its progress, monitor its real token/cost usage, resume/steer it, and forcibly stop it if it runs away. Use this whenever you'd otherwise reach for the `claude-code` MCP server's `Agent`/`Task` tool: as of mid-2026 that tool's agent registry comes back empty over `claude mcp serve` (a known, still-open upstream bug — anthropics/claude-code#41973, #64622), so `Agent`/`Task` calls fail with "Agent type '...' not found. Available agents:" for every subagent_type, including builtins. `claude --bg` + friends is the load-bearing workaround. Also use when asked to check on, babysit, or kill a runaway/looping Claude Code background session.
---

# Claude Code Background Sessions

## Version this was validated against

Every specific in this skill (error strings, timings, file formats, field names) was empirically measured against `claude` (Claude Code CLI) **v2.1.201** (2026-07-05) — see `references/methodology.md` for exactly how each finding was derived and which specifics are most likely to have drifted since.

Run `claude --version` at the start of a task and keep it in mind. A mismatch isn't itself a problem worth act­ing on or mentioning up front — don't re-verify the findings below just because the version differs. If something in this skill *does* misbehave (a command errors differently, a timing assumption is way off, a file isn't where expected), that's the actual signal: mention the version difference then, and offer to re-run the relevant probe from `references/methodology.md` to reconcile this skill rather than treating the whole thing as unreliable.

## Why this exists

The `claude-code` MCP server (started via `claude mcp serve`, the same command Codex's `claude-code` MCP entry runs) exposes an `Agent` tool for spawning subagents. That tool is currently non-functional over MCP: its agent registry is never populated in `mcp serve` mode, so every call — even `subagent_type: "general-purpose"` — fails with an empty "Available agents:" list. This reproduces on a fresh `claude mcp serve` process (not just nested calls), across OSes, and has for many CLI versions with no fix landed. Don't spend time debugging your own MCP config for this; it isn't a config problem.

The rest of Claude Code's MCP-exposed tools (Bash, Read, Edit, Write, TaskCreate/List/Get/Update, WebSearch, WebFetch) work fine over MCP. Only the `Agent`/`Task` tool is broken.

**Workaround:** shell out to the `claude` CLI directly (e.g. via Codex's own Bash tool) instead of the MCP `Agent` tool. `claude --bg` gives you a real background session with a registry that loads normally, plus CLI-level primitives (`agents`, `logs`, `attach`, `stop`, `--resume`) that roughly replicate spawn / inspect / steer / stop.

## Quick reference

| Need | Command | Notes |
|---|---|---|
| Spawn a background session | `claude --bg "<prompt>"` | Prints `backgrounded · <short-id>`. Returns immediately. |
| List / check status | `claude agents --json` | Structured: `pid`, `sessionId` (full UUID), `status` (`busy`/`idle`), `state`, `startedAt` (epoch ms), `cwd`. No TTY needed. |
| Inspect progress | `claude logs <short-id>` | Raw terminal capture (ANSI escapes, redraws, spinner frames) — not a clean transcript. Strip with `perl -pe 's/\x1b\[[0-9;?]*[a-zA-Z]//g; s/\x1b\][^\x07]*\x07//g'` and expect it to still be messy. Fine for a quick eyeball, bad for machine parsing. |
| Inspect daemon health | `claude daemon status` | Shows whether the transient supervisor is running, whether `control.sock` is reachable, roster count, and daemon log path. Useful when `claude logs <id>` fails. |
| Inspect daemon events | `timeout 1s claude daemon logs` | Tails `~/.claude/daemon.log`; `timeout` exit code 124 is expected. Shows supervisor starts/exits, control socket binding, and bg worker claim/settle events. |
| Monitor real cost/tokens | Read the session's own transcript JSONL directly | See "Cost/token monitoring" below — much more reliable than parsing `logs`. |
| Steer / send a follow-up | `claude stop <id>` then `claude --resume <full-uuid> -p "<followup>"` | See "Resuming" below — you cannot inject a message into a still-running bg session headlessly. |
| Kill a runaway session | `claude stop <id>` | Real process kill, not graceful. Measured ~1-1.5s from call to the OS process actually disappearing. Safe circuit breaker for cost control. |

## Spawning

```
claude --bg "<prompt>"
```

If this fails with `EPERM ... mkdir '.../.claude/jobs/...'`, you're in a filesystem sandbox that blocks writes to `~/.claude/jobs`; it's not a real product bug, just re-run with sandboxing disabled for that command.

Useful flags: `--model <alias>` (e.g. `haiku` for cheap smoke tests), `--agent <name>` / `--agents <json>` to pick a persona.

## Delegating implementation or review work

Use `claude --bg` as a supervised worker, not as a fire-and-forget replacement
for your own final judgment. This flow is worth the overhead for larger
implementation spikes, independent design attempts, or second-pass code review;
for routine small fixes, the process is usually more ceremony than value.

Before spawning, decide the worktree policy and say it explicitly in the
prompt:

- **Candidate branch/worktree:** ask Claude to isolate the work in a dedicated
  worktree/branch and produce a candidate patch there. This is the preferred
  default for substantial implementation work because it avoids mixing
  Claude's edits with the caller's dirty working tree.
- **Current checkout only:** ask Claude not to enter a worktree if the desired
  output is a direct edit to the current checkout.
- **Review only:** tell Claude the run is read-only and must not edit files,
  commit, push, or open pull requests.

Unless the user explicitly requested publishing, include a hard scope limit in
the prompt:

```
Do not commit, push, or open a pull request unless explicitly requested.
When finished, summarize the files changed and verification performed.
```

If you do allow a commit, still treat it as a candidate artifact. Inspect the
diff yourself, run the relevant checks in the directory where Claude actually
worked, and report the result as "Claude produced this candidate" rather than
as complete until you have verified it.

Practical handoff checklist after Claude reports completion:

1. `claude agents --json` to confirm whether the session is still busy and to
   see its actual `cwd` (it may be a retained `.claude/worktrees/...` checkout).
2. Read the transcript JSONL for the final summary and failed tool calls; prefer
   this over `claude logs` for anything machine-readable.
3. Run `git status`, `git show --stat`, and the task's relevant checks in
   Claude's actual worktree.
4. Review the resulting API/diff yourself, especially for overreach beyond the
   prompt.
5. Stop any still-busy background worker before ending your turn.

## Inspecting

`claude agents --json` is the structured source of truth for whether a session is still running (`"status":"busy"` vs `"idle"`) and how long it's been going (`startedAt`). Prefer polling this over screen-scraping `claude logs`.

`claude logs <id>` gives you the current terminal buffer, which is genuinely useful for a human glance but is a rendered TUI snapshot, not discrete messages — don't build automated loop-detection on top of it (see below for a better signal).

If `claude logs <id>` fails with `connect ECONNREFUSED .../control.sock`, check the daemon layer:

```
claude daemon status
timeout 1s claude daemon logs
```

`claude daemon logs` has no non-follow mode, so wrap it with `timeout`; exit code 124 means the timeout stopped the tail successfully. The daemon log is not a session transcript, but it is useful for lifecycle debugging: supervisor start/idle-exit, control socket binding, spare worker spawn, and which short ids were claimed or settled. In this Codex environment, `claude logs <id>` may need to run outside the filesystem sandbox to reach the daemon socket; `claude daemon status`, `timeout 1s claude daemon logs`, and direct transcript JSONL reads worked sandboxed in the observed setup.

If sandboxed `claude agents --json` reports no background session while
`claude --resume <uuid> -p ...` still says the session is bg-owned, trust the
daemon-side view: rerun `claude agents --json` and `claude stop <id>` outside
the sandbox. This mismatch has been observed when a background job moved into a
retained Claude worktree; the sandboxed roster and daemon state disagreed until
the daemon was queried directly.

## Resuming / steering — what actually works

- `claude --resume <full-uuid> -p "<followup>"` **fails** while the session is still bg-owned/running: `Session is currently running as a background agent (bg). Use 'claude agents' to find and attach to it, or add --fork-session to branch off a copy.`
- `--fork-session` works immediately, but it **branches a new session** (different `session_id` in the result) — it continues the *history*, not the live process. Fine for "ask it something based on what it did," not for steering the original run.
- The only way to continue the **same** session: `claude stop <id>` first (or wait for it to go idle), then `claude --resume <full-uuid> -p "<followup>"` (no fork flag). Verified this is genuine continuation, not a rebuild: the resumed call's `cache_read_input_tokens` matched the original session's accumulated context and `total_cost_usd` dropped by ~20x vs the forked version, because it read cache instead of recreating it.
- `claude attach <id>` opens the session in a real terminal for live back-and-forth, but it requires an actual TTY. Run from a non-interactive shell (no pty) and it just flashes alternate-screen escape codes and exits instantly — it cannot be used to steer a session headlessly (e.g. from Codex's plain Bash tool). If real mid-run steering is a hard requirement, the only path is allocating a pty (`script`, a pty library) and driving `attach` interactively, or using `claude -p --input-format stream-json --output-format stream-json` (a long-lived pipeable session) — both are meaningfully heavier than a shell-out.

## Killing a runaway session

`claude stop <id>` is a real, fast process kill:

- Measured: the `stop` command itself returned in ~0.7s, and the target PID was confirmed gone (checked via `ps -p <pid>`) within well under another second after that — total budget roughly 1-1.5s from calling `stop` to the process being dead.
- It is **not** "let the current turn finish, then stop." It interrupts mid-turn, mid-tool-call.
- Side effect: once stopped, `claude logs <id>` immediately reports `job not found — it may have already exited`. You lose the ability to inspect exactly where it was cut off. Doesn't matter for cost containment, matters for post-mortems — pull `claude logs <id>` *before* you call `stop` if you want the last state.
- The conversation itself is **not** deleted — `claude stop <id>` keeps it resumable (see Resuming above); it's only the live process and its terminal buffer that go away.

## Cost/token monitoring (the reliable signal for "is this looping expensively?")

Don't try to infer runaway cost from `claude logs` text patterns. Instead read the session's transcript file directly:

```
~/.claude/projects/<cwd-with-slashes-replaced-by-dashes>/<full-session-uuid>.jsonl
```

(e.g. `/Users/you/.claude/projects/-Users-you-Projects-myrepo/<uuid>.jsonl`). This file is appended to **live**, in near-real-time, as the session runs — confirmed by tailing it during an active loop and watching new entries land in sync with the loop's actual cadence, not all at once at exit.

Each line is one JSON event. Lines with `"type": "assistant"` carry a `message.usage` object (`input_tokens`, `output_tokens`, `cache_read_input_tokens`, `cache_creation_input_tokens`) and `message.model`.

**Critical gotcha:** one billed turn is split across *multiple* JSONL lines — one per content block (`thinking`, `text`, `tool_use`) — and every one of those lines repeats the *identical, full-turn* usage total. Summing every assistant-type line overcounts by however many content blocks the average turn has (commonly 2-3x). You must dedupe by `message.id` (not `uuid`/`parentUuid`, and not `requestId` if a turn is ever retried) before summing:

```python
seen_msg_ids = set()
totals = {"input_tokens": 0, "output_tokens": 0, "cache_read_input_tokens": 0, "cache_creation_input_tokens": 0}
for line in open(jsonl_path):
    d = json.loads(line)
    if d.get("type") != "assistant":
        continue
    msg = d["message"]
    if msg["id"] in seen_msg_ids:
        continue
    seen_msg_ids.add(msg["id"])
    u = msg.get("usage", {})
    for k in totals:
        totals[k] += u.get(k, 0)
```

No dollar figure is stored in the transcript — only raw token counts plus `message.model`. Apply that model's published per-token rate yourself to get a running `$` estimate (mirrors what `-p --output-format json`'s `total_cost_usd` computes internally, which you only get at the very end of a `-p` call).

**Practical watchdog recipe:**
1. `claude --bg "<task>"`, capture the short id.
2. Resolve the full UUID + jsonl path via `claude agents --json` (match on `id`).
3. Poll the jsonl file, dedupe by `message.id`, accumulate tokens; cross-check `startedAt` from `agents --json` for a simple elapsed-time signal too.
4. If cumulative tokens or elapsed time blows past what the task should reasonably need, `claude stop <id>` — it'll be dead in about a second.

See `references/session-jsonl-shape.md` for the full set of line `type`s seen in a real transcript, if you need to parse more than just usage.
