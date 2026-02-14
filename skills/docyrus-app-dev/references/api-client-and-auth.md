# @docyrus/api-client & @docyrus/signin Reference

## Table of Contents

1. [RestApiClient](#restapiclient)
2. [Authentication with @docyrus/signin](#authentication-with-docyrussignin)
3. [API Client Module Pattern](#api-client-module-pattern)
4. [Interceptors](#interceptors)
5. [Error Handling](#error-handling)
6. [Advanced Features](#advanced-features)

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
  status,   // 'loading' | 'authenticated' | 'unauthenticated'
  mode,     // 'standalone' | 'iframe'
  client,   // RestApiClient | null
  tokens,   // { accessToken, refreshToken, ... } | null
  signIn,   // () => void — redirects to Docyrus login
  signOut,  // () => void — logout and clear tokens
  error,    // Error | null
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

## API Client Module Pattern

For non-React modules (collections, utilities), `src/lib/api.ts` exposes a setter/proxy pattern:

```typescript
import type { RestApiClient } from '@docyrus/api-client'

let apiClient: RestApiClient | null = null

// Proxy prevents access before initialization
const apiClientProxy = new Proxy({} as RestApiClient, {
  get(_target, prop) {
    if (!apiClient) throw new Error('API client not initialized.')
    const value = apiClient[prop as keyof RestApiClient]
    if (typeof value === 'function') return value.bind(apiClient)
    return value
  },
})

export function setApiClient(client: RestApiClient) {
  apiClient = client
  // Add interceptors here
}

export function getApiClient(): RestApiClient {
  if (!apiClient) throw new Error('API client not initialized.')
  return apiClient
}

export { apiClientProxy as apiClient }
```

**App.tsx syncs the client after auth:**

```typescript
const client = useDocyrusClient()

useEffect(() => {
  if (client) setApiClient(client)
}, [client])
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
