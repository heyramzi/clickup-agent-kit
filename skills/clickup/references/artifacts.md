# Brain² artifacts

An artifact is a page Brain² builds from a prompt: a dashboard, calculator or small app. There's
no public route, so `cu artifacts` drives the web app's own calls on a captured session. The `pk_`
token never works here, and some workspaces deny it outright (`can_use_public_api_dev_key`).

```
cu net capture https://app.clickup.com/<workspaceId>/ai
cu artifacts create --file prompt.md --save ./artifacts/
cu artifacts revise <thread> --file change.md --save ./artifacts/
cu artifacts get <id> --save ./artifacts/
cu artifacts
```

- **The session picks the workspace, not `teamId`.** Capture the workspace the artifact belongs
  in. Capturing another one moves every frontdoor command there too, so re-capture afterwards.
- **A build takes 2 to 3 minutes.** `create` and `revise` poll every 10 seconds and print the
  artifact id, its thread and Brain's reply. With `--wait 0` they return at once, and the run picks up with
  `cu artifacts`.
- **Keep the thread id.** A follow-up goes in the same thread, pointed at the artifact, and Brain
  edits that artifact rather than starting a new one. `--artifact <id>` overrides the target.
- **`--save` is the only export.** It writes the page as one self-contained HTML file.
- **Brain reads the workspace it lives in.** Name the space, folder and lists in the prompt, and
  it builds from the real tasks. A demo goes in a demo workspace with invented data, never a client's.
- **Say the style in every prompt.** Brain picks a new look each time. A set that should match
  needs the same written style block (fonts, hex colours, radius, spacing) in each prompt, and a
  restyle is a `revise` carrying that block.
- **The inline editor has no CLI route yet.** The artifact's Edit toolbar (Edit Inline, Select
  item, "Queue edit for Brain") runs inside a sandboxed iframe, so a script can't drive it. Use
  `revise` and name the element in words.

Routes and their bodies: `api/artifacts-api.ts` in the library.
