# Create a Jira ticket

Follow this from step **5** of the skill. The flow rules in `SKILL.md` apply here too.

## Pick the project

If a project was already chosen earlier in this run (tickets-only loop), reuse it. Otherwise call `executeRead` with `listJiraProjects`, `inputs: { "action": "create" }` and `responseFields: ["values.key", "values.name"]`. Ask with AskUserQuestion which project to create the ticket in, offering the best matches; the user can pick Other to type a key. Never guess the project. Use its key as `<PROJECT>` below.

## Look everything up at once

Make all of these calls in one message. In the tickets-only loop, run only the duplicate search and reuse the rest.

- **Duplicates:** `searchJiraIssuesUsingJql` with `project = <PROJECT> AND statusCategory != Done AND text ~ "<keywords>"`, using 2 or 3 distinctive keywords from the summary, plus `fields: ["summary", "status"]` and `maxResults: 5`.
- **Epics:** `searchJiraIssuesUsingJql` with `project = <PROJECT> AND issuetype = Epic AND statusCategory != Done` and `fields: ["summary"]`.
- **User:** `atlassianUserInfo`, if the planner isn't `none` and you don't have the `accountId` yet.
- **Boards:** `executeRead` with `listJiraBoards`, `inputs: { "projectKeyOrId": "<PROJECT>", "type": "all" }` and `responseFields: ["values.id", "values.name", "values.type"]`.
- **Components:** `executeRead` with `listJiraProjectComponents`, `inputs: { "projectIdOrKey": "<PROJECT>" }` and `responseFields: ["values.id", "values.name"]`.
- **Issue types:** `executeRead` with `listJiraProjectIssueTypesMetadata`, `inputs: { "projectIdOrKey": "<PROJECT>" }` and `responseFields: ["issueTypes.id", "issueTypes.name", "issueTypes.subtask"]`.

Kanban boards have no sprints, so keep only the board whose `type` is `scrum`. If there isn't exactly one, show the boards and ask which to use. Then, in one message, call `executeRead` with `listJiraBoardSprints` twice, for `"state": "active"` and `"state": "future"`, each with that `boardId` and `responseFields: ["values.id", "values.name", "values.state"]`. If there isn't exactly one active sprint, show them (or say there are none) and ask which to use.

Keep every sprint's name and numeric ID. Always use the ID from here on: sprint names repeat across boards and years. Some teams use a future sprint as their backlog (e.g. one named "Backlog"). That is still a sprint, with an ID, and is never the same as the board's backlog.

If the duplicate search found anything plausible, show the matches (key, title, status, link) and ask with AskUserQuestion whether to create a new ticket anyway or use one of them. If the user picks an existing ticket, use its key from here on, skip creation and go to **6** (or **5A** if the planner is `none`).

## Ask for the ticket details

Ask in one AskUserQuestion call, as the only thing in that message. Never show them as a table or a list of proposed values to confirm. Build the options from what you fetched. The user can always pick Other to type a value instead.

1. **Epic**: the 3 open epics that best match the confirmed summary.
2. **Sprint**: `<current sprint name> (current)`, up to 2 future sprints by their exact names, and `No sprint (board backlog)`.
3. **Component**: the 3 components that best match the affected areas.
4. **Ticket type**: up to 4 of the project's issue types, the best fit first.

A picked option already has its ID. Resolve anything typed with Other to its Jira ID. Never guess a match. If a typed name matches nothing, or matches more than one thing, show the user the candidates and ask which one they meant.

- **Epic:** search with `searchJiraIssuesUsingJql` using `project = <PROJECT> AND issuetype = Epic AND summary ~ "<epic name>"`.
- **Sprint:** look it up with `listJiraBoardSprints` on the same board, using `"state": "all"`, and take its numeric ID.
- **Component** and **ticket type:** match against the lists you fetched.

## Write the description

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
- The clarification Q&A. Decisions belong in the summary file and the optional comment in **6**, not the description.

Before showing the payload, trim the draft: cut every word, bullet and sentence that a developer or tester wouldn't miss.

## Confirm the payload

Before creating anything, show the user the exact payload: the summary, the description, the type, the epic, the sprint and the component. If the planner isn't `none`, also show `Assignee: you (say no to leave it unassigned)`.

In the same message, ask for "yes", plus any links. Links are optional; with none, leave out the Design (or Design/Evidence) section.

- For a Bug, ask for Figma designs and links to browser screenshots or screen recordings.
- For any other type, ask for a Figma link.

If the reply has links and a "yes", add the links and create the ticket. If it has links or corrections but no "yes", show the updated payload and ask again.

## Create the ticket

Call `createJiraIssue` with the description as Markdown (its default `contentFormat`). Pass:

- the epic key in `parent`
- the component in `additional_fields: { "components": [{ "name": "<component>" }] }`
- the user's `accountId` in `assignee`, unless the planner is `none` or the user said no
- the numeric sprint ID as a string in `assignToSprint` (e.g. `"375"`)

Never pass the sprint name or `"active"`. Any sprint the user picked, active or future, goes in `assignToSprint` by ID, even when its name contains "Backlog". Leave out `assignToSprint` only when the user picked `No sprint (board backlog)`.

The ticket is created even when the sprint assignment fails, so never create it again. If a sprint was picked and the response doesn't show `sprintAssignment.assigned: true`:

1. Call `transitionJiraIssue` with only `issueIdOrKey` and `sprintId` (a number, not a string).
2. Confirm it with `searchJiraIssuesUsingJql` using `key = <KEY> AND sprint = <sprint ID>` and `fields: ["summary"]`. If that returns nothing, retry step 1 once and check again. If it still fails, tell the user the ticket is not in the sprint and give them the error.

Report the new key and its link back to the user, and use that key from here on. Go to **6**, or **5A** if the planner is `none`.
