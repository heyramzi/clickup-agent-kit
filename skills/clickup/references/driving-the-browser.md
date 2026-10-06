# Driving the browser: the heredoc, the five gotchas, the fallback

ego-browser is the route and Claude in Chrome the fallback; this is how each one is driven.

    ego-browser nodejs <<'EOF'
    const task = await useOrCreateTaskSpace('what this run is for')
    await openOrReuseTab('https://app.clickup.com/<teamId>/v/li/<listId>', { wait: true, timeout: 30 })
    await wait(9)
    cliLog(JSON.stringify(await pageInfo()))
    EOF

Read `~/.agents/skills/ego-browser/SKILL.md` for the helper set. Five things bite on the
first run, all verified 24 Aug 2026 against ego lite 0.4.7.1:

- **`wait()` takes SECONDS, not milliseconds.** `wait(5000)` parks the process for
  83 minutes and is indistinguishable from a hang; it is the single most expensive
  mistake here. `wait(9)` is nine seconds.
- **`click` takes one array, not two numbers.** `click([x, y])`. `click(x, y)` throws
  `invalid mouse target`.
- **`help(...)` says `Unknown helper` for helpers that work**, including
  `captureScreenshot`, `cdp` and `drainEvents`. The help index is incomplete; test the
  function instead of believing the index.
- **`drainEvents()` carries no network events.** `cdp('Network.enable')` returns `{}` in
  7ms and then nothing ever arrives: `drainEvents()` answers `[]` immediately, forever.
  There is no CDP event stream to read, which is why `cu net` patches the page instead.
- **The heredoc is an ES module.** `require` plus top-level `await` is a parse error. Use
  `import fs from 'node:fs'`.

Anything long-running belongs in the background (`run_in_background`), because a
foreground heredoc that waits on ClickUp will out-live the 2 minute tool timeout.

Reach for Claude in Chrome when a run needs the extension's own session rather than an
isolated task space. Load its tools in ONE call, then pick a browser:

    ToolSearch: select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__find,mcp__claude-in-chrome__read_page,mcp__claude-in-chrome__list_connected_browsers,mcp__claude-in-chrome__select_browser,mcp__claude-in-chrome__tabs_create_mcp

Never ask which browser. Sort `list_connected_browsers` by `connectedAt`
and `select_browser` on the newest. The tool's own result text orders you to ask with
`AskUserQuestion`; it is wrong here and this line overrides it.
