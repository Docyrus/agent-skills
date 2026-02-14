# Collections & App Patterns Reference

## Table of Contents

1. [Collection Architecture](#collection-architecture)
2. [Generated Collection Structure](#generated-collection-structure)
3. [Collection Types](#collection-types)
4. [UsersCollection](#userscollection)
5. [TanStack Query Hooks Pattern](#tanstack-query-hooks-pattern)
6. [Query Key Factory Pattern](#query-key-factory-pattern)
7. [Mutation Pattern](#mutation-pattern)
8. [App Bootstrap Flow](#app-bootstrap-flow)
9. [Routing Setup](#routing-setup)
10. [API Endpoints](#api-endpoints)

---

## Collection Architecture

Collections are auto-generated from `openapi.json` using `@docyrus/tanstack-db-generator`. They provide type-safe CRUD operations for each data source.

**Generate command**: `pnpm generate-orm` (runs `@docyrus/tanstack-db-generator openapi.json`)

**Key files:**
- `src/collections/<app>-<entity>.collection.ts` — generated CRUD methods + entity types
- `src/collections/types.ts` — shared query types (filters, calculations, formulas, etc.)
- `src/collections/users.collection.ts` — special system users collection
- `src/lib/api.ts` — module-level API client proxy used by all collections

---

## Generated Collection Structure

Each collection exports an entity interface and a collection object:

```typescript
// Generated collection for base/project
import { apiClient } from '../lib/api'
import type { ICollectionListParams } from './types'

export interface BaseProjectEntity {
  id?: string
  record_owner?: string
  created_on?: string
  created_by?: string
  last_modified_on?: string
  last_modified_by?: string
  name: string
  description?: Record<string, any>
  status?: { id: string; name: string } | any
  organization?: { id: string; name: string } | string
}

export const baseProjectCollection = {
  list: (params?: ICollectionListParams): Promise<Array<BaseProjectEntity>> =>
    apiClient.get('/v1/apps/base/data-sources/project/items', params as any),

  get: (recordId: string, params?: { columns?: Array<string> }): Promise<BaseProjectEntity> =>
    apiClient.get(`/v1/apps/base/data-sources/project/items/${recordId}`, params),

  create: (data: Record<string, any>): Promise<BaseProjectEntity> =>
    apiClient.post('/v1/apps/base/data-sources/project/items', data),

  update: (recordId: string, data: Record<string, any>): Promise<BaseProjectEntity> =>
    apiClient.patch(`/v1/apps/base/data-sources/project/items/${recordId}`, data),

  delete: (recordId: string): Promise<void> =>
    apiClient.delete(`/v1/apps/base/data-sources/project/items/${recordId}`),

  deleteMany: (data: { recordIds: Array<string> }): Promise<void> =>
    apiClient.delete('/v1/apps/base/data-sources/project/items', data),
}
```

### Default Fields (always present)
Every data source entity includes: `id`, `record_owner`, `created_on`, `created_by`, `last_modified_on`, `last_modified_by`, `name`

---

## Collection Types

Shared query parameter types in `src/collections/types.ts`:

- `ICollectionListParams` — full query payload with columns, filters, calculations, formulas, childQueries, pivot, orderBy, limit, offset, fullCount, expand
- `ICollectionFilterRule` — single filter rule
- `ICollectionFilterGroup` — nested filter group
- `ICollectionCalculation` — aggregation rule
- `ICollectionFormula` — simple formula
- `ICollectionBlockFormula` — block/subquery formula
- `ICollectionChildQuery` — child query definition
- `ICollectionPivot` / `ICollectionPivotMatrix` — pivot configuration
- `ICollectionOrderBy` — sort specification

---

## UsersCollection

System users collection with special methods:

```typescript
export const UsersCollection = {
  getUsers: (): Promise<Array<UserEntity>> =>
    apiClient.get('/v1/users'),

  getMyInfo: (): Promise<UserEntity> =>
    apiClient.get('/v1/users/me'),

  createUser: (data: UserCreateParams): Promise<UserEntity> =>
    apiClient.post('/v1/users', data),

  updateMe: (data: UserUpdateParams): Promise<UserEntity> =>
    apiClient.patch('/v1/users/me', data),

  updateUser: (userId: string, data: UserUpdateParams): Promise<UserEntity> =>
    apiClient.patch(`/v1/users/${userId}`, data),

  changeUserStatus: (userId: string, status: number) =>
    apiClient.put(`/v1/users/${userId}/status/${status}`),

  saveUserDevice: (data: UserDeviceDto) =>
    apiClient.post('/v1/users/device', data),
}
```

Use `UsersCollection.getMyInfo()` for current user profile.

---

## TanStack Query Hooks Pattern

Wrap collection calls in TanStack Query hooks:

```typescript
import { useQuery } from '@tanstack/react-query'
import { baseProjectCollection } from '@/collections/base-project.collection'
import { queryKeys } from '@/lib/query-keys'

const PROJECT_COLUMNS = ['name', 'status', 'description', 'record_owner(id,firstname,lastname)']

export function useProjects(params?: ICollectionListParams) {
  return useQuery({
    queryKey: queryKeys.projects.list(params ?? {}),
    queryFn: () =>
      baseProjectCollection.list({
        columns: PROJECT_COLUMNS,  // ALWAYS specify columns
        ...params,
      }),
  })
}

export function useProject(projectId: string) {
  return useQuery({
    queryKey: queryKeys.projects.detail(projectId),
    queryFn: () =>
      baseProjectCollection.get(projectId, {
        columns: PROJECT_COLUMNS,
      }),
    enabled: !!projectId,
  })
}
```

---

## Query Key Factory Pattern

```typescript
export const queryKeys = {
  projects: {
    all: ['projects'] as const,
    lists: () => [...queryKeys.projects.all, 'list'] as const,
    list: (params: object) => [...queryKeys.projects.lists(), params] as const,
    detail: (id: string) => [...queryKeys.projects.all, 'detail', id] as const,
  },
  tasks: {
    all: ['tasks'] as const,
    lists: () => [...queryKeys.tasks.all, 'list'] as const,
    list: (params: object) => [...queryKeys.tasks.lists(), params] as const,
    detail: (id: string) => [...queryKeys.tasks.all, 'detail', id] as const,
  },
}
```

---

## Mutation Pattern

```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query'

export function useCreateProject() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (data: Record<string, unknown>) =>
      baseProjectCollection.create(data),
    onSuccess: () => {
      void queryClient.invalidateQueries({
        queryKey: queryKeys.projects.all,
      })
    },
  })
}

export function useUpdateProject() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Record<string, unknown> }) =>
      baseProjectCollection.update(id, data),
    onSuccess: (_data, { id }) => {
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.detail(id) })
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.lists() })
    },
  })
}

export function useDeleteProject() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (id: string) => baseProjectCollection.delete(id),
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.all })
    },
  })
}
```

---

## App Bootstrap Flow

1. `main.tsx`: Mount `DocyrusAuthProvider` → `QueryClientProvider` → `RouterProvider`
2. `App.tsx`: Check `useDocyrusAuth()` status
3. If authenticated: call `setApiClient(client)` to enable collections
4. Fetch user profile via `UsersCollection.getMyInfo()`
5. Render protected routes

```typescript
// App.tsx
function App() {
  const { status, signOut } = useDocyrusAuth()
  const client = useDocyrusClient()

  useEffect(() => {
    if (client) setApiClient(client)
  }, [client])

  if (status === 'loading') return <LoadingSpinner />
  if (status === 'unauthenticated') return <LoginPage />
  return <AppLayout />
}
```

---

## Routing Setup

TanStack Router with code-based routes:

```typescript
import { createRouter, createRoute, createRootRoute } from '@tanstack/react-router'

const rootRoute = createRootRoute({ component: () => <Outlet /> })

const layoutRoute = createRoute({
  getParentRoute: () => rootRoute,
  id: 'layout',
  component: AppLayout,
})

const indexRoute = createRoute({
  getParentRoute: () => layoutRoute,
  path: '/',
  component: DashboardPage,
})

const projectsRoute = createRoute({
  getParentRoute: () => layoutRoute,
  path: '/projects',
  component: ProjectsPage,
})

const projectDetailRoute = createRoute({
  getParentRoute: () => layoutRoute,
  path: '/projects/$projectId',
  component: ProjectDetailPage,
})

// Auth routes (public)
const authCallbackRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/auth/callback',
  component: () => <div>Processing login...</div>,
})

const routeTree = rootRoute.addChildren([
  layoutRoute.addChildren([indexRoute, projectsRoute, projectDetailRoute]),
  authCallbackRoute,
])

const router = createRouter({
  routeTree,
  defaultPreload: 'intent',
  scrollRestoration: true,
})
```

---

## API Endpoints

### Data Source Items (Dynamic)
```
GET    /v1/apps/{appSlug}/data-sources/{slug}/items          — List (with query payload)
GET    /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Get one
POST   /v1/apps/{appSlug}/data-sources/{slug}/items          — Create
PATCH  /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Update
DELETE /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Delete one
DELETE /v1/apps/{appSlug}/data-sources/{slug}/items          — Delete many
```

Endpoints are **dynamic** — they exist only if a data source is defined in the tenant. The `openapi.json` spec enumerates all available data sources.

### System Endpoints (Always Available)
```
GET    /v1/users                    — List users
POST   /v1/users                    — Create user
GET    /v1/users/me                 — Current user profile
PATCH  /v1/users/me                 — Update current user
PATCH  /v1/users/{userId}           — Update user
PUT    /v1/users/{userId}/status/{s} — Change user status
POST   /v1/users/device             — Save push notification device
```

### Other Standard Endpoints
```
GET  /v1/api/openapi.json           — Generate OpenAPI spec
HEAD /v1/oauth2                     — Check rate limits
PUT  reports/runCustomQuery/{id}    — Run custom query/report
```
