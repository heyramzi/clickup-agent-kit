# Demo workspace cleanup

Read `demo-workspace.md` for the workspace and list ids before running any of this.

## Workflow

1. Fetch all tasks per list with `include_closed=true`.
2. Identify stale by pattern: single-char names (`d`, `e`, `ss`); generic tests (`TEST`,
   `hello`, `Hello`); celebrity names (Angelina Jolie, Brad Pitt, Britney Spears); obvious
   placeholders (`New Employee`, `John Smith` as the only candidate); duplicate task names
   in the same list.
3. Delete stale tasks in bulk.
4. Enrich remaining tasks with realistic descriptions.
5. Add new tasks to lists with fewer than 5 tasks (especially Onboarding, Templates).
6. Scan all views across spaces/folders/lists.
7. Delete junk views: unnamed, duplicate types on the same parent, empty/broken.
8. Update column configs on key views to show the most relevant fields.

## Data quality standard

**Task names**: Position — `Job Title` (no prefix, title case). Time-Off — `Type — Employee
Name` or `Type — Context`. Expenses — `Item — Context` with amounts where relevant.
Employees/Talents — `First Last` (real-sounding, diverse names).

**Descriptions**: every task gets one, 1-3 sentences: what it is / what the person does, key
context (tech stack, scope, timeline), any notable detail (budget, status reason, coverage
plan).

**Dates**: past dates on done/validated tasks are fine as-is. Active tasks get near-future
dates, 2-8 weeks out. Never leave `null` on a task that logically needs a due date.

**Statuses**: match the list's status schema — don't leave a task in "to-do" if context
implies "in progress" or "validated".
