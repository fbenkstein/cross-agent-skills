# TODO

Code work to make the cross-agent background workflows easier to reuse.

## Codex-side Claude wrappers

- Add a small `claude-bg-start` helper that runs `claude --bg`, captures the
  short id, resolves the full session UUID with `claude agents --json`, records
  the inferred transcript path, and prints the session's actual `cwd`.
- Add `claude-bg-status` to print concise state from `claude agents --json`:
  status, cwd/worktree, elapsed time, and the latest meaningful transcript
  events.
- Add `claude-bg-tail` to parse the session JSONL transcript and show only
  assistant text, tool calls, command failures, commits, pushes, and final
  summaries. Truncate large tool outputs and avoid `claude logs` except as a
  last-resort human TUI snapshot.
- Add `claude-bg-usage` to dedupe assistant transcript lines by `message.id`
  and report token totals.
- Add `claude-bg-stop` to run `claude stop <id>` and verify the worker is gone
  with `claude agents --json`, retrying outside a sandbox when needed.
- Add `claude-bg-result` to show `git status`, `git show --stat HEAD`, changed
  files, and the pushed branch if Claude committed in a retained worktree.

## Launch guardrails

- Make the launch helper append this default instruction unless overridden:
  `Do not commit, push, or open a pull request unless explicitly requested.`
- Provide explicit opt-in flags such as `--allow-commit`, `--allow-push`, and
  `--allow-pr` that add matching prompt text and make the scope visible in the
  wrapper output.
- Add a `--review-only` mode that tells Claude not to edit files or run
  mutating commands.
- Add a `--worktree`/`--no-worktree` option so the caller decides whether
  Claude should isolate implementation work.

## Cross-agent symmetry

- Mirror the same helper shape for the Claude-side `codex exec` workflow where
  it makes sense, using Codex's clean stdout JSONL instead of transcript
  scraping.
- Document wrapper output formats so both agents can consume them without
  brittle terminal-log parsing.

## Verification

- Add smoke tests or golden transcript fixtures for the transcript parsers.
- Include examples for an implementation delegation, a read-only code review,
  and a runaway-session stop.
