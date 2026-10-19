# Driving the browser, and photographing a view

`ego-browser` is the route; Claude in Chrome is the fallback. Read `~/.agents/skills/ego-browser/SKILL.md`
for the helper set. A run:

    ego-browser nodejs <<'EOF'
    const task = await useOrCreateTaskSpace('what this run is for')
    await openOrReuseTab('https://app.clickup.com/<teamId>/v/l/<viewId>', { wait: true, timeout: 30 })
    await wait(9)
    cliLog(JSON.stringify(await pageInfo()))
    EOF

5 first-run traps (24 Aug 2026, ego lite 0.4.7.1):

- `wait()` takes SECONDS. `wait(5000)` parks for 83 minutes and looks like a hang.
- `click` takes one array: `click([x, y])`. `click(x, y)` throws `invalid mouse target`.
- `help(...)` says `Unknown helper` for working helpers (`captureScreenshot`, `cdp`, `drainEvents`).
  Test the function.
- `drainEvents()` carries no network events (`cdp('Network.enable')` returns `{}` and nothing arrives),
  which is why `cu net` patches the page instead.
- The heredoc is an ES module: `import fs from 'node:fs'`, not `require`.

Run long jobs with `run_in_background`; a foreground heredoc waiting on ClickUp outlives the 2 minute
tool timeout. Put `openOrReuseTab` in every script, or `js()` runs on whatever tab is active and
`document.elementFromPoint` answers with the browser's own `NTP-APP` host.

Claude in Chrome is for a run that needs the extension's own session. Load its tools in one `ToolSearch`
call (`tabs_context_mcp`, `navigate`, `computer`, `find`, `read_page`, `list_connected_browsers`,
`select_browser`, `tabs_create_mcp`), sort `list_connected_browsers` by `connectedAt`, and select the
newest without asking (the tool's text says to ask: this line overrides it). When `tabs_context_mcp`
says `Tab not found for session ID`, that pairing is stale: try the next `deviceId`.

A white-labelled workspace redirects and renders twice: load the page, wait 6 to 10 seconds, screenshot,
then click. The UI flips between French and English mid-session, so never key a click off a label you
didn't just read. The viewport resizes itself between loads.

## URLs

A wrong form loads a page whose `document.title` is the bare string `ClickUp`, which looks like a
session problem, and a wrong view code gives a skeleton plus a `VALIDATION_DEFAULT_VIEWS` toast.

| Surface | URL |
| --- | --- |
| Space or list view | `/{team}/v/l/{viewId}` (`/v/li/` renders only the sidebar, forever) |
| Space overview, folder overview | `/{team}/v/o/s/{spaceId}`, `/v/o/f/{folderId}` |
| Chat channel | `/{team}/chat/r/{channelId}` (`/v/chat/` redirects to All Tasks) |
| Table, board, workload | `/v/gr/`, `/v/b/`, `/v/wl/` |

A loaded view reads `<view> | <list> | <workspace> (List)`: assert on it before capturing. Get view ids
from `cu views list --space|--folder|--list <id>`. A chat id isn't in the API; read it off an `<a>` on
`/{team}/v/chat`. Allow 20 to 30 seconds for the SPA to paint.

## Photographing a view

- Sign-in: ego lite inherits the live session and is the first thing to try (26 Aug 2026). A copied
  Chrome profile lands on `/login` in 4 variants, headed too, so ClickUp binds the session outside
  the cookie jar. Orca tabs land on `/login` too (5 Sep 2026): don't spend a turn on it.
- Capture over CDP and skip the helper. `captureScreenshot(path)` is 1x, ignores `deviceScaleFactor`
  and takes a plain string path:

      await cdp('Emulation.setDeviceMetricsOverride', { width:1600, height:950, deviceScaleFactor:2, mobile:false })
      const r = await cdp('Page.captureScreenshot', { format:'png' })
      fs.writeFileSync(path, Buffer.from(r.data, 'base64'))

- On list views, crop and don't hide. `display:none` on the sidebar blanks the virtualised rows.
  Capture the full frame and cut the content pane with PIL (at 2x the sidebar ends near x=660 and the
  tabs and banner sit above y=445). Cropping the rail is also what keeps a real space name out of a
  demo frame: the blast radius is the whole screen (sidebar, agent list, search, breadcrumb).
- A task detail isn't virtualised, so hide-then-crop works: hide the `Ask Brain` promo strip, the empty
  `Add description` band, the `Fields from <type> type` header and empty `-` fields (walk 3
  ancestors up from the text node or a black bar stays), then crop out the sidebar, Activity panel and
  icon rail.
- Set columns before you shoot: a `Recommendation` column at width 520 reads, at 420 it ellipses.
  Park the mouse outside the table (the hover row shows buttons). Freshly opened views paint headers
  first and field cells later: scroll and back. Everything in one bucket is a ClickApp that's off.
- The sidebar is pinned collapsed and a JS `.click()` on `[data-test="simple-bar__expand-sidebar-button"]`
  collapses again before the shot. Dispatch `Input.dispatchMouseEvent` (`mouseMoved`, `mousePressed`,
  `mouseReleased`) at the button's centre, then `mouseMoved` to about (150, 520) to hold the hover.
- Public `clickup.com` pages shoot fine headless: dismiss the OneTrust banner (button text `ok`), hide
  fixed or sticky elements over 40px, capture with a `clip` from `getBoundingClientRect` and
  `captureBeyondViewport: true`. Create the tab on `about:blank` and `Page.navigate` (`/json/new?<url>`
  plus `Runtime.evaluate` hangs); the evaluate result is at `res.result.result.value`. Pricing
  selectors are hashed CSS modules (2026-08-15), so re-probe. `help.clickup.com` is Cloudflare-gated;
  AI prices live on `/pricing` and `/brain/pricing` (`/ai/pricing` is a 404).

## The Workload view

Open `https://app.clickup.com/<teamId>/v/wl/<viewId>`. One menu picks what's summed (Tasks, Time
Estimates, or any number or dropdown field, summed by option label), another what a cell shows
(Weekly Capacity is the percent mode, red past 100%). A person's row exists only once they have
scheduled work in the window. At 1900x861:

- Collapse a row (559px to 47px, so 7 people fit a frame): `.row-details-assignee-settings`
  inside `cu-timeline-group-row`, about x=600. Rows are virtualised: collapse the topmost expanded
  one, re-probe, repeat.
- To set capacity, use `.cu-timeline-group-row__capacity-toggle` reads `24/15` or `Set capacity` and opens a
  popover whose input has `placeholder="-"`. Double-click the input, type, click Save. A click plus
  `Meta+a` kept the old value.
- State is per session until you Save view, so a reload re-expands every row.
- A synthetic `.click()` does nothing here: 12 runs moved nothing, and `click([x, y])` collapsed
  on the first try. Probe with `js()` for coordinates, act with the mouse.

## Demos and subagents

- Demos are invented, always, in their own list. Never a real client's board with the logo swapped,
  even temporarily, because the intermediate state ends up in a take.
- Verify with the API. A screenshot proves a dialog closed and nothing about whether an
  automation fired. `cu task create`, wait about 20 seconds, `cu task get <id> --fields`, `cu task
  delete <id>`. Export `CU_TEAM_ID` for the workspace under test or the CLI hits your default.
- Lazy renders: an agent image in a comment shows as `"text": ""` in `GET .../comment` (the image is a
  sibling block) and the Activity panel loads it off-screen.
- A concurrent session renaming a source task mid-run orphans a template's pointer: re-read from the
  CLI, save a new template and swap it into the rule.
- Subagents parallelize badly here: one tab per agent, and closing the window kills the whole tab
  group. Forbid `select_browser`, `switch_browser` and `tabs_create_mcp`, tell each to report rather
  than improvise when its tab dies, and expect to finish the tail yourself. An agent also reports
  success when a click landed but the confirmation dialog was missed: re-read the object before redoing.
- Dashboards and cards are UI-only and not mapped yet: map the click path the next time one comes up.
