# Collections & App Patterns Reference

## Table of Contents

1. [Collection Architecture](#collection-architecture)
2. [Generated Collection Structure](#generated-collection-structure)
3. [Collection Types](#collection-types)
4. [useUsersCollection](#useuserscollection)
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
- `src/collections/<app>-<entity>.collection.ts` — generated React hooks with CRUD methods + entity types
- `src/collections/types.ts` — shared query types (filters, calculations, formulas, etc.)
- `src/collections/users.collection.ts` — special system users collection hook

---

## Generated Collection Structure

Each collection exports an entity interface and a React hook that returns CRUD methods:

```typescript
// Generated collection for base/project
import { useDocyrusClient } from '@docyrus/signin'
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

export function useBaseProjectCollection() {
  const client = useDocyrusClient()

  return {
    list: (params?: ICollectionListParams): Promise<Array<BaseProjectEntity>> =>
      client!.get('/v1/apps/base/data-sources/project/items', params as any),

    get: (recordId: string, params?: { columns?: Array<string> }): Promise<BaseProjectEntity> =>
      client!.get(`/v1/apps/base/data-sources/project/items/${recordId}`, params),

    create: (data: Record<string, any>): Promise<BaseProjectEntity> =>
      client!.post('/v1/apps/base/data-sources/project/items', data),

    update: (recordId: string, data: Record<string, any>): Promise<BaseProjectEntity> =>
      client!.patch(`/v1/apps/base/data-sources/project/items/${recordId}`, data),

    delete: (recordId: string): Promise<void> =>
      client!.delete(`/v1/apps/base/data-sources/project/items/${recordId}`),

    deleteMany: (data: { recordIds: Array<string> }): Promise<void> =>
      client!.delete('/v1/apps/base/data-sources/project/items', data),
  }
}
```

Collections are hooks because they use `useDocyrusClient()` internally, which provides the authenticated `RestApiClient` from `DocyrusAuthProvider`. This means collections must be called inside React components.

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

## useUsersCollection

System users collection hook with special methods:

```typescript
export function useUsersCollection() {
  const client = useDocyrusClient()

  return {
    getUsers: (): Promise<Array<UserEntity>> =>
      client!.get('/v1/users'),

    getMyInfo: (): Promise<UserEntity> =>
      client!.get('/v1/users/me'),

    createUser: (data: UserCreateParams): Promise<UserEntity> =>
      client!.post('/v1/users', data),

    updateMe: (data: UserUpdateParams): Promise<UserEntity> =>
      client!.patch('/v1/users/me', data),

    updateUser: (userId: string, data: UserUpdateParams): Promise<UserEntity> =>
      client!.patch(`/v1/users/${userId}`, data),

    changeUserStatus: (userId: string, status: number) =>
      client!.put(`/v1/users/${userId}/status/${status}`),

    saveUserDevice: (data: UserDeviceDto) =>
      client!.post('/v1/users/device', data),

    getMyTenants: () =>
      client!.get('/v1/users/me/tenants'),
  }
}
```

Use `useUsersCollection().getMyInfo()` for current user profile.

---

## TanStack Query Hooks Pattern

Wrap collection hook methods in TanStack Query hooks. Since collections are themselves hooks, call them inside the component/hook, then pass the returned methods to TanStack Query:

```typescript
import { useQuery } from '@tanstack/react-query'
import { useBaseProjectCollection } from '@/collections/base-project.collection'
import { queryKeys } from '@/lib/query-keys'

const PROJECT_COLUMNS = ['name', 'status', 'description', 'record_owner(id,firstname,lastname)']

export function useProjects(params?: ICollectionListParams) {
  const { list } = useBaseProjectCollection()
  return useQuery({
    queryKey: queryKeys.projects.list(params ?? {}),
    queryFn: () =>
      list({
        columns: PROJECT_COLUMNS,  // ALWAYS specify columns
        ...params,
      }),
  })
}

export function useProject(projectId: string) {
  const { get } = useBaseProjectCollection()
  return useQuery({
    queryKey: queryKeys.projects.detail(projectId),
    queryFn: () =>
      get(projectId, {
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
import { useBaseProjectCollection } from '@/collections/base-project.collection'

export function useCreateProject() {
  const { create } = useBaseProjectCollection()
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (data: Record<string, unknown>) => create(data),
    onSuccess: () => {
      void queryClient.invalidateQueries({
        queryKey: queryKeys.projects.all,
      })
    },
  })
}

export function useUpdateProject() {
  const { update } = useBaseProjectCollection()
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Record<string, unknown> }) =>
      update(id, data),
    onSuccess: (_data, { id }) => {
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.detail(id) })
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.lists() })
    },
  })
}

export function useDeleteProject() {
  const { delete: deleteProject } = useBaseProjectCollection()
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (id: string) => deleteProject(id),
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: queryKeys.projects.all })
    },
  })
}
```

---

## App Bootstrap Flow

1. `main.tsx`: Mount `DocyrusAuthProvider` → `QueryClientProvider` → `RouterProvider`
2. `App.tsx`: Check `useDocyrusAuth()` status — `user` is auto-fetched from `/v1/users/me`
3. Use `hasRole()` / `hasPermission()` from `useDocyrusAuth()` for authorization checks
4. Use collection hooks (e.g., `useUsersCollection()`) for data access — they get the authenticated client via `useDocyrusClient()` internally
5. Render protected routes

```typescript
// App.tsx
function App() {
  const { status, user, hasRole, hasPermission } = useDocyrusAuth()

  if (status === 'loading') return <LoadingSpinner />
  if (status === 'unauthenticated') return <LoginPage />

  // user auto-fetched, hasRole/hasPermission ready
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
