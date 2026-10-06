# Doc page naming cleanup

Standardizes ClickUp doc pages with inconsistent naming (emoji prefixes, auto-generated
meeting names, mixed formats) to a scannable format.

Announce at start: "I'm using the clickup skill, doc page naming mode."

## Naming convention

Format: `Company Name - Type - MM/DD/YYYY`

| Type      | Keywords to detect                            | Example output                       |
| --------- | ---------------------------------------------- | ------------------------------------- |
| Discovery | `discovery`, `appel decouverte`, `decouverte`   | `Acme Corp - Discovery - 11/15/2025`  |
| Proposal  | `proposal`, `proposition`, `accompagnement`     | `Acme Corp - Proposal - 11/18/2025`   |
| Kickoff   | `kickoff`, `demarrage`, `startup`               | `Acme Corp - Kickoff - 11/20/2025`    |
| Demo      | `demo`, `presentation`                          | `Acme Corp - Demo - 11/22/2025`       |
| Clarity   | `clarity`, `session`, `follow-up`               | `Acme Corp - Clarity - 11/25/2025`    |

Fallback: undetermined type → `Meeting`. No date found → omit it: `Company Name - Type`.

## Client/company name extraction

Fetch content via `clickup_get_document_pages` with `content_format: text/md`. Search in
priority order: attendees section, overview section, transcript, filename in attachments;
`{Company} x {your agency}` pattern in the title; company name before the `-` separator;
contact name as fallback.

Remove from names: emojis, `ClickUp` references, your own agency's name and any `x {agency}`
suffix, special chars (period, hyphen, times), extra whitespace. Normalize to Title Case.

Date extraction: look for a date at the end of the name, in `MM/DD/YYYY`, `DD/MM/YYYY` or
`YYYY-MM-DD`.

## Execution flow

1. Fetch structure via `clickup_list_document_pages` (document_id + workspace_id from URL).
2. Identify pages: with `parent_page_id`, process direct children + sub-pages; without,
   process all.
3. Read content via `clickup_get_document_pages` in batches of 8.
4. Analyze: detect meeting type from title keywords; extract company from content (preferred)
   or title; extract date.
5. Create a rename plan listing before/after per page.
6. Execute via `clickup_update_document_page` in batches of 8; report progress.
7. Summary: count by type, report totals.

Edge cases: proposal sub-pages (children of discovery) keep their type. Multiple calls with
the same client: the date differentiates them. Missing date: use `Type - Client`; never
invent one.

## Red flags

- Don't rename a page you didn't understand — skip and note it.
- Don't change meeting types incorrectly (a proposal is not a discovery).
- Don't process root/container pages — only actual meeting pages.

MCP tools used: `clickup_list_document_pages`, `clickup_get_document_pages`,
`clickup_update_document_page`.
