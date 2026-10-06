# Writing ClickUp comments

`cu comments add --text` posts **plain text**. ClickUp renders it verbatim: no headings,
no bold, no lists, and every newline you typed becomes a hard line break in the reader's
width. A brief written as markdown and posted with `--text` arrives as a wall of
mid-sentence breaks with `##` sitting in it as literal characters.

Use `--text` only for a single short sentence. Anything with structure goes through
`--from`.

## The format

ClickUp comments are **Quill Delta**: an array of ops on `comment`. Two rules explain
the whole shape.

- **Inline formatting rides on the text op.** `{ "text": "Title", "attributes": { "bold": true } }`
- **Line formatting rides on the trailing newline**, and applies to the line before it.
  `{ "text": "\n", "attributes": { "header": 2 } }`

So a heading is two ops: the words, then a newline carrying `header`. A bullet is the
words, then a newline carrying `list`.

Attributes worth knowing, from `types/clickup-comment-types.ts` in the CLI source:

- Inline: `bold`, `italic`, `underline`, `strike`, `code`, `link` (a URL string)
- Line: `header` (1, 2 or 3), `list` (`{ "list": "bullet" | "ordered" | "checked" | "unchecked" }`),
  `blockquote`, `code-block`, `divider`, `indent`
- `{ "type": "tag", "user": { "id": 123 }, "text": "@Name" }` for a mention

```bash
cu comments add <taskId> --from comment.json
```

where `comment.json` is `{ "comment": [ ...ops ], "notify_all": false }`.

## Never hard-wrap

This is the failure that actually shipped. A brief wrapped at 90 characters for the
terminal arrived in ClickUp with a break after every 90 characters, mid-sentence, on
every line, because ClickUp already wraps to the reader's column width.

**Write each paragraph as one long line and let ClickUp wrap it.** The only newlines in
a comment are the ones ending a heading, a paragraph or a list item.

## Build the ops, do not hand-write them

79 ops is normal for a one-page brief and hand-maintaining that JSON is miserable. A
dozen lines of helpers make it readable:

```js
const ops = [];
const t = (text, attributes) => ops.push(attributes ? { text, attributes } : { text });
const line = (attributes) => ops.push(attributes ? { text: "\n", attributes } : { text: "\n" });
const h = (text, level = 2) => { t(text); line({ header: level }); };
const p = (...segs) => { for (const s of segs) typeof s === "string" ? t(s) : t(s[0], s[1]); line(); };
const li = (...segs) => { for (const s of segs) typeof s === "string" ? t(s) : t(s[0], s[1]); line({ list: { list: "bullet" } }); };
const b = (s) => [s, { bold: true }];
const code = (s) => [s, { code: true }];
```

Then `h("Hard constraints", 3)`, `li(b("One person. "), "Never a second face.")`.

## Fixing one you already posted

There is no `cu comments delete`. The v2 route exists, so go straight at it:

```bash
TOKEN=$(python3 -c "import json;d=json.load(open('$HOME/.config/clickup/config.json'));print(d['tokens'][0]['token'])")
curl -s -X DELETE -H "Authorization: $TOKEN" "https://api.clickup.com/api/v2/comment/<commentId>"
```

It answers `{}` with HTTP 200. Get the id from `cu comments list <taskId>`.

## Reading it back

`cu comments list --json` prints a banner line before the JSON, so piping it into a
parser fails on "Extra data". Read the comment back off the API instead, and check that
the ops carry the attributes you meant:

```bash
curl -s -H "Authorization: $TOKEN" "https://api.clickup.com/api/v2/task/<taskId>/comment" \
  | jq '.comments[0].comment[] | {text, attributes}'
```

## Descriptions are the other half

A task description posted without `--markdown` keeps `##` and `**` as literal
characters. The flag is a boolean and the body always goes in `--description`:

```bash
cu task update <taskId> --markdown --description "$(cat body.md)"
```

Doc pages have their own trap, tables. See [`doc-authoring.md`](doc-authoring.md).
