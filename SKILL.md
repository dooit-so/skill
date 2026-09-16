---
name: dooit
description: >-
  Run the user's Dooit task board — plan their day, capture and schedule tasks,
  work a weekly review, and track goals. Use when the user asks what's on their
  plate, wants something added to their to-do list, asks you to reschedule or
  clear their day, mentions their board, tasks, projects or weekly/monthly/
  yearly goals, or wants help planning a week. Works through Dooit's remote MCP
  server — reads the real board and writes to it, in the user's own timezone.
---

# Dooit — run the user's task board

Dooit is a minimal, keyboard-first personal task manager. Tasks sit on a kanban
board of user-defined status columns, optionally belong to a project, and can
carry a due date or a recurring rule. Above them sit goals scoped to a week, a
month or a year.

It is one person's board — no teams, no assignees, no sprints. You are that
person's hands: everything they can do by clicking, you can do by calling, on
the same data, with their permissions.

## Setup (once)

The user needs a Dooit account (app.dooit.so). Connect the remote MCP server:

- **Claude Code:** `claude mcp add --transport http dooit https://app.dooit.so/mcp`
- **Claude Desktop / claude.ai / ChatGPT:** add a custom connector with URL
  `https://app.dooit.so/mcp`
- **Cursor / VS Code / others:** standard remote MCP config, same URL

OAuth sign-in, no API keys. If a tool answers that no Dooit account exists for
this login, the user authenticated with an identity that has never opened the
app — ask them to open app.dooit.so once, then retry.

## The two rules that prevent most mistakes

**1. Call `get_board` first, every conversation.** It returns today's date in
the user's timezone plus the status column ids and project ids that every write
needs. Ids are account-scoped and unguessable. Never invent one, and never
reuse one you remember from an earlier session — statuses get renamed and
projects get archived.

**2. Dates are the user's, not yours.** Everything is ISO `YYYY-MM-DD`
interpreted in the user's IANA timezone. Derive "tomorrow" and "next Friday"
from the `today` field `get_board` returns — never from your own clock, and
never from UTC. Getting this wrong silently files work on the wrong day.

## The jobs and their tools

**"What's on my plate?"** — `get_agenda`. One call returns overdue, due-today,
upcoming (next 7 days) and unscheduled open tasks, plus this week's and this
month's goals. Don't reconstruct this from several `list_tasks` calls; it is
cheaper and it is the shape the user thinks in.

**"Show me X"** — `list_tasks` filters by project (`projectId`, or `noProject`
for loose tasks), status column (`statusId`), completion (`done`), archive
state (`includeArchived`) and date range (`dueOnOrBefore` / `dueOnOrAfter`),
with a `limit`. It returns terse rows without descriptions, open tasks in
column order (top to bottom within each column). `day: "today"` (or an ISO
date) returns one day's list instead, in the order the Today page or the
Calendar shows it. Reach for `get_task` when the user asks about one task's
detail. `list_goals` takes the same period shortcuts as `create_goal`.

**"Add this"** — `create_task` with `name`, and optionally `description`,
`projectId`, `statusId`, `schedule`, and a place (below). Defaults are no
project, no date, and the user's default status, at the bottom of that column.
Worth knowing:

- **Scheduling routes the task for you.** Setting a date moves it between the
  Today and Scheduled columns automatically. Don't also set a `statusId` to
  mimic that — you'll fight the router.
- **Recurring rules are structured, and carry their own prose.** Pass
  `schedule: { type: "recurring", recurringRule: { granularity, interval,
  anchor, … }, recurringText: "Every Monday" }`. `recurringText` is what the
  user reads on the card, so write it the way they said it.
- **Descriptions are Markdown (GFM).** Headings, lists, task lists
  (`- [ ]`), links, code — they render in the app exactly as you'd expect,
  and `get_task` returns the description as Markdown too.

**"Move / rename / reschedule"** — `update_task`. Only the fields you pass
change. `schedule: null` clears the date or recurrence; `projectId: null` pulls
the task out of its project. To move a task between columns, pass a `statusId`
from `get_board`.

**"Put it first / move it up / order my day"** — a task has a place in two
orders, so first work out which one the user means:

- **`columnPosition`** — its status column: the board's columns, project boards,
  and Today's List and Kanban layouts. Read it with `list_tasks` + `statusId`.
- **`dayPosition`** — its day: Today's flat list (everything due today or
  earlier) and the Calendar. Read it with `list_tasks` + `day`. Open, dated
  tasks only.

"My day" and "today's list" mean `dayPosition`; a column, a project or a
status means `columnPosition`. When it's unclear, set both — unless the task
has no date, in which case it only has a column. Each takes `"top"`,
`"bottom"`, or `{ aboveTaskId }` / `{ belowTaskId }` with an id from the
matching read, on `update_task` or `create_task`. Naming a neighbour also
moves the task into that neighbour's column or day, the same as dragging it
there, so "put X right after Y" is one call; don't add a `statusId` or
`schedule` that disagrees with it. The Done column is ordered by completion
time and takes no position.

To set a whole order — "today: A, then B, then C" — use `move_tasks` with the
ids in that order and a place for the block, e.g. `dayPosition: "top"`. It is
all or nothing and counts as one write. If a day move would give a repeating
task a new date, the call fails asking for `keepRecurring`: ask the user
whether the rule stays (`true` moves just this occurrence, `false` makes it a
one-off), then retry. Never answer that for them.

**"Done"** — `set_task_done` with `done: true` (or `false` to reopen). It's
idempotent, so a double-call is harmless. Completing a recurring task schedules
its next occurrence automatically — don't create the next one yourself.

**"Get rid of it"** — `archive_task` by default. It's a reversible soft-delete:
the task leaves the board and comes back with `restore: true`. Reach for
`delete_task` only when the user has explicitly said permanent; it refuses the
first call and requires `confirm: true` on the second, and that gate exists so
you ask a human, not so you retry.

**"Organize this"** — `create_project` (emoji defaults to 📁) and
`update_project` for name, emoji, description or done state. Projects are
buckets, not milestones — one per ongoing area of work, not one per deliverable.

**"What am I aiming at?"** — `create_goal` and `update_goal`. Prefer the
`period` shortcut (`this-week`, `next-week`, `this-month`, `this-year`) over
computing a `periodKey` yourself; it resolves in the user's timezone, so you
never have to reason about ISO week numbers. Explicit `periodType` +
`periodKey` (`2026-W35`, `2026-08`, `2026`) is there for anything the shortcuts
don't reach.

## Operating doctrine

1. **Reads are free — use them.** Read the board before you plan, and re-read
   after a batch of writes rather than assuming your model of it is still
   right. Reads never need a subscription and aren't rate limited.
2. **Batch thoughtfully, not blindly.** Writes are capped at 30/minute per
   account. That's plenty for a person planning a week with you, and not enough
   to import a backlog — if the user wants a bulk import, tell them so rather
   than half-finishing one.
3. **Don't reorganize unasked.** Renaming projects, re-emojiing, bulk-moving
   columns, or archiving "stale" tasks are the kinds of tidy-up that feel
   helpful and read as vandalism on someone's personal system. Propose; don't
   perform.
4. **Say what you did.** After a batch, name the tasks you touched. The user's
   board is the source of truth and they'll be looking at it.
5. **Surface, don't nag.** If the agenda shows a pile of overdue work, say so
   once, plainly, with the count — then let the user decide.

## Named plays

- **"Plan my day"** — `get_agenda`, then propose a shortlist: overdue first,
  then due-today, then a couple of unscheduled items that fit this week's
  goals. Schedule only what the user agrees to, then — if they want an order —
  set it with one `move_tasks` call (`dayPosition: "top"`).
- **"Clear my day"** — pull today's open tasks, then `update_task` to push what
  the user picks to a real date. Don't blanket-move; ask what actually moves.
- **"Weekly review"** — `list_goals` with `period: "this-week"`, `list_tasks`
  with `done: true` and a `dueOnOrAfter` of Monday for what got finished, and
  the unscheduled pile for what didn't. Close out the goals with `update_goal`,
  then set next week's.
- **"Capture this"** — from a conversation, a doc or a meeting: one
  `create_task` per real commitment, with the context in `description` so the
  task still makes sense in a week. Ask which project before guessing.

## Failure modes

Errors name their own next step; two are typed and mean different things:

- `[SUBSCRIPTION_REQUIRED]` — writes are off because the subscription lapsed.
  **Do not retry.** Tell the user to manage billing in the app. Reads still
  work, so keep answering questions.
- `[RATE_LIMITED]` — wait the stated seconds, then retry.

A "not found" almost always means a stale id: re-run `get_board`, `list_tasks`
or `list_goals` and retry with a current one. An invalid-status error means the
`statusId` isn't one of this user's columns — same fix.
