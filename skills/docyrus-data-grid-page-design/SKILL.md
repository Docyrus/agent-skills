---
name: docyrus-data-grid-page-design
description: Build Docyrus React data-grid and record-list pages with `DataGrid`, `DataGridViewSelect`, `useDataGrid`, `useDocyrusDataViewSelect`, and `useDocyrusDataGrid`. Use when asked to build or refactor a list page, records table, CRM or ERP grid, saved-view tabs, row actions, manual DataGrid toolbar composition, or Docyrus data-source grid queries with filtering, sorting, grouping, search, display modes, paging, and reload behavior.
---

# Docyrus Data Grid Page Design

Build full Docyrus list pages around the web `DataGrid` stack.

## Choose the build mode

1. **Standard Docyrus page** → use `useDocyrusDataGrid`.
   - Best when rows come from a Docyrus data source or generated collection.
   - Gives you metadata-driven columns, saved views, toolbar wiring, search, filters (with async option search for relation/user fields), grouping, sorting, row-height, display mode, paging, and reload.
   - Auto-wires reference field expansion, tenant-aware formatters, and a shared users list when supplied.
   - Built-in inline change tracking + save banner (`trackChanges`, `onSaveChanges`).
   - Built-in selection-bar bulk actions (`bulkActions`: `update`, `delete`, `export`) with `BulkUpdateDialog` + `RecordDeleteConfirmDialog`.
   - Built-in server-side export menu (`enableServerExportMenu`, `exportColumns`, `exportFileName`, `serverExportLimit`).
   - Read `references/hook-pages.md`.

2. **Custom layout or custom row-query lifecycle** → use `useDocyrusDataViewSelect` + `useDataGrid`.
   - Best when you need a custom toolbar arrangement, local/demo data, or a non-standard fetch cycle.
   - Read `references/manual-pages.md`.

3. **No backend saved views** → use `useDataGrid` with local `SavedDataGridView[]` or `DataGridViewMenu`.
   - Also covered in `references/manual-pages.md`.

4. **Complex Docyrus query payloads** → also load the `docyrus-api-dev` skill.
5. **Advanced saved-view persistence or manual view-driven queries** → read `references/advanced-saved-view-query-patterns.md`.
6. **Tenant-aware formatters and shared users list** → read `references/tenant-and-users-providers.md`.

## Default page workflow

1. Confirm the `appSlug`, `dataSourceSlug`, and whether a generated collection already exists.
2. Pick hook mode or manual mode.
3. App-level: ensure a `TenantProvider` (preferences → date/number utils) and `UsersProvider` (`/v1/users` cache) wrap the route, and pull `formatDate` / `formatDateTime` / `formatNumber` / `users` into the grid hook from those contexts.
4. Add reserved columns in this order: select first, actions second. They are pinned left automatically by `useDocyrusDataGrid`.
5. Keep the page in a flex column layout with the grid body wrapped in `min-h-0 flex-1`.
6. Add create/edit/view/delete dialogs around row actions.
7. Verify initial saved view, search, filters, grouping, sorting, paging, and reload behavior.

## Non-negotiables

- Render page-sized grids with the toolbar in a shrink-0 row and the grid inside `min-h-0 flex-1`.
- Prefer `<DataGrid table={table} {...gridProps} height="auto" />` for full-page layouts.
- Pass the TanStack `table` instance to every grid menu and to `DataGridViewSelect`.
- `useDocyrusDataViewSelect` manages view metadata only. It does **not** fetch rows and does **not** accept `table`.
- Manual Docyrus item queries must always send `columns`. Reference fields (`field-userSelect`, `field-userMultiSelect`, `field-relation`, `field-relatedField`, `field-select`, `field-radioGroup`, `field-enum`, `field-systemEnum`, `field-multiSelect`, `field-tagSelect`, `field-status`, `field-approvalStatus`) must also be added to the `expand` array so the API returns `{ id, name, ... }` payloads instead of bare IDs.
- Backend saved-view filters need enum options; `useDocyrusDataViewSelect` already fetches the data source with `expand=enums`.
- Manual pages must apply the initial active view with `applyViewToTable(table, activeView)` after views and columns are ready. User-triggered view switches through `DataGridViewSelect` are applied automatically.
- Prefer `useDocyrusDataGrid` unless you truly need custom row fetching or custom toolbar composition. The hook handles auto-expand, async filter options, formatters, and reserved-column pinning for you.
- When you wire manual view CRUD into `DataGridViewSelect`, pass `isSaving` and `isLoading` so the editor UX stays correct during saves and background fetches.
- For tenant-aware date/datetime/number formatting, fetch `getTenantPreferences(client)` once at app boot and pass the resulting `formatDate` / `formatDateTime` / `formatNumber` callbacks into `useDocyrusDataGrid`.
- For instant avatar + label rendering on user-typed cells, fetch `/v1/users` once at app boot and pass the resulting `CellUserOption[]` into `useDocyrusDataGrid` via the `users` option.

## References

- `references/hook-pages.md` — one-call Docyrus grid pages with direct API mode, collection mode, extension points, formatters, and shared-users wiring.
- `references/manual-pages.md` — local/manual grid composition, local views, backend view-select wiring, and initial-view handling.
- `references/advanced-saved-view-query-patterns.md` — saved-view persistence, system views, hidden views, paging ownership, and translating active views into manual server queries.
- `references/tenant-and-users-providers.md` — app-level providers for tenant preferences and the shared users list, plus how to plug them into `useDocyrusDataGrid`.
