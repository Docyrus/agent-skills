# @docyrus/api-client & @docyrus/signin Reference

## Table of Contents

1. [RestApiClient](#restapiclient)
2. [Authentication with @docyrus/signin](#authentication-with-docyrussignin)
3. [Authorization (Roles & Permissions)](#authorization-roles--permissions)
4. [API Client Access Pattern](#api-client-access-pattern)
5. [Interceptors](#interceptors)
6. [Error Handling](#error-handling)
7. [Advanced Features](#advanced-features)

---

## RestApiClient

Type-safe REST API client from `@docyrus/api-client`.

### HTTP Methods

```typescript
import { RestApiClient } from '@docyrus/api-client'

client.get<T>(endpoint, params?)    // GET
client.post<T>(endpoint, data)      // POST
client.patch<T>(endpoint, data)     // PATCH
client.put(endpoint, data)          // PUT
client.delete(endpoint, data?)      // DELETE
```

### Typed Responses

```typescript
interface User { id: string; name: string; email: string }
const response = await client.get<User[]>('/v1/users')
```

### Config Options

```typescript
interface ApiClientConfig {
  baseURL?: string
  tokenManager?: TokenManager
  headers?: Record<string, string>
  timeout?: number
  fetch?: typeof fetch
  FormData?: typeof FormData
  AbortController?: typeof AbortController
  storage?: Storage
}
```

---

## Authentication with @docyrus/signin

Package: `@docyrus/signin` (peer dep: `@docyrus/api-client >= 0.0.10`, `react >= 18`)

### DocyrusAuthProvider Setup

Wrap application root:

```tsx
import { DocyrusAuthProvider } from '@docyrus/signin'

<DocyrusAuthProvider
  apiUrl={import.meta.env.VITE_API_BASE_URL}
  clientId={import.meta.env.VITE_OAUTH2_CLIENT_ID}
  redirectUri={import.meta.env.VITE_OAUTH2_REDIRECT_URI}
  scopes={['offline_access', 'Read.All', 'DS.ReadWrite.All', 'Users.Read']}
  callbackPath="/auth/callback"
>
  <App />
</DocyrusAuthProvider>
```

### Provider Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `apiUrl` | `string` | `https://alpha-api.docyrus.com` | API base URL |
| `clientId` | `string` | Built-in default | OAuth2 client ID |
| `redirectUri` | `string` | `origin + callbackPath` | OAuth2 redirect URI |
| `scopes` | `string[]` | `['offline_access', 'Read.All', ...]` | OAuth2 scopes |
| `callbackPath` | `string` | `/auth/callback` | OAuth callback route |
| `forceMode` | `'standalone' \| 'iframe'` | Auto-detected | Force auth mode |
| `storageKeyPrefix` | `string` | `docyrus_oauth2_` | localStorage prefix |
| `allowedHostOrigins` | `string[]` | `undefined` | Extra trusted iframe origins |

### Auth Modes

- **Standalone**: OAuth2 Authorization Code + PKCE via page redirect. Tokens stored in localStorage, auto-refreshed.
- **Iframe**: Receives tokens via `window.postMessage` from `*.docyrus.app` hosts. Requests refresh from host when expired.

### useDocyrusAuth() Hook

```typescript
const {
  status,        // 'loading' | 'authenticated' | 'unauthenticated'
  mode,          // 'standalone' | 'iframe'
  client,        // RestApiClient | null
  tokens,        // { accessToken, refreshToken, ... } | null
  user,          // DocyrusUser | null — auto-fetched from /v1/users/me
  signIn,        // () => void — redirects to Docyrus login
  signOut,       // () => void — logout and clear tokens
  hasRole,       // (role: string | string[]) => boolean — check role by slug or uid
  hasPermission, // (operation: string, dataSourceId?: string) => boolean — check ACL permission
  refreshUser,   // () => Promise<void> — re-fetch user from API
  error,         // Error | null
} = useDocyrusAuth()
```

### useDocyrusClient() Hook

```typescript
const client = useDocyrusClient() // RestApiClient | null
```

### SignInButton Component

```tsx
// Basic
<SignInButton />

// Styled
<SignInButton className="btn" label="Log in with Docyrus" />

// Render prop
<SignInButton>
  {({ signIn, isLoading }) => (
    <button onClick={signIn} disabled={isLoading}>
      {isLoading ? 'Redirecting...' : 'Sign in'}
    </button>
  )}
</SignInButton>
```

### Environment Variables (.env)

```bash
VITE_API_BASE_URL=https://localhost:3366
VITE_OAUTH2_CLIENT_ID=your-client-id
VITE_OAUTH2_REDIRECT_URI=http://localhost:3000/auth/callback
VITE_OAUTH2_SCOPES=openid profile offline_access Users.Read DS.ReadWrite.All
```

---

## Authorization (Roles & Permissions)

The provider auto-fetches the current user from `/v1/users/me` after authentication. The `user`, `hasRole`, and `hasPermission` are available on the `useDocyrusAuth()` hook.

### Role-Based UI Gating

```tsx
function Dashboard({ dataSourceId }: { dataSourceId: string }) {
  const { user, hasRole, hasPermission } = useDocyrusAuth()

  if (!user) return <Spinner />

  const canEdit = hasPermission('edit', dataSourceId)
  const canDelete = hasPermission('delete', dataSourceId)
  const isAdmin = hasRole('super_admin')

  return (
    <div>
      {canEdit && <Button>Edit</Button>}
      {canDelete && <Button variant="destructive">Delete</Button>}
      {isAdmin && <AdminPanel />}
    </div>
  )
}
```

### Permission Resolution Order

1. `super_admin` role → always granted
2. `global_editor` role → granted for: view, create, edit, delete, create_bulk, export, import, print
3. `global_viewer` role → granted only for: view
4. Always-permitted system data sources (reports, todos, notes, etc.)
5. User's `aclRules` array (merged from all roles by the server)

### Pure Functions (No React)

```typescript
import { hasRole, hasPermission } from '@docyrus/signin/core'
import type { DocyrusUser } from '@docyrus/signin/core'

hasRole(user, 'super_admin')
hasPermission(user, 'edit', 'some-ds-id')
```

---

## API Client Access Pattern

Generated collections are React hooks that use `useDocyrusClient()` internally to get the authenticated `RestApiClient` from `DocyrusAuthProvider`. No manual client syncing is needed.

```typescript
// Collections get the client automatically via useDocyrusClient()
function useBaseProjectCollection() {
  const client = useDocyrusClient()
  return {
    list: (params?) => client!.get('/v1/apps/base/data-sources/project/items', params),
    // ... other CRUD methods
  }
}

// In your component — just call the collection hook
function ProjectList() {
  const { list } = useBaseProjectCollection()
  const { data } = useQuery({
    queryKey: ['projects'],
    queryFn: () => list({ columns: ['name', 'status'] }),
  })
}
```

For direct API access outside collections, use the `useDocyrusClient()` hook:

```typescript
const client = useDocyrusClient()
const data = await client!.get<MyType>('/v1/custom-endpoint')
```

---

## Interceptors

Add request/response interceptors via `client.use()`:

```typescript
client.use({
  request: (config) => {
    // Transform columns array to comma-separated string
    if (config.params?.columns && Array.isArray(config.params.columns)) {
      config.params.columns = config.params.columns.join(',')
    }
    return config
  },
  response: (response) => {
    // Unwrap nested .data property
    if (response.data?.data && !Array.isArray(response.data)) {
      response.data = response.data.data
    }
    return response
  },
  error: (error, request, response) => {
    if (error.status === 401) { /* handle auth error */ }
    return { error, request, response }
  },
})
```

---

## Error Handling

```typescript
import {
  ApiError, NetworkError, TimeoutError,
  AuthenticationError, AuthorizationError,
  NotFoundError, RateLimitError, ValidationError,
  OAuth2Error, InvalidGrantError,
} from '@docyrus/api-client'

try {
  await client.get('/resource')
} catch (error) {
  if (error instanceof AuthenticationError) { /* 401 */ }
  else if (error instanceof AuthorizationError) { /* 403 */ }
  else if (error instanceof NotFoundError) { /* 404 */ }
  else if (error instanceof RateLimitError) { /* 429 - error.retryAfter */ }
  else if (error instanceof NetworkError) { /* network issue */ }
  else if (error instanceof TimeoutError) { /* timeout */ }
}
```

---

## Advanced Features

### SSE (Server-Sent Events)

```typescript
const eventSource = client.sse('/events', {
  onMessage(data) { console.log(data) },
  onError(error) { console.error(error) },
  onComplete() { console.log('done') },
})
eventSource.close()
```

### File Upload

```typescript
const formData = new FormData()
formData.append('file', fileInput.files[0])
await client.post('/upload', formData)
```

### File Download

```typescript
const response = await client.get('/download/file.pdf', { responseType: 'blob' })
const url = URL.createObjectURL(response.data)
```

### HTML to PDF

```typescript
await client.html2pdf({
  html: '<html><body>Content</body></html>',
  options: { format: 'A4', margin: { top: 10, bottom: 10 } },
})
```

### Retry Logic

```typescript
import { withRetry } from '@docyrus/api-client'
const response = await withRetry(() => client.get('/endpoint'), {
  retries: 3, retryDelay: 1000,
  retryCondition: (error) => error.status >= 500,
})
```
