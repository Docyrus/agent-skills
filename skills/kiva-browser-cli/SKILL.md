---
name: kiva-browser-cli
description: Use KIVA CLI browser commands for browser automation (local Chrome or remote Cloudflare Browser Rendering). Use when you need to start a browser session, navigate pages, inspect page state, interact with elements, capture screenshots, evaluate JavaScript, read console output, inspect network requests, or run CDP scripts.
---

# KIVA Browser CLI

Browser automation via a persistent daemon that holds a raw CDP (Chrome DevTools Protocol) WebSocket connection. No Puppeteer/Playwright framework — direct CDP for speed and flexibility.

Works in both **local mode** (Chrome on `:9222`) and **remote sandbox mode** (Cloudflare Browser Rendering).

## Architecture

```
kiva browser <command>
  → tool script (Node.js)
    → HTTP request to daemon (localhost:9333)
      → raw CDP WebSocket to Chrome/Cloudflare
```

The daemon starts automatically on the first command and stays alive for 5 minutes of idle time. All commands share the same browser connection — no reconnection overhead.

## Core Workflow for Self-Testing

```bash
# 1. Start browser + daemon
kiva browser start

# 2. Navigate to the app preview (a /path resolves against the preview URL,
#    so you never need to know the sandbox hostname)
kiva browser nav /

# 3. Wait for page to fully load
kiva browser wait --idle

# 4. Discover interactive elements
kiva browser snapshot

# 5. Interact using refs from snapshot
kiva browser fill @e2 "user@example.com"
kiva browser click @e3

# 6. Verify results
kiva browser wait --selector ".success-message"
kiva browser screenshot
kiva browser console --level error
kiva browser network --method POST --status 2xx
```

## Commands

### Session Management

```bash
kiva browser start                     # Start Chrome + daemon
kiva browser start --profile           # Start with user's Chrome profile (local only)
kiva browser close                     # Stop daemon and disconnect
```

### Navigation

```bash
kiva browser nav <url>                 # Navigate active tab
kiva browser nav /settings             # Path → resolved against the preview origin
kiva browser nav <url> --new           # Open in new tab
kiva browser nav <url> --reload        # Navigate and force reload
```

A target with no scheme resolves against the app preview origin (see
[Preview URL](#preview-url)), so `/settings` and `settings` both land on the
running app. Resolution is against the ORIGIN, not the current path, so the
result does not depend on which page is open. `host:port` shorthand
(`localhost:3000/foo`) is expanded to `http://`. Absolute URLs pass through
untouched.

### Preview URL

```bash
kiva browser url                       # Where the app under development is served
```

Returns `{ previewUrl, origin, headers? }`. Needs no daemon and no open tab.

In a sandbox the dev server binds to localhost *inside* the sandbox while the
browser runs outside it, so `http://localhost:3000` is not reachable and the
external hostname cannot be derived locally. The API injects it as
`KIVA_PREVIEW_URL` when it starts the agent server, and it is persisted in
`.kiva/browser.json`. `kiva browser start` also opens the app in the first
tab automatically, already signed in — so the usual first step is `wait --idle`
and `snapshot`, not `nav`.

If `url` reports nothing, the preview never came up: restart it rather than
guessing a URL.

### Waiting

```bash
kiva browser wait --idle               # Wait for network idle
kiva browser wait --selector "h1"      # Wait for element to appear
kiva browser wait --url "**/dashboard" # Wait for URL to match pattern
kiva browser wait 2000                 # Wait fixed milliseconds
kiva browser wait --idle --timeout 30000
```

### Page Snapshot (Element Discovery)

```bash
kiva browser snapshot                  # Interactive elements with refs (@e1, @e2, ...)
kiva browser snapshot --all            # Include non-interactive elements
kiva browser snapshot --selector "#app" # Scope to a subtree
```

Returns:
```json
{
  "snapshot": [
    { "ref": "@e1", "tag": "input", "type": "email", "placeholder": "Email" },
    { "ref": "@e2", "tag": "input", "type": "password" },
    { "ref": "@e3", "tag": "button", "text": "Sign In" }
  ]
}
```

### Element Interaction

Three targeting modes — refs, CSS selectors, or x,y coordinates:

```bash
kiva browser click @e3                 # Click by snapshot ref
kiva browser click "button.submit"     # Click by CSS selector
kiva browser click 350 200             # Coordinate click (compositor-level)
kiva browser fill @e1 "user@test.com"  # Clear + type into input
kiva browser select @e4 "Option A"    # Select dropdown option
```

Coordinate clicks pass through iframes, shadow DOM, and cross-origin frames at the compositor level — no DOM piercing needed. Use `info` for viewport dimensions.

### JavaScript Evaluation

```bash
kiva browser eval 'document.title'
kiva browser eval 'const rows = document.querySelectorAll("tr"); return rows.length'
kiva browser eval 'await fetch("/api/health").then(r => r.json())'
```

Supports expressions and multi-statement code. Uses CDP `Runtime.evaluate` with `awaitPromise: true`.

### Screenshots

```bash
kiva browser screenshot               # Viewport screenshot → /tmp
kiva browser screenshot --full         # Full page
kiva browser screenshot --base64       # Base64 in JSON (for remote/sandbox mode)
```

### Console Messages

```bash
kiva browser console                   # Captured logs (last 50)
kiva browser console --level error     # Only errors
kiva browser console --listen 5000     # Listen for 5 seconds via CDP events
```

Run `console` once early to install the interceptor.

### Network Inspection

```bash
kiva browser network                   # All captured requests
kiva browser network --method POST     # Only POST requests
kiva browser network --status 4xx      # Only 4xx responses
kiva browser network --url "/api/"     # URL substring filter
kiva browser network --listen 5000     # Listen for 5 seconds
```

Network events are captured automatically by the daemon (CDP `Network.enable`).

### KIVA Devtools

Read runtime diagnostics from `@kivacore/devtools` (must be loaded on the page):

```bash
kiva browser devtools state              # Full devtools state
kiva browser devtools errors             # Collected API errors
kiva browser devtools issues             # Detected API usage issues (e.g. missing fields, bad queries)
kiva browser devtools console            # Console entries captured by devtools
kiva browser devtools console --level error  # Filtered by level
```

Use after interacting with a KIVA-backed app to catch API misuse, failed requests, and runtime issues that `console --level error` alone would miss.

### Content Extraction

```bash
kiva browser content <url>             # Extract readable markdown from URL
```

### Cookies

```bash
kiva browser cookies                   # All cookies (via CDP Network.getCookies)
kiva browser cookies --name "session"
kiva browser cookies --domain ".app.com"
```

### Page Info

```bash
kiva browser info                      # Preview URL, page URL, title, viewport, scroll, dimensions
```

### Tab Management

```bash
kiva browser tabs                      # List all tabs
kiva browser tabs --switch 1           # Switch to tab by index
```

### CDP Scripts

```bash
kiva browser run-script script.js
```

The script receives raw CDP helpers: `cdp`, `evaluate`, `navigate`, `captureScreenshot`, `clickAt`, `typeText`, `pressKey`, `pageInfo`, `drainEvents`, `waitForCondition`.

## Tips for AI Agents

1. `start` auto-launches the daemon — subsequent commands are fast (no reconnection)
2. In a sandbox, `start` already opens the app you are building: `wait --idle` and `snapshot`, no `nav` needed
3. Navigate the app by path (`nav /settings`); `browser url` shows the origin that fills in
4. Always `wait --idle` after `nav` before taking snapshots or screenshots
5. Use `snapshot` → refs → `click`/`fill` for reliable element targeting
6. Re-snapshot after interactions to get fresh refs
7. Use coordinate `click 350 200` for elements inside iframes or shadow DOM
8. Check `console --level error` and `network --status 5xx` to catch runtime issues
9. Use `screenshot --base64` in remote/sandbox mode
10. `close` stops the daemon; it auto-stops after 5 minutes idle anyway
