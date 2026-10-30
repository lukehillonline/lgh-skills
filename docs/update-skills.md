# update-skills

[← Back to lgh-skills](../README.md) · [SKILL.md](../skills/update-skills/SKILL.md)

## What it does

`update-skills` updates everything you have installed from this repo in one go, so you don't have to remember the separate commands.

## Requirements

- The `claude` CLI on your `PATH`.
- `lgh-skills` version 0.8.0 or later. Older installs need updating by hand once to get this skill:
  ```bash
  claude plugin marketplace update lgh-skills
  claude plugin update lgh-skills@lgh-skills
  ```

## How to use it

```
/lgh-skills:update-skills     # plugin install
/update-skills                # symlinked install
```

Restart Claude Code afterwards. Updates only apply after a restart.

## How it works

1. **Find what's installed:** plugins from the `lgh-skills` marketplace, and symlinks in `~/.claude/skills` that point into a clone of this repo.
2. **Update plugins:** refreshes the marketplace, then updates each installed plugin, including `dev-talk` if you have it.
3. **Update clones:** runs `git pull --ff-only` in each clone. It never stashes or discards your changes.
4. **Report:** shows old and new versions, and anything that failed.

It only updates what you already have, and never installs anything new.
