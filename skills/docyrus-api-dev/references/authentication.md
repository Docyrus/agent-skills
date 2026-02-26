# @docyrus/signin — React Authentication Reference

## Table of Contents

1. [Overview](#overview)
2. [Installation](#installation)
3. [DocyrusAuthProvider](#docyrusauthprovider)
4. [Auth Hooks](#auth-hooks)
5. [SignInButton](#signinbutton)
6. [Auth Modes](#auth-modes)
7. [Environment Variables](#environment-variables)
8. [App Integration Pattern](#app-integration-pattern)
9. [Advanced Usage](#advanced-usage)

---

## Overview

`@docyrus/signin` provides "Sign in with Docyrus" for React apps. Auto-detects environment:
- **Standalone**: OAuth2 Authorization Code + PKCE via page redirect
- **Iframe**: Receives tokens via `window.postMessage` from `*.docyrus.app` hosts

Peer dependencies: `react >= 18`, `@docyrus/api-client >= 0.0.10`

---

## Installation

```bash
pnpm add @docyrus/signin @docyrus/api-client
```

---

## DocyrusAuthProvider

Wrap application root:

```tsx
import { DocyrusAuthProvider } from '@docyrus/signin'

<DocyrusAuthProvider
  apiUrl="https://alpha-api.docyrus.com"
  clientId="your-oauth2-client-id"
  redirectUri="http://localhost:3000/auth/callback"
  scopes={['offline_access', 'Read.All', 'DS.ReadWrite.All', 'Users.Read']}
  callbackPath="/auth/callback"
>
  <App />
</DocyrusAuthProvider>
```

### Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `apiUrl` | `string` | `https://alpha-api.docyrus.com` | API base URL |
| `clientId` | `string` | Built-in default | OAuth2 client ID |
| `redirectUri` | `string` | `origin + callbackPath` | OAuth2 redirect URI |
| `scopes` | `string[]` | `['offline_access', 'Read.All', ...]` | OAuth2 scopes |
| `callbackPath` | `string` | `/auth/callback` | Path to detect OAuth callback |
| `forceMode` | `'standalone' \| 'iframe'` | Auto-detected | Force a specific auth mode |
| `storageKeyPrefix` | `string` | `docyrus_oauth2_` | localStorage key prefix |
| `allowedHostOrigins` | `string[]` | `undefined` | Extra trusted iframe origins |

---

## Auth Hooks

### useDocyrusAuth()

Full authentication context:

```typescript
import { useDocyrusAuth } from '@docyrus/signin'

const {
  status,   // 'loading' | 'authenticated' | 'unauthenticated'
  mode,     // 'standalone' | 'iframe'
  client,   // RestApiClient | null — configured API client with tokens
  tokens,   // { accessToken, refreshToken, ... } | null
  signIn,   // () => void — redirects to Docyrus login page
  signOut,  // () => void — logout and clear tokens
  error,    // Error | null
} = useDocyrusAuth()
```

### useDocyrusClient()

Shorthand for just the API client:

```typescript
import { useDocyrusClient } from '@docyrus/signin'

const client = useDocyrusClient() // RestApiClient | null

if (client) {
  const user = await client.get('/v1/users/me')
  const items = await client.get('/v1/apps/base/data-sources/project/items', queryPayload)
}
```

---

## SignInButton

Unstyled button. Automatically hidden when authenticated or in iframe mode.

```tsx
import { SignInButton } from '@docyrus/signin'

// Basic
<SignInButton />

// Styled
<SignInButton className="btn btn-primary" label="Log in with Docyrus" />

// Render prop for full control
<SignInButton>
  {({ signIn, isLoading }) => (
    <button onClick={signIn} disabled={isLoading}>
      {isLoading ? 'Redirecting...' : 'Sign in with Docyrus'}
    </button>
  )}
</SignInButton>
```

---

## Auth Modes

### Standalone (OAuth2 PKCE)

For apps running directly in the browser:

1. User clicks sign-in
2. Page redirects to Docyrus authorization endpoint
3. After login, redirects back with authorization code
4. Provider automatically exchanges code for tokens
5. Tokens stored in localStorage, auto-refreshed before expiry

### Iframe (postMessage)

For apps embedded in an iframe on `*.docyrus.app`:

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
import { DocyrusAuthProvider } from '@docyrus/signin'

const scopes = (import.meta.env.VITE_OAUTH2_SCOPES || '').split(' ').filter(Boolean)

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <DocyrusAuthProvider
      apiUrl={import.meta.env.VITE_API_BASE_URL}
      clientId={import.meta.env.VITE_OAUTH2_CLIENT_ID}
      redirectUri={import.meta.env.VITE_OAUTH2_REDIRECT_URI}
      scopes={scopes}
      callbackPath="/auth/callback"
    >
      <App />
    </DocyrusAuthProvider>
  </StrictMode>,
)
```

### Auth-Gated App (App.tsx)

```tsx
import { useDocyrusAuth, useDocyrusClient, SignInButton } from '@docyrus/signin'

function App() {
  const { status, signOut } = useDocyrusAuth()
  const client = useDocyrusClient()

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

In React components, use `useDocyrusClient()` to get the authenticated client. Generated collections are hooks that call `useDocyrusClient()` internally, so no manual client syncing is needed:

```typescript
// Generated collections use useDocyrusClient() internally
const { list, get, create } = useBaseProjectCollection()

// For direct API access in React components
const client = useDocyrusClient()
const data = await client!.get('/v1/custom-endpoint')
```

---

## Advanced Usage

Core classes exported for advanced scenarios:

```typescript
import { AuthManager, StandaloneOAuth2Auth, IframeAuth, detectAuthMode } from '@docyrus/signin'
```
