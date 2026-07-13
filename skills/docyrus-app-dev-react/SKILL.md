---
name: docyrus-app-dev-react
description: Build Docyrus React TypeScript web applications end-to-end, combining authentication, API access, @docyrus/app-utils runtime helpers, @docyrus/devtools in-app debugging, generated collections, TanStack Query/Form patterns, and production-grade UI implementation with preferred component libraries. Use when creating or modifying Docyrus-backed apps that use @docyrus/api-client, @docyrus/signin, @docyrus/app-utils, @docyrus/devtools, Docyrus collections, queries, dashboards, forms, tables, layouts, or Docyrus UI components.
---

# Docyrus App Dev React

Build Docyrus React TypeScript applications end-to-end. This skill combines app architecture, authentication, data access, query patterns, and production-grade UI guidance in one place.

## Recommended Tech Stack

- React 19 + TypeScript + Vite
- TanStack Router (code-based), TanStack Query (server state), TanStack Form
- Tailwind CSS v4, shadcn/ui components
- `@docyrus/api-client` + `@docyrus/signin` + `@docyrus/app-utils`
- `@docyrus/devtools` — recommended in-app developer panel for every Docyrus app during development (network, errors, issues, console, iframe messages, OpenAPI request explorer, DOM element picker); gate it to non-production builds
- Auto-generated collections from OpenAPI spec
- Preferred UI libraries: shadcn, diceui, animate-ui, docyrus-ui, reui

## When to Use This Skill

Use this skill when you are:

- Building or modifying a Docyrus-backed React app
- Setting up authentication with `@docyrus/signin`
- Bootstrapping tenant-aware runtime utilities with `@docyrus/app-utils`
- Fetching or mutating data with generated collections or `@docyrus/api-client`
- Persisting app-level config or user-level config or saved grid views with `AppConfig`, `UserAppConfig`, and `DataViews`
- Building record sharing, role management, or ACL-driven UI flows
- Designing feature UIs such as dashboards, forms, tables, layouts, dialogs, analytics, or detail pages
- Selecting between shadcn, diceui, animate-ui, docyrus-ui, and reui components
- Implementing complete feature flows that combine data access and polished UI

## End-to-End Feature Workflow

1. Set up app auth, routing, and query providers. `@docyrus/signin` already fetches the signed-in user from `/v1/users/me` — read it from `useDocyrusAuth().user`, do not add your own user call.
2. Mount `@docyrus/devtools` near the root (dev builds only) and register the authenticated client so requests, errors, and console are instrumented from the start.
3. After sign-in, create **one** shared `InventoryClient` (`createInventoryClient`) and `load()` it behind a progress bar to warm apps, data sources, users, brands, preferences, and this app's config.
4. Bootstrap `TenantPreferences`, date/number utilities, and shared app runtime helpers from `@docyrus/app-utils`, wiring the shared inventory into every app-utils client.
5. Use generated Docyrus collection hooks or the REST client for data access.
6. Define `columns`, filters, formulas, child queries, and mutations correctly.
7. Use `AppConfig` for per-app persisted settings, `UserAppConfig` for per-user per-app settings, and `DataViews` for saved grid views.
8. Check preferred UI components before building anything custom.
9. Use Docyrus form and detail patterns for create, edit, item detail, and editable grid flows.
10. Connect UI actions to TanStack Query mutations and invalidate relevant queries.

## Quick Start: App Bootstrap

### Root provider setup

```tsx
import { DocyrusAuthProvider } from '@docyrus/signin'

<DocyrusAuthProvider
  apiUrl={import.meta.env.VITE_API_BASE_URL}
  clientId={import.meta.env.VITE_OAUTH2_CLIENT_ID}
  redirectUri={import.meta.env.VITE_OAUTH2_REDIRECT_URI}
  scopes={['offline_access', 'Read.All', 'DS.ReadWrite.All', 'Users.Read']}
  callbackPath="/auth/callback"
>
  <QueryClientProvider client={queryClient}>
    <RouterProvider router={router} />
  </QueryClientProvider>
</DocyrusAuthProvider>
```

### Auth gate and current-user access

```tsx
const { status, user, hasRole, hasPermission } = useDocyrusAuth()

if (status === 'loading') return <Spinner />
if (status === 'unauthenticated') return <SignInButton />

// user is auto-fetched from /v1/users/me after authentication
// hasRole('super_admin') — check role by slug or uid
// hasPermission('edit', dataSourceId) — check ACL permission on a data source
```

**`@docyrus/signin` already fetches the signed-in user.** On authentication it calls `/v1/users/me` once and exposes the result (roles, permissions, `aclRules`, identity) as `useDocyrusAuth().user`. Read the current user and do all auth/role/permission checks from there — **never add your own `/v1/users/me` request**. Call `refreshUser()` to re-fetch after a role change. Only reach for `useUsersCollection().getMyInfo()` when you need profile fields that are not on the auth user.

### Inventory cache warm-up after sign-in

Right after `@docyrus/signin` reports an authenticated session, create **one** shared `InventoryClient` from `@docyrus/app-utils` and `load()` it behind a progress bar. The inventory is a tenant-wide in-memory cache of apps, data sources (with embedded views/forms/fields), users, brands, tenant preferences, and this app's config/user-config. Warming it once turns nearly every later metadata read across the app into a cache hit (no network), and the same instance backs every app-utils client you pass it to.

Create the single instance at bootstrap and wire every app-utils client to it:

```ts
// services/docyrus.ts — one cache, shared by every client
import {
  createInventoryClient,
  createDataSourceClient,
  createBrandClient,
  createAppConfigClient,
  createUserAppConfigClient,
  createDataViewClient,
} from '@docyrus/app-utils'

const appId = import.meta.env.VITE_APP_ID

export const inventory = createInventoryClient(apiClient)

export const dataSources = createDataSourceClient(apiClient, { inventory })
export const brands = createBrandClient(apiClient, { inventory })
export const appConfig = createAppConfigClient(apiClient, appId, { inventory })
export const userAppConfig = createUserAppConfigClient(apiClient, appId, { inventory })

// View/form clients are per data source — construct on demand, always sharing `inventory`
export const viewsFor = (appSlug: string, dsSlug: string) =>
  createDataViewClient(apiClient, appSlug, dsSlug, { inventory })
```

Warm the cache once, after sign-in, before routing to the home page:

```ts
// Runs after useDocyrusAuth() reports an authenticated session.
async function bootstrapAfterSignIn(setProgress: (p: { label: string; value: number }) => void) {
  await inventory.load({
    // Drop 'users' from `include` if the signed-in user may lack the Users.Read.All scope.
    onProgress: ({ message, ratio, phase }) => {
      setProgress({ label: message, value: Math.round(ratio * 100) })
      if (phase === 'complete') setProgress({ label: 'Ready', value: 100 })
    },
  })
}
```

- Create the inventory **after** sign-in — it needs the authenticated `apiClient`.
- Pass the **same** `inventory` to every app-utils client so they share one cache; mutations through those clients patch the cache automatically.
- `load()` warms apps, data sources, users, brands, and preferences, plus this app's config/user-config when `VITE_APP_ID` is set; it emits progress events for a post-sign-in progress bar.
- A full page reload starts cold — that is why `load()` runs after each sign-in. On a **tenant switch** without reload, call `inventory.refresh()` so the new tenant's data replaces the old.

See `references/app-utils-readme.md` (the "createInventoryClient" section) for `load()` options, scope handling, and cache-invalidation helpers.

### Tenant-aware app utilities

Use `@docyrus/app-utils` as the default runtime layer for tenant-level formatting and persisted app/grid preferences.

```tsx
import {
  createAppConfigClient,
  createUserAppConfigClient,
  createDataViewClient,
  createDateUtils,
  createNumberUtils,
  getTenantPreferences,
} from '@docyrus/app-utils'

function useAppRuntime(appId: string) {
  const client = useDocyrusClient()
  const { getMyInfo } = useUsersCollection()

  return useQuery({
    queryKey: ['app-runtime', appId],
    enabled: !!client && !!appId,
    queryFn: async () => {
      const [preferences, me] = await Promise.all([
        getTenantPreferences(client!),
        getMyInfo(),
      ])

      return {
        preferences,
        me,
        dateUtils: createDateUtils({
          preferences,
          userTimezone: me.timeZone?.id,
        }),
        numberUtils: createNumberUtils({ preferences }),
        appConfig: createAppConfigClient(client!, appId),
        userConfig: createUserAppConfigClient(client!, appId),
        dataViews: createDataViewClient(client!, appId),
      }
    },
  })
}
```

Use this runtime to:

- Format dates and datetimes with tenant format strings and the user's timezone.
- Format numbers, currency-like values, and decimals using tenant separators and precision.
- Read and upsert the app's single persisted `AppConfig` document.
- Read and upsert the current user's `UserAppConfig` document (per-user per-app settings).
- Read and persist saved grid views through `DataViews`.

When you have warmed a shared inventory (see above), pass `{ inventory }` to each of these clients and read `preferences` from `inventory.getPreferences()` instead of a fresh `getTenantPreferences(client)` call — every read then resolves from the warmed cache.

### Data fetching with generated collections

```tsx
const { list } = useBaseProjectCollection()

const { data: projects } = useQuery({
  queryKey: ['projects'],
  queryFn: () =>
    list({
      columns: ['name', 'status', 'record_owner(firstname,lastname)'],
      filters: { rules: [{ field: 'status', operator: '!=', value: 'archived' }] },
      orderBy: 'created_on DESC',
      limit: 50,
    }),
})
```

### ACL, roles, and record sharing

Use direct `useDocyrusClient()` calls for ACL features. These routes may be hidden from generated OpenAPI output, so they are typically not available through generated collection hooks.

```tsx
const client = useDocyrusClient()

const { data: roles } = useQuery({
  queryKey: ['acl', 'roles'],
  queryFn: () => client!.get('/v1/users/acl/roles'),
})

const replaceUserRoles = useMutation({
  mutationFn: ({ userId, roleIds }: { userId: string; roleIds: string[] }) =>
    client!.put(`/v1/users/acl/users/${userId}/roles`, { roleIds }),
})

const createRoleQuery = useMutation({
  mutationFn: (payload: Record<string, unknown>) =>
    client!.post('/v1/users/acl/role-queries', payload),
})
```

Prefer role `uid` values returned by the API when sending `roleIds` for user-role updates or role-query payloads.

### Saved data grid views

Use `DataGridViewSelect` as the default saved-view UI for Docyrus grids, and persist those views with `createDataViewClient(client, appId)`.

- `DataGridViewSelect` is the default component for showing and editing saved grid views.
- Pass the TanStack table instance via `table` so the selector/editor can read column definitions.
- Pass `fields` when you want the built-in filter builder enabled in the editor.
- Back `views`, `onViewCreate`, `onViewSave`, `onViewDelete`, `onViewHide`, and `onViewUnhide` with `DataViews` CRUD.
- Use `DataGridViewEditor` separately only when you need a standalone editor outside the selector.

### Dev tooling with `@docyrus/devtools`

Wire `@docyrus/devtools` into every Docyrus app. It mounts a floating trigger and a docked panel with `Network`, `Errors`, `Issues`, `Explore`, `Console`, and `Messages` tabs — instrumenting `@docyrus/api-client` requests (with duplicate/slow detection), TanStack Query issues, console output, and iframe ↔ host `postMessage` traffic, plus an OpenAPI-driven request explorer that replays endpoints through the app's authenticated client.

```tsx
import { DocyrusDevtools, useRegisterDocyrusClient } from '@docyrus/devtools'
import { DocyrusAuthProvider, useDocyrusClient } from '@docyrus/signin'

function RegisterDevtoolsClient() {
  // Register the auth-managed client once it exists so requests are instrumented.
  useRegisterDocyrusClient(useDocyrusClient())
  return null
}

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      {/* enabled gates ALL instrumentation — keep devtools out of production */}
      <DocyrusDevtools enabled={import.meta.env.DEV} queryClient={queryClient} openApiSpecPath="/openapi-spec.json">
        <DocyrusAuthProvider apiUrl={/* … */} clientId={/* … */}>
          <RegisterDevtoolsClient />
          <YourRoutes />
        </DocyrusAuthProvider>
      </DocyrusDevtools>
    </QueryClientProvider>
  )
}
```

- Gate it with `enabled={import.meta.env.DEV}` (or ship it dormant behind `activationShortcut` for on-demand use in production). When `enabled` is `false`, it only renders children — no listeners, no UI.
- Pass the existing `queryClient` so TanStack Query issues surface in the `Issues` tab.
- Call `useRegisterDocyrusClient(useDocyrusClient())` after mount so the auth-managed client is instrumented as soon as it exists.
- Point `openApiSpecPath` at the served spec to enable the request explorer.

See `references/devtools-readme.md` for the full prop list, host bridge, iframe-message tap, DOM element picker, and the CDP/agent in-page API.

## Critical App/Data Rules

1. **Always send `columns`** in `.list()` and `.get()` calls. Without it, only `id` is returned.
2. **Collections are React hooks** — call `useBaseProjectCollection()`, `useUsersCollection()`, and similar hooks inside React components.
3. **Data source endpoints are dynamic** — they only exist if the data source is defined in the tenant OpenAPI spec.
4. **Use `id` for `count` calculations**. Use actual field slugs for `sum`, `avg`, `min`, and `max`.
5. **Child query keys must appear in `columns`**.
6. **Formula keys must appear in `columns`**.
7. **Read the signed-in user from `useDocyrusAuth().user`** — `@docyrus/signin` already fetches it from `/v1/users/me` on sign-in (roles, permissions, `aclRules`, identity). Never add your own `/v1/users/me` call; use `refreshUser()` after a role change, and `useUsersCollection().getMyInfo()` only for profile fields not on the auth user.
8. **Initialize `TenantPreferences` once per app runtime** and create shared `dateUtils` / `numberUtils` instances from `@docyrus/app-utils`.
9. **Formatting functions from `@docyrus/app-utils` are regionalized** — do not hardcode locale, date format, decimal separator, thousand separator, or decimal precision when tenant preferences should drive them.
10. **Use `createAppConfigClient(client, appId)`** for the app's single persisted config document; `upsert` is the default write path.
11. **Use `createUserAppConfigClient(client, appId)`** for the current user's persisted config document scoped to an app (e.g. theme, layout preferences, sidebar state); `upsert` is the default write path.
12. **Use `createDataViewClient(client, appId)`** for saved grid-view CRUD.
13. **Use `DataViews` with `DataGridViewSelect`** to show, create, edit, reorder, hide, unhide, soft-delete, and hard-delete saved data grid views.
14. **`DataGridViewSelect` needs a TanStack table instance** and should receive `fields` when you want the built-in filter builder/editor experience.
15. **Data view creation requires `name` and `tenant_data_source_id`**.
16. **Use `dataViews.update(viewId, { archived: true })` for soft-delete** and `dataViews.remove(viewId)` only for irreversible hard-delete.
17. **Regenerate collections after schema changes** by rebuilding the tenant OpenAPI spec, downloading the latest `openapi.json`, and re-running the collection generator.
18. **ACL endpoints are usually raw-client integrations** — use `useDocyrusClient()` or `RestApiClient` for roles, user-role assignments, role queries, record sharing, and ownership transfer.
19. **Prefer role `uid` values** for ACL role writes, user-role `roleIds`, and role-query `roleIds`.
20. **Treat `PUT /v1/users/acl/users/:userId/roles` as full replacement** and `POST /v1/users/acl/users/:userId/roles` as additive.
21. **Send role-query `query` as raw JSON** and omit `tenantAppId` when `dataSourceId` is present; backend derives it.
22. **After deleting a role, invalidate dependent app queries** for role lists, user-role lists, role-query lists, and any UI that renders primary-role labels.
23. **Create one shared `InventoryClient` after sign-in and `load()` it** — pass that single instance to every app-utils client so they share the warmed cache; call `inventory.refresh()` on tenant switch.
24. **Mount `@docyrus/devtools` in dev builds** gated by `enabled={import.meta.env.DEV}`, pass the app's `queryClient`, and register the auth client with `useRegisterDocyrusClient(useDocyrusClient())`.

## Critical UI/UX Rules

1. **Always check preferred components first** before creating anything custom.
2. **Use `AwesomeCard` for dashboards** unless the user explicitly wants a different card style.
3. **Use animate-ui `Sidebar` for app layouts** unless another layout is requested.
4. **Prefer Recharts for charts** in general feature UI. **Exception — dashboards/analytics/reporting:** use the **docyrus-dashboard-design** skill, which mandates Bklit UI charts, `AwesomeCard` panels, and `AwesomeStats` KPI cards.
5. **Use icons in this order**: hugeicons, then fontawesome light, then lucide.
6. **Use `AwesomeDialog` for item create forms**.
   - Small/simple forms: `container="sheet"` with `side="right"`
   - Long/complex forms: `container="modal"` or `container="drawer"`
7. **Choose detail containers based on item complexity**.
   - Large items: dedicated page
   - Small items: `AwesomeDialog` right sheet
8. **All forms must use TanStack Form + the Docyrus form system**. Do not build feature forms with plain HTML forms or React Hook Form directly.
9. **Use `EditableRecordDetail` for inline editing** in item detail views.
10. **Always enable `trackChanges`** for editable detail and grid experiences.
11. **Use `DataGridViewSelect` for saved grid views** and back it with `DataViews` from `@docyrus/app-utils`.
12. **Prefer `DataGridViewEditor` only when you need a standalone grid-view editor** outside the selector component.

## Default UI Choices

| Use Case | Default Component | Library |
|----------|-------------------|---------|
| Item create form | `AwesomeDialog` | docyrus |
| Quick record create | `CreateRecordDialog` | docyrus |
| Item detail (small) | `AwesomeDialog` sheet right | docyrus |
| Item detail (large) | Dedicated page | — |
| Inline editing | `EditableRecordDetail` | docyrus |
| Dashboard card | `AwesomeCard` | docyrus |
| Stat dashboards | `AwesomeStats` | docyrus |
| App navigation | `Sidebar` | animate-ui |
| Data table | `DataTable` | diceui |
| Editable grid | `Data Grid` | docyrus |
| Grid saved views | `DataGridViewSelect` + `DataViews` | docyrus + `@docyrus/app-utils` |
| Forms | Docyrus form fields + TanStack Form | docyrus |
| Charts (general) | shadcn chart + Recharts | shadcn |
| Dashboards / analytics / KPIs | Bklit charts + `AwesomeCard` + `AwesomeStats` (see **docyrus-dashboard-design**) | bklit + docyrus |
| File upload | File Upload | diceui |
| Gantt/project scheduling | `Gantt` | docyrus |
| Resource scheduling | `ResourceSchedulerPanel` | docyrus |
| Team chat | `TeamChatChannel` | docyrus |
| AI interface | `DocyrusAgent` | docyrus |
| Pricing / quoting | `PricingEnginePanel` | docyrus |
| Analytics / pivot | `PivotGrid` | docyrus |

## Quick UI Patterns

### Item create form

```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right" size="default">
  <AwesomeDialogHeader title="Create Task" icon="far-plus" />
  <AwesomeDialogBody>
    <form.Field name="title">{(field) => <TextFormField field={field} label="Title" />}</form.Field>
    <form.Field name="status">{(field) => <SelectFormField field={field} label="Status" />}</form.Field>
  </AwesomeDialogBody>
  <AwesomeDialogFooter>
    <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleSubmit}>Create</Button>
  </AwesomeDialogFooter>
</AwesomeDialog>
```

### Item detail with inline editing

```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right" size="lg" fullscreenable>
  <AwesomeDialogHeader
    title="Task Detail"
    description="Review and edit task fields inline"
    headerButtons={<Button variant="outline" size="sm" onClick={switchToFullForm}>Edit All</Button>}
  />
  <AwesomeDialogBody>
    <EditableRecordDetail fields={fields} record={record} onSave={handleSave} trackChanges>
      <EditableRecordDetailField slug="title" />
      <EditableRecordDetailField slug="status" />
      <EditableRecordDetailField slug="assignee" />
      <EditableRecordDetailField slug="due_date" />
    </EditableRecordDetail>
  </AwesomeDialogBody>
</AwesomeDialog>
```

## TanStack Query Pattern

```typescript
function useProjects(params?: ICollectionListParams) {
  const { list } = useBaseProjectCollection()
  return useQuery({
    queryKey: ['projects', 'list', params],
    queryFn: () => list({ columns: PROJECT_COLUMNS, ...params }),
  })
}

function useCreateProject() {
  const { create } = useBaseProjectCollection()
  const qc = useQueryClient()
  return useMutation({
    mutationFn: (data: Record<string, unknown>) => create(data),
    onSuccess: () => {
      void qc.invalidateQueries({ queryKey: ['projects'] })
    },
  })
}
```

## Collection CRUD Methods

```typescript
const { list, get, create, update, delete: deleteOne, deleteMany } = useBaseProjectCollection()

list(params?: ICollectionListParams)
get(id, { columns })
create(data)
update(id, data)
deleteOne(id)
deleteMany({ recordIds })
```

API endpoint pattern: `/v1/apps/{appSlug}/data-sources/{slug}/items`

## Query Capabilities Summary

The `.list()` method supports:

- `columns`
- `filters`
- `filterKeyword`
- `orderBy`
- `limit` and `offset`
- `fullCount`
- `calculations`
- `formulas`
- `childQueries`
- `pivot`
- `expand`

## Component Installation Pattern

```bash
pnpm dlx shadcn@latest add button
pnpm dlx shadcn@latest add @diceui/data-table
pnpm dlx shadcn@latest add @animate-ui/sidebar
pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card
pnpm dlx shadcn@latest add @reui/file-upload-default
```

## References

For deep dives, read:

- `references/README.md` — merged reference map for app development and UI design
- `references/api-client-and-auth.md`
- `references/collections-and-patterns.md`
- `references/signin-readme.md` — full `@docyrus/signin` package README (auth provider, hooks, iframe/host bridge, SSR, roles & permissions). Published on npm as `@docyrus/signin`; source at `docyrus-devkit/packages/signin`
- `references/host-communication-integration.md` — embedded-app ↔ host shell `postMessage` integration, **excluding** Guidy: host→app navigation (`useDocyrusHostNavigation`) & notifications (`useDocyrusHostNotification`), app→host route sync (`syncRouteToHost` / `useDocyrusHostRouteSync` / `notifyRouteChange`) and navigation requests (`requestHostNavigation`), origin validation, Vue equivalents
- `references/guidy-integration.md` — make an embedded app Guidy-compatible (host AI assistant driving the app's UI): `enableGuidyBridge`, targetable controls (stable ids + accessible labels), and declared routes (`guidyRoutes` / `useDocyrusGuidyRoutes`); routes also need `useDocyrusHostNavigation` from the host-communication reference
- `references/app-utils-readme.md` — full `@docyrus/app-utils` package README (tenant preferences, date/number formatting, app/user config, data views/forms, data-source metadata, and the Inventory Cache). Published on npm as `@docyrus/app-utils`; source at `docyrus-devkit/packages/app-utils`
- `references/devtools-readme.md` — full `@docyrus/devtools` package README (in-app dev panel, request/console/message instrumentation, OpenAPI explorer, element picker, CDP/agent API). Published on npm as `@docyrus/devtools`; source at `docyrus-devkit/packages/devtools`
- the **docyrus-app-settings** skill — designing and using app-scoped (`AppConfig`) and per-user (`UserAppConfig`) settings documents with `createAppConfigClient` / `createUserAppConfigClient`: scope choice, wholesale-replace and empty-document gotchas, and the REST/CLI contract
- the **docyrus-acl-design** skill — ACL roles, permissions, and role-queries (`docyrus acl`); plus the ACL REST endpoints in **docyrus-api-dev**'s SKILL.md
- `../docyrus-api-dev/references/data-source-query-guide.md`
- `../docyrus-api-dev/references/formula-design-guide-llm.md`
- `../docyrus-api-dev/references/query-guide.md`
- `references/preferred-components-catalog.md`
- `references/component-selection-guide.md`
- `references/icon-usage-guide.md`
