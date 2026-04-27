---
name: docyrus-data-grid-pages
description: Build Docyrus React data-grid and record-list pages with `DataGrid`, `DataGridViewSelect`, `useDataGrid`, `useDocyrusDataViewSelect`, and `useDocyrusDataGrid`. Use when asked to build or refactor a list page, records table, CRM or ERP grid, saved-view tabs, row actions, manual DataGrid toolbar composition, or Docyrus data-source grid queries with filtering, sorting, grouping, search, display modes, paging, and reload behavior.
---

# Docyrus Data Grid Pages

Build full Docyrus list pages around the web `DataGrid` stack.

## Choose the build mode

1. **Standard Docyrus page** → use `useDocyrusDataGrid`.
   - Best when rows come from a Docyrus data source or generated collection.
   - Gives you metadata-driven columns, saved views, toolbar wiring, search, filters, grouping, sorting, row-height, display mode, paging, and reload.
   - Read `references/hook-pages.md`.

2. **Custom layout or custom row-query lifecycle** → use `useDocyrusDataViewSelect` + `useDataGrid`.
   - Best when you need a custom toolbar arrangement, local/demo data, or a non-standard fetch cycle.
   - Read `references/manual-pages.md`.

3. **No backend saved views** → use `useDataGrid` with local `SavedDataGridView[]` or `DataGridViewMenu`.
   - Also covered in `references/manual-pages.md`.

4. **Complex Docyrus query payloads** → also load the `docyrus-api-dev` skill.
5. **Advanced saved-view persistence or manual view-driven queries** → read `references/advanced-saved-view-query-patterns.md`.

## Default page workflow

1. Confirm the `appSlug`, `dataSourceSlug`, and whether a generated collection already exists.
2. Pick hook mode or manual mode.
3. Add reserved columns in this order: select first, actions second.
4. Keep the page in a flex column layout with the grid body wrapped in `min-h-0 flex-1`.
5. Add create/edit/view/delete dialogs around row actions.
6. Verify initial saved view, search, filters, grouping, sorting, paging, and reload behavior.

## Non-negotiables

- Render page-sized grids with the toolbar in a shrink-0 row and the grid inside `min-h-0 flex-1`.
- Prefer `<DataGrid table={table} {...gridProps} height="auto" />` for full-page layouts.
- Pass the TanStack `table` instance to every grid menu and to `DataGridViewSelect`.
- `useDocyrusDataViewSelect` manages view metadata only. It does **not** fetch rows and does **not** accept `table`.
- Manual Docyrus item queries must always send `columns`.
- Backend saved-view filters need enum options; `useDocyrusDataViewSelect` already fetches the data source with `expand=enums`.
- Manual pages must apply the initial active view with `applyViewToTable(table, activeView)` after views and columns are ready. User-triggered view switches through `DataGridViewSelect` are applied automatically.
- Prefer `useDocyrusDataGrid` unless you truly need custom row fetching or custom toolbar composition.
- `field-userSelect`, `field-userMultiSelect`, `field-relation`, and `field-relatedField` need custom column mapping for rich cells; otherwise they fall back to short text.
- When you wire manual view CRUD into `DataGridViewSelect`, pass `isSaving` and `isLoading` so the editor UX stays correct during saves and background fetches.

## References

- `references/hook-pages.md` — one-call Docyrus grid pages with direct API mode, collection mode, and extension points.
- `references/manual-pages.md` — local/manual grid composition, local views, backend view-select wiring, and initial-view handling.
- `references/advanced-saved-view-query-patterns.md` — saved-view persistence, system views, hidden views, paging ownership, and translating active views into manual server queries.
