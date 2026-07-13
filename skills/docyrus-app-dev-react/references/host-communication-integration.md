<!--
Host ↔ embedded-app communication for `@docyrus/signin`, EXCLUDING the Guidy
bridge (see `guidy-integration.md` for that). Source of truth:
`docyrus-devkit/packages/signin` (core/iframe-auth.ts, core/auth-manager.ts,
hooks/use-host-messages.ts, types.ts). Refresh this file if that package changes.
-->

# Host ↔ App Communication Integration (`@docyrus/signin`)

How an embedded Docyrus app talks to the host shell (the "super app") over the
`postMessage` channel — **every message direction except the Guidy bridge**, which is
documented separately in `guidy-integration.md`.

Everything here is **iframe / WebView only** and a **no-op in standalone and
React-Native modes**, so all of it is safe to mount unconditionally.

## Mental model

The Docyrus shell embeds each app as a **cross-origin `<iframe>`** (or, on mobile, a
React Native WebView). `@docyrus/signin` runs an `IframeAuth` runtime inside the app that
exchanges `postMessage` payloads with the shell. The same origin-validated channel carries
auth **and** the side-channel messages below.

```
┌───────────────────────── Docyrus shell (host) ─────────────────────────┐
│  address bar   menus / deep links   notification center                │
│      ▲                │                    │                            │
│      │ route-change   │ navigation         │ notification               │
│      │ navigation-req │                    │                            │
│  ┌───┴────────────────▼────────────────────▼──── cross-origin iframe ─┐ │
│  │  YOUR EMBEDDED APP (@docyrus/signin)                               │ │
│  │  in : useDocyrusHostNavigation, useDocyrusHostNotification         │ │
│  │  out: syncRouteToHost / notifyRouteChange, requestHostNavigation   │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────┘
```

### Message catalog (non-Guidy)

| Direction | Message | Trigger / API |
|---|---|---|
| app → host | `{ type: 'signin-ready' }` | Auto, on iframe-mode start. Tells the host the app is ready to receive tokens. |
| host → app | `{ type: 'signin', accessToken, refreshToken }` | Host delivers tokens. Handled by the provider — you never see it. |
| app → host | `{ type: 'token-refresh-request' }` | Auto, when the token nears expiry. Host replies with a fresh `signin`. |
| host → app | `{ type: 'navigation', url }` | **`useDocyrusHostNavigation`** — host asks the app to navigate. |
| host → app | `{ type: 'notification', notification }` | **`useDocyrusHostNotification`** — host pushes a `DocyrusNotification`. |
| app → host | `{ type: 'route-change', path, search, hash, url }` | **`syncRouteToHost` / `useDocyrusHostRouteSync` / `notifyRouteChange`** — report the app's route. |
| app → host | `{ type: 'navigation-request', url, replace?, newTab? }` | **`requestHostNavigation` / `useDocyrusRequestHostNavigation`** — ask the host to navigate. |

The auth handshake (`signin-ready` / `signin` / `token-refresh-request`) is fully managed
by `DocyrusAuthProvider`; it's listed for context only. The rest of this doc is the four
APIs you actually wire.

## Origin validation

Every inbound message's `event.origin` is checked before processing:

- The built-in pattern trusts `*.docyrus.app` hosts.
- Add extra trusted origins with the `allowedHostOrigins` prop:
  ```tsx
  <DocyrusAuthProvider allowedHostOrigins={['https://shell.mycorp.com']} … />
  ```
- In a **React Native WebView** the origin check is skipped (the RN bridge is trusted);
  outbound messages go through `window.ReactNativeWebView.postMessage` instead of
  `window.parent.postMessage`. Your app code is identical either way.

Handlers are wrapped in try/catch — a throwing handler is logged, never crashes the app.
Handlers registered before iframe mode finishes initializing are retained and fire once
messages arrive, so mounting order doesn't matter.

---

## 1. Host → App: navigation (`useDocyrusHostNavigation`)

The shell (a host menu, a deep link, the notification center — and, when the Guidy bridge
is enabled, a Guidy route jump) asks the app to navigate.

```tsx
import { useDocyrusHostNavigation, type HostNavigationMessage } from '@docyrus/signin';

useDocyrusHostNavigation((message: HostNavigationMessage) => {
  // message = { type: 'navigation', url: string }
  navigate(message.url);
});
```

- Subscribes on mount, unsubscribes on unmount.
- **The latest closure is always invoked** — no memoization needed; close over props/state freely.
- Empty / missing `url` messages are dropped before your handler runs.
- `url` may be **absolute or relative** — normalize before handing to the router (see §5).
- No-op outside iframe/WebView mode.

Alternatively subscribe imperatively (returns an unsubscribe function; the underlying primitive for Vue):

```ts
const { onHostNavigation } = useDocyrusAuth();
useEffect(() => onHostNavigation(({ url }) => navigate(url)), [onHostNavigation, navigate]);
```

## 2. Host → App: notification (`useDocyrusHostNotification`)

The host pushes a `DocyrusNotification` (mirrors the `NotificationEntity` API contract) —
surface it however the app already surfaces notifications (a toast, a bell badge, etc.).

```tsx
import { useDocyrusHostNotification, type DocyrusNotification } from '@docyrus/signin';
import { toast } from 'sonner';

useDocyrusHostNotification((n: DocyrusNotification) => {
  toast(n.subject, { description: n.message });
});
```

`DocyrusNotification` shape:

| Field | Type | Notes |
|---|---|---|
| `id` | `string` | Stable id (messages without a string `id` are dropped). |
| `subject` | `string` | Title. |
| `message` | `string` | Body. |
| `status` | `string` | Server status. |
| `seen` | `boolean` | Read state. |
| `created_on` / `notify_on` | `string` | Timestamps. |
| `created_by` / `created_by_id` / `created_by_fullname` / `created_by_photo` | `string` | Author. |
| `record_owner` | `string` | Owner id. |
| `params` | `Record<string, unknown> \| null` | Optional payload. |
| `tenant_app_id` | `string \| null` | Optional originating app. |
| `output_render_template` | `Record<string, unknown> \| null` | Optional render hint. |

Same guarantees as navigation: latest closure always invoked, no-op outside iframe mode,
handler errors caught.

## 3. App → Host: route reporting (address-bar sync)

Keep the shell's own address bar reflecting the app's current route (so users can
copy / share / bookmark the URL). The app posts
`{ type: 'route-change', path, search, hash, url }` where `url = path + search + hash`.

Pick **one** of three approaches — don't combine an automatic one with the manual one, or
you double-report:

**A. `syncRouteToHost` prop (simplest, router-agnostic).** Patches
`history.pushState`/`replaceState` and listens for `popstate`/`hashchange`, so every route
change posts — regardless of router — and it fires an initial sync on start.

```tsx
<DocyrusAuthProvider apiUrl="…" clientId="…" syncRouteToHost>
  <App />
</DocyrusAuthProvider>
```

**B. `useDocyrusHostRouteSync()` hook.** Same behavior as the prop, scoped to where it's
mounted (typically the app shell). Safe to call once.

```tsx
import { useDocyrusHostRouteSync } from '@docyrus/signin';

function Shell() {
  useDocyrusHostRouteSync();
  return <Routes />;
}
```

**C. `notifyRouteChange(payload?)` manual.** Report from a router location subscription and
keep the router as the single source of truth (no `history` patching).

```tsx
import { useDocyrusAuth, type RouteChangePayload } from '@docyrus/signin';

const { notifyRouteChange } = useDocyrusAuth();

notifyRouteChange();                                    // no arg → reads window.location
notifyRouteChange({ path: '/customers/123', search: '?tab=notes', hash: '' }); // explicit
```

- No argument → reads `window.location` (`pathname` / `search` / `hash`).
- `payload: { path?, search?, hash? }` → each field defaults to `''`.
- No-op after teardown or outside iframe mode.

Manual reporting on a location subscription:

```tsx
function RouteReporter() {
  const { notifyRouteChange } = useDocyrusAuth();
  const location = useLocation(); // React Router / TanStack Router / etc.
  useEffect(() => {
    notifyRouteChange({ path: location.pathname, search: location.search, hash: location.hash });
  }, [location, notifyRouteChange]);
  return null;
}
```

## 4. App → Host: navigation request (`requestHostNavigation`)

The reverse of §1: an in-app action asks the **host shell** to move — deep-link the host,
switch the host view, or open a sibling app. The app posts
`{ type: 'navigation-request', url, replace?, newTab? }`; the host decides how to honour it.
Unlike `route-change` (which passively reports where the app already is), this is an
explicit request to move the host.

```tsx
import { useDocyrusRequestHostNavigation } from '@docyrus/signin';

function OpenBillingButton() {
  const requestHostNavigation = useDocyrusRequestHostNavigation();
  return <button onClick={() => requestHostNavigation('/settings/billing')}>Go to billing</button>;
}
```

With options (`HostNavigationRequestOptions`):

```tsx
const { requestHostNavigation } = useDocyrusAuth();
requestHostNavigation('/reports/q3', { replace: true });               // replace host history entry
requestHostNavigation('https://docs.docyrus.com', { newTab: true });   // open in a new tab/window
```

- `useDocyrusRequestHostNavigation()` returns a stable function; or read
  `requestHostNavigation` from `useDocyrusAuth()`.
- Empty `url` is ignored. `replace` / `newTab` are sent only when truthy.
- No-op outside iframe/WebView mode.

---

## 5. Router adapters (normalizing host navigation)

The host may send an **absolute URL or a relative path**. Resolve against
`window.location.origin`, push the relative part (so the router owns search/hash parsing),
and fall back to the raw value if it isn't parseable.

**TanStack Router**
```ts
import { useRouter } from '@tanstack/react-router';
const router = useRouter();
useDocyrusHostNavigation(({ url }) => {
  if (!url) return;
  try { const r = new URL(url, window.location.origin); router.history.push(`${r.pathname}${r.search}${r.hash}`); }
  catch { router.history.push(url); }
});
```

**React Router**
```ts
import { useNavigate } from 'react-router-dom';
const navigate = useNavigate();
useDocyrusHostNavigation(({ url }) => {
  if (!url) return;
  try { const r = new URL(url, window.location.origin); navigate(`${r.pathname}${r.search}${r.hash}`); }
  catch { navigate(url); }
});
```

**Next.js (App Router)**
```ts
import { useRouter } from 'next/navigation';
const router = useRouter();
useDocyrusHostNavigation(({ url }) => {
  if (!url) return;
  try { const r = new URL(url, window.location.origin); router.push(`${r.pathname}${r.search}${r.hash}`); }
  catch { router.push(url); }
});
```

## 6. The single `useHostBridge()` pattern

Collapse the inbound handlers into one hook and mount it **unconditionally**, near the top
of the root component, **before any auth/loading early return**, and **inside the router
provider** (so navigation has router context). All hooks are no-ops outside iframe mode, so
unconditional mounting is safe.

```tsx
import { useRouter } from '@tanstack/react-router';
import { useDocyrusHostNavigation, useDocyrusHostNotification } from '@docyrus/signin';
import { toast } from 'sonner';

export function useHostBridge() {
  const router = useRouter();

  useDocyrusHostNavigation(({ url }) => {
    if (!url) return;
    try {
      const r = new URL(url, window.location.origin);
      router.history.push(`${r.pathname}${r.search}${r.hash}`);
    } catch {
      router.history.push(url);
    }
  });

  useDocyrusHostNotification((n) => toast(n.subject, { description: n.message }));
}
```

Route reporting is enabled separately on the provider (`syncRouteToHost`) or via one of the
alternatives in §3.

## 7. Vue

The React convenience hooks (`useDocyrusHostNavigation`, `useDocyrusHostNotification`,
`useDocyrusHostRouteSync`, `useDocyrusRequestHostNavigation`) are **React-only**. In Vue,
read the same primitives from `getDocyrusAuth()` and manage the subscription lifecycle
yourself. `onHostNavigation` / `onHostNotification` return an unsubscribe function.

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue';
import { getDocyrusAuth } from '@docyrus/signin/vue';
import { useRouter } from 'vue-router';

const auth = getDocyrusAuth();
const router = useRouter();
let stopNav: (() => void) | undefined;
let stopNote: (() => void) | undefined;

onMounted(() => {
  stopNav = auth.onHostNavigation(({ url }) => router.push(url));
  stopNote = auth.onHostNotification((n) => toast.show(n.subject, n.message));
});
onUnmounted(() => { stopNav?.(); stopNote?.(); });

// Reverse direction:
function goToBilling() { auth.requestHostNavigation('/settings/billing'); }
</script>
```

For route sync in Vue, use the `syncRouteToHost` prop (router-agnostic) or call
`auth.notifyRouteChange({ path, search, hash })` from a `vue-router` `afterEach` guard.

---

## 8. Implementation task prompt (paste this)

> **Wire this embedded Docyrus app's host-shell communication using `@docyrus/signin`
> (auth is already configured; do NOT touch the Guidy bridge here).**
>
> The app runs inside the Docyrus super-app shell iframe. Implement:
>
> 1. **Host → App navigation** — apply shell navigation requests to this app's router.
> 2. **Host → App notifications** — surface pushed notifications via this repo's existing
>    toast/notification UI (`subject` → title, `message` → body).
> 3. **App → Host route reporting** — keep the shell address bar in sync with the app's route.
> 4. **App → Host navigation requests** (only if the app has actions that should move the host,
>    e.g. "open billing in the shell") — wire `requestHostNavigation`.
>
> **Requirements**
>
> - Create a single `useHostBridge()` hook calling `useDocyrusHostNavigation` and
>   `useDocyrusHostNotification`. For navigation, the host may send an **absolute URL or a
>   relative path** — resolve against `window.location.origin`, navigate to the relative
>   `pathname + search + hash` with this repo's router primitive, and fall back to the raw
>   value in a try/catch.
> - Call `useHostBridge()` **unconditionally**, near the top of the root component, **before**
>   any auth/loading early return, and **inside** the router provider. Every hook is a no-op
>   outside iframe mode, so unconditional mounting is safe.
> - Enable route reporting: add `syncRouteToHost` to `<DocyrusAuthProvider>` (simplest), or —
>   if the router must stay the single source of truth — call `notifyRouteChange` from a
>   location subscription instead. Do not do both.
> - Match the repo's existing router / toast library and code style. Run the project's
>   typecheck/build afterward to confirm it compiles.

## 9. Testing checklist

Embedded in a (local) shell, with devtools' Messages tab (or the `@docyrus/devtools` panel):

1. **Route reporting** — navigate in the app → outbound `route-change` fires; shell address bar updates.
2. **Host → app navigation** — trigger a shell deep-link → the app's router navigates.
3. **Notification** — host pushes a notification → it surfaces in the app UI.
4. **Navigation request** — trigger the in-app "move the host" action → outbound `navigation-request` fires and the host moves.
5. **Origin** — confirm messages from a non-`*.docyrus.app` origin (not in `allowedHostOrigins`) are ignored.

## 10. Gotchas

- **Iframe / WebView mode only.** Every API here is inert in standalone / RN. Test embedded.
- **Absolute vs relative nav URLs.** Always resolve host `navigation` URLs against
  `window.location.origin`; push the relative part with a raw-value fallback.
- **Don't double-report routes.** Use `syncRouteToHost` / `useDocyrusHostRouteSync` **or** a
  manual `notifyRouteChange` effect — never both.
- **Mount inside the router provider, before early returns.** Otherwise navigation has no
  router context and handlers may miss messages.
- **`route-change` vs `navigation-request`.** The first passively reports where the app is;
  the second explicitly asks the host to move. Don't use route reporting to try to drive the host.
- **No memoization required** for the React hook handlers (latest closure wins), but the
  imperative `onHostNavigation`/`onHostNotification` subscriptions returned functions must be
  cleaned up (React `useEffect` return, Vue `onUnmounted`).
- **Extra trusted hosts** go through `allowedHostOrigins`; don't loosen origin checks any other way.
- **Guidy is separate.** Host-driven pointing/clicking and Guidy route declaration are **not**
  covered here — see `guidy-integration.md`. (Guidy route jumps do arrive as the same host → app
  `navigation` message, so the `useDocyrusHostNavigation` handler above is also what makes Guidy
  navigation land.)
