# lgh-skills

A collection of [Claude Code](https://claude.com/claude-code) skills, installable globally as a plugin.

| Skill | What it does |
| --- | --- |
| [`start-ticket`](skills/start-ticket/SKILL.md) | Takes a Jira ticket (or a problem with no ticket yet) from "just picked up" to "ready to plan" |

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

---

## start-ticket

### What it does

`start-ticket` front-loads the thinking on a piece of work. Before any code is planned or written it makes sure the ticket is fully understood, every open question has an answer, and the decisions are written down. It then claims the ticket, creates a branch and hands off to the planning skill of your choice.

It can also be used purely as a ticket writer: describe a problem, and it turns it into a well-structured Jira ticket in the right epic, sprint and component.

### Requirements

- **Atlassian MCP** (required): used to read and create Jira tickets, sprints, epics and Confluence pages.
- **Sentry MCP** (optional): used to find and link a Sentry issue when the work is a bug. Without it you can paste a Sentry link instead.
- **A planning skill** (optional): PAUL, GSD, [superpowers](https://github.com/obra/superpowers), or any other installed skill.

### How to use it

```
/start-ticket                 # asks for everything
/start-ticket PROJ-123        # ticket key given
/start-ticket PROJ-123 gsd    # ticket key and planner given
```

You'll be asked which ticket you're starting (or "No ticket") and which planner to hand off to (`paul`, `gsd`, `superpowers`, another skill, or `none` to only create tickets).

### How it works

The skill follows a strict, one-question-at-a-time flow. It never batches questions from different steps, never pre-fills answers and never skips steps.

1. **Opening questions:** ticket and planner. If a summary for the ticket was saved earlier, you can resume from it.
2. **Gather the problem:** fetches the ticket (with its parent, epic, linked issues and Confluence pages) or asks you to describe the problem.
3. **Clarify:** reads the ticket alongside the code it touches and asks about every gap (acceptance criteria, affected apps, edge cases, UX, API contracts, scope) in rounds until nothing is left. Answers it can infer from the code are offered as `(Recommended)`, and you can accept all recommendations at once. For bugs it offers to link a Sentry issue.
4. **Confirm understanding:** shows a summary (goal, scope, acceptance criteria, affected areas, decisions, risks) and loops on corrections until you confirm. The confirmed summary is saved to `.planning/tickets/<KEY>.md`, and `.planning/tickets/` is added to `.gitignore` if needed.
5. **Create a ticket (no-ticket path):** asks which Jira project to use, checks for duplicates, looks up the active sprint, open epics, components and issue types, and asks you to pick each one. Writes the description in a fixed format (Background, Acceptance Criteria, Tech Notes, Design; bugs get Expected/Actual steps too), shows the exact payload, and only creates it after an explicit "yes".
6. **Claim and branch:** optionally assigns the ticket to you, moves it to In Progress, and creates a branch from `master`/`develop`. The name comes from `AGENTS.md` if it defines a pattern, otherwise `feature/<KEY>-<title>` or `bug/<KEY>-<title>`, capped at 60 characters. It never touches uncommitted changes.
7. **Hand off:** optionally posts the clarification decisions to the ticket as a comment, checks the planner is installed and set up, then invokes it with the saved summary.

With planner `none`, it loops: create a ticket, then offer to create another, and finish with a list of everything it created.

---

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
```

### Adding or changing a skill

1. Fork the repo and create a branch.
2. Add a folder under `skills/` with a `SKILL.md`. The frontmatter needs a `name` (matching the folder) and a `description` that says what the skill does and when Claude should use it. Supporting files (templates, reference docs) can sit next to `SKILL.md`.
3. Test it locally by loading the plugin from your checkout:
   ```bash
   claude --plugin-dir /path/to/lgh-skills
   ```
   Run the skill end to end, including the edge cases you changed.
4. Add the skill to the table at the top of this README, with a section explaining what it does and how to use it.
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
