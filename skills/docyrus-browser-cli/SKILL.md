---
name: docyrus-browser-cli
description: Use Docyrus CLI browser commands for browser automation (local Chrome or remote Cloudflare Browser Rendering). Use when you need to start a browser session, navigate pages, inspect page state, interact with elements, capture screenshots, evaluate JavaScript, read console output, inspect network requests, or run CDP scripts.
---

# Docyrus Browser CLI

Browser automation via a persistent daemon that holds a raw CDP (Chrome DevTools Protocol) WebSocket connection. No Puppeteer/Playwright framework — direct CDP for speed and flexibility.

Works in both **local mode** (Chrome on `:9222`) and **remote sandbox mode** (Cloudflare Browser Rendering).

## Architecture

```
docyrus browser <command>
  → tool script (Node.js)
    → HTTP request to daemon (localhost:9333)
      → raw CDP WebSocket to Chrome/Cloudflare
```

The daemon starts automatically on the first command and stays alive for 5 minutes of idle time. All commands share the same browser connection — no reconnection overhead.

## Core Workflow for Self-Testing

```bash
# 1. Start browser + daemon
docyrus browser start

# 2. Navigate to preview
docyrus browser nav https://preview-url.example.com

# 3. Wait for page to fully load
docyrus browser wait --idle

# 4. Discover interactive elements
docyrus browser snapshot

# 5. Interact using refs from snapshot
docyrus browser fill @e2 "user@example.com"
docyrus browser click @e3

# 6. Verify results
docyrus browser wait --selector ".success-message"
docyrus browser screenshot
docyrus browser console --level error
docyrus browser network --method POST --status 2xx
```

## Commands

### Session Management

```bash
docyrus browser start                     # Start Chrome + daemon
docyrus browser start --profile           # Start with user's Chrome profile (local only)
docyrus browser close                     # Stop daemon and disconnect
```

### Navigation

```bash
docyrus browser nav <url>                 # Navigate active tab
docyrus browser nav <url> --new           # Open in new tab
docyrus browser nav <url> --reload        # Navigate and force reload
```

### Waiting

```bash
docyrus browser wait --idle               # Wait for network idle
docyrus browser wait --selector "h1"      # Wait for element to appear
docyrus browser wait --url "**/dashboard" # Wait for URL to match pattern
docyrus browser wait 2000                 # Wait fixed milliseconds
docyrus browser wait --idle --timeout 30000
```

### Page Snapshot (Element Discovery)

```bash
docyrus browser snapshot                  # Interactive elements with refs (@e1, @e2, ...)
docyrus browser snapshot --all            # Include non-interactive elements
docyrus browser snapshot --selector "#app" # Scope to a subtree
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
docyrus browser click @e3                 # Click by snapshot ref
docyrus browser click "button.submit"     # Click by CSS selector
docyrus browser click 350 200             # Coordinate click (compositor-level)
docyrus browser fill @e1 "user@test.com"  # Clear + type into input
docyrus browser select @e4 "Option A"    # Select dropdown option
```

Coordinate clicks pass through iframes, shadow DOM, and cross-origin frames at the compositor level — no DOM piercing needed. Use `info` for viewport dimensions.

### JavaScript Evaluation

```bash
docyrus browser eval 'document.title'
docyrus browser eval 'const rows = document.querySelectorAll("tr"); return rows.length'
docyrus browser eval 'await fetch("/api/health").then(r => r.json())'
```

Supports expressions and multi-statement code. Uses CDP `Runtime.evaluate` with `awaitPromise: true`.

### Screenshots

```bash
docyrus browser screenshot               # Viewport screenshot → /tmp
docyrus browser screenshot --full         # Full page
docyrus browser screenshot --base64       # Base64 in JSON (for remote/sandbox mode)
```

### Console Messages

```bash
docyrus browser console                   # Captured logs (last 50)
docyrus browser console --level error     # Only errors
docyrus browser console --listen 5000     # Listen for 5 seconds via CDP events
```

Run `console` once early to install the interceptor.

### Network Inspection

```bash
docyrus browser network                   # All captured requests
docyrus browser network --method POST     # Only POST requests
docyrus browser network --status 4xx      # Only 4xx responses
docyrus browser network --url "/api/"     # URL substring filter
docyrus browser network --listen 5000     # Listen for 5 seconds
```

Network events are captured automatically by the daemon (CDP `Network.enable`).

### Docyrus Devtools

Read runtime diagnostics from `@docyrus/devtools` (must be loaded on the page):

```bash
docyrus browser devtools state              # Full devtools state
docyrus browser devtools errors             # Collected API errors
docyrus browser devtools issues             # Detected API usage issues (e.g. missing fields, bad queries)
docyrus browser devtools console            # Console entries captured by devtools
docyrus browser devtools console --level error  # Filtered by level
```

Use after interacting with a Docyrus-backed app to catch API misuse, failed requests, and runtime issues that `console --level error` alone would miss.

### Content Extraction

```bash
docyrus browser content <url>             # Extract readable markdown from URL
```

### Cookies

```bash
docyrus browser cookies                   # All cookies (via CDP Network.getCookies)
docyrus browser cookies --name "session"
docyrus browser cookies --domain ".app.com"
```

### Page Info

```bash
docyrus browser info                      # URL, title, viewport, scroll, page dimensions
```

### Tab Management

```bash
docyrus browser tabs                      # List all tabs
docyrus browser tabs --switch 1           # Switch to tab by index
```

### CDP Scripts

```bash
docyrus browser run-script script.js
```

The script receives raw CDP helpers: `cdp`, `evaluate`, `navigate`, `captureScreenshot`, `clickAt`, `typeText`, `pressKey`, `pageInfo`, `drainEvents`, `waitForCondition`.

## Tips for AI Agents

1. `start` auto-launches the daemon — subsequent commands are fast (no reconnection)
2. Always `wait --idle` after `nav` before taking snapshots or screenshots
3. Use `snapshot` → refs → `click`/`fill` for reliable element targeting
4. Re-snapshot after interactions to get fresh refs
5. Use coordinate `click 350 200` for elements inside iframes or shadow DOM
6. Check `console --level error` and `network --status 5xx` to catch runtime issues
7. Use `screenshot --base64` in remote/sandbox mode
8. `close` stops the daemon; it auto-stops after 5 minutes idle anyway
