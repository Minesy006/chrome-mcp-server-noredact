# chrome-mcp-server (no-redact)

Fork of [hangwin/mcp-chrome](https://github.com/hangwin/mcp-chrome) unpacked MV3 extension, patched for pentest-style browser automation with Claude Code.

Current version: **1.0.4** (2026-09-27)

This is **not** a drop-in replacement of the official store build. It keeps the original browser-operator tools (click / fill / navigate / screenshot) and opens the data path: cookies, full headers, curl export, no JS redaction.

## What this fork changes

See [CHANGELOG-vs-upstream.md](./CHANGELOG-vs-upstream.md) for the full table. Short version:

| Area | Upstream 1.0.0 | This fork 1.0.4 |
|---|---|---|
| `chrome_javascript` | redacts cookies / JWT / tokens | returns plaintext |
| JS output cap | 50KB | 512KB |
| debugger busy | silent isolated-world fallback | MAIN world, or reuse capture CDP |
| cookies | none | `chrome_cookies` including HttpOnly |
| `chrome_network_request` | active tab page `fetch` (CORS) | `tabId` + background fetch with cookie jar |
| capture | 100 req, stripped headers, HTML dropped | 2000, full headers, HTML/XHR kept, per-request `curl` |
| MCP initialize | stale session → 400 | recreate transport (bridge patch, not in this folder) |

## This folder is the unpacked Chrome extension

Load it in Chrome:

1. Open `chrome://extensions`
2. Enable **Developer mode**
3. **Load unpacked** → select this directory
4. If Chrome asks for the extra `cookies` permission, allow it

The local install that keeps the original extension ID (`hbdgbgagpkpjffpklnamcljpakneikee`) still has `manifest.key` **on disk only**. That field is a **private key** and is **not** in this repository. A fresh unpacked load of this public copy gets a **new extension ID**. That is expected.

## This folder is not a complete MCP install

Claude Code talks to Native Messaging host `com.chromemcp.nativehost` on `http://127.0.0.1:12306/mcp`.

On the machine that will use the tools:

```bash
npm install -g mcp-chrome-bridge
```

Then connect the loaded extension to the native host (the extension UI / original docs).

Two extra patches live in the global npm package, **not** in this repo:

- `chrome-mcp-shared` tool schemas (`chrome_cookies`, `tabId`, curl description, 512KB default)
- `mcp-chrome-bridge` `initialize` recovering a stale `mcp-session-id`

`npm update` / reinstall of `mcp-chrome-bridge` can overwrite those.

## Intended workflow

Chrome = logged-in browser operator + credential / traffic extractor.  
Burp / curl = mutate, replay, batch IDOR.  
Claude Code orchestrates.

Do not expect this extension to become Repeater / Intruder.

## Upstream

- https://github.com/hangwin/mcp-chrome
- License: MIT (see `LICENSE`)
