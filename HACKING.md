# Hacking on these skills

The install methods in `README.md` (plugin marketplace for Claude, the
skill-installer for Codex) copy files with no git metadata — the installed
copy can't be recognized as this repo, edited in place usefully, or pushed
back, and picking up updates means a manual reinstall dance (Claude:
`marketplace update` → `plugin update` → restart; Codex: no update command at
all, delete and reinstall).

If you plan to maintain or contribute fixes to these skills, clone the repo
and symlink the live skill directories straight into place instead. Edits made
through the normal skill path *are* edits to this repo, ready to commit and
push directly, by either agent, as they learn things.

## Setup

```bash
git clone git@github.com:fbenkstein/cross-agent-skills.git ~/Projects/cross-agent-skills

mkdir -p ~/.claude/skills ~/.codex/skills
ln -sfn ~/Projects/cross-agent-skills/claude-skill ~/.claude/skills/codex-cli-bg-sessions
ln -sfn ~/Projects/cross-agent-skills/codex-skill ~/.codex/skills/claude-code-bg-sessions
```

Restart the respective CLI (or Claude: `/reload-plugins`) to pick up the
skill.

## Getting updates

Since the skill directories are symlinks into a normal clone, there's no
reinstall step — just update the checkout and the running skill updates with
it:

```bash
cd ~/Projects/cross-agent-skills && git pull
```

## Contributing a correction

Edit the file through the live skill path (or the clone directly — it's the
same file), then commit and push from `~/Projects/cross-agent-skills` as
usual. If a finding turns out wrong (confirmed by testing, not just
suspected), update the skill's `references/methodology.md` to say so rather
than leaving a stale claim in place — that file exists specifically to track
what was verified versus inferred.
