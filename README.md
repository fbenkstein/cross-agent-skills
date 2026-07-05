# cross-agent-skills

Skills for driving Claude Code from Codex and Codex from Claude Code as local background workers — no cloud.

- **`claude-skill/`** — installs into Claude Code as `codex-cli-bg-sessions`. Drive the `codex` CLI as a background worker: spawn, inspect, monitor real token usage, resume, and kill `codex exec` sessions.
- **`codex-skill/`** — installs into Codex as `claude-code-bg-sessions`. Drive the `claude` CLI as a background worker: spawn, inspect, monitor real cost/token usage, resume, and kill `claude --bg` sessions.

Both exist because each agent's own MCP tool for driving the other is either broken (`claude-code` MCP's `Agent` tool has an empty agent registry — anthropics/claude-code#41973) or fully synchronous with no background/cost/kill primitives (Codex's `codex`/`codex-reply` MCP tools).

## Delegation model

These skills are best used for supervised delegation: ask the other agent to
produce a candidate implementation, independent design pass, or read-only code
review, then have the caller inspect the diff, run checks, and decide what to
keep. For routine small edits, the process overhead usually outweighs the
benefit.

By default, prompts should tell background workers not to commit, push, or open
pull requests unless the user explicitly requested that publishing step. If a
worker uses a retained worktree, run verification in that worktree rather than
assuming the original checkout changed.

See `TODO.md` for planned wrapper scripts that will make this workflow less
dependent on noisy terminal logs and manual transcript parsing.

## These are living skills, not a fixed reference

Everything in both `SKILL.md`s was derived by live experimentation against specific CLI versions (see each skill's `references/methodology.md`), not from stable documentation. Version numbers drift, error strings change, timings shift, upstream bugs get fixed. Each `SKILL.md` says so up front and asks whoever's using it to note a version mismatch without treating it as a crisis — and, if something in the skill actually turns out wrong (like the untrusted-project approval assumption that got corrected after being tested for real), to fix the file in place rather than work around it silently. That only works if the running skill and the repo are the same files — see below.

## Setup (recommended: clone + symlink — lets both agents maintain these skills in place)

Both agents' plugin/skill installers copy files with no git metadata — the installed copy can't be recognized as this repo, edited in place usefully, or pushed back, and picking up updates means a manual reinstall dance (Claude: `marketplace update` → `plugin update` → restart; Codex: no update command at all, delete and reinstall). Cloning and symlinking the live skill directories straight into the working tree avoids all of that: edits made through the normal skill path *are* edits to this repo, ready to commit and push directly, by either agent, as they learn things.

```
git clone git@github.com:fbenkstein/cross-agent-skills.git ~/Projects/cross-agent-skills
ln -s ~/Projects/cross-agent-skills/claude-skill ~/.claude/skills/codex-cli-bg-sessions
ln -s ~/Projects/cross-agent-skills/codex-skill ~/.codex/skills/claude-code-bg-sessions
```

Restart the respective CLI (or Claude: `/reload-plugins`) to pick up the skill.

**Getting updates from elsewhere:** since the skill directories are symlinks into a normal clone, there's no reinstall step — just update the checkout and the running skill updates with it:

```
cd ~/Projects/cross-agent-skills && git pull
```

**Contributing a correction:** edit the file through the live path (or the clone directly, it's the same file), then commit and push from `~/Projects/cross-agent-skills` as usual. If a finding turns out wrong (confirmed by testing, not just suspected), update the skill's `references/methodology.md` to say so rather than leaving a stale claim in place — that file exists specifically to track what was verified versus inferred.

## Read-only convenience consumption (no push-back)

For a quick "just use it, once, no intent to maintain it" install, both agents' native mechanisms still work — the `.claude-plugin/` manifests are kept valid for this purpose. The tradeoff versus clone + symlink: what you get is a disconnected snapshot with no git metadata, so you can't push fixes back, and picking up upstream changes later means an explicit reinstall (Codex: delete and reinstall; Claude: `marketplace update` → `plugin update` → restart), not a `git pull`.

**Into Claude Code:**

```
claude plugin marketplace add fbenkstein/cross-agent-skills
claude plugin install codex-cli-bg-sessions@cross-agent-skills
```

**Into Codex:** ask Codex (its built-in `skill-installer` skill handles this):

> Install the skill from github.com/fbenkstein/cross-agent-skills, path `codex-skill`

or run its installer script directly:

```
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo fbenkstein/cross-agent-skills --path codex-skill
```
