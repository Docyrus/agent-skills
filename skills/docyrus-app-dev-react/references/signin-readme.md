<!--
Mirror of the published `@docyrus/signin` package README (npm: `@docyrus/signin`,
source: `docyrus-devkit/packages/signin`). Kept here as the authoritative auth
reference for this skill. If the package README changes, refresh this file.
-->

# @docyrus/signin

Authentication provider for Docyrus apps. Provides "Sign in with Docyrus" experience with automatic environment detection — standalone apps use OAuth2 Authorization Code + PKCE via page redirect, while iframe-embedded apps receive tokens via `window.postMessage` from the host.

Supports **React**, **Vue**, **React Native**, **Electron**, and **Next.js SSR** via separate entrypoints.

## Features

- **Auto Environment Detection**: Detects standalone vs iframe mode automatically
- **OAuth2 PKCE**: Full Authorization Code flow with PKCE (S256) for standalone apps
- **Iframe Auth**: Receives tokens via `postMessage` from `*.docyrus.app` hosts
- **Host Message Hooks**: Subscribe to `navigation` and `notification` messages pushed by the host into the embedded app
- **Route Sync**: Mirror the embedded app's route to the host so the host can reflect it in its address bar
- **Token Auto-Refresh**: Proactive token refresh before expiry in both modes (opt out with `autoRefresh: false` when an external system owns the token lifecycle)
- **Token Bootstrap**: Start a session from existing access/refresh tokens (e.g. device flow handoff)
- **React Hooks**: `useDocyrusAuth()` and `useDocyrusClient()` for easy integration
- **Vue Composables**: `useDocyrusAuth()` and `useDocyrusClient()` for Vue 3
- **Authorization Helpers**: `hasRole()` and `hasPermission()` for role-based access control
- **Electron Support**: `ElectronTokenStorage` for IPC-based token storage + `getAuthorizationUrl()` for external browser login
- **Ready-to-Use API Client**: Exposes a configured `RestApiClient` from `@docyrus/api-client`
- **SignInButton**: Unstyled, customizable sign-in button component (React + Vue)
- **SSR Support**: Sync access token to cookie for server-side rendering (Next.js) with `max-age` and `Secure` flag
- **Next.js SSR Helpers**: `createServerClient()`, `getSession()`, `authMiddleware()` for server components, actions, and middleware
- **TypeScript**: Full type definitions included

## Installation

```bash
npm install @docyrus/signin
```

```bash
pnpm add @docyrus/signin
```

### Peer Dependencies

- `@docyrus/api-client` >= 0.0.4
- `react` >= 19.2.0 (optional — required for React/Next.js entrypoint)
- `vue` >= 3.5.0 (optional — required for Vue entrypoint)

## Entrypoints

| Import Path | Description | Framework |
|-------------|-------------|-----------|
| `@docyrus/signin` | React provider, hooks, components | React, Next.js |
| `@docyrus/signin/nextjs` | Server-side helpers (middleware, server client, session) | Next.js |
| `@docyrus/signin/vue` | Vue provider, composables, components | Vue 3 |
| `@docyrus/signin/electron` | Electron token storage | Electron |
| `@docyrus/signin/react-native` | React Native OAuth2 auth (in-app browser) | React Native / Expo |
| `@docyrus/signin/core` | Pure TypeScript core (no framework dependency) | Any |

## Quick Start (React)

```tsx
import { DocyrusAuthProvider, useDocyrusAuth, useDocyrusClient, SignInButton } from '@docyrus/signin';

function App() {
  return (
    <DocyrusAuthProvider
      apiUrl="https://alpha-api.docyrus.com"
      clientId="your-oauth2-client-id"
      redirectUri="http://localhost:3000/callback"
      scopes={['offline_access', 'Read.All', 'Users.Read']}
      callbackPath="/callback">
      <Dashboard />
    </DocyrusAuthProvider>
  );
}

function Dashboard() {
  const { status, signOut } = useDocyrusAuth();
  const client = useDocyrusClient();

  if (status === 'loading') return <div>Loading...</div>;
  if (status === 'unauthenticated') return <SignInButton />;

  return (
    <div>
      <p>Authenticated!</p>
      <button onClick={() => client!.get('/v1/users/me').then(console.log)}>
        Fetch user
      </button>
      <button onClick={signOut}>Sign out</button>
    </div>
  );
}
```

## Quick Start (Vue)

```vue
<!-- App.vue -->
<script setup lang="ts">
import { RouterView } from 'vue-router';
import { DocyrusAuthProvider } from '@docyrus/signin/vue';
</script>

<template>
  <DocyrusAuthProvider
    apiUrl="https://alpha-api.docyrus.com"
    clientId="your-oauth2-client-id"
    redirectUri="http://localhost:5173/callback"
    :scopes="['offline_access', 'Read.All', 'Users.Read']"
    callbackPath="/callback">
    <RouterView />
  </DocyrusAuthProvider>
</template>
```

```vue
<!-- Dashboard.vue -->
<script setup lang="ts">
import { useDocyrusAuth } from '@docyrus/signin/vue';

const { status, client, signIn, signOut } = useDocyrusAuth();
</script>

<template>
  <div v-if="status === 'loading'">Loading...</div>
  <button v-else-if="status === 'unauthenticated'" @click="signIn">Sign in</button>
  <div v-else>
    <p>Authenticated!</p>
    <button @click="signOut">Sign out</button>
  </div>
</template>
```

## Quick Start (Electron)

```tsx
import { DocyrusAuthProvider, useDocyrusAuth } from '@docyrus/signin';
import { ElectronTokenStorage } from '@docyrus/signin/electron';

const tokenStorage = new ElectronTokenStorage();

function App() {
  return (
    <DocyrusAuthProvider
      apiUrl="https://alpha-api.docyrus.com"
      clientId="your-oauth2-client-id"
      redirectUri="docyrus://callback"
      scopes={['offline_access', 'Read.All', 'Users.Read']}
      callbackPath="/callback"
      tokenStorage={tokenStorage}>
      <LoginPage />
    </DocyrusAuthProvider>
  );
}

function LoginPage() {
  const { status, getAuthorizationUrl } = useDocyrusAuth();

  const handleLogin = async () => {
    const url = await getAuthorizationUrl();
    if (url) {
      // Open in system browser instead of navigating
      await window.electronAPI.openExternal(url);
    }
  };

  return <button onClick={handleLogin} disabled={status === 'loading'}>Sign in</button>;
}
```

## Quick Start (React Native / Expo)

```tsx
import { DocyrusAuthProvider, useDocyrusAuth } from '@docyrus/signin/react-native';
import { createTokenStorage, ReactNativeCryptoProvider } from '@docyrus/api-client/react-native';
import * as WebBrowser from 'expo-web-browser';
import * as ExpoCrypto from 'expo-crypto';
import * as SecureStore from 'expo-secure-store';

const tokenStorage = createTokenStorage({
  getItem: (key) => SecureStore.getItemAsync(key),
  setItem: (key, value) => SecureStore.setItemAsync(key, value),
  removeItem: (key) => SecureStore.deleteItemAsync(key),
});

function App() {
  return (
    <DocyrusAuthProvider
      clientId="your-oauth2-client-id"
      forceMode="react-native"
      nativeRedirectUri="myapp://auth/callback"
      openAuthSession={WebBrowser.openAuthSessionAsync}
      tokenStorage={tokenStorage}
      cryptoProvider={new ReactNativeCryptoProvider(ExpoCrypto)}>
      <HomeScreen />
    </DocyrusAuthProvider>
  );
}

function HomeScreen() {
  const { status, signIn, signOut } = useDocyrusAuth();
  const client = useDocyrusClient();

  if (status === 'loading') return <Text>Loading...</Text>;
  if (status === 'unauthenticated') return <Button title="Sign In" onPress={signIn} />;

  return (
    <View>
      <Text>Authenticated!</Text>
      <Button title="Sign Out" onPress={signOut} />
    </View>
  );
}
```

### React Native Props

| Prop | Type | Description |
|------|------|-------------|
| `forceMode` | `'react-native'` | **Required** — enables mobile OAuth2 flow |
| `nativeRedirectUri` | `string` | **Required** — deep link URI scheme (e.g., `myapp://auth/callback`) |
| `openAuthSession` | `(url, redirectUri) => Promise<{type, url?}>` | **Required** — `WebBrowser.openAuthSessionAsync` from expo-web-browser |
| `tokenStorage` | `OAuth2TokenStorage` | Token storage (use `createTokenStorage()` from api-client) |
| `cryptoProvider` | `CryptoProvider` | PKCE crypto (use `ReactNativeCryptoProvider` from api-client) |

## Configuration

Pass props to `DocyrusAuthProvider` to override defaults:

```tsx
<DocyrusAuthProvider
  apiUrl="https://alpha-api.docyrus.com"
  clientId="your-oauth2-client-id"
  redirectUri="http://localhost:3000/callback"
  scopes={['offline_access', 'Read.All', 'Users.Read']}
  callbackPath="/callback"
  forceMode="standalone"
>
  <App />
</DocyrusAuthProvider>
```

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `apiUrl` | `string` | `https://alpha-api.docyrus.com` | Docyrus API base URL |
| `clientId` | `string` | **Required** | OAuth2 client ID (throws if missing) |
| `redirectUri` | `string` | `window.location.origin + callbackPath` | OAuth2 redirect URI |
| `scopes` | `string[]` | `['offline_access', 'Read.All', ...]` | OAuth2 scopes |
| `callbackPath` | `string` | `/auth/callback` | Path to detect OAuth callback |
| `forceMode` | `'standalone' \| 'iframe' \| 'react-native'` | Auto-detected | Force a specific auth mode |
| `nativeRedirectUri` | `string` | — | Deep link URI for React Native OAuth2 callback |
| `openAuthSession` | `Function` | — | `WebBrowser.openAuthSessionAsync` for React Native |
| `cryptoProvider` | `CryptoProvider` | Browser crypto | Custom crypto for PKCE (React Native) |
| `storageKeyPrefix` | `string` | `docyrus_oauth2_` | localStorage key prefix |
| `tokenStorage` | `OAuth2TokenStorage` | Browser localStorage | Custom token storage (e.g., `ElectronTokenStorage`) |
| `initialTokens` | `OAuth2Tokens` | `undefined` | Bootstrap a standalone / React Native session from existing tokens |
| `autoRefresh` | `boolean` | `true` | When `false`, the provider never refreshes tokens itself (no pre-expiry timer, no pre-request refresh, no reactive 401 refresh). The current access token is used until replaced via `signInWithTokens`. Use when an external system owns the token lifecycle. |
| `allowedHostOrigins` | `string[]` | `undefined` | Extra trusted iframe origins |
| `ssr` | `boolean` | `false` | Sync access token to cookie for server components |
| `ssrCookieKey` | `string` | `docyrus-token` | Cookie name for SSR token sync |
| `syncRouteToHost` | `boolean` | `false` | Iframe mode: post `route-change` to the host on every history change |
| `enableGuidyBridge` | `boolean` | `false` | Iframe mode: let the host assistant (Guidy) scan, highlight, and click this app's visible UI (see below) |
| `guidyRoutes` | `GuidyRoute[]` | `undefined` | Iframe mode: routes Guidy may navigate the user to inside this app (see below). Requires `enableGuidyBridge` |

## Guidy Bridge (host-driven guidance)

When an app runs embedded in the Docyrus shell, the host AI assistant (Guidy)
can guide users _through the app_ — pointing at and clicking real in-app
buttons — once the app opts in:

```tsx
<DocyrusAuthProvider apiUrl="..." clientId="..." enableGuidyBridge>
  <App />
</DocyrusAuthProvider>
```

This starts a small runtime (`GuidyBridge`) that, in iframe mode only:

- pushes the app's clickable inventory (`a[id]` / `button[id]` that are visible
  and labeled) to the host on mount, route change, and DOM mutation;
- on a host `guidy:point`, scrolls to, highlights, optionally annotates, and
  optionally clicks the target element, then acknowledges.

It is opt-in and intentionally narrow: it only exposes already-visible
interactive elements and only runs a fixed command vocabulary — **no arbitrary
script is accepted from the host**. All traffic rides the same origin-validated
postMessage channel as auth.

```text
App  → Host: { type: 'guidy:elements', path, elements: [{ id, label, tag }] }
App  → Host: { type: 'guidy:routes', routes: [{ path, label }] }
Host → App:  { type: 'guidy:scan' }                       // re-request inventory + routes
Host → App:  { type: 'guidy:point', id, click?, message? } // point / click / annotate
App  → Host: { type: 'guidy:point-ack', id, ok }
```

Give the elements Guidy should reach a stable `id` and an accessible label
(`aria-label`, text content, or `title`).

### In-app navigation (routes)

Elements are auto-discovered from the DOM; routes are not, so the app declares
which pages Guidy may send the user to. The host merges them into Guidy's
navigation tree as deep links and drives them through the existing host→app
`navigation` message. Declare them statically:

```tsx
<DocyrusAuthProvider
  apiUrl="..."
  clientId="..."
  enableGuidyBridge
  guidyRoutes={[
    { path: "/leads", label: "Leads" },
    { path: "/leads/new", label: "New Lead" }
  ]}
>
  <App />
</DocyrusAuthProvider>
```

…or dynamically (e.g. when routes depend on permissions) with the hook:

```tsx
import { useDocyrusGuidyRoutes } from "@docyrus/signin"

useDocyrusGuidyRoutes(visibleRoutes) // re-syncs whenever the array changes
```

Driving a route requires the app to handle inbound host navigation — wire
`useDocyrusHostNavigation` (or `syncRouteToHost`) so the deep link reaches your
router. `path` is app-internal (e.g. `/leads`); the host prefixes it with the
app's mount path.

## Hooks / Composables

### useDocyrusAuth()

Returns the full authentication context. Available in both React and Vue (`@docyrus/signin/vue`).

```tsx
const {
  status,              // 'loading' | 'authenticated' | 'unauthenticated'
  mode,                // 'standalone' | 'iframe'
  client,              // RestApiClient | null
  tokens,              // { accessToken, refreshToken, ... } | null
  user,                // DocyrusUser | null (auto-fetched from /v1/users/me)
  signIn,              // () => void (redirects to Docyrus login)
  signInWithTokens,    // (tokens) => Promise<void> (bootstrap from device flow / external auth)
  getAuthorizationUrl, // () => Promise<string | null> (returns URL without navigating — for Electron)
  signOut,             // () => Promise<void>
  hasRole,             // (role) => boolean — check if user has a role by slug or uid
  hasPermission,       // (operation, dataSourceId?) => boolean — check ACL permission
  refreshUser,         // () => Promise<void> — re-fetch user from API
  onHostNavigation,    // (handler) => () => void — subscribe to host navigation messages (iframe mode)
  onHostNotification,  // (handler) => () => void — subscribe to host notification messages (iframe mode)
  enableHostRouteSync, // () => void — patch history + fire `route-change` on every change (iframe mode)
  notifyRouteChange,   // (payload?) => void — post a single `route-change` to the host (iframe mode)
  requestHostNavigation, // (url, options?) => void — ask the host shell to navigate (iframe mode)
  error                // Error | null
} = useDocyrusAuth();
```

### useDocyrusHostNavigation() / useDocyrusHostNotification()

React-only convenience hooks for subscribing to host postMessage events
from within an embedded iframe/WebView. Both subscribe on mount and
unsubscribe on unmount; the latest closure is always invoked so the
handler does not need to be memoized. No-op outside iframe mode.

```tsx
import {
  useDocyrusHostNavigation,
  useDocyrusHostNotification,
  type DocyrusNotification
} from '@docyrus/signin';

useDocyrusHostNavigation(({ url }) => {
  navigate(url);
});

useDocyrusHostNotification((notification: DocyrusNotification) => {
  showToast(notification.subject, notification.message);
});
```

### useDocyrusClient()

Shorthand hook/composable that returns just the API client:

```tsx
const client = useDocyrusClient(); // RestApiClient | null

if (client) {
  const response = await client.get('/v1/users/me');
}
```

## Components

### SignInButton

Unstyled button that triggers sign-in. Automatically hidden when authenticated or in iframe mode. Available in both React and Vue.

**React:**
```tsx
// Basic
<SignInButton />

// With custom styling
<SignInButton className="btn btn-primary" label="Log in with Docyrus" />

// Render prop for full customization
<SignInButton>
  {({ signIn, isLoading }) => (
    <button onClick={signIn} disabled={isLoading}>
      {isLoading ? 'Redirecting...' : 'Sign in'}
    </button>
  )}
</SignInButton>
```

**Vue:**
```vue
<!-- Basic -->
<SignInButton />

<!-- With custom styling -->
<SignInButton label="Login" class="btn-primary" />

<!-- Scoped slot for custom rendering -->
<SignInButton v-slot="{ signIn, status }">
  <MyCustomButton @click="signIn" :loading="status === 'loading'" />
</SignInButton>
```

## Electron Token Storage

`ElectronTokenStorage` stores OAuth2 tokens via IPC bridge to the Electron main process. Falls back to localStorage if the Electron API is not available.

```tsx
import { ElectronTokenStorage } from '@docyrus/signin/electron';

// Auto-detects window.electronAPI
const storage = new ElectronTokenStorage();

// Or pass a custom IPC bridge
const storage = new ElectronTokenStorage(myElectronAPI);

// Custom key prefix
const storage = new ElectronTokenStorage(undefined, 'my_app_oauth2');
```

The Electron preload script must expose `storeGet`, `storeSet`, and `storeDelete` IPC methods via `contextBridge`.

## Bootstrap From Existing Tokens

If you already have Docyrus `accessToken` and `refreshToken` from another flow such as device authorization, you can start a web session without redirecting through the browser login flow again.

The simplest option is `initialTokens`:

```tsx
<DocyrusAuthProvider
  apiUrl="https://alpha-api.docyrus.com"
  clientId="your-oauth2-client-id"
  initialTokens={deviceFlowTokens}
  forceMode="standalone">
  <App />
</DocyrusAuthProvider>
```

Or bootstrap later with `signInWithTokens()`:

```tsx
import { useEffect } from 'react';
import { useDocyrusAuth } from '@docyrus/signin';

function TokenHandoff({ deviceFlowTokens }) {
  const { signInWithTokens } = useDocyrusAuth();

  useEffect(() => {
    if (!deviceFlowTokens) return;
    signInWithTokens(deviceFlowTokens).catch(console.error);
  }, [deviceFlowTokens, signInWithTokens]);

  return null;
}
```

`@docyrus/signin` will persist those tokens into its configured storage, create the authenticated API client, and continue normal refresh-token handling. In iframe mode this bootstrap path is intentionally unsupported; use the host `postMessage` signin flow instead.

## Auth Modes

### Standalone Mode (OAuth2 PKCE)

Used when the app runs directly in the browser (not in an iframe). The flow:

1. User clicks sign-in
2. Page redirects to Docyrus authorization endpoint
3. After login, redirects back with authorization code
4. Provider automatically exchanges code for tokens
5. Tokens stored in localStorage (or custom `tokenStorage`), auto-refreshed before expiry

### Iframe Mode (postMessage)

Used when the app is embedded in an iframe on `*.docyrus.app`. The flow:

1. Provider detects iframe environment and validates host origin
2. Host sends `{ type: 'signin', accessToken, refreshToken }` via `postMessage`
3. Provider creates API client with received tokens
4. When tokens expire, provider sends `{ type: 'token-refresh-request' }` to host
5. Host responds with fresh tokens

In addition to auth messages, the host may push two side-channel messages
that the embedded app can subscribe to:

| Message | Payload | Purpose |
|---------|---------|---------|
| `{ type: 'navigation', url }` | `url: string` | Host asks the app to navigate (e.g. deep-link from a host menu) |
| `{ type: 'notification', notification }` | `DocyrusNotification` | Host pushes a notification to the embedded app |

**React** — register handlers with the provided hooks (latest closure is
always invoked, no need to memoize):

```tsx
import {
  useDocyrusHostNavigation,
  useDocyrusHostNotification
} from '@docyrus/signin';

function Shell() {
  const navigate = useNavigate();

  useDocyrusHostNavigation(({ url }) => {
    navigate(url);
  });

  useDocyrusHostNotification((notification) => {
    toast({
      title: notification.subject,
      description: notification.message
    });
  });

  return <Routes />;
}
```

**Vue** — register handlers via the composable; both return an
unsubscribe function:

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue';
import { getDocyrusAuth } from '@docyrus/signin/vue';

const auth = getDocyrusAuth();
let stopNav: (() => void) | undefined;
let stopNote: (() => void) | undefined;

onMounted(() => {
  stopNav = auth.onHostNavigation(({ url }) => {
    router.push(url);
  });
  stopNote = auth.onHostNotification((notification) => {
    toast.show(notification.subject, notification.message);
  });
});

onUnmounted(() => {
  stopNav?.();
  stopNote?.();
});
</script>
```

Both subscriptions are no-ops outside iframe/WebView mode, so handlers
are safe to register unconditionally.

#### Route → Host sync (address bar reflection)

When the embedded app is inside a host shell, the shell often wants its
own browser address bar to reflect the app's current route (so users can
copy/share/bookmark URLs). The embedded app reports route changes with:

```
App → Host: { type: 'route-change', path, search, hash, url }
```

The simplest setup — pass `syncRouteToHost` to the provider. It patches
`history.pushState/replaceState` and listens for `popstate`/`hashchange`,
so every route change in the embedded app — regardless of router
(React Router, TanStack Router, Next, plain history) — posts a message:

```tsx
<DocyrusAuthProvider apiUrl="..." clientId="..." syncRouteToHost>
  <App />
</DocyrusAuthProvider>
```

Or opt in scoped to a component with the hook:

```tsx
import { useDocyrusHostRouteSync } from '@docyrus/signin';

function Shell() {
  useDocyrusHostRouteSync();
  return <Routes />;
}
```

Or report routes manually from a router subscription (avoids history
patching, lets the router be the source of truth):

```tsx
import { useDocyrusAuth } from '@docyrus/signin';

function RouteReporter() {
  const { notifyRouteChange } = useDocyrusAuth();
  const location = useLocation(); // React Router / TanStack Router / etc.

  useEffect(() => {
    notifyRouteChange({
      path: location.pathname,
      search: location.search,
      hash: location.hash
    });
  }, [location, notifyRouteChange]);

  return null;
}
```

All three are no-ops outside iframe/WebView mode.

#### App → Host navigation request

The host can ask the app to navigate (`{ type: 'navigation' }`, above). The
reverse direction lets the embedded app ask the **host shell** to navigate —
for example, a button inside the app that should deep-link the host, switch the
host view, or open a sibling app. The app posts:

```
App → Host: { type: 'navigation-request', url, replace?, newTab? }
```

The host decides how to honour the request (push/replace its own router, open a
new tab, etc.). Unlike `route-change` — which passively reports where the app
already is — this is an explicit request to move the host.

**React** — call `requestHostNavigation` from `useDocyrusAuth()`, or use the
`useDocyrusRequestHostNavigation` convenience hook:

```tsx
import { useDocyrusRequestHostNavigation } from '@docyrus/signin';

function OpenSettingsButton() {
  const requestHostNavigation = useDocyrusRequestHostNavigation();

  return (
    <button onClick={() => requestHostNavigation('/settings/billing')}>
      Go to billing
    </button>
  );
}
```

```tsx
// With options
const { requestHostNavigation } = useDocyrusAuth();

requestHostNavigation('/reports/q3', { replace: true });
requestHostNavigation('https://docs.docyrus.com', { newTab: true });
```

**Vue** — call `requestHostNavigation` from the composable:

```vue
<script setup lang="ts">
import { getDocyrusAuth } from '@docyrus/signin/vue';

const auth = getDocyrusAuth();

function goToBilling() {
  auth.requestHostNavigation('/settings/billing');
}
</script>
```

No-op outside iframe/WebView mode.

## SSR Support (Next.js)

Enable `ssr` on the client-side provider to sync the access token to a cookie. The cookie is written with `max-age` (matching the token lifetime) and `Secure` flag (on HTTPS origins) automatically.

### 1. Client — Enable SSR in AuthProvider

```tsx
<DocyrusAuthProvider
  apiUrl="https://alpha-api.docyrus.com"
  clientId="your-oauth2-client-id"
  scopes={['offline_access', 'Read.All', 'Users.Read']}
  callbackPath="/auth/callback"
  ssr>
  {children}
</DocyrusAuthProvider>
```

The cookie is automatically **set** on login and token refresh, **cleared** on sign out, and **expires** with the token (via `max-age`).

### 2. Middleware — Route Protection

Use the pre-built `authMiddleware` for automatic route protection:

```ts
// middleware.ts
import { authMiddleware } from '@docyrus/signin/nextjs';

export default authMiddleware({
  publicRoutes: ['/login', '/callback'],
  loginPath: '/login',
});

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\..*).*)'],
};
```

Or use `getMiddlewareSession` for custom middleware logic:

```ts
// middleware.ts
import { NextResponse, type NextRequest } from 'next/server';
import { getMiddlewareSession } from '@docyrus/signin/nextjs';

export function middleware(request: NextRequest) {
  const session = getMiddlewareSession(request);
  if (!session.isAuthenticated && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  return NextResponse.next();
}
```

### 3. Server Components — Authenticated API Calls

```tsx
import { cookies } from 'next/headers';
import { createServerClient, getSession } from '@docyrus/signin/nextjs';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const cookieStore = await cookies();
  const session = getSession(cookieStore);
  if (!session.isAuthenticated) redirect('/login');

  const client = createServerClient(cookieStore);
  const { data: user } = await client.get('/v1/users/me');
  return <div>Welcome, {user.name}</div>;
}
```

### 4. Server Actions

```ts
'use server';
import { cookies } from 'next/headers';
import { createServerClient } from '@docyrus/signin/nextjs';

export async function getUser() {
  const client = createServerClient(await cookies());
  return client.get('/v1/users/me');
}
```

### Next.js SSR API Reference

| Function | Import | Usage |
|----------|--------|-------|
| `getSession(cookies)` | `@docyrus/signin/nextjs` | Read auth session in server components/actions |
| `createServerClient(cookies)` | `@docyrus/signin/nextjs` | Create authenticated `RestApiClient` on server |
| `authMiddleware(config)` | `@docyrus/signin/nextjs` | Pre-built route protection middleware |
| `getMiddlewareSession(request)` | `@docyrus/signin/nextjs` | Read auth session in custom middleware |
| `createMiddlewareClient(request)` | `@docyrus/signin/nextjs` | Create authenticated client in middleware |

### Cookie Behavior

| Attribute | Value | Reason |
|-----------|-------|--------|
| `path` | `/` | Available to all routes |
| `SameSite` | `Lax` | Sent on same-site navigations |
| `Secure` | Auto (`https:` only) | Only sent over HTTPS in production |
| `max-age` | Token lifetime (seconds) | Expires with the token, survives browser restart |
| `httpOnly` | No | Client-side auth provider must read/write it |

### Cookie Helper (Client-Side)

```tsx
import { getTokenFromCookie } from '@docyrus/signin';

const token = getTokenFromCookie(); // reads 'docyrus-token' cookie
const token = getTokenFromCookie('custom-key'); // custom cookie name
```

## Authorization (Roles & Permissions)

The provider auto-fetches the current user from `/v1/users/me` after authentication and exposes `hasRole` and `hasPermission` helpers.

### Via Hooks (React / React Native)

```tsx
function DataSourceView({ dataSourceId }: { dataSourceId: string }) {
  const { user, hasRole, hasPermission } = useDocyrusAuth();

  if (!user) return <div>Loading user...</div>;

  // Check role by slug or uid
  if (hasRole('super_admin')) {
    return <AdminPanel />;
  }

  // Check data source permission
  const canEdit = hasPermission('edit', dataSourceId);
  const canDelete = hasPermission('delete', dataSourceId);

  // Check multiple roles
  if (hasRole(['editor', 'reviewer'])) {
    return <EditorView canEdit={canEdit} canDelete={canDelete} />;
  }

  return <ReadOnlyView />;
}
```

### Via Pure Functions (Framework-Agnostic)

```tsx
import { hasRole, hasPermission } from '@docyrus/signin/core';
import type { DocyrusUser } from '@docyrus/signin/core';

function checkAccess(user: DocyrusUser) {
  // hasRole returns true when role arg is null/undefined (no requirement)
  hasRole(user, null);              // true
  hasRole(user, 'super_admin');     // checks slug or uid
  hasRole(user, ['editor', 'viewer']); // any match

  // hasPermission checks: super_admin → global_editor → global_viewer → always-permitted → aclRules
  hasPermission(user, 'view', 'some-ds-id');
  hasPermission(user, 'edit', 'some-ds-id');
}
```

### Permission Resolution Order

1. `super_admin` role → always granted
2. `global_editor` role → granted for: view, create, edit, delete, create_bulk, export, import, print
3. `global_viewer` role → granted only for: view
4. Always-permitted system data sources (reports, todos, notes, etc.)
5. User's `aclRules` array (merged from all roles by the server)

### Refreshing User Data

If roles or permissions change while the app is running:

```tsx
const { refreshUser } = useDocyrusAuth();

// After a role change
await refreshUser();
```

## Advanced Usage

### Core (Framework-Agnostic)

Core classes are exported for advanced use cases without any framework dependency:

```tsx
import {
  AuthManager,
  StandaloneOAuth2Auth,
  IframeAuth,
  detectAuthMode,
} from '@docyrus/signin/core';
```

### React Entrypoint

```tsx
import {
  DocyrusAuthProvider,
  useDocyrusAuth,
  useDocyrusClient,
  SignInButton,
  AuthManager,
  detectAuthMode
} from '@docyrus/signin';
```

### Vue Entrypoint

```tsx
import {
  DocyrusAuthProvider,
  useDocyrusAuth,
  useDocyrusClient,
  SignInButton
} from '@docyrus/signin/vue';
```

## Development

```bash
# Install dependencies
pnpm install

# Development mode with watch
pnpm dev

# Build the package
pnpm build

# Run linting
pnpm lint

# Type checking
pnpm typecheck
```

## License

MIT
