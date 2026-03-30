---
name: docyrus-app-dev-react
description: Build Docyrus React TypeScript web applications end-to-end, combining authentication, API access, generated collections, TanStack Query/Form patterns, and production-grade UI implementation with preferred component libraries. Use when creating or modifying Docyrus-backed apps that use @docyrus/api-client, @docyrus/signin, Docyrus collections, queries, dashboards, forms, tables, layouts, or Docyrus UI components.
---

# Docyrus App Dev React

Build Docyrus React TypeScript applications end-to-end. This skill combines app architecture, authentication, data access, query patterns, and production-grade UI guidance in one place.

## Tech Stack

- React 19 + TypeScript + Vite
- TanStack Router (code-based), TanStack Query (server state), TanStack Form
- Tailwind CSS v4, shadcn/ui components
- `@docyrus/api-client` + `@docyrus/signin`
- Auto-generated collections from OpenAPI spec
- Preferred UI libraries: shadcn, diceui, animate-ui, docyrus-ui, reui

## When to Use This Skill

Use this skill when you are:

- Building or modifying a Docyrus-backed React app
- Setting up authentication with `@docyrus/signin`
- Fetching or mutating data with generated collections or `@docyrus/api-client`
- Building record sharing, role management, or ACL-driven UI flows
- Designing feature UIs such as dashboards, forms, tables, layouts, dialogs, analytics, or detail pages
- Selecting between shadcn, diceui, animate-ui, docyrus-ui, and reui components
- Implementing complete feature flows that combine data access and polished UI

## End-to-End Feature Workflow

1. Set up app auth, routing, and query providers.
2. Use generated Docyrus collection hooks or the REST client for data access.
3. Define `columns`, filters, formulas, child queries, and mutations correctly.
4. Check preferred UI components before building anything custom.
5. Use Docyrus form and detail patterns for create, edit, and item detail flows.
6. Connect UI actions to TanStack Query mutations and invalidate relevant queries.

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

## Critical App/Data Rules

1. **Always send `columns`** in `.list()` and `.get()` calls. Without it, only `id` is returned.
2. **Collections are React hooks** — call `useBaseProjectCollection()`, `useUsersCollection()`, and similar hooks inside React components.
3. **Data source endpoints are dynamic** — they only exist if the data source is defined in the tenant OpenAPI spec.
4. **Use `id` for `count` calculations**. Use actual field slugs for `sum`, `avg`, `min`, and `max`.
5. **Child query keys must appear in `columns`**.
6. **Formula keys must appear in `columns`**.
7. **Use `useUsersCollection().getMyInfo()`** for current user profile instead of making a direct profile call.
8. **Regenerate collections after schema changes** by rebuilding the tenant OpenAPI spec, downloading the latest `openapi.json`, and re-running the collection generator.
9. **ACL endpoints are usually raw-client integrations** — use `useDocyrusClient()` or `RestApiClient` for roles, user-role assignments, role queries, record sharing, and ownership transfer.
10. **Prefer role `uid` values** for ACL role writes, user-role `roleIds`, and role-query `roleIds`.
11. **Treat `PUT /v1/users/acl/users/:userId/roles` as full replacement** and `POST /v1/users/acl/users/:userId/roles` as additive.
12. **Send role-query `query` as raw JSON** and omit `tenantAppId` when `dataSourceId` is present; backend derives it.
13. **After deleting a role, invalidate dependent app queries** for role lists, user-role lists, role-query lists, and any UI that renders primary-role labels.

## Critical UI/UX Rules

1. **Always check preferred components first** before creating anything custom.
2. **Use `AwesomeCard` for dashboards** unless the user explicitly wants a different card style.
3. **Use animate-ui `Sidebar` for app layouts** unless another layout is requested.
4. **Prefer Recharts for charts**. shadcn chart primitives are the default wrapper.
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
| Grid saved views | `DataGridViewSelect` | docyrus |
| Forms | Docyrus form fields + TanStack Form | docyrus |
| Charts | shadcn chart + Recharts | shadcn |
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
- `../docyrus-api-dev/references/acl-endpoints-frontend.md`
- `../docyrus-api-dev/references/data-source-query-guide.md`
- `../docyrus-api-dev/references/formula-design-guide-llm.md`
- `../docyrus-api-dev/references/query-guide.md`
- `references/preferred-components-catalog.md`
- `references/component-selection-guide.md`
- `references/icon-usage-guide.md`
