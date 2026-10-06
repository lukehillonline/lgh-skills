# lgh-skills

A collection of [Claude Code](https://claude.com/claude-code) skills, installable globally as a plugin.

| Skill | What it does |
| --- | --- |
| [`start-ticket`](docs/start-ticket.md) | Takes a Jira ticket (or a problem with no ticket yet) from "just picked up" to "ready to plan" |

| Output style | What it does |
| --- | --- |
| [`dev-talk`](docs/dev-talk.md) | Always on. Short, plain, factual, direct responses: no praise, enthusiasm or filler. Also applies to commits, PRs, tickets and code comments |

## Installation

### As a plugin (recommended)

Inside Claude Code:

```
/plugin marketplace add lukehillonline/lgh-skills
/plugin install lgh-skills@lgh-skills
/reload-plugins
```

The skills are then available in every project. Plugin skills are namespaced, so run it as `/lgh-skills:start-ticket`.

To update later: `/plugin marketplace update lgh-skills`.

### Manually

Clone the repo and symlink the skill into your personal skills folder, so `git pull` keeps it up to date:

```bash
git clone https://github.com/lukehillonline/lgh-skills.git
ln -s "$(pwd)/lgh-skills/skills/start-ticket" ~/.claude/skills/start-ticket
```

Installed this way it runs as `/start-ticket`.

## Contributing

Contributions are welcome, whether that's a fix, an improvement to an existing skill, or a new skill.

### Repo layout

```
.claude-plugin/
  plugin.json         # plugin manifest
  marketplace.json    # lets the repo be added with /plugin marketplace add
skills/
  <skill-name>/
    SKILL.md          # the skill itself
output-styles/        # every .md here loads as a style, so no README
  <style-name>.md     # output styles
hooks/
  hooks.json          # plugin hooks
docs/
  <name>.md           # one doc per skill or output style
```

### Adding or changing a skill

1. Fork the repo and create a branch.
2. Add a folder under `skills/` with a `SKILL.md`. The frontmatter needs a `name` (matching the folder) and a `description` that says what the skill does and when Claude should use it. Supporting files (templates, reference docs) can sit next to `SKILL.md`.
3. Test it locally by loading the plugin from your checkout:
   ```bash
   claude --plugin-dir /path/to/lgh-skills
   ```
   Run the skill end to end, including the edge cases you changed.
4. Add a `docs/<name>.md` explaining what it does and how to use it, and add a row linking to it in the table at the top of this README.
5. Bump `version` in `.claude-plugin/plugin.json`.
6. Open a pull request describing the change and how you tested it.

### Guidelines

- Write skills as clear, numbered steps. Claude follows them literally, so say exactly what to ask, when to stop, and what never to do.
- Don't hardcode company-specific values (project keys, app names, URLs). Ask for them or read them from the repo.
- Keep each skill focused on one job.

### Issues

Found a bug or have an idea? [Open an issue](https://github.com/lukehillonline/lgh-skills/issues).

## License

[MIT](LICENSE)
