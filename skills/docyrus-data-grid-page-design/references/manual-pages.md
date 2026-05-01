# Manual DataGrid Page Patterns

## Use this path when

- The page uses local/demo data.
- You need a custom toolbar layout.
- You want backend-saved views but your own row-query lifecycle.
- You only need local views and not Docyrus view persistence.

## Local views with full manual composition

```tsx
'use client';

import { useEffect, useMemo, useState } from 'react';

import {
  DataGrid,
  DataGridFilterMenu,
  DataGridGroupMenu,
  DataGridRowHeightMenu,
  DataGridSortMenu,
  applyViewToTable,
  getDataGridActionsColumn,
  getDataGridSelectColumn,
  useDataGrid,
  type ColumnDef,
  type SavedDataGridView
} from '@docyrus/ui/components/data-grid';
import { DataGridViewSelect } from '@docyrus/ui/components/data-grid-view-select';

const columns: ColumnDef<Row>[] = [
  getDataGridSelectColumn<Row>(),
  getDataGridActionsColumn<Row>({ cell: ({ row }) => <OpenButton row={row.original} /> }),
  ...dataColumns
];

export function TasksPage() {
  const [views, setViews] = useState<SavedDataGridView[]>(INITIAL_VIEWS);
  const [activeViewId, setActiveViewId] = useState('all');
  const activeView = useMemo(() => views.find(view => view.id === activeViewId), [views, activeViewId]);

  const { table, ...gridProps } = useDataGrid<Row>({
    data: rows,
    columns,
    enableSearch: true,
    enableGrouping: true
  });

  useEffect(() => {
    if (!activeView) return;
    applyViewToTable(table, activeView);
  }, [table, activeView]);

  return (
    <div className="flex h-full flex-col gap-4 overflow-hidden">
      <div className="shrink-0 flex items-center gap-2">
        <DataGridViewSelect
          table={table}
          variant="horizontal-tabs"
          editable
          views={views}
          activeViewId={activeViewId}
          onViewChange={(view) => setActiveViewId(view.id)}
          onViewSave={(view) => setViews(prev => prev.map(item => item.id === view.id ? view : item))}
          onViewCreate={(view) => setViews(prev => [...prev, view])}
          onViewDelete={(viewId) => setViews(prev => prev.filter(item => item.id !== viewId))} />
        <DataGridFilterMenu table={table} />
        <DataGridGroupMenu table={table} />
        <DataGridSortMenu table={table} />
        <DataGridRowHeightMenu table={table} />
      </div>
      <div className="min-h-0 flex-1">
        <DataGrid table={table} {...gridProps} height="auto" />
      </div>
    </div>
  );
}
```

`DataGridViewSelect` applies views automatically when the user clicks a different tab. The `useEffect` above is still needed for the initial/default view.

## Backend saved views with custom row fetching

```tsx
const viewSelect = useDocyrusDataViewSelect({
  client,
  appSlug: 'crm',
  dataSourceSlug: 'contacts',
  defaultRowGroupingColumn: 'status',
  systemViews: [ALL_CONTACTS_VIEW]
});

const activeView = useMemo(
  () => viewSelect.views.find(view => view.id === viewSelect.activeViewId),
  [viewSelect.views, viewSelect.activeViewId]
);

const { table, ...gridProps } = useDataGrid<Row>({ data: rows, columns, enableGrouping: true });

useEffect(() => {
  if (!activeView) return;
  applyViewToTable(table, activeView);
}, [table, activeView]);

<DataGridViewSelect table={table} editable {...viewSelect.gridViewSelectProps} />
```

Important: `useDocyrusDataViewSelect` gives you `views`, `fields`, `activeViewId`, CRUD callbacks, hidden-view state, and persistence. It does **not** fetch rows. If the active view should also shape the backend query, translate `activeView.columnVisibility`, `columnOrder`, `sorting`, and `filterQuery` into your request yourself or switch to `useDocyrusDataGrid`.

## Manual toolbar building blocks

Use these when you do not want the prebuilt hook toolbar:

- `DataGridViewSelect`
- `DataGridFilterMenu`
- `DataGridSortMenu`
- `DataGridGroupMenu`
- `DataGridRowHeightMenu`
- `DataGridDisplayMenu`
- `DataGridViewMenu`
- your own search input wired to `table.setGlobalFilter(...)` or to backend query state

### Extra toolbar buttons + selection actions

- `<DataGridToolbar startContent={...} endContent={...}>` accepts arbitrary nodes prepended/appended to the standard menus. Use this when composing the toolbar yourself.
- `<DataGrid actions={[{ label, icon, onAction(selectedRows) }, ...]}>` renders custom buttons in the floating action bar when rows are selected. Each entry can supply either `onAction(selectedRows)` (simple `<Button>`) or `render(selectedRows)` (full custom node, e.g. a dropdown). `DataGridAction<TData>` is exported from `@docyrus/ui/components/data-grid`.

For persistence, system views, paging ownership, or manual translation of the active view into Docyrus query params, also read `advanced-saved-view-query-patterns.md`.

## What you lose by going manual

`useDocyrusDataGrid` does several things automatically that manual mode does **not**:

- **Auto-`expand`** for reference fields. In manual mode you must add reference field slugs to your request's `expand` array yourself, or rich cells (user / relation / select / status) will only see bare IDs.
- **Reserved-column pinning.** `select` and `actions` are not auto-pinned to the left in manual mode — call `table.setColumnPinning({ left: ['select', 'actions'], right: [] })` after applying the active view.
- **Filter menu refetch wiring.** `DataGridFilterMenu` writes through to `table.setColumnFilters`, but a manual page's row query doesn't watch that state. You need to read `table.getState().columnFilters` (or lift the state into the host) and translate the filter values into your request's `filters` payload. Reset `pageIndex` to 0 on changes.
- **Async option search for relation/user filter columns.** The filter menu will show empty option lists unless you pass a `getAsyncOptions` resolver (signature: `(column) => AsyncOptionsConfig | undefined`) that calls `/v1/users` for user fields and `/v1/apps/{appSlug}/data-sources/{slug}/items` for relations.
- **Tenant-aware formatters.** Pass `formatDate`, `formatDateTime`, `formatNumber` through `useDataGrid({ meta: { ... } })`. Cells read them from `tableMeta`.
- **Shared users list.** Pass `users: ReadonlyArray<CellUserOption>` to `buildTanstackColumnDef({ users })` (or set the `cell.options` directly via `mapColumn`) so user cells get avatar + label from the global list.

If any of these matter to the page, prefer `useDocyrusDataGrid` and use `mapColumn` / `extraColumns` / toolbar enable flags to customize.

## Gotchas

- Local/manual `SavedDataGridView` objects can include reserved columns like `select` and `actions` in `columnOrder`.
- Backend Docyrus-saved views store real field slugs only. If you want the standard reserved-column behavior, `useDocyrusDataGrid` is the safest path.
- Pass `fields` into `DataGridViewSelect` when you want the filter builder inside the editor.
- Pass `isSaving` and `isLoading` when the host owns async view CRUD.
- For manual Docyrus item requests, always send `columns`. Add reference-type field slugs (`field-userSelect`, `field-userMultiSelect`, `field-relation`, `field-relatedField`, `field-select`, `field-radioGroup`, `field-enum`, `field-systemEnum`, `field-multiSelect`, `field-tagSelect`, `field-status`, `field-approvalStatus`) to `expand` so the API returns `{ id, name, ... }` payloads.
