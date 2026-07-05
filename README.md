# cross-agent-skills

Skills for driving Claude Code from Codex and Codex from Claude Code as local background workers - no cloud.

## Install

```bash
git clone git@github.com:fbenkstein/cross-agent-skills.git ~/Projects/cross-agent-skills

mkdir -p ~/.claude/skills ~/.codex/skills
ln -sfn ~/Projects/cross-agent-skills/claude-skill ~/.claude/skills/codex-cli-bg-sessions
ln -sfn ~/Projects/cross-agent-skills/codex-skill ~/.codex/skills/claude-code-bg-sessions
```

Restart Codex and Claude Code. In Claude Code, `/reload-plugins` may be enough.

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

## Updating

Because the installed skills are symlinks into this clone, updates are just:

```bash
cd ~/Projects/cross-agent-skills
git pull
```

Edits made through the live skill path are edits to this repo, so corrections
can be committed and pushed normally.

## Notes

These are living skills, not stable API references. The details in each
`SKILL.md` were measured against specific CLI versions and may drift. If a
finding turns out wrong, update the skill and its methodology notes instead of
working around stale guidance.

See `TODO.md` for planned helper scripts that will reduce manual transcript
parsing and noisy terminal-log handling.
