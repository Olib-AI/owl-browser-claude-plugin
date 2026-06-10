# Owl Browser plugin for Claude Code

Agent-native browser automation for Claude Code. This plugin bundles:

- **The Owl Browser MCP server** (`@olib-ai/owl-browser-mcp`), which exposes the browser tools and defaults to the agent-native render.
- **An `owl-browser` skill** that teaches Claude the observe-and-act loop: read pages as **OwlMark** (a compact, handle-addressable text render) and act on elements by handle, instead of screenshots or pixel coordinates.

## Why

A screenshot costs a vision model a lot of tokens and still does not tell it where to
click, so it guesses coordinates and misfires. Owl returns the page as structured,
handle-addressable text. On a Hacker News front page that is about 753 tokens covering
240 interactive elements, versus roughly 1,365 tokens for a screenshot. It is a renderer,
not a JavaScript plugin on top of a normal browser.

## Prerequisites

A running Owl Browser instance (Docker or standalone) and two environment variables in
your shell before launching Claude Code:

```bash
export OWL_API_ENDPOINT="http://localhost:8080"   # or your Docker/nginx URL, e.g. http://localhost:80
export OWL_API_TOKEN="your-owl-http-token"         # matches OWL_HTTP_TOKEN on the server
# optional: which MCP toolset to advertise (agent | automation | webdev | full); default agent
export OWL_MCP_PROFILE="agent"
```

`npx` (Node.js) must be available on PATH for the bundled MCP server.

## Install

From a published marketplace repo:

```
/plugin marketplace add Olib-AI/owl-browser-claude-plugin
/plugin install owl-browser@olib-ai
```

Then restart Claude Code (or reload plugins). The `browser_*` tools and the
`owl-browser` skill become available.

## Use

Ask Claude to do anything on the web and it will follow the loop:

```
create_context(render_mode=agent) -> navigate(url) -> observe -> click/type(handle) -> observe -> ...
```

See the bundled skill (`skills/owl-browser/SKILL.md`) for the full tool reference,
params, edge-case handling, and examples.

## Layout

```
.claude-plugin/
  plugin.json          plugin manifest (bundles the MCP server via .mcp.json)
  marketplace.json     marketplace listing this plugin (source ".")
.mcp.json              Owl Browser MCP server registration
skills/
  owl-browser/
    SKILL.md           the agent-usage skill
```

## Publishing

This directory is a self-contained marketplace plus plugin. To publish to the Claude
plugin directory, push it as a git repository; users then run
`/plugin marketplace add <owner>/<repo>` and `/plugin install owl-browser@olib-ai`.

By Olib AI. https://www.owlbrowser.net
