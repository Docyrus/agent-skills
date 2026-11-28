---
name: kiva-e2e-browser-testing
description: End-to-end self-test a KIVA-backed web app in a real browser — sign in headlessly, drive the UI, then read runtime problems back from @kivacore/devtools. Use after building or changing a KIVA app feature to prove it works in the browser — authenticate without typing credentials via `kiva auth sso-session`, navigate/click/fill with the kiva browser tools, and fetch collected API errors, usage issues, and console errors via `kiva browser devtools`. Triggers on "test the app in the browser", "e2e test this flow", "sign me into the app", "self-test the feature I built", "verify it works end to end", "check for runtime errors", "what's broken on the page", "is @kivacore/devtools installed".
---

# KIVA E2E Browser Testing

The loop for proving a KIVA-backed app actually works after you change it: **sign in headlessly → drive the real UI → read the runtime problems the app collected.** It ties together three tools:

1. **`kiva auth sso-session`** — mint a short-lived token so the app logs in with no credential typing.
2. **The kiva browser tools** — navigate, snapshot, click, fill, screenshot. Full reference: **`kiva-browser-cli` skill**. This skill only covers the e2e-specific use.
3. **`@kivacore/devtools`** — an in-page diagnostics layer most KIVA pi-built apps already ship. Read its collected errors/issues/console with `kiva browser devtools`.

## The e2e loop

```
1. Confirm CLI auth, env, tenant     kiva auth who / env which / auth tenants use
2. Mint an SSO token                 kiva auth sso-session --clientId <appClientId>
3. Navigate WITH the token           kiva browser nav "<appUrl>?sso_token=<token>"  → app auto-signs-in
4. Drive the changed flow            kiva browser snapshot / click / fill / wait
5. Pull the problems                 kiva browser devtools issues|errors|console --level error
6. Triage → fix → repeat
```

The browser tools come in two equivalent forms — use whichever your runtime exposes:

- **CLI form** (shell): `kiva browser nav <url>`, `kiva browser devtools issues`, … (used throughout this skill)
- **Desktop pi-tool form** (when `KIVA_DESKTOP_TOOLS=1`): `kiva_browser_navigate`, `kiva_browser_devtools`, … — same commands, same args.

Both drive the same browser. SSO sign-in is tool-agnostic: mint the token with the CLI, then navigate that browser to the token URL.

`<appUrl>` is the app's preview origin. Don't guess it and don't assume
`localhost` — in a sandbox the dev server is only reachable through the exposed
preview URL, which `kiva browser url` prints (`{ previewUrl, origin }`).

## 1. Sign in with `kiva auth sso-session`

`sso-session` exchanges your **already-authenticated CLI session** for a short-lived `sso_token` the app can redeem. No login UI, no password.

### Preconditions

```bash
kiva auth who              # you are signed in as the user you want to test as
kiva env which             # active env points at the SAME backend the app uses
kiva auth tenants use <n>  # active tenant is the one you want to test against
```

The app must be configured (its `KivaAuthProvider apiUrl`) against the **same KIVA environment** as the CLI's active env, or the exchange will reject the token.

### Mint the token

```bash
kiva auth sso-session --clientId <appClientId>
# optional: --targetOrigin https://app-preview.example.com   (restrict the token to one origin)
# optional: --scope "<oauth2 scopes>"                          (defaults to the standard login scopes)
```

Returns:

```json
{ "sso_token": "…", "expires_in": 120, "url": "https://app-preview.example.com?sso_token=…" }
```

`url` is only populated when you pass `--targetOrigin` — it is the ready-to-navigate URL.

**`--clientId` is required and must equal the OAuth2 client id the app's `<KivaAuthProvider clientId=…>` uses** — the backend exchange validates `client_id`. Find it in the app source: grep for `KivaAuthProvider`, `clientId`, or an env var like `VITE_KIVA_CLIENT_ID` / `KIVA_CLIENT_ID`.

### Redeem it (sign the app in)

Navigate the browser to the app origin with `?sso_token=` appended:

```bash
kiva browser start
kiva browser nav "https://app-preview.example.com/?sso_token=<token>"
kiva browser wait --idle
```

On load, `@kivacore/signin` detects `?sso_token=`, `POST`s it to `/v1/oauth2/sso/exchange`, stores the real OAuth2 tokens, and strips the param from the URL. The app is now authenticated for the session.

### Verify you're in

```bash
kiva browser snapshot          # should show the app shell, NOT a login screen
kiva browser console --level error
```

### Gotchas

- **Short-lived** (`expires_in`, seconds). Mint it *immediately* before navigating; don't reuse a stale token. If the app shows the login screen, the token likely expired — mint a fresh one.
- **One client id.** A token minted for client A cannot sign into an app running as client B.
- **Origin restriction.** `--targetOrigin` binds the token to that origin; use it when testing a known preview URL.
- **Wrong env/tenant** is the most common failure — re-check `env which` / `auth who` before blaming the token.

## 2. Drive the flow

Use the standard browser loop (see the `kiva-browser-cli` skill for the full command set):

```bash
kiva browser snapshot                 # discover refs (@e1, @e2, …)
kiva browser fill @e2 "Acme Corp"
kiva browser click @e5
kiva browser wait --selector ".saved" # or --idle / --url "**/detail/*"
kiva browser screenshot               # visual proof
```

Re-`snapshot` after each navigation/interaction to get fresh refs. Always `wait --idle` after navigations before snapshotting or reading devtools.

## 3. Fetch existing problems from `@kivacore/devtools`

`@kivacore/devtools` instruments `@kivacore/api-client` / `@kivacore/app-client` / `fetch` and the console, collecting failed requests, API-misuse issues, and console errors **as the app runs**. This catches problems that `console --level error` alone misses (e.g. a request that 4xx'd but was swallowed, a duplicated query, an N+1 pattern).

### Is it installed?

```bash
kiva browser devtools state
```

- Success → devtools is loaded; the JSON is the current diagnostics state.
- `✗ @kivacore/devtools is not loaded on this page` → not present (or not yet mounted). Also confirm with:

```bash
kiva browser eval "typeof window.__KIVA_DEVTOOLS__ !== 'undefined'"
```

Most apps scaffolded by the KIVA pi coding agent already ship it. If it is genuinely missing, add it (then rebuild/reload):

- `pnpm add @kivacore/devtools`
- Wrap the app near the root with `<KivaDevtools>` (pass the existing `queryClient`; if the app uses `useKivaClient()`, register it with `useRegisterKivaClient(...)`). The package README ships a ready-made "LLM Install Prompt" — follow it rather than guessing the wiring.
- Note: devtools only attaches when `enabled` (defaults `true`, but apps often gate it to `import.meta.env.DEV`). If the preview is a production build, devtools may be intentionally off.

### Pull the problems

```bash
kiva browser devtools issues               # detected API-usage / perf issues
kiva browser devtools errors               # requests that failed (outcome = error)
kiva browser devtools console --level error # console errors, window errors, unhandled rejections
kiva browser devtools state                # everything at once (entries + errors + issues + console + route/url)
```

Read **after** exercising the flow — devtools accumulates from page load, so drive the feature first, then collect.

### What the output means

**`issues`** — `DevtoolsIssue[]`. The high-signal "you're using the API wrong / slowly" list:

| `code` | meaning |
|---|---|
| `duplicate-request` | same API request fired more than once (often missing dedup/caching) |
| `slow-request` | request exceeded `slowThresholdMs` (default 1000ms) |
| `duplicate-query` | duplicate TanStack Query for the same key |
| `slow-query` / `slow-mutation` | slow TanStack Query / mutation |

Each issue carries `severity` (`warning`/`error`), `title`, `message`, `count`, `routeKey`, and `entryIds` (the requests it derived from).

**`errors`** — `DevtoolsEntry[]` where `outcome === "error"`: a failed API/fetch call. Useful fields: `operation`, `method`, `target` (URL), `status`, `durationMs`, `routeKey`, and `error` (`{ name, message, stack }`). Cross-check `status` against the endpoint you expected to hit.

**`console`** — `DevtoolsConsoleEntry[]`: `level` (`log`/`info`/`warn`/`error`/`debug`) and `source` (`console`, `window-error`, `unhandledrejection`), plus `message`, `args`, `stack`. `--level error` is the fast path to crashes and uncaught rejections.

### Triage

1. **`errors` first** — a failed request usually explains broken UI directly (wrong endpoint, bad payload, missing field, 401/403 ⇒ the SSO sign-in didn't take).
2. **`console --level error`** — uncaught exceptions / rejected promises that broke rendering.
3. **`issues`** — correctness-adjacent and performance smells (duplicate/slow). Fix these to keep the app healthy even when nothing is visibly broken.

Map each finding back to the code you changed, fix, reload (re-mint the SSO token), and re-run the loop until `errors` and error-level `console` are clean.

## End-to-end example

```bash
# context
kiva auth who && kiva env which

# headless sign-in
TOKEN=$(kiva auth sso-session --clientId acme-web --targetOrigin https://acme-preview.dev | jq -r .sso_token)
kiva browser start
kiva browser nav "https://acme-preview.dev/?sso_token=$TOKEN"
kiva browser wait --idle

# exercise the new "create customer" flow
kiva browser nav "https://acme-preview.dev/customers/new"
kiva browser wait --idle
kiva browser snapshot
kiva browser fill @e2 "Acme Corp"
kiva browser click @e7          # Save
kiva browser wait --selector ".toast-success"

# collect problems
kiva browser devtools errors
kiva browser devtools issues
kiva browser devtools console --level error
kiva browser screenshot
```

## Tips

- Drive the flow, **then** read devtools — diagnostics accumulate from load.
- A login screen or a wave of `401`/`403` in `errors` means the SSO step failed (expired token, wrong `--clientId`, or env/tenant mismatch), not the feature.
- `devtools errors` + `devtools console` together catch most regressions; `devtools issues` catches the slow/duplicate ones you won't see by eye.
- Re-mint the SSO token on every fresh reload — it is single-use and short-lived.
- For the complete browser command reference (waiting, selectors, network, CDP scripts, remote/sandbox mode), use the **`kiva-browser-cli`** skill.
