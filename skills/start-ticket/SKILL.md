---
name: start-ticket
description: Start work on a Jira ticket (or on a problem with no ticket yet) — fetch it via the Atlassian (Jira) MCP or gather the problem from the user, clarify every open question, optionally create the ticket, confirm understanding, then hand off to the chosen planning skill (paul, gsd, superpowers, or another the user names), or stop there if the user only wants tickets created. Use when the user runs /start-ticket or asks to "start ticket PROJ-123".
---

# Start ticket

## Flow rules (apply to every step)

Follow this flow rigidly. These rules override any urge to be efficient.

- **One step at a time.** Do the current step, ask its question, then stop and wait for the answer. Never start the next step, or show anything from it, in the same message.
- **Never group.** Never combine questions or confirmations from different steps in one message. This includes the summary confirmation, the ticket details (epic, sprint, component, type), the links request and the payload confirmation: each is its own step. In the tickets-only loop, each ticket goes through the whole flow on its own; never batch tickets.
- **Ask the way the step says.** If a step says to use AskUserQuestion, use AskUserQuestion with the options the step lists. Never swap it for a table, a list of proposed values, a "confirm all of these" prompt, or plain-text questions.
- **Never pre-fill later steps.** Don't propose or decide answers to a question before reaching its step, even if you can guess them.
- **Corrections restart the step.** When the user corrects something, apply the change, show the full updated result, and ask for confirmation again with the same step's question. Move on only after an explicit confirmation.
- **No shortcuts.** Never skip, reorder or merge steps, and never improvise a different approach. If a step can't be followed as written, stop and tell the user why.

## 0. Preconditions

The `mcp__atlassian__*` tools must be available. They may be deferred: load the ones this skill uses with `ToolSearch` (e.g. `select:mcp__atlassian__getJiraIssue,mcp__atlassian__getAccessibleAtlassianResources`) before calling them. If no `mcp__atlassian__*` tools exist at all, stop and tell the user to connect the Atlassian MCP.

## 1. Ask the two opening questions

If the args contain a Jira key, run **1A** for it first, before asking anything else. Then continue here.

First read the skill's args. A Jira key answers the ticket question, and a planner name answers the planner question. Ask only the questions the args didn't answer. If both are answered, skip the AskUserQuestion call.

Ask the remaining questions in a single AskUserQuestion call:

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

- A ticket key was given: if **1A** hasn't run for it yet, run it now. Then check for a saved summary at `.planning/tickets/<KEY>.md` (see **4**). If it exists, ask with AskUserQuestion whether to resume from it or start fresh.
  - **Resume:** read the file, show the summary, and go to **6**. Skip fetching and clarifying: the file is the confirmed summary.
  - **Start fresh** (or no file): go to **2A**.
- `No ticket` was picked: go to **2B**.

## 1A. Claim the ticket

Runs as soon as a ticket key is known, before anything else happens with it. Ask each question on its own, and wait for the answer.

1. Call `mcp__atlassian__getAccessibleAtlassianResources` once to get the `cloudId`. Call `mcp__atlassian__getJiraIssue` for the key to read its assignee and status. If the ticket can't be found, stop and tell the user.
2. **Assign.** Get the user's `accountId` from `atlassianUserInfo`. If the ticket is already assigned to the user, say so and skip this question. Otherwise ask with AskUserQuestion whether to assign it to them, naming the current assignee if there is one. If yes, set it with `editJiraIssue` (`fields: { "assignee": { "accountId": "<id>" } }`).
3. **Move.** List the ticket's transitions (find the operation with `discover`). Ask with AskUserQuestion which column to move the ticket to, naming its current status. Offer up to 3 transitions, the most likely first (usually In Progress), plus `Leave it in <current status>`. The user can pick Other to type a column. Match a typed name against the transitions, and if it matches none or more than one, show the options and ask again. Never guess. If the user picks a column, move the ticket with `transitionJiraIssue`.

## 2A. Ticket path: fetch the ticket

1. Call `mcp__atlassian__getAccessibleAtlassianResources` once to get the `cloudId`.
2. Call `mcp__atlassian__getJiraIssue` for the key with `view: "full"`. This returns the description, acceptance criteria, comments, custom fields and links.
3. If the description depends on the parent, the epic, or linked or blocking issues, fetch those too. Fetch any linked Confluence pages with `getConfluenceContent`.
4. If the ticket can't be found, stop and tell the user.

Then go to **3**.

## 2B. No-ticket path: describe the problem

Ask the user, in plain chat, to describe the problem or change in their own words. Wait for their answer, then go to **3**.

## 3. Clarify until there is 100% clarity

Read the ticket or the user's description together with the parts of the codebase it touches. Where helpful, use `.planning/codebase/*.md` for orientation. Then list every gap, such as:

- ambiguous or missing acceptance criteria
- which apps, packages or modules are affected
- edge cases, error states, and empty or loading states
- design or UX details that aren't specified
- API contracts and backend dependencies
- conflicts with the current code
- scope boundaries: what is explicitly out of scope

Ask these with AskUserQuestion, up to 4 per call. Keep asking rounds until **no** open question remains. Where the code or ticket points to a sensible answer, put it first and mark it `(Recommended)`, with a one-line reason.

From the second round on, if open questions remain, make the first question of the call: "<N> questions left. Keep going, or take the recommended answer for all of them?" If the user takes the recommended answers, list each question with the answer you chose, record them as "Decided by Claude, delegated by user", and go to **4**. Any question without a recommended answer still has to be asked.

If the work is a bug (the ticket type is Bug, or the user describes broken behaviour), ask with AskUserQuestion whether they want to link a Sentry issue. If they do:

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

Ask the user to confirm the summary or correct it, then stop and wait. After each correction, show the full updated summary and ask again. Loop until they explicitly confirm. Don't move to the next step, or mention it, until then.

Then save the confirmed summary to `.planning/tickets/<KEY>.md`. With no ticket yet, use `.planning/tickets/<kebab-case-title>.md`, and rename it to the key if a ticket is created in **5**. Before writing it, run `git check-ignore -q .planning/tickets/`. If the folder isn't ignored, append `.planning/tickets/` to the repo root `.gitignore` and tell the user you did. Update this file whenever a later step changes something it records (new key, branch name).

- Ticket path: go to **6**.
- No-ticket path: go to **5**.

## 5. No-ticket path: offer to create a ticket

Ask with AskUserQuestion whether you should create a Jira ticket from the confirmed summary.

- **No:** if the planner is `none`, go to **5A**. Otherwise go to **7**, and use "none" as the ticket key. There is no key to name a branch after, so skip branch creation.
- **Yes:** first pick the Jira project. If a project was already chosen earlier in this run (tickets-only loop), reuse it. Otherwise list the projects the user can see (find the operation with `discover`) and ask with AskUserQuestion which one to create the ticket in, offering the best matches; the user can pick Other to type a key. Never guess the project. Use its key as `<PROJECT>` below.

  Then look for duplicates. Run `searchJiraIssuesUsingJql` with `project = <PROJECT> AND statusCategory != Done AND text ~ "<keywords>"`, using 2 or 3 distinctive keywords from the summary. If anything plausibly matches, show the matches (key, title, status, link) and ask with AskUserQuestion whether to create a new ticket anyway or use one of them. If the user picks an existing ticket, use its key from here on, skip creation and go to **6** (or **5A** if the planner is `none`).

  Before asking for the details, look up the current sprint with `executeRead`:
  1. `listJiraBoards` with `{ "projectKeyOrId": "<PROJECT>", "type": "all" }`. Kanban boards have no sprints, so keep only the board whose `type` is `scrum`. If there isn't exactly one, show the boards and ask which to use.
  2. `listJiraBoardSprints` with that `boardId` and `"state": "active"`, then again with `"state": "future"`. If there isn't exactly one active sprint, show them (or say there are none) and ask which to use.

  Keep every sprint's name and numeric ID. Always use the ID from here on: sprint names repeat across boards and years. Some teams use a future sprint as their backlog (e.g. one named "Backlog"). That is still a sprint, with an ID, and is never the same as the board's backlog.

  Also fetch, with their IDs:
  - the open epics: `searchJiraIssuesUsingJql` with `project = <PROJECT> AND issuetype = Epic AND statusCategory != Done`
  - the project's components and issue types (find the operations with `discover`)

  Ask for the ticket details in one AskUserQuestion call, as the only thing in that message. Never show them as a table or a list of proposed values to confirm. Build the options from what you fetched. The user can always pick Other to type a value instead.
  1. **Epic**: the 3 open epics that best match the confirmed summary.
  2. **Sprint**: `<current sprint name> (current)`, up to 2 future sprints by their exact names, and `No sprint (board backlog)`.
  3. **Component**: the 3 components that best match the affected areas.
  4. **Ticket type**: up to 4 of the project's issue types, the best fit first.

  A picked option already has its ID. Resolve anything typed with Other to its Jira ID. Never guess a match. If a typed name matches nothing, or matches more than one thing, show the user the candidates and ask which one they meant.
  - **Epic:** search with `searchJiraIssuesUsingJql` using `project = <PROJECT> AND issuetype = Epic AND summary ~ "<epic name>"`.
  - **Sprint:** look it up with `listJiraBoardSprints` on the same board, using `"state": "all"`, and take its numeric ID.
  - **Component** and **ticket type:** match against the lists you fetched.

  Ask the user, in plain chat, for links. These are optional; if they have none, leave out the Design (or Design/Evidence) section.
  - For a Bug, ask for Figma designs and links to browser screenshots or screen recordings.
  - For any other type, ask for a Figma link.

  Write the description in Markdown. The sections depend on the ticket type.

  **Bug:** use these sections, in this order:

  - **Background:** one or two sentences describing the bug.
  - **Expected:** numbered steps describing how it is supposed to work, 4 at most.
  - **Actual:** numbered steps to recreate the bug, ending with what goes wrong, 4 at most. Don't repeat setup steps already in Expected; start from "Same as Expected 1–2, then".
  - **Acceptance Criteria:** the expected end result once the ticket is done, as testable bullets a non-developer understands.
  - **Tech Notes:** optional. Follow the same rules as Tech Notes below.
  - **Design/Evidence:** optional. Figma links, screenshots, screen recordings and the linked Sentry issue.

  **Any other type:** use these sections, in this order:

  - **Background:** two sentences at most: what is changing and why.
  - **Acceptance Criteria:** a bullet list. Each item must be an action someone can test, written so a non-developer understands it. No code names, file paths or store fields.
  - **Tech Notes:** only what a developer needs to start: where the change lives, the approach, real caveats, links to documentation, and shared prerequisites (e.g. "may already be done by PROJ-123"). Leave out anything they'd find in the first few minutes of reading the code. Where possible, don't hardcode technical details such as variable names, types, payload fields or values. Link to the documentation instead (Miro, Confluence, API docs): that is the source of truth, and hardcoded details go out of date. Describe the approach in words, and ask the user for documentation links if you don't have any.
  - **Design:** the Figma link. Leave this section out if the user didn't give one.

  Keep tickets short and precise. Only Tech Notes may be technical; every other section must make sense to a non-developer.

  Length limits, for every type:
  - Every bullet and step is one line: one sentence, no sub-clauses chained with semicolons.
  - Acceptance Criteria: 5 bullets at most. Tech Notes: 5 bullets at most. No tables, sub-bullets or bold lead-ins ("**Scope:** …").
  - If the work genuinely needs more, say so to the user and suggest splitting the ticket instead of writing a longer one.

  Leave out:
  - How you worked things out: no "confirmed in code", "likely", "I checked", or explanations of why an alternative won't work.
  - Anything already said in another section, or in the title.
  - Out-of-scope notes, unless the user asked for one or it prevents an obvious mistake.
  - The clarification Q&A. Decisions belong in the summary file and the optional comment in **7**, not the description.

  Before showing the payload, trim the draft: cut every word, bullet and sentence that a developer or tester wouldn't miss.

  Before creating anything, show the user the exact payload: the summary, the description, the type, the epic, the sprint and the component. Wait for an explicit "yes".

  Then call `createJiraIssue` with the description as Markdown (its default `contentFormat`) and the numeric sprint ID as a string in `assignToSprint` (e.g. `"375"`). Never pass the sprint name or `"active"`. Any sprint the user picked, active or future, goes in `assignToSprint` by ID, even when its name contains "Backlog". Leave out `assignToSprint` only when the user picked `No sprint (board backlog)`.

  The ticket is created even when the sprint assignment fails, so never create it again. If a sprint was picked, check it:
  1. If the response doesn't show `sprintAssignment.assigned: true`, call `transitionJiraIssue` with only `issueIdOrKey` and `sprintId` (a number, not a string).
  2. Then confirm it with `searchJiraIssuesUsingJql` using `key = <KEY> AND sprint = <sprint ID>`. If that returns nothing, retry step 1 once and check again. If it still fails, tell the user the ticket is not in the sprint and give them the error.

  Report the new key and its link back to the user, and use that key from here on. Go to **6**, or **5A** if the planner is `none`.

## 5A. Tickets-only loop (planner `none`)

Don't claim the ticket, create a branch or plan. Ask with AskUserQuestion whether to create another ticket or stop.

- **Another ticket:** go to **2B** and run the no-ticket path again for the new problem. Keep the planner as `none`.
- **Stop:** end the skill. List every ticket created in this run with its link.

## 6. Claim the ticket and create the branch

If there is a ticket key and **1A** hasn't run for it (a ticket created or picked in **5**), run **1A** now.

Then ask with AskUserQuestion whether the user wants a new branch. If they say no, skip to the end of this step.

If they say yes:

1. Run `git ls-remote --heads origin master develop` and offer only the bases that exist. If just one exists, tell the user you're using it instead of asking.
2. Run `git status --porcelain`. If there are uncommitted changes, stop and ask the user how to handle them. Never stash, commit or discard them yourself.
3. Work out `<branch>`:
   - If `AGENTS.md` exists at the repo root and defines a branch naming pattern, use that pattern instead of the defaults below.
   - Otherwise, Story or Task: `feature/{ticket-number}-{ticket-title}`; Bug: `bug/{ticket-number}-{ticket-title}`; any other type: ask the user which prefix to use.

   `{ticket-number}` is the full key (e.g. `PROJ-123`). `{ticket-title}` is the ticket summary with spaces and special characters replaced by `-`, runs of `-` collapsed, and no leading or trailing `-` (e.g. `feature/PROJ-123-Add-Dark-Mode-Toggle`). If the whole branch name is longer than 60 characters, cut `{ticket-title}` at the last `-` that keeps it within 60.

4. Check whether `<branch>` already exists locally (`git branch --list <branch>`) or on the remote (`git ls-remote --heads origin <branch>`). If it does, ask whether to check it out or use a different name.
5. `git checkout <base> && git pull`, then `git checkout -b <branch>`. When checking out an existing remote branch, use `git checkout <branch>` instead.

Then:

- Ticket path: go to **7**.
- No-ticket path: ask with AskUserQuestion whether to continue to planning or stop here.
  - **Continue:** go to **7**.
  - **Stop:** end the skill. Tell the user they can plan later by running `/start-ticket <KEY>`, which resumes from the saved summary.

## 7. Hand off to the planner

If there is a ticket key, offer with AskUserQuestion to post the **Decisions from clarification** to the ticket as a comment, so the team can see them. Show the exact comment text first. If the user says yes, post it with `addOrEditJiraIssueComment`. Skip this if there were no decisions.

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

Use the Skill tool to invoke the planner chosen in step 1. Pass the path of the saved summary file, the ticket key if there is one, and a one-line goal as args:

| Planner       | Handoff                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `paul`        | Read `.paul/STATE.md` and `.paul/ROADMAP.md`, tell the user which milestone and phase are current, and ask with AskUserQuestion where the work goes: a new phase in the current milestone (`paul:add-phase`), or a new milestone (`paul:milestone`). Name it `<TICKET-KEY> <title>`, or just `<title>` when there is no ticket. Then invoke `paul:plan` for the new phase. Never plan into an existing phase. |
| `gsd`         | Invoke `gsd-phase` to add a phase named `<TICKET-KEY> <title>`, or just `<title>` when there is no ticket, with the summary as its goal. Then invoke `gsd-plan-phase` with the new phase number.                                                                                                                                                                                                              |
| `superpowers` | Invoke `superpowers:brainstorming`, which chains into `superpowers:writing-plans`.                                                                                                                                                                                                                                                                                                                            |
| Other         | Invoke the skill the user chose.                                                                                                                                                                                                                                                                                                                                                                              |

From here on, the planner's workflow is in charge. Do not re-ask questions the summary already settles.
