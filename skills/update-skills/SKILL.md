---
name: update-skills
description: Update every installed skill and plugin from the lgh-skills repo to the latest version — plugin installs through the claude plugin CLI, symlinked installs through git pull. Use when the user runs /update-skills or asks to "update my skills" or "update lgh-skills".
---

# Update skills

Updates everything installed from `lgh-skills`, then reports what changed. Run the steps in order.

## Rules

- Never pass `-y` or `--yes`, and never accept a marketplace-declared command for the user. If an update needs confirming, stop and show them the command to run themselves.
- Never stash, reset or discard anything in a clone. If a pull fails, report it and carry on.
- Only update what is already installed. Never install anything new.

## 1. Find what's installed

Run these in one Bash call:

```bash
claude plugin list
find ~/.claude/skills -maxdepth 1 -type l 2>/dev/null | while read -r link; do
  git -C "$link" remote get-url origin 2>/dev/null | grep -q 'lgh-skills' && git -C "$link" rev-parse --show-toplevel
done | sort -u
```

- **Plugins:** every entry in `claude plugin list` ending in `@lgh-skills`. Note each one's version.
- **Clones:** every path the second command prints. These are symlinked installs.

If there are neither, stop and point the user to the install steps: https://github.com/lukehillonline/lgh-skills#installation

## 2. Update the plugins

If there are plugins, refresh the marketplace, then update each one:

```bash
claude plugin marketplace update lgh-skills
claude plugin update <plugin>@lgh-skills
```

If the marketplace update fails, stop and show the error, because the plugin updates would find nothing new. If one plugin fails, keep going with the others.

## 3. Update the clones

For each clone:

```bash
git -C <clone> pull --ff-only
```

## 4. Report

Run `claude plugin list` again, then report:

- each plugin's old and new version, or that it was already up to date
- each clone's result
- anything that failed, with its error

If any plugin changed, end with one line telling the user to restart Claude Code, because updates only apply after a restart.
