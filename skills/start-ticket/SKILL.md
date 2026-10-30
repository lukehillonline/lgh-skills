---
name: start-ticket
description: Start work on a Jira ticket (or on a problem with no ticket yet) — fetch it via the Atlassian (Jira) MCP or gather the problem from the user, clarify every open question, optionally create the ticket, confirm understanding, then hand off to the chosen planning skill (paul, gsd, superpowers, or another the user names), or stop there if the user only wants tickets created. Use when the user runs /start-ticket or asks to "start ticket PROJ-123".
---

# Start ticket

## Flow rules (apply to every step)

Follow this flow in order. Every extra message and tool result costs tokens, so batch wherever a step allows it.

- **Batch each step.** Make a step's reads in one message. Ask all its questions in one AskUserQuestion call (4 at most). Make its writes in one message.
- **Two gates stand alone.** The summary confirmation in **4** and the payload confirmation in **5** are always the only thing in their message, because the user has to read them. In the tickets-only loop, each ticket goes through the whole flow on its own; never batch tickets.
- **Ask the way the step says.** If a step says to use AskUserQuestion, use AskUserQuestion with the options the step lists. Never swap it for a table, a list of proposed values, a "confirm all of these" prompt, or plain-text questions.
- **Never pre-fill later steps.** Don't propose or decide answers to a question before reaching its step, even if you can guess them.
- **Corrections restart the step.** When the user corrects something, apply the change, show what changed, and ask for confirmation again with the same step's question. Move on only after an explicit confirmation.
- **No shortcuts.** Never skip or reorder steps, and never improvise a different approach. If a step can't be followed as written, stop and tell the user why.
- **Fetch once, fetch small.** Reuse the `cloudId`, the `accountId` and every list you already fetched in this run. Pass `responseFields` to `executeRead`, and `fields` to `searchJiraIssuesUsingJql`. This skill names every operation it needs: use `discover` only if one is rejected as unknown.

## 0. Preconditions

The `mcp__atlassian__*` tools must be available. If no `mcp__atlassian__*` tools exist at all, stop and tell the user to connect the Atlassian MCP.

They may be deferred. Load them in one `ToolSearch` call as soon as you know the path:

- **Ticket key known:** `getAccessibleAtlassianResources`, `atlassianUserInfo`, `getJiraIssue`, `editJiraIssue`, `transitionJiraIssue`, `listJiraIssueComments`, `executeRead`.
- **Creating a ticket:** `getAccessibleAtlassianResources`, `atlassianUserInfo`, `createJiraIssue`, `searchJiraIssuesUsingJql`, `transitionJiraIssue`, `executeRead`.

Load any other tool the same way when you first need it, in the same message as other loads.

## 1. Ask the opening questions

First read the skill's args. A Jira key answers the ticket question, and a planner name answers the planner question.

If the args contain a Jira key, go straight to **1A**. It asks the planner question too.

Otherwise ask the remaining questions in a single AskUserQuestion call:

1. **Ticket** (header `Ticket`): "Which Jira ticket are you starting? Pick Other and type the key, e.g. PROJ-123."
   - `No ticket`: I'll describe the problem instead.
   - `I'll paste it in chat`: the user will send the key as their next message.
2. **Planner** (header `Planner`): "Which planning skill should we use?"
   - `paul`
   - `gsd`
   - `superpowers`
   - `none`: I just want to create tickets.

   The user can pick Other to name a different planning skill.

Rules for the ticket answer:

- If the user types only a number (no project key), ask for the full key. Never guess the project.
- If they pick `I'll paste it in chat`, wait for the key.

Branch on the ticket answer:

- A ticket key was given: go to **1A**.
- `No ticket` was picked: go to **2B**.

## 1A. Claim the ticket

Runs as soon as a ticket key is known, before anything else happens with it.

**Reads.** Call `getAccessibleAtlassianResources` for the `cloudId` and `atlassianUserInfo` for the `accountId`, in one message. Then make these in one message:

- `getJiraIssue` for the key with `view: "evidence"`. This is the only fetch of the ticket, so **2A** reuses it. If the ticket can't be found, stop and tell the user.
- `executeRead` with `listJiraIssueTransitions`, `inputs: { "issueIdOrKey": "<KEY>" }` and `responseFields: ["transitions.id", "transitions.name", "transitions.to.name"]`.
- `test -f .planning/tickets/<KEY>.md` to check for a saved summary (see **4**).

**Questions.** Ask whichever of these apply in one AskUserQuestion call:

1. **Assign:** whether to assign the ticket to the user, naming the current assignee if there is one. If it's already assigned to the user, say so and leave this out.
2. **Move:** which column to move the ticket to, naming its current status. Offer up to 3 transitions, the most likely first (usually In Progress), plus `Leave it in <current status>`. The user can pick Other to type a column. Match a typed name against `to.name`, and if it matches none or more than one, show the options and ask again. Never guess.
3. **Resume:** if a saved summary exists, whether to resume from it or start fresh.
4. **Planner:** the planner question from **1**, if the args didn't answer it.

**Writes.** Make these in one message:

- If the user said yes to Assign, `editJiraIssue` with `fields: { "assignee": { "accountId": "<id>" } }`.
- If the user picked a column, `transitionJiraIssue` with its `transitionId`. If it fails, retry it once now the assign has finished, because some workflows only let the assignee move a ticket.

Then branch:

- **Resume:** read the file, show the summary, and go to **6**. Skip fetching and clarifying: the file is the confirmed summary.
- **Start fresh** (or no file): go to **2A**.

## 2A. Ticket path: read the rest of the ticket

Use the ticket from **1A**. In one message, fetch what it doesn't include:

- If `comment.total` is above 0, call `listJiraIssueComments` for the comments.
- If the description depends on the parent, the epic, or linked or blocking issues, fetch those with `getJiraIssue`. The default `compact` view is enough unless you need their links.
- Fetch any linked Confluence pages with `getConfluenceContent`.

Then go to **3**.

## 2B. No-ticket path: describe the problem

Ask the user, in plain chat, to describe the problem or change in their own words. Wait for their answer, then go to **3**.

## 3. Clarify until there is 100% clarity

Read the ticket or the user's description together with the parts of the codebase it touches. Search first, and read only the files and ranges that matter. Where helpful, use `.planning/codebase/*.md` for orientation. Then list every gap, such as:

- ambiguous or missing acceptance criteria
- which apps, packages or modules are affected
- edge cases, error states, and empty or loading states
- design or UX details that aren't specified
- API contracts and backend dependencies
- conflicts with the current code
- scope boundaries: what is explicitly out of scope

Ask these with AskUserQuestion, up to 4 per call. Keep asking rounds until **no** open question remains. Where the code or ticket points to a sensible answer, put it first and mark it `(Recommended)`, with a one-line reason.

From the second round on, if open questions remain, make the first question of the call: "<N> questions left. Keep going, or take the recommended answer for all of them?" If the user takes the recommended answers, list each question with the answer you chose, record them as "Decided by Claude, delegated by user", and go to **4**. Any question without a recommended answer still has to be asked.

If the work is a bug (the ticket type is Bug, or the user describes broken behaviour), include in the first round whether they want to link a Sentry issue. If they do:

- If no `mcp__sentry__*` tools exist, tell the user the Sentry MCP isn't connected. Offer to take a pasted Sentry link instead, or carry on without one.
- Otherwise, load the tools with `ToolSearch` and use `search_issues` with keywords from the bug to find candidates. Show the best matches (title, event count, last seen, link) and let the user pick one, paste a link, or skip. Never link an issue the user hasn't picked.

Keep the chosen Sentry link for the summary and, when you create a ticket, for its Design/Evidence section.

Assume nothing. Every fact must come from the ticket, the code, or the user. If you catch yourself inferring something, turn it into a question instead. If the user answers "your call" (or similar), choose the recommended option and record it in the summary as "Decided by Claude, delegated by user". That is not an assumption, because the user handed over the choice.

## 4. Confirm understanding

Present a summary to the user:

- **Ticket:** key, title and link, or "none yet"
- **Goal:** one or two sentences
- **Scope:** in scope and out of scope
- **Acceptance criteria:** as clarified
- **Affected areas:** apps, libs and key files
- **Decisions from clarification:** each question and its answer
- **Risks:** anything to watch for
- **Sentry:** the linked issue, if there is one

Ask the user to confirm the summary or correct it, then stop and wait. After each correction, show the sections that changed and ask again. Loop until they explicitly confirm. Don't move to the next step, or mention it, until then.

Then save the confirmed summary to `.planning/tickets/<KEY>.md`. With no ticket yet, use `.planning/tickets/<kebab-case-title>.md`, and rename it to the key if a ticket is created in **5**. Before writing it, run `git check-ignore -q .planning/tickets/`. If the folder isn't ignored, append `.planning/tickets/` to the repo root `.gitignore` and tell the user you did. Update this file whenever a later step changes something it records (new key, branch name).

- Ticket path: go to **6**.
- No-ticket path: go to **5**.

## 5. No-ticket path: create a ticket

If the planner is `none`, the user already said they want tickets, so don't ask. Otherwise ask with AskUserQuestion whether you should create a Jira ticket from the confirmed summary.

- **No:** go to **7**, and use "none" as the ticket key. There is no key to name a branch after, so skip branch creation.
- **Yes:** read `create-ticket.md` in this skill's directory and follow it.

## 5A. Tickets-only loop (planner `none`)

Don't claim the ticket, create a branch or plan. Ask in plain chat for the next problem to turn into a ticket, or "stop" to finish.

- **A new problem:** go to **3** with that description. Keep the planner as `none`.
- **Stop:** end the skill. List every ticket created in this run with its link.

## 6. Claim the ticket and create the branch

**Reads.** In one message:

- Run `git ls-remote --heads origin master develop`, `git status --porcelain` and, if `AGENTS.md` exists at the repo root, `grep -in branch AGENTS.md`, chained in one Bash call.
- If there is a ticket key and **1A** hasn't run for it, get what the Claim questions need. For a ticket created in **5**, that is `listJiraIssueTransitions` as in **1A**. For an existing ticket picked in **5**, use all of **1A**'s reads except the saved summary check.

**Questions.** Ask whichever of these apply in one AskUserQuestion call. If there are more than 4, the last ones go in a second call.

1. **Claim:** Assign and Move from **1A**, if **1A** hasn't run. Leave out Assign if the ticket was created with the user as assignee, or the user said no to that.
2. **Branch:** whether the user wants a new branch and from which base. Offer `Branch from <base>` for each base that exists, plus `No branch`.
3. **Comment:** if there is a ticket key and there were decisions, whether to post the **Decisions from clarification** to the ticket as a comment, so the team can see them. Show the exact comment text in the question.
4. **Continue:** on the no-ticket path, whether to continue to planning or stop here.

**Writes.** In one message, make the claim writes as in **1A**, and post the comment with `addOrEditJiraIssueComment` if the user said yes.

If the user picked a branch:

1. If `git status --porcelain` showed uncommitted changes, stop and ask the user how to handle them. Never stash, commit or discard them yourself.
2. Work out `<branch>`:
   - If `AGENTS.md` defines a branch naming pattern, use that pattern instead of the defaults below.
   - Otherwise, Story or Task: `feature/{ticket-number}-{ticket-title}`; Bug: `bug/{ticket-number}-{ticket-title}`; any other type: ask the user which prefix to use.

   `{ticket-number}` is the full key (e.g. `PROJ-123`). `{ticket-title}` is the ticket summary with spaces and special characters replaced by `-`, runs of `-` collapsed, and no leading or trailing `-` (e.g. `feature/PROJ-123-Add-Dark-Mode-Toggle`). If the whole branch name is longer than 60 characters, cut `{ticket-title}` at the last `-` that keeps it within 60.

3. In one Bash call, check whether `<branch>` already exists locally (`git branch --list <branch>`) or on the remote (`git ls-remote --heads origin <branch>`). If neither prints anything, continue with `git checkout <base> && git pull && git checkout -b <branch>`. If it exists, ask whether to check it out or use a different name. When checking out an existing remote branch, use `git checkout <branch>` instead.

Then:

- Ticket path, or the user chose to continue: go to **7**.
- The user chose to stop: end the skill. Tell the user they can plan later by running `/start-ticket <KEY>`, which resumes from the saved summary.

## 7. Hand off to the planner

If the planner is `none`, end the skill here. Tell the user where the saved summary is, and that they can plan later by running `/start-ticket <KEY>`, which resumes from it.

Check that the chosen planner is ready:

- **superpowers:** if no `superpowers:*` skills are listed, ask with AskUserQuestion whether to install it or pick a different planner. To install, tell the user to run these themselves (you can't run `/plugin` commands), then `/reload-plugins`:
  ```
  /plugin marketplace add obra/superpowers-marketplace
  /plugin install superpowers@superpowers-marketplace
  ```
  Wait for them to confirm, then check that the `superpowers:*` skills are listed. If they still aren't, stop and tell the user.
- **Other:** match the name to a skill in the available-skills list. If nothing matches, or more than one skill does, show the user the candidates and ask which one they meant, or ask them to pick one of the three suggestions. Never guess.
- **gsd:** if `.planning/ROADMAP.md` doesn't exist, GSD isn't set up for this repo. Ask with AskUserQuestion whether to set it up now or pick a different planner. To set it up, invoke `gsd-onboard` and come back to this skill when it finishes.
- **paul:** if `.paul/` doesn't exist, PAUL isn't set up for this repo. Ask with AskUserQuestion whether to set it up now (invoke `paul:init`, then come back to this skill) or pick a different planner.

Use the Skill tool to invoke the planner chosen earlier. Pass the path of the saved summary file, the ticket key if there is one, and a one-line goal as args:

- **`paul`:** Read `.paul/STATE.md` and `.paul/ROADMAP.md`, tell the user which milestone and phase are current, and ask with AskUserQuestion where the work goes: a new phase in the current milestone (`paul:add-phase`), or a new milestone (`paul:milestone`). Name it `<TICKET-KEY> <title>`, or just `<title>` when there is no ticket. Then invoke `paul:plan` for the new phase. Never plan into an existing phase.
- **`gsd`:** Invoke `gsd-phase` to add a phase named `<TICKET-KEY> <title>`, or just `<title>` when there is no ticket, with the summary as its goal. Then invoke `gsd-plan-phase` with the new phase number.
- **`superpowers`:** Invoke `superpowers:brainstorming`, which chains into `superpowers:writing-plans`.
- **Other:** Invoke the skill the user chose.

From here on, the planner's workflow is in charge. Do not re-ask questions the summary already settles.
