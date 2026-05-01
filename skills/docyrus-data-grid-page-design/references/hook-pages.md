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

  const { users } = useUsers();
  const { formatDate, formatDateTime, formatNumber } = useGridFormatters();

  const { table, gridProps, toolbar, resolvedListParams } = useDocyrusDataGrid<OrganizationRow>({
    client,
    appSlug: 'crm',
    dataSourceSlug: 'organization',
    actionsColumn,
    users,
    formatDate,
    formatDateTime,
    formatNumber,
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

### Page shell with `vertical-tabs` view picker

When `viewSelectVariant === 'vertical-tabs'`, render `sidePanel` to the left of the grid:

```tsx
const { table, gridProps, toolbar, sidePanel } = useDocyrusDataGrid<Row>({
  client, appSlug, dataSourceSlug,
  viewSelectVariant: 'vertical-tabs'
});

return (
  <div className="flex h-full flex-col overflow-hidden">
    <div className="shrink-0">{toolbar}</div>
    <div className="flex min-h-0 flex-1">
      {sidePanel}
      <div className="min-h-0 flex-1">
        <DataGrid table={table} {...gridProps} height="auto" />
      </div>
    </div>
  </div>
);
```

`sidePanel` is `null` for `'horizontal-tabs'` and `'dropdown'` — rendering it conditionally is safe.

## Data modes

1. `data`: pass pre-resolved rows when another part of the page owns fetching.
2. `collection`: pass a generated collection hook; the hook calls `collection.list(resolvedListParams)`.
3. direct API: pass only `client`, `appSlug`, and `dataSourceSlug`.

Use `onReload` when you use `data` mode, because the hook cannot refetch rows on its own there.

## High-value options

### Columns and toolbar

- `actionsColumn`: add per-row actions right after the select column.
- `extraColumns`: prepend custom columns before metadata-generated Docyrus fields.
- `mapColumn`: override or skip generated field columns (`(field, defaultColumn) => ColumnDef | null`).
- `defaultRowGroupingColumn`: seed a default grouping for views that do not define one.
- `systemViews`: add static developer-defined views before saved backend views.
- `viewSelectVariant` (default `'horizontal-tabs'`): view picker layout. `'dropdown'` → compact `<Select>`-style trigger for tight toolbars. `'vertical-tabs'` → moves the picker out of the toolbar into a side panel (see "Side panel" in **Hook result** below); the consumer renders `sidePanel` to the left of the grid.
- `viewSelectMaxVisible?: number`: caps inline tabs in the `'horizontal-tabs'` variant; overflow collapses into the active view's overflow menu. Ignored for `'dropdown'` / `'vertical-tabs'`.
- `enableViewSelect`, `enableSearchInput`, `enableFilterMenu`, `enableGroupMenu`, `enableSortMenu`, `enableRowHeightMenu`, `enableDisplayMenu`, `enableReloadButton`, `enableServerExportMenu`: trim the standard toolbar (all default `true`).
- `showSelectColumn` (default `true`), `enableRowMarkers` (default `true`): control the left-most reserved column.
- `selectColumn`: override `getDataGridSelectColumn` entirely.
- `searchPlaceholder` (default `"Search…"`), `searchDebounceMs` (default `300`), `toolbarClassName`.
- `enableSearch` (default `false`), `enableGrouping` (default `true`), `readOnly` (default `true`) — forwarded to `useDataGrid`.

### Query shape

- `listParams`: append query params like `limit`, `fullCount`, `expand`, or custom backend flags. `listParams` overrides win against the hook's defaults, so use this to pin paging or add tenant-specific flags.
- `defaultLimit` (default `100`): page size used when no `limit` is supplied via `listParams`.
- `enableItemsQuery` (default `true` when no `data` prop is given): set `false` to wire the toolbar without fetching rows.

### Inline change tracking

- `trackChanges` (default `true`): enables the save/discard banner above the grid for inline cell edits.
- `onSaveChanges?: (changes: Array<RowChange>, data: Array<TData>) => void | Promise<void>`: custom save handler. By default the hook batches changes into a single `PATCH /items/bulk` call (with `enableAutomation: false`, `enableChangeLogging: false`) using `collection.updateMany` when supplied, otherwise the direct items endpoint.

### Bulk actions (selection bar)

- `bulkActions` (default `['update', 'delete', 'export']`): array of built-in bulk actions to surface when rows are selected. Pass `false` or `[]` to hide the built-ins (custom `extraBulkActions` still render if provided).
  - `update` → opens the `BulkUpdateDialog`. Calls `collection.updateMany` when present, otherwise the direct items endpoint.
  - `delete` → opens `RecordDeleteConfirmDialog`. Calls `collection.deleteMany` when present, otherwise the direct items endpoint.
  - `export` → opens the export menu for the selection (subset of the toolbar export).
- `extraBulkActions?: Array<DataGridAction<TData>>`: extra row-selection actions appended after the built-ins. Each entry's `onAction(selectedRows)` (or `render(selectedRows)`) receives the currently selected rows so handlers can drive their own logic. Use for app-specific operations like "Send email", "Assign owner", etc.

### Toolbar slots

- `toolbarStartContent?: ReactNode`: prepended to the left side of the built-in toolbar (next to view tabs and search). Use for app-specific filters or buttons that should sit alongside standard controls.
- `toolbarEndContent?: ReactNode`: appended to the right side of the built-in toolbar (after the reload button). Use for app-specific actions like "Add record", "Refresh metrics", etc.

### Server export

- `enableServerExportMenu` (default `true`): toolbar export dropdown that pulls all rows matching the active view's filters/keyword from `POST /v1/edge/run/query-export`.
- `serverExportLimit` (default `10000`): row cap forwarded to the server export endpoint.
- `serverExportExcludedFieldTypes`: field types to skip when picking export columns. Defaults to virtual / non-stored types like `field-action`.
- `serverExportExcludedSlugs`: field slugs to skip when picking export columns. Defaults to internal metadata columns mirroring the legacy Vue exporter (`data`, `document`, `parent_data_source_id`, `parent_record_id`, `icon`, `color`, `mentions`, `followers`, `type`, `tenant_view_id`, `tenant_data_source_id`, `sort_order`, `editor_view_id`).
- `exportColumns` (default `'visible'`): `'visible'` uses the table's currently-visible columns; `'all'` includes every data-source field; an explicit `Array<string>` of slugs picks specific fields in order.
- `exportFileName`: file name (without extension) for exports. Defaults to `dataSourceSlug`.

### Lifecycle hooks

- `onReload?: () => void`: called after the reload button's internal `refetch` (data source + views + items) runs.

### Tenant-aware formatters

- `formatDate?: (value) => string` — wired into `DateCell` display.
- `formatDateTime?: (value) => string` — wired into `DateTimeCell` display.
- `formatNumber?: (value, opts?: { variant?: 'number' | 'currency' | 'percent'; currency?: string }) => string` — wired into `NumberCell` / `CurrencyCell` / `PercentCell` display.

Build these from `@docyrus/app-utils` (`createDateUtils` + `createNumberUtils` over `getTenantPreferences`). See `tenant-and-users-providers.md` for the standard provider pattern.

### Shared users list

- `users?: ReadonlyArray<CellUserOption>` — seeds the static option list for `field-userSelect` and `field-userMultiSelect` cells.
  - When supplied, those cells render avatars + labels from this list immediately.
  - When not supplied, cells fall back to the row's expanded `{ id, name, photo }` payload (which is why the auto-expand for reference fields matters — see below).
  - Type with `import { type CellUserOption } from '@docyrus/ui/components/data-grid'`.

## How backend query params are derived

The hook builds `resolvedListParams` from the active view and toolbar state:

- `columns`: visible field slugs, with `id` always first.
- `orderBy`: mapped from `view.sorting`.
- `filters`: AND-merge of `view.filterQuery` (saved view) and the toolbar filter menu's transient state. Toolbar filter changes refetch the items query automatically (no extra wiring needed in standard mode).
- `filterKeyword`: mapped from the debounced toolbar search input.
- `expand`: every visible reference-type column slug is added automatically. Covered types: `field-userSelect`, `field-userMultiSelect`, `field-relation`, `field-relatedField`, `field-select`, `field-radioGroup`, `field-enum`, `field-systemEnum`, `field-multiSelect`, `field-tagSelect`, `field-status`, `field-approvalStatus`.
- `limit` / `offset`: default to `100` / `0`, then `listParams` wins.
- grouping column safety: the active grouping field is appended even if the saved view hides it.

Inspect `resolvedListParams` when you need debugging, analytics, export, or a "copy query" action.

## Filter menu (`DataGridFilterMenu`) behavior

The hook's toolbar wires the filter menu so it pulls live data from the backend instead of only filtering loaded rows.

- Toolbar filter additions/changes trigger a refetch with the merged `filters` payload.
- Filter changes reset `pageIndex` to 0 so a stale offset can't land out of bounds.
- Toolbar filters are session-transient and clear when the active view changes.
- Field type to filter type mappings:
  - `field-userSelect`, `field-userMultiSelect`, `field-relation`, `field-relatedField`, `field-multiSelect`, `field-tagSelect` → `multiOption` (server `in` operator), with debounced async option search.
  - `field-status`, `field-approvalStatus`, `field-enum`, `field-systemEnum`, `field-select`, `field-radioGroup` → `option` with the field's static enum options.
  - `field-checkbox`, `field-switch` → `boolean` (true / false / empty).
  - `field-date`, `field-dateTime`, `field-dateRange` → `date` with `is between` for ranges.
  - `field-money`, `field-currency`, `field-percent`, `field-rating`, `field-duration`, `field-number`, `field-autonumber` → `number`.
  - `field-file`, `field-image`, `field-chart`, etc. are excluded from the filter menu entirely.
- Async option search calls `/v1/users` for user fields and `/v1/apps/{appSlug}/data-sources/{slug}/items` for relations. The hook caches the data-sources lookup needed to resolve `relationDataSourceId` (UUID) to the URL slugs, valid for 5 minutes.

## Reserved columns

`useDocyrusDataGrid` always:

- Re-prepends `select` and `actions` to the column order after a saved view is applied (saved views only know real field slugs).
- Pins both reserved columns to the left so they stay visible during horizontal scroll.

You don't need to configure pinning yourself; the hook merges with whatever the saved view declares.

## Cell variants for reference fields

Out of the box (no `mapColumn` needed):

- `field-userSelect` → `UserCell` with avatar + name. If a static `users` list is passed, options come from there; otherwise the cell reads the row's expanded `{ id, name, photo }` payload directly.
- `field-userMultiSelect` → `UserMultiSelectCell` (avatar stack). Same options/expanded-payload fallback.
- `field-relation`, `field-relatedField` → `RelationCell` showing the related record's name (from `expand`).
- `field-status`, `field-approvalStatus` → `StatusCell` with color + icon from the field's enums.
- `field-select`, `field-radioGroup` → `SelectCell` with color/icon support.
- `field-multiSelect`, `field-tagSelect` → multi-select cells with chips.
- `field-identity` → `UuidCell`. Defaults to `showCopyButton: true` (60px fixed-width icon-only column). Override `mapColumn` to set `showCopyButton: false` for a 300px full-text column.
- `field-url` → `UrlCell` rendering as a link with `target="_blank"`.
- `field-money`, `field-currency`, `field-percent`, `field-number` → `NumberCell` family. Honor the `formatNumber` option for tenant-aware locale formatting.
- `field-date`, `field-dateTime` → `DateCell` / `DateTimeCell`. Honor `formatDate` / `formatDateTime`.

Use `mapColumn` only when you need to override these defaults.

## Hook result

`useDocyrusDataGrid<TData>` returns:

- `table` — TanStack Table instance. Pass to `<DataGrid>` and any toolbar building blocks.
- `gridProps` — spread onto `<DataGrid table={table} {...gridProps} />`. Includes `actions: Array<DataGridAction<TData>>` so the floating selection bar lights up automatically when `bulkActions` are enabled.
- `toolbar` — pre-wired toolbar `ReactNode` ready to render above the grid.
- `sidePanel` — pre-wired side panel `ReactNode`. `null` for `'horizontal-tabs'` and `'dropdown'` view variants. When `viewSelectVariant === 'vertical-tabs'`, it returns a `<DataGridSidePanel>` containing the vertical view picker — render it to the left of `<DataGrid>` (e.g. inside a flex row).
- `items: Array<TData>` — resolved rows from `data`, `collection.list()`, or the direct items fetch.
- `resolvedListParams: DocyrusDataGridListParams` — final params sent to the backend (after merging view state, search, and `listParams`). Use for export/analytics/copy-query.
- `pagingMode: 'standard' | 'virtual-scroll' | undefined` — resolved paging mode for the active view. Pass to `<DataGrid pagingMode>` so the standard footer renders only when the view enables it.
- `reload: () => void` — triggers `refetch()` (data source + views + items) and the optional `onReload` callback.
- Plus everything from `useDocyrusDataViewSelect` except `gridViewSelectProps`: `views`, `fields`, `dataSource`, `activeViewId`, `setActiveViewId`, `isLoading`, `error`, `refetch`.

## Important behavior

- `gridProps` already carries the active view's paging settings. Usually you should just spread it into `<DataGrid>`.
- Backend-saved views store field slugs, not reserved columns like `select` or `actions`. The hook re-prepends and re-pins those reserved columns for you.
- In `data` mode the search box becomes client-side global filtering. In backend modes it becomes `filterKeyword`.
- The hook controls TanStack `columnFilters` state. If you build a manual page, replicate this so toolbar filter changes can drive your row query.
- `collection.updateMany` / `collection.deleteMany` are required for bulk update/delete to use the collection layer; otherwise the hook falls back to the direct items endpoints.

## Default recommendation

If the page is a CRUD-style Docyrus list page, start here first. Drop to manual composition only when the toolbar, saved-view lifecycle, or row fetching needs to diverge from the standard behavior.

For system views, hidden views, paging ownership, or translating active views into a custom query pipeline, also read `advanced-saved-view-query-patterns.md`.
