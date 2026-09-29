# Writing ClickUp Doc pages

## Never write markdown tables into a Doc page

ClickUp renders a markdown table as a real table block: fixed narrow columns,
every cell wrapping over three or four lines, a drag handle on each row. A
two-column term-and-definition table becomes unreadable, and the reader cannot
scan it on mobile at all.

Write the same content as a bulleted list with a bold lead-in instead:

```markdown
- **Workspace.** Your whole company. Keep exactly one, nothing moves between them.
- **Space.** A department or business entity. Marketing, Sales, Delivery.
```

Reserve tables for really tabular data with short cells (three or more
columns of values, no prose). When in doubt, use the list.

## `cu docs update --mode replace` destroys a page's embed cards

The card ClickUp draws for a pasted Descript, Loom or YouTube link is an
editor-only block. It reads back as markdown, but that string is not valid
markdown input, so sending it back escapes the brackets and the card becomes
literal text. Measured 5 Sep 2026 adding a SOP to a client tutorials doc:
nothing put it back. A bare URL, `<url>`, `[label](url)`, a ```bookmark fence
and `![](url)` all store as a plain link, `content_format` accepts only
`text/md` and `text/plain`, and `cu` has no frontdoor doc-content route.

**On any page that already holds an embed, a table or a callout, append or
prepend, never replace.** Replace re-imports every existing block through the
markdown parser, which is where they are lost. Read the page with `cu docs get`
first: an embed block in it makes the write an append.
