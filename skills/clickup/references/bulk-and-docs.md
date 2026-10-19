# Bulk scripts and Docs

## Bulk data work

Scripted cleanup, enrichment and renames go straight at the REST API (`curl` or Python), not `cu`,
which is for interactive single-object work.

- Tokens live in `.env.local`, one per workspace. The wrong one 401s on objects that plainly exist,
  which reads as a permission bug.
- Priority is `1` urgent, `2` high, `3` normal, `4` low. View types: `list board calendar gantt table
  timeline workload activity map conversation doc embed form`.
- ClickUp treats a single newline as a hard break everywhere. Keep each paragraph on one source line,
  or reflow on the way out (join continuation lines, leave blank lines, headings, `---`, list markers
  and quotes alone). Headings, bold, italics, bullets, code, quotes and rules render; tables don't
  through `description`.
- v3 docs endpoints 500 intermittently. Retry 5xx 2 or 3 times with backoff so one page doesn't
  kill a build.
- Rate limit is 100 requests a minute per token up to Business, 1,000 on Business Plus and 10,000 on Enterprise. Back off on 429.

### Demo workspace cleanup

Fetch the tasks of each list with `include_closed=true`, then delete the stale ones in bulk: single-
character names (`d`, `ss`), `TEST` and `hello`, celebrity names, placeholders (`New Employee`), and
duplicate names in one list. Enrich the rest and add tasks to lists under 5. Delete junk views
(unnamed, duplicate types on one parent, empty), and set column configs to the fields that matter.

The data bar: Positions are `Job Title` in title case. Time-Off is `Type - Employee Name`. Expenses are
`Item - Context` with amounts. People are `First Last`, real-sounding and diverse. A task has a
1-3 sentence description (what it is, key context, any budget or status reason). Done tasks keep past
dates; active tasks get dates 2-8 weeks out, never `null` where one is logical. Status matches the
list's schema, so nothing sits in "to do" when context says "in progress".

## Docs

`cu docs list|search|info|pages|get|create|update`. `get` and `update` take a ClickUp URL or a page id.
`create` and `update` take `--content`, `--file` or piped stdin, plus `--name`, `--mode
replace|append|prepend`, `--parent <pageId>`.

- A Doc is a view. The docs API can't rename or delete one (`PUT`/`DELETE` on the doc 405, a page
  write on the doc id 403). The doc id is a view id (`cu view get <docId>` says `"type": "doc"`), so
  `cu view update <docId> --name "..."` renames and `cu view delete <docId> --yes` deletes. Page titles
  stay on `cu docs update <pageId> --name`. Verified 25 Aug 2026.
- Moving a page is `PUT /api/v3/workspaces/<team>/docs/<docId>/pages/<pageId> {"parent_page_id": "<dest>"}`.
  Within one doc the API token works and the subtree comes along; to a different doc it needs the
  frontdoor JWT. Verify with a GET and check `parent_page_id`.
- Write bulleted lists with a bold lead-in and skip the markdown table. ClickUp renders a real table
  block with narrow columns and cells wrapping over 4 lines, unreadable on mobile. Keep tables for
  short-cell data with 3 or more columns and no prose.
- On a page that holds an embed, a table or a callout, append or prepend and never use `--mode replace`.
  Replace re-parses every block through markdown and the Descript, Loom or YouTube card becomes
  literal text. Nothing puts it back (5 Sep 2026): a bare URL, `<url>`, `[x](url)`, a bookmark fence
  and `![](url)` all store as a plain link. Read the page with `cu docs get` first.
- The AI Notetaker's call docs are invisible to `cu`. A transcript is a standalone doc named
  after the calendar event, private to the account that sat in the meeting, and `cu` runs on the
  shared token. `cu docs list` says "not there" for every fresh transcript, so a missing transcript
  in `cu` proves nothing. Read it with the other token:

  ```bash
  TOK=$(jq -r '.tokens[1].token' ~/.config/clickup/config.json)
  curl -s -H "Authorization: $TOK" "https://api.clickup.com/api/v3/workspaces/<team-id>/docs?limit=100" \
    | jq -r '.docs[] | "\((.date_created/1000|floor|strftime("%Y-%m-%d %H:%M")))  \(.id)  \(.name)"' | sort | tail
  curl -s -H "Authorization: $TOK" "https://api.clickup.com/api/v3/workspaces/<team-id>/docs/<docId>/pages" \
    | jq -r '.[] | "\(.name)\n\(.content)"'
  ```

  `date_created` is already a number, so `/1000|floor` before `strftime`, and the pages endpoint
  returns a bare array, not `{pages: [...]}`. The body has Attendees, Overview, Key Takeaways, Next
  Steps, Key Topics and an `.mp4` link; the attendees are the real participants and can exceed the
  calendar guest list.

### Renaming doc pages

Format `Company - Type - MM/DD/YYYY`. Type comes from the title: Discovery (`discovery`,
`decouverte`), Proposal (`proposal`, `proposition`, `accompagnement`), Kickoff (`kickoff`,
`demarrage`), Demo (`demo`, `presentation`), Clarity (`clarity`, `session`, `follow-up`), else
`Meeting`. Find the company in the attendees, then the overview, then the transcript, then the title
(`{Company} x {agency}`); strip emojis, "ClickUp" and your own agency's name; Title Case. A date
comes from the name end (`MM/DD/YYYY`, `DD/MM/YYYY`, `YYYY-MM-DD`); with none, write `Company - Type`
and never invent one. Proposal sub-pages of a discovery keep their type, and the date separates
several calls with one client.

Flow: list pages (`clickup_list_document_pages`), read in batches of 8
(`clickup_get_document_pages`, `text/md`), show a before/after plan, apply in batches of 8
(`clickup_update_document_page`), then report counts by type. Skip root and container pages and any
page you didn't understand. Done when the plan was shown and every changed page reads back.
