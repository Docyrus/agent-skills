<!--
Guidy bridge integration for `@docyrus/signin` — making an embedded Docyrus app
navigable and drivable by the host AI assistant (Guidy). Source of truth:
`docyrus-devkit/packages/signin` (core/guidy-bridge.ts, core/auth-manager.ts,
hooks/use-host-messages.ts, types.ts). For all NON-Guidy host messaging (host
navigation, notifications, route sync, navigation requests) see
`host-communication-integration.md`. Refresh this file if the package changes.
-->

# Guidy Integration (`@docyrus/signin`)

Make an embedded Docyrus app **Guidy-compatible** — so the host AI assistant (Guidy) can
guide users *through* the app: point at, annotate, and click real in-app controls, and
navigate the app to its own pages. This is the **app side only**; the host shell already
ships the other end.

Guidy is **iframe-mode only** and a **no-op in standalone / React-Native modes**. It is
strictly **opt-in** — bumping the package version does nothing until you set
`enableGuidyBridge`.

## How it works (1-minute model)

The shell embeds apps as cross-origin `<iframe>`s. Guidy runs in the shell and cannot read
from or draw into a cross-origin iframe, so a small runtime in `@docyrus/signin`
(`GuidyBridge`) runs **inside the app** and talks to the host over the same
origin-validated `postMessage` channel used for auth.

```text
app  → host :  guidy:elements   { path, elements: [{ id, label, tag }] }  (auto-scanned from the DOM)
app  → host :  guidy:routes     { routes: [{ path, label }] }             (declared by the app)
host → app  :  guidy:scan       re-request inventory + routes
host → app  :  guidy:point      { id, click?, message? }  → scroll, highlight, annotate, (click)
app  → host :  guidy:point-ack  { id, ok }
host → app  :  navigation       { url }  → app navigates its own router (for declared routes)
```

The host namespaces in-app element ids with an `app:` prefix and turns declared routes into
`/apps/<slug>/<path>` deep links. **You never deal with the prefix or the deep-link path** —
that's host-side. Your job is the three steps below.

## Prerequisites

- The app renders `<DocyrusAuthProvider>` from `@docyrus/signin` at its root.
- Package versions:
  - Elements / point / click: **`>= 0.13.0`**.
  - **Routes** (`guidyRoutes` / `useDocyrusGuidyRoutes`): **`>= 0.14.0`**.
- The app actually runs **embedded** in the shell (iframe mode). The bridge is a no-op in
  standalone / RN — that's expected.

---

## Step 1 — Opt in (`enableGuidyBridge`, required)

Set the boolean prop. This is the entire activation; the bridge auto-starts in iframe mode,
scans the DOM, pushes the inventory, and handles host commands.

```tsx
<DocyrusAuthProvider apiUrl={apiUrl} clientId={clientId} enableGuidyBridge>
  <App />
</DocyrusAuthProvider>
```

Once running, the bridge:

- **Scans the live DOM** for `<a id>` / `<button id>` elements and pushes the inventory
  (`guidy:elements`) to the host on mount, on route change (`popstate` / `hashchange`), and
  on DOM mutation (debounced ~250 ms via a `MutationObserver` watching `id`, `hidden`,
  `aria-label`, `title`, `style`).
- **Answers `guidy:scan`** by re-posting elements + routes (recovers from a late host mount
  that missed the proactive push).
- **Executes `guidy:point`**: scrolls the target into view, highlights it (~5.2 s pulsing
  blue outline it draws itself), optionally shows an annotation bubble (`message`), and — if
  `click: true` — calls the real element's `.click()` after ~450 ms (so the highlight is seen
  first), then replies `guidy:point-ack { id, ok }`.

Without this flag the app is invisible to Guidy.

## Step 2 — Make controls targetable (required for point / click)

The scan only collects elements that are **all** of:

1. **`<a>` or `<button>`** (a `div[role=button]` is *not* scanned),
2. **visible** — non-zero box, not `display:none` / `visibility:hidden`,
3. carry a **stable `id`**, and
4. expose an **accessible label** — `aria-label` → trimmed text content → `title`
   (first non-empty wins).

Ids beginning with `guidy-` are reserved for the host and skipped; duplicate ids are
de-duped (first wins).

```tsx
<button id="new-lead">New Lead</button>                                   {/* ✅ id + text label */}
<button id="save-record" aria-label="Save record"><SaveIcon /></button>   {/* ✅ icon-only needs aria-label */}
<a id="reports-link" href="/reports">Reports</a>                          {/* ✅ anchor with label */}
<button>New Lead</button>                                                 {/* ❌ no id → invisible */}
<div id="new-lead" role="button" onClick={…}>New Lead</div>               {/* ❌ not an <a>/<button> */}
```

Guidelines:

- **Ids must be stable across renders** — the model references a control by id between
  turns; a regenerated/random id breaks targeting.
- **Ids must be unique** — duplicates are ambiguous (first wins).
- Use real `<button>` / `<a>` for primary actions; render custom controls *as* (or wrap the
  action in) a `button`/`a`, or they won't be seen.
- **Labels should read like what the user sees** ("New Lead", "Convert to Deal"), so Guidy
  can map the user's words to the right control.
- Dynamically-rendered controls need no extra wiring — the re-scan picks them up; just give
  them an id + label.

Nothing to wire for clicking: `guidy:point { click: true }` calls the real element's
`.click()`, so your normal click handler fires.

## Step 3 — Declare navigable routes

Elements are auto-discovered; **routes cannot be**, so the app declares which pages Guidy
may send the user to. `path` is **app-internal** (your router's path); the host builds the
deep link. Two ways:

**Static — `guidyRoutes` prop** (routes known up front):

```tsx
<DocyrusAuthProvider
  apiUrl={apiUrl}
  clientId={clientId}
  enableGuidyBridge
  guidyRoutes={[
    { path: '/leads',     label: 'Leads' },
    { path: '/leads/new', label: 'New Lead' },
    { path: '/deals',     label: 'Deals' },
    { path: '/reports',   label: 'Reports' },
  ]}
>
  <App />
</DocyrusAuthProvider>
```

**Dynamic — `useDocyrusGuidyRoutes` hook** (routes depend on permissions / runtime state):

```tsx
import { useDocyrusGuidyRoutes, useDocyrusAuth } from '@docyrus/signin';
import { useMemo } from 'react';

function GuidyRouteRegistrar() {
  const { hasPermission } = useDocyrusAuth();

  const routes = useMemo(() => {
    const list = [{ path: '/leads', label: 'Leads' }];
    if (hasPermission('view', REPORTS_DS_ID)) list.push({ path: '/reports', label: 'Reports' });
    return list;
  }, [hasPermission]);

  useDocyrusGuidyRoutes(routes); // re-posts guidy:routes whenever the array identity changes
  return null;
}
```

- Type: `GuidyRoute[]` where `GuidyRoute = { path: string; label: string }`.
- **Both `guidyRoutes` and `useDocyrusGuidyRoutes` require `enableGuidyBridge`** — they do
  nothing on their own.
- The hook replaces the whole list on each change (not a merge) and re-syncs whenever the
  **array identity** changes — so **`useMemo` the array** (or keep it referentially stable),
  or it re-posts every render. You can seed the initial list with the prop and update it with
  the hook, or use the hook alone.
- Imperative equivalent (rarely needed): `setGuidyRoutes(routes)` from `useDocyrusAuth()`.

### Routes also require inbound host-navigation handling

Driving a declared route sends a **host → app `navigation` message** — so declaring routes
without applying that message to your router makes them *appear* in Guidy but *not*
navigate. Wire `useDocyrusHostNavigation` (documented in full in
`host-communication-integration.md`):

```tsx
import { useDocyrusHostNavigation } from '@docyrus/signin';
import { useNavigate } from 'react-router-dom';

function HostNavBridge() {
  const navigate = useNavigate();
  useDocyrusHostNavigation(({ url }) => navigate(url)); // normalize absolute/relative — see host-communication doc
  return null;
}
```

If the shell can already deep-link into your app on load (or you use `syncRouteToHost` with
existing deep-link handling), this is effectively wired and routes will work.

### Only declare concrete, statically-navigable paths

**Never declare parameterized / templated routes** — `/leads/$leadId`, `/leads/:id`,
`/deals/*`. The model has no real id to fill and would navigate to a literal broken URL.
Declare the index page and let Guidy reach a specific record by **clicking** its row/link
(an element target), not by route-jumping.

```tsx
guidyRoutes={[
  { path: '/leads',     label: 'Leads' },     // ✅ concrete
  { path: '/leads/new', label: 'New Lead' },  // ✅ concrete
  // { path: '/leads/$leadId', label: 'Lead' } ❌ skip — no real id to fill
]}
```

If a specific dynamic destination is genuinely needed ("open lead 42"), expose it as a real
concrete route at the moment it's relevant via `useDocyrusGuidyRoutes` — never as a
templated path.

---

## What you do NOT need to do

- **No message handling, highlight rendering, or overlay code** — the bridge does scanning,
  the pulsing highlight, the annotation bubble, and the click.
- **No `app:` prefixing or `/apps/<slug>/…` path building** — host-side.
- **No manual "push inventory" calls** — it auto-posts on mount, route change, and DOM
  mutation, and answers `guidy:scan`.

## Security posture

Intentionally narrow and **do not try to widen it per-app**:

- Only **already-visible** `<a>` / `<button>` elements are exposed (labels + ids only).
- Only a **fixed command vocabulary** runs — scan / point / click. **No arbitrary script is
  ever accepted from the host.**
- All traffic rides the same **origin-validated** `postMessage` channel as auth.
- Cross-origin is by design: the app draws its own highlight/bubble; the host never reaches
  into the iframe.
- The bridge stops automatically on sign-out and on provider teardown.

## Minimal complete example

```tsx
import { DocyrusAuthProvider, useDocyrusHostNavigation } from '@docyrus/signin';
import { useNavigate } from 'react-router-dom';

function HostNavBridge() {
  const navigate = useNavigate();
  useDocyrusHostNavigation(({ url }) => navigate(url));
  return null;
}

export function Root() {
  return (
    <DocyrusAuthProvider
      apiUrl={import.meta.env.VITE_DOCYRUS_API_BASE_URL}
      clientId={import.meta.env.VITE_DOCYRUS_CLIENT_ID}
      enableGuidyBridge
      guidyRoutes={[
        { path: '/leads',     label: 'Leads' },
        { path: '/leads/new', label: 'New Lead' },
        { path: '/deals',     label: 'Deals' },
      ]}
    >
      <HostNavBridge />
      <App /> {/* buttons/links carry stable ids + labels */}
    </DocyrusAuthProvider>
  );
}
```

That's the complete app-side surface to be fully Guidy-compatible: opt in, give controls
ids + labels, declare routes, and handle host navigation.

## Implementation task prompt (paste this)

> **Make this embedded Docyrus app Guidy-compatible using `@docyrus/signin`.**
>
> The app runs inside the Docyrus super-app shell iframe. Do all of:
>
> 1. **Opt in** — set `enableGuidyBridge` on `<DocyrusAuthProvider>`.
> 2. **Targetable controls** — give every control Guidy should reach a **stable, unique `id`**
>    and an accessible **label** (`aria-label`, text, or `title`). Use real `<button>` / `<a>`
>    elements (not `div[role=button]`).
> 3. **Declare routes** — pass `guidyRoutes` for statically-known pages, or use
>    `useDocyrusGuidyRoutes(routes)` with a `useMemo`'d array if the set is permission-dependent.
>    Declare **concrete paths only** — never parameterized ones like `/leads/$leadId`; use the
>    index page + element clicks to reach records.
> 4. **Handle inbound navigation** — wire `useDocyrusHostNavigation` to this app's router so
>    declared route jumps actually navigate (resolve absolute/relative URLs against
>    `window.location.origin`). If the app already handles shell deep-links, this may be done.
>
> Do not write any message-handling, highlight, or overlay code — the bridge handles it. Match
> the repo's router / code style and run typecheck/build afterward.

## Testing checklist

Embedded in a (local) shell that has the Guidy host changes, devtools open:

1. **Opt-in** — on load an outbound `guidy:elements` fires (and `guidy:routes` if declared).
2. **Elements** — ask Guidy "show me where I create a lead" → the New Lead control highlights
   with an annotation bubble.
3. **Click** — ask Guidy "create a new lead for me" → the control highlights, then actually
   clicks (your handler runs).
4. **Re-scan** — navigate within the app → a fresh `guidy:elements` posts; newly visible
   controls become targetable.
5. **Navigation** — ask Guidy "go to Reports" → the app's router navigates to the declared
   route (shell address bar shows `/apps/<slug>/reports`).
6. **Dynamic routes** — flip a permission → `useDocyrusGuidyRoutes` re-posts an updated
   `guidy:routes`.
7. **Negative** — a control with no id/label is *not* offered by Guidy (expected).

## Gotchas & rules

- **Opt-in only.** Without `enableGuidyBridge` nothing happens — updating the package isn't enough.
- **Iframe mode only.** Standalone / RN are no-ops; test embedded.
- **`guidyRoutes` / `useDocyrusGuidyRoutes` require `enableGuidyBridge`** — no bridge, no effect.
- **Routes need both** the declaration *and* an inbound-navigation handler
  (`useDocyrusHostNavigation`); declaring routes alone makes them appear but not navigate.
- **Stable, unique ids + meaningful labels** are the whole element contract — most "Guidy
  can't find it" issues are a missing id, a missing label, or a non-`button`/`a` element.
- **Memoize dynamic routes** passed to `useDocyrusGuidyRoutes`, or it re-posts every render.
- **No parameterized routes.** Declare only concrete paths; use the index page + an in-app
  click to reach a record.
- **Cross-origin is by design** — the app draws its own highlight/bubble; don't try to widen
  the exposed surface per-app.
- **Non-Guidy host messaging** (notifications, route→host sync, app→host navigation requests)
  lives in `host-communication-integration.md`.
