# @kivacore/signin — React Authentication Reference

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [KivaAuthProvider](#kivaauthprovider)
4. [Auth Hooks](#auth-hooks)
5. [Authorization (Roles & Permissions)](#authorization-roles--permissions)
6. [SignInButton](#signinbutton)
7. [Auth Modes](#auth-modes)
8. [Environment Variables](#environment-variables)
9. [App Integration Pattern](#app-integration-pattern)
10. [Advanced Usage](#advanced-usage)

---

## Overview

`@kivacore/signin` provides "Sign in with KIVA" for React apps. Auto-detects environment:
- **Standalone**: OAuth2 Authorization Code + PKCE via page redirect
- **Iframe**: Receives tokens via `window.postMessage` from `*.kiva.app` hosts

Peer dependencies: `react >= 18`, `@kivacore/api-client >= 0.0.10`

---

## Installation

```bash
pnpm add @kivacore/signin @kivacore/api-client
```

---

## KivaAuthProvider

Wrap application root:

```tsx
import { KivaAuthProvider } from '@kivacore/signin'

<KivaAuthProvider
  apiUrl="https://api.kiva.app"
  clientId="your-oauth2-client-id"
  redirectUri="http://localhost:3000/auth/callback"
  scopes={['offline_access', 'Read.All', 'DS.ReadWrite.All', 'Users.Read']}
  callbackPath="/auth/callback"
>
  <App />
</KivaAuthProvider>
```

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `apiUrl` | `string` | `https://api.kiva.app` | API base URL |
| `clientId` | `string` | Built-in default | OAuth2 client ID |
| `redirectUri` | `string` | `origin + callbackPath` | OAuth2 redirect URI |
| `scopes` | `string[]` | `['offline_access', 'Read.All', ...]` | OAuth2 scopes |
| `callbackPath` | `string` | `/auth/callback` | Path to detect OAuth callback |
| `forceMode` | `'standalone' \| 'iframe'` | Auto-detected | Force a specific auth mode |
| `storageKeyPrefix` | `string` | `kiva_oauth2_` | localStorage key prefix |
| `allowedHostOrigins` | `string[]` | `undefined` | Extra trusted iframe origins |

---

## Auth Hooks

### useKivaAuth()

Full authentication context:

```typescript
import { useKivaAuth } from '@kivacore/signin'

const {
  status,        // 'loading' | 'authenticated' | 'unauthenticated'
  mode,          // 'standalone' | 'iframe'
  client,        // RestApiClient | null — configured API client with tokens
  tokens,        // { accessToken, refreshToken, ... } | null
  user,          // KivaUser | null — auto-fetched from /v1/users/me
  signIn,        // () => void — redirects to KIVA login page
  signOut,       // () => void — logout and clear tokens
  hasRole,       // (role: string | string[]) => boolean — check role by slug or uid
  hasPermission, // (operation: string, dataSourceId?: string) => boolean — check ACL permission
  refreshUser,   // () => Promise<void> — re-fetch user from API
  error,         // Error | null
} = useKivaAuth()
```

### useKivaClient()

Shorthand for just the API client:

```typescript
import { useKivaClient } from '@kivacore/signin'

const client = useKivaClient() // RestApiClient | null

if (client) {
  const user = await client.get('/v1/users/me')
  const items = await client.get('/v1/apps/base/data-sources/project/items', queryPayload)
}
```

---

## Authorization (Roles & Permissions)

The provider auto-fetches the current user from `/v1/users/me` after authentication and exposes `hasRole` and `hasPermission` helpers. The `user` object is `null` until the fetch completes (shortly after `status` becomes `'authenticated'`).

### Role Checking

```typescript
const { hasRole } = useKivaAuth()

hasRole(null)                      // true — no role requirement
hasRole('super_admin')             // checks slug or uid match
hasRole(['editor', 'reviewer'])    // true if user has any of these roles
```

Checks both `primaryRole` and all additional `roles` from the user object.

### Permission Checking

```typescript
const { hasPermission } = useKivaAuth()

hasPermission('view', dataSourceId)    // can view this data source?
hasPermission('edit', dataSourceId)    // can edit?
hasPermission('delete', dataSourceId)  // can delete?
```

**Permission resolution order:**
1. `super_admin` role → always granted
2. `global_editor` role → granted for: view, create, edit, delete, create_bulk, export, import, print
3. `global_viewer` role → granted only for: view
4. Always-permitted system data sources (reports, todos, notes, etc.)
5. User's `aclRules` array (merged from all roles by the server)

### Pure Functions (Framework-Agnostic)

```typescript
import { hasRole, hasPermission } from '@kivacore/signin/core'
import type { KivaUser } from '@kivacore/signin/core'

// Use with any user object (e.g., server-side, tests)
hasRole(user, 'super_admin')
hasPermission(user, 'edit', 'some-ds-id')
```

### Refreshing User Data

```typescript
const { refreshUser } = useKivaAuth()
await refreshUser() // re-fetch after role/permission changes
```

---

## SignInButton

Unstyled button. Automatically hidden when authenticated or in iframe mode.

```tsx
import { SignInButton } from '@kivacore/signin'

// Basic
<SignInButton />

// Styled
<SignInButton className="btn btn-primary" label="Log in with KIVA" />

// Render prop for full control
<SignInButton>
  {({ signIn, isLoading }) => (
    <button onClick={signIn} disabled={isLoading}>
      {isLoading ? 'Redirecting...' : 'Sign in with KIVA'}
    </button>
  )}
</SignInButton>
```

---

## Auth Modes

### Standalone (OAuth2 PKCE)

For apps running directly in the browser:

1. User clicks sign-in
2. Page redirects to KIVA authorization endpoint
3. After login, redirects back with authorization code
4. Provider automatically exchanges code for tokens
5. Tokens stored in localStorage, auto-refreshed before expiry

### Iframe (postMessage)

For apps embedded in an iframe on `*.kiva.app`:

1. Provider detects iframe environment and validates host origin
2. Host sends `{ type: 'signin', accessToken, refreshToken }` via `postMessage`
3. Provider creates API client with received tokens
4. When tokens expire, provider sends `{ type: 'token-refresh-request' }` to host
5. Host responds with fresh tokens

---

## Environment Variables

```bash
# .env
VITE_API_BASE_URL=https://localhost:3366
VITE_OAUTH2_CLIENT_ID=your-client-id
VITE_OAUTH2_REDIRECT_URI=http://localhost:3000/auth/callback
VITE_OAUTH2_SCOPES=openid profile offline_access Users.Read DS.ReadWrite.All
```

Access in code: `import.meta.env.VITE_API_BASE_URL`

---

## App Integration Pattern

### Minimal Setup (main.tsx)

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { KivaAuthProvider } from '@kivacore/signin'

const scopes = (import.meta.env.VITE_OAUTH2_SCOPES || '').split(' ').filter(Boolean)

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <KivaAuthProvider
      apiUrl={import.meta.env.VITE_API_BASE_URL}
      clientId={import.meta.env.VITE_OAUTH2_CLIENT_ID}
      redirectUri={import.meta.env.VITE_OAUTH2_REDIRECT_URI}
      scopes={scopes}
      callbackPath="/auth/callback"
    >
      <App />
    </KivaAuthProvider>
  </StrictMode>,
)
```

### Auth-Gated App (App.tsx)

```tsx
import { useKivaAuth, useKivaClient, SignInButton } from '@kivacore/signin'

function App() {
  const { status, signOut } = useKivaAuth()
  const client = useKivaClient()

  if (status === 'loading') return <div>Loading...</div>
  if (status === 'unauthenticated') return <SignInButton />

  // client is guaranteed non-null when authenticated
  return (
    <div>
      <p>Authenticated!</p>
      <button onClick={() => client!.get('/v1/users/me').then(console.log)}>My Profile</button>
      <button onClick={signOut}>Sign Out</button>
    </div>
  )
}
```

### Accessing the API Client

In React components, use `useKivaClient()` to get the authenticated client. Generated collections are hooks that call `useKivaClient()` internally, so no manual client syncing is needed:

```typescript
// Generated collections use useKivaClient() internally
const { list, get, create } = useBaseProjectCollection()

// For direct API access in React components
const client = useKivaClient()
const data = await client!.get('/v1/custom-endpoint')
```

---

## Advanced Usage

Core classes and permission functions exported for advanced scenarios:

```typescript
import { AuthManager, StandaloneOAuth2Auth, IframeAuth, detectAuthMode } from '@kivacore/signin'
import { hasRole, hasPermission, getAllRoles } from '@kivacore/signin/core'
import type { KivaUser, KivaRole, KivaAclRule, AclOperation, PermissionConfig } from '@kivacore/signin/core'
```
