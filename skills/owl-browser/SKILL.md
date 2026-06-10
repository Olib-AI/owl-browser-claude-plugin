---
name: owl-browser
description: Drive Owl Browser as an agent. Read web pages as compact, handle-addressable OwlMark text and click or type by handle instead of screenshots or pixel coordinates. Use when navigating sites, scraping content, filling forms, logging in, or automating any web task through the Owl Browser tools (browser_create_context, browser_navigate, browser_observe, browser_click, browser_type).
license: MIT
---

# Owl Browser (agent rendering)

Owl Browser is an AI-native browser. Instead of screenshots or raw HTML, it renders
each page as **OwlMark**: a compact, handle-addressable text view of what is actually
on screen. You **observe** the page, then **act on handles**. This is far cheaper than
a screenshot and removes pixel-coordinate guessing.

This plugin bundles the Owl Browser MCP server, so the tools below are available as
tool calls (`browser_create_context`, `browser_navigate`, `browser_observe`, ...).
They require `OWL_API_ENDPOINT` and `OWL_API_TOKEN` to point at a running Owl Browser
instance (Docker or standalone). The same tools exist over REST at
`POST $OWL_API_ENDPOINT/execute/<tool>`.

## The loop (do this, keep it short)

```
browser_create_context(render_mode="agent")
  -> browser_navigate(url)
  -> browser_observe                       # OwlMark text + handle table
  -> browser_click(handle) / browser_type(handle, text)
  -> browser_observe                       # see the result, repeat
  -> browser_close_context
```

- Call `browser_observe` after navigating and after every action. It is the only way you see the page.
- Act using the handle tokens `observe` prints (e.g. `l5`, `b12`, `x27`). No CSS selectors, no pixel coordinates.
- `browser_observe` blocks until the page is ready. Do not add a separate wait step.
- Never screenshot to read text or find elements. Screenshot only to judge visual design or layout.

## Reading an OwlMark render

`browser_observe` returns `render` (the text view), `handles` (actionable elements),
`metadata`, and `token_estimate`. In the render, an element looks like:

```
- link "Pricing" [#l5]
- button "Sign in" [#b12]
  textbox "Email" [#x27 val=""]
```

The token in brackets is the handle. Pass it to click/type. Markers like `[#R1]` or
`T1 x8` are collapsed groups you can `browser_expand`.

## Core tools

- **browser_create_context** `{render_mode}` -> returns `result.context_id`. Use `"agent"` (token-efficient render), `"both"` (agent + pixels for screenshots), or `"pixel"` (legacy). Do NOT pass `context_id` in; it is generated.
- **browser_navigate** `{context_id, url}` -> does not return the page; observe next.
- **browser_observe** `{context_id, detail?, region?, max_tokens?}` -> the render + handles. `detail` = `min`|`normal`(default)|`full`|`outline`. Use `outline` for a headings-only map of a long page, then expand/read the part you want.
- **browser_click** `{context_id, selector}` where `selector` is the handle token. Returns an `effect`: `navigated`, `dom-changed`, or `no-effect`. Trust it, then re-observe.
- **browser_type** `{context_id, selector, text}`. Use **browser_clear_input** `{context_id, selector}` first to replace text, and **browser_press_key** `{context_id, key:"Enter"}` to submit.
- **browser_screenshot** `{context_id}` for a visual check of design or layout only.
- **browser_close_context** `{context_id}` when done.

Drill-downs (rare): **browser_expand** `{context_id, handle}` re-serializes a collapsed
region; **browser_read_node** `{context_id, handle}` returns one node's full text.

## Edge cases and recovery

Check `metadata.status` on every observe:
- `ready` — act on it.
- `pending` — content has not rendered yet (a lazy client-rendered shell). The envelope has `reason` and `retry_after_ms`; re-observe after that delay. Do NOT treat a pending render as an empty page.
- `incomplete` — chrome rendered but main content did not; re-observe once, then use vision.

`metadata.dropped_surfaces` says what the text render could not capture:
- `canvas` / `webgl` / `image:N` — a visual surface: create the context with `render_mode:"both"`, screenshot, and read the pixels with your own vision.
- `sparse_main` / `shell_unhydrated` / `main_content_unrendered` — content late or withheld: re-observe, then vision.
- `first_tree_timeout` — slow or bot-blocked: read it with a screenshot.

Other cases:
- **Handles are per-document.** After any navigation (a click whose `effect` is `navigated`, or a `browser_navigate`), re-observe to get fresh handles. A stale handle returns `STALE_HANDLE`; re-observe.
- **Same-page anchors scroll, they do not navigate.** Clicking an `href="#section"` link returns `effect:"scrolled"` and moves the viewport. Expected.
- **Rare click-nav crash:** on a few slow sites, a click that triggers a cross-document navigation can crash and auto-respawn the browser (~1s). If a context is lost right after such a click, recreate it and `browser_navigate` directly to the destination URL.
- **PDF / embedded plugins:** read with a screenshot (`render_mode:"both"`); in-page plugin controls may not be actionable handles.

## Do and do not

- DO observe before acting and re-observe after every action.
- DO act on the exact handle tokens `observe` printed.
- DO read the `effect` of a click/type before assuming it worked.
- DO use `detail:"outline"` on long reference pages, then expand/read the part you need.
- DO NOT screenshot to read a page or find elements.
- DO NOT guess pixel coordinates. Owl gives you handles so you never have to.
- DO NOT pass `context_id` to `browser_create_context`; it is returned to you.

## Minimal example: search and open a result

```
browser_create_context {"render_mode":"agent"}                 -> ctx
browser_navigate        {"context_id":ctx,"url":"https://duckduckgo.com"}
browser_observe         {"context_id":ctx}                     # find the search box, e.g. x4
browser_type            {"context_id":ctx,"selector":"x4","text":"owl browser olib ai"}
browser_press_key       {"context_id":ctx,"key":"Enter"}
browser_observe         {"context_id":ctx}                     # results appear, pick a link, e.g. l31
browser_click           {"context_id":ctx,"selector":"l31"}    # effect: navigated
browser_observe         {"context_id":ctx}                     # read the opened page
browser_close_context   {"context_id":ctx}
```
