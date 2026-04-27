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

For persistence, system views, paging ownership, or manual translation of the active view into Docyrus query params, also read `advanced-saved-view-query-patterns.md`.

## Gotchas

- Local/manual `SavedDataGridView` objects can include reserved columns like `select` and `actions` in `columnOrder`.
- Backend Docyrus-saved views store real field slugs only. If you want the standard reserved-column behavior, `useDocyrusDataGrid` is the safest path.
- Pass `fields` into `DataGridViewSelect` when you want the filter builder inside the editor.
- Pass `isSaving` and `isLoading` when the host owns async view CRUD.
- For manual Docyrus item requests, always send `columns`.
