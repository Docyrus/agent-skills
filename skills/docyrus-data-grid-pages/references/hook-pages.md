# Hook-first Docyrus DataGrid Pages

## Use this path when

- The page is a normal Docyrus records/index page.
- Rows come from a Docyrus items endpoint or a generated collection hook.
- Saved views should drive visible columns, sorting, filters, grouping, paging, and toolbar state.
- You want the fastest path to a production page.

## Minimal page shell

```tsx
'use client';

import { useMemo } from 'react';

import { useDocyrusAuth } from '@docyrus/signin';
import {
  DataGrid,
  getDataGridActionsColumn,
  type ColumnDef
} from '@docyrus/ui/components/data-grid';
import { useDocyrusDataGrid } from '@docyrus/ui/library/hooks/use-docyrus-data-grid';
import { Button } from '@docyrus/ui/primitives/ui/button';

type OrganizationRow = { id: string; name?: string };

export function OrganizationsPage() {
  const { client } = useDocyrusAuth();

  if (!client) return null;

  return <OrganizationsPageInner client={client} />;
}

function OrganizationsPageInner({ client }: { client: NonNullable<ReturnType<typeof useDocyrusAuth>['client']> }) {
  const actionsColumn = useMemo<ColumnDef<OrganizationRow>>(
    () => getDataGridActionsColumn<OrganizationRow>({
      cell: ({ row }) => (
        <Button size="sm" variant="ghost" onClick={() => { void row.original.id; }}>
          Open
        </Button>
      )
    }),
    []
  );

  const { table, gridProps, toolbar, resolvedListParams } = useDocyrusDataGrid<OrganizationRow>({
    client,
    appSlug: 'crm',
    dataSourceSlug: 'organization',
    actionsColumn,
    listParams: { limit: 50 },
    defaultRowGroupingColumn: 'status'
  });

  return (
    <div className="flex h-full flex-col gap-4 overflow-hidden px-6 py-5">
      <div className="shrink-0">{toolbar}</div>
      <div className="min-h-0 flex-1">
        <DataGrid table={table} {...gridProps} height="auto" />
      </div>
    </div>
  );
}
```

If a generated collection already exists, pass `collection` to `useDocyrusDataGrid` and let the hook call `collection.list(resolvedListParams)` instead of the direct items endpoint.

## Data modes

1. `data`: pass pre-resolved rows when another part of the page owns fetching.
2. `collection`: pass a generated collection hook; the hook calls `collection.list(resolvedListParams)`.
3. direct API: pass only `client`, `appSlug`, and `dataSourceSlug`.

Use `onReload` when you use `data` mode, because the hook cannot refetch rows on its own there.

## High-value options

- `actionsColumn`: add per-row actions right after the select column.
- `extraColumns`: prepend custom columns before metadata-generated Docyrus fields.
- `mapColumn`: override or skip generated field columns.
- `listParams`: append query params like `limit`, `fullCount`, `expand`, or custom backend flags.
- `defaultRowGroupingColumn`: seed a default grouping for views that do not define one.
- `systemViews`: add static developer-defined views before saved backend views.
- `enableViewSelect`, `enableSearchInput`, `enableFilterMenu`, `enableGroupMenu`, `enableSortMenu`, `enableRowHeightMenu`, `enableDisplayMenu`, `enableReloadButton`: trim the standard toolbar.
- `showSelectColumn`, `enableRowMarkers`: control the left-most reserved column.

## How backend query params are derived

The hook builds `resolvedListParams` from the active view and toolbar state:

- `columns`: visible field slugs, with `id` always first.
- `orderBy`: mapped from `view.sorting`.
- `filters`: mapped from `view.filterQuery`.
- `filterKeyword`: mapped from the debounced toolbar search input.
- `limit` / `offset`: default to `100` / `0`, then `listParams` wins.
- grouping column safety: the active grouping field is appended even if the saved view hides it.

Inspect `resolvedListParams` when you need debugging, analytics, export, or a “copy query” action.

## Important behavior

- `gridProps` already carries the active view's paging settings. Usually you should just spread it into `<DataGrid>`.
- Backend-saved views store field slugs, not reserved columns like `select` or `actions`. The hook re-prepends those reserved columns for you.
- In `data` mode the search box becomes client-side global filtering. In backend modes it becomes `filterKeyword`.
- Relation and user-ish field types need extra metadata; use `mapColumn` when you want richer rendering than the fallback short-text cell.

## Default recommendation

If the page is a CRUD-style Docyrus list page, start here first. Drop to manual composition only when the toolbar, saved-view lifecycle, or row fetching needs to diverge from the standard behavior.

For system views, hidden views, paging ownership, or translating active views into a custom query pipeline, also read `advanced-saved-view-query-patterns.md`.
