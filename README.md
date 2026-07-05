# cross-agent-skills

Skills for driving Claude Code from Codex and Codex from Claude Code as local background workers - no cloud.

## Install

### If you use Claude Code

Run these commands in your terminal:

```bash
claude plugin marketplace add fbenkstein/cross-agent-skills
claude plugin install codex-cli-bg-sessions@cross-agent-skills
```

### If you use Codex

Type this into a Codex session:

> Install the skill from github.com/fbenkstein/cross-agent-skills, path `codex-skill`

Or run its installer script directly in your terminal:

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo fbenkstein/cross-agent-skills --path codex-skill
```

## What You Get

- `claude-skill/` installs into Claude Code as `codex-cli-bg-sessions`.
  It drives `codex exec` as a local background worker: spawn, inspect, monitor
  token usage, resume, and kill.
- `codex-skill/` installs into Codex as `claude-code-bg-sessions`.
  It drives `claude --bg` as a local background worker: spawn, inspect, monitor
  cost/token usage, resume, and stop.

Both skills exist because the built-in cross-agent MCP paths are not enough for
background work: Claude Code's MCP `Agent` tool currently has an empty agent
registry, and Codex's MCP tools are synchronous with no background, status,
token, or kill primitives.

## How To Use

Use these for supervised delegation: candidate implementations, independent
design passes, or read-only code review. The caller still reviews the diff,
runs checks, and decides what to keep.

Default prompt guardrails:

```text
Edit files only in the requested checkout. Do not create or enter a worktree.
Do not create a branch, commit, push, or open a pull request unless explicitly requested.
When finished, summarize the files changed and verification performed.
```

For small edits, just do the work directly. The background-agent workflow pays
off when independent reasoning or parallel implementation is worth the overhead.

## Notes

These are living skills, not stable API references. The details in each
`SKILL.md` were measured against specific CLI versions and may drift. If a
finding turns out wrong, update the skill and its methodology notes instead of
working around stale guidance.

See `TODO.md` for planned helper scripts that will reduce manual transcript
parsing and noisy terminal-log handling.

Want to edit these skills in place, get updates via `git pull`, or contribute
a fix? See `HACKING.md`.
