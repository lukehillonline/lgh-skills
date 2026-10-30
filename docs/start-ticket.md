# start-ticket

[← Back to lgh-skills](../README.md) · [SKILL.md](../skills/start-ticket/SKILL.md)

## What it does

`start-ticket` front-loads the thinking on a piece of work. Before any code is planned or written it makes sure the ticket is fully understood, every open question has an answer, and the decisions are written down. It then claims the ticket, creates a branch and hands off to the planning skill of your choice.

It can also be used purely as a ticket writer: describe a problem, and it turns it into a well-structured Jira ticket in the right epic, sprint and component.

## Requirements

- **Atlassian MCP** (required): used to read and create Jira tickets, sprints, epics and Confluence pages.
- **Sentry MCP** (optional): used to find and link a Sentry issue when the work is a bug. Without it you can paste a Sentry link instead.
- **A planning skill** (optional): PAUL, GSD, [superpowers](https://github.com/obra/superpowers), or any other installed skill.

## How to use it

```
/start-ticket                 # asks for everything
/start-ticket PROJ-123        # ticket key given
/start-ticket PROJ-123 gsd    # ticket key and planner given
```

You'll be asked which ticket you're starting (or "No ticket") and which planner to hand off to (`paul`, `gsd`, `superpowers`, another skill, or `none` to only create tickets).

## How it works

The skill follows a strict, ordered flow built to use as few tokens as it can. Each step makes its Jira reads together, asks its questions in one prompt and makes its Jira writes together. The summary and the ticket payload are always confirmed on their own. It never pre-fills answers and never skips steps.

1. **Opening questions:** ticket and planner. As soon as a ticket key is known, it fetches the ticket and asks in one prompt whether to assign it to you, which column to move it to, and which planner to use if you haven't said. If a summary for the ticket was saved earlier, you can resume from it.
2. **Gather the problem:** adds what the first fetch doesn't cover (comments, and the parent, epic, linked issues and Confluence pages where needed), or asks you to describe the problem.
3. **Clarify:** reads the ticket alongside the code it touches and asks about every gap (acceptance criteria, affected apps, edge cases, UX, API contracts, scope) in rounds until nothing is left. Answers it can infer from the code are offered as `(Recommended)`, and you can accept all recommendations at once. For bugs it offers to link a Sentry issue.
4. **Confirm understanding:** shows a summary (goal, scope, acceptance criteria, affected areas, decisions, risks) and loops on corrections until you confirm. The confirmed summary is saved to `.planning/tickets/<KEY>.md`, and `.planning/tickets/` is added to `.gitignore` if needed.
5. **Create a ticket (no-ticket path):** asks which Jira project to use, then checks for duplicates and looks up the active sprint, open epics, components and issue types together. You pick each one in one prompt. Writes the description in a fixed format (Background, Acceptance Criteria, Tech Notes, Design; bugs get Expected/Actual steps too), shows the exact payload, asks for any links, and only creates it after an explicit "yes". The ticket is assigned to you on creation unless you say no. These instructions live in `create-ticket.md`, so they only load when needed.
6. **Claim and branch:** asks in one prompt about the column for a ticket created in step 5, whether to branch from `master`/`develop`, and whether to post the clarification decisions to the ticket as a comment. The branch name comes from `AGENTS.md` if it defines a pattern, otherwise `feature/<KEY>-<title>` or `bug/<KEY>-<title>`, capped at 60 characters. It never touches uncommitted changes.
7. **Hand off:** checks the planner is installed and set up, then invokes it with the saved summary.

With planner `none`, it loops: create a ticket, then ask for the next problem, and finish with a list of everything it created.
