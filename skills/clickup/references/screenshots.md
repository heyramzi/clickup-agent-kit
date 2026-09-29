# Photographing a ClickUp view

What to do when the extension is unavailable and a view still has to be captured.

## Photograph a view, without the extension

A screenshot of a view for a deck or a report does not need Claude in Chrome, and
should not wait for it. Copy the signed-in Chrome profile, run a second Chrome
headless against the copy on a port of your own, and speak CDP to it. The
`proposal` skill (`references/slide-images.md`) holds the profile copy recipe and the two traps in it (the
zero-byte `Cookies` race, and `IndexedDB` carrying ClickUp's stale offline cache
into every screenshot). What matters here is what to point it at.

The whole capture is four CDP calls: `Page.enable`,
`Emulation.setDeviceMetricsOverride` with `deviceScaleFactor: 2`, `Page.navigate`,
then `Page.captureScreenshot` after 15 to 20 seconds. ClickUp renders correctly
headless. Then crop the content pane out of the frame rather than trying to hide
the chrome:

- **The title is the load check.** A wrong view code loads a page whose
  `document.title` is the bare string `ClickUp`. A loaded view reads
  `<view name> | <list name> | <workspace> (List)`. Assert on it before capturing.
- **Crop, do not hide.** `display:none` on the sidebar collapses ClickUp's
  virtualised rows to blank. Capture the whole 1700x1200 frame and cut the content
  pane out with PIL: the sidebar ends around x=660 at `deviceScaleFactor: 2`, the
  view tabs and the notification banner sit above y=445, and the row area runs to
  the horizontal scrollbar. Read the picture and adjust; those numbers move with
  the window.
- **Set the columns before you shoot.** `cu view create --list <id> --name X
  --type list --group-by priority --columns-json '{"fields":[...]}'` decides what
  the screenshot says. A `Recommendation` column at `"width":520` reads; at 420 it
  ellipses mid sentence.
- **Grouping that comes back all in one bucket is a ClickApp, not a view.** See
  the ClickApps note in the `clickup` skill: the API stores a priority on a space that
  has Priorities switched off, and the UI then draws every task under
  "No Priority".
- **The hover row is a different row.** Whatever the cursor sits on renders its
  action buttons. Park the mouse outside the table before every capture.

## The helper is 1x, CDP is not

`captureScreenshot(path)` writes at CSS pixels and **ignores
`Emulation.setDeviceMetricsOverride`'s `deviceScaleFactor`**: a 1600x950 viewport at factor
2 still came back 1600x950. Driving the protocol directly honours it, so for anything going
on camera:

    await cdp('Emulation.setDeviceMetricsOverride', { width:1600, height:950, deviceScaleFactor:2, mobile:false })
    const r = await cdp('Page.captureScreenshot', { format:'png', captureBeyondViewport:false })
    fs.writeFileSync(path, Buffer.from(r.data, 'base64'))

That returns 3200x1900. Note the helper takes a **plain string path**; passing `{ path }`
throws inside `writeFile`.

Then crop rather than hide, per `references/screenshots.md`. Cropping the left rail is also
what keeps a real client's space name out of a demo frame, which is the blast-radius rule
above: the demo workspace's own space name sits in the sidebar and lands in every
uncropped shot.

## A copied browser profile never reaches a signed-in ClickUp page

Four variants all landed on `app.clickup.com/login` with a valid, unexpired `cu_jwt`: Chrome's
`Default` profile copied and driven headless, the same copy with the real Keychain
(`ignoreDefaultArgs: ["--use-mock-keychain"]`), the same copy headed, and a clean profile with all
71 cookies injected and `navigator.webdriver` masked. Dia's own profile copied and driven by Dia's
binary over CDP failed identically. **Headed failing the same way rules out headless detection**, so
ClickUp binds the session to something outside the cookie jar. This does not generalise from Skool,
which the copy route handles fine.

**A task detail is hide-then-crop, not crop alone.** A task page carries three things a deck does
not want and cropping cannot remove, because they sit between the parts that matter: the
`Ask Brain² for a presentation` promo strip, the empty `Add description` band, and the
`Fields from <type> type` header with its `Create field` button. Hide those three with
`el.style.display='none'` from `js()` first, then capture, then crop the content pane. The
virtualised-rows warning above is about list views; a task detail is not virtualised, so hiding is
safe there. An empty custom field (a row reading `–`) is worth hiding for the same reason.
Then crop out the black sidebar on the left, the Activity panel on the right and the icon rail at
the pane's right edge, and stop the frame on the last field rather than on `Add subtask`. Measured
26 Aug 2026 shooting a freelancer invoice task for the Clic Tempo carousel.

**ego lite reaches a signed-in ClickUp, and it is the first thing to try** (measured 26 Aug 2026,
capturing the demo billing system). It inherits the live session rather than copying a profile,
which is the whole reason the copy route above fails and this one does not. Capture with
`cdp('Emulation.setDeviceMetricsOverride', {deviceScaleFactor: 2})` then
`cdp('Page.captureScreenshot')`, and write the file with `await import('node:fs')`, because the
heredoc is an ES module. See the ego lite section of `chrome-devtools`.

**Orca does not reach ClickUp, so do not spend a turn on it.** `orca tab create` on an
`app.clickup.com` URL lands on `/login`, whatever `chrome-devtools` says about trying Orca
first: that profile is signed out. Measured 5 Sep 2026 shooting a demo workspace build for the
agency blueprint. For ClickUp, ego lite is the first thing to try and the only one that worked.

**`openOrReuseTab` goes in every ego script, not only the first.** Without it `js()` runs
against whatever tab is active, and `document.elementFromPoint` answers with the browser's own
`NTP-APP` host element rather than anything on the page, which reads exactly like a broken
selector. The same run also learned that hiding only the node carrying the AI-credits banner's
text leaves its black bar behind: walk three ancestors up and hide each one.

**The sidebar is pinned collapsed, and a JS click does not hold it open.** `.click()` on
`[data-test="simple-bar__expand-sidebar-button"]` expands the rail and it is collapsed again
before the capture fires, twice in a row. Drive it through CDP and then park the pointer inside
the rail, so the hover state survives the shot:

    const [x, y] = /* the button's rect centre, read through js() */
    await cdp('Input.dispatchMouseEvent', {type:'mouseMoved', x, y, buttons:0})
    await cdp('Input.dispatchMouseEvent', {type:'mousePressed', x, y, button:'left', clickCount:1, buttons:1})
    await cdp('Input.dispatchMouseEvent', {type:'mouseReleased', x, y, button:'left', clickCount:1, buttons:0})
    await cdp('Input.dispatchMouseEvent', {type:'mouseMoved', x:150, y:520, buttons:0})

That is what got all 5 spaces plus DELIVERY's folders and its standing list into one frame.

The extension is the fallback. When `tabs_context_mcp` answers
`Failed to query tabs: Tab not found for session ID`, that browser's pairing is stale and **another
connected one may be fine**: call `list_connected_browsers` and try the next `deviceId` rather than
concluding the extension is dead.

### The URL forms, and two of them are not what the API id suggests

Getting these wrong costs a whole run, because a wrong form loads a page whose `document.title` is
the bare string `ClickUp` and looks exactly like a session problem.

| Surface | URL | Loaded title reads |
| --- | --- | --- |
| Space or list **view** | `/{team}/v/l/{viewId}` | `<view> \| <space> \| <workspace> (List)` |
| Space overview | `/{team}/v/o/s/{spaceId}` | `Overview \| <space> \| <workspace> (Overview)` |
| **Chat channel** | `/{team}/chat/r/{channelId}` | `<channel> \| <workspace>` |

**`/v/li/{viewId}` is the trap.** It is the form most docs and older sessions reach for, and on a
view created through `POST /space/{id}/view` it renders the sidebar and nothing else, forever. The
working form is `/v/l/`. **`/v/chat/{channelId}` is the same trap for chat**: it silently redirects
to `All Tasks` (`/v/l/7-{team}-1`). The chat id is not guessable from the API either, which returns
only `links.members` and `links.followers`; read a real one off any `<a>` on `/{team}/v/chat`.

Assert on the title before capturing, and allow **20 to 30 seconds** for the SPA to paint. Navigate
in one `ego-browser` round and capture in the next, or the call is killed at the shell timeout.

## Public clickup.com pages screenshot fine headless

The opposite case. Launch headless Chrome with `--remote-debugging-port`, drive CDP from Node,
dismiss the OneTrust banner by clicking the button whose text is `ok`, hide every `position: fixed`
or `sticky` element over 40px tall, then `Page.captureScreenshot` with a `clip` from
`getBoundingClientRect` and `captureBeyondViewport: true`.

Two mechanics: `/json/new?<url>` followed by `Runtime.evaluate` **hangs**, so create the tab on
`about:blank` and `Page.navigate`; and a CDP evaluate result sits at `res.result.result.value`, two
levels deep.

Pricing selectors are CSS-module hashed and were stable on 2026-08-15:
`.PricingV4CardsContainer_cardsGrid__NLbNu` for the four plan cards,
`.PricingV4AIPricing_cardsContainer__IdMUU` for the AI cards, and
`.BrainPricing_cardsContainer__nDUSk` on `/brain/pricing` for the add-ons. Re-probe the DOM rather
than trusting a hash after a redesign.

**`help.clickup.com` is Cloudflare bot-gated** and answers "Performing security verification" to
headless, so take help-centre facts from `clickup.com/pricing` or through the extension.
`clickup.com/ai/pricing` is a 404: the AI prices live on `/pricing` and `/brain/pricing`.

## View URL type codes

Guessing one wastes turns, because a wrong code renders a skeleton plus a
`VALIDATION_DEFAULT_VIEWS` toast rather than a clean 404.

| View type | Code | Example |
|---|---|---|
| table / grid | `gr` | `app.clickup.com/<team>/v/gr/8cbypq9-103975` |
| list | `li` | |
| board | `b` | |
| workload | `wl` | |
| space overview | `o/s` | `.../v/o/s/90154546093` |
| folder overview | `o/f` | |

Get the ids from `cu views list --space <id>` (also `--folder`, `--list`) rather than reading them
off a URL. Safest of all: open the space and click the view tab.

