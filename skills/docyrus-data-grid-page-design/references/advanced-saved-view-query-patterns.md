# Advanced Saved-View and Query Patterns

## Read this when

- You need backend-saved views but do not want `useDocyrusDataGrid` to own row fetching.
- You want system views plus user-saved views in one selector.
- You need to debug why a saved view, grouping rule, or paging setting is not reflected in the grid.
- You need to translate `SavedDataGridView` into a custom Docyrus items query.

## Saved-view ownership model

- `DataGridViewSelect` applies the selected view to the TanStack table when the user clicks a tab or dropdown item.
- `useDocyrusDataGrid` also reapplies the active view on mount and whenever the active view changes, so the initial view works without extra code.
- Manual pages using `useDocyrusDataViewSelect` plus `useDataGrid` must still apply the initial active view with `applyViewToTable(table, activeView)`.

## System views, hidden views, and persistence

`useDocyrusDataViewSelect` supports three useful layers of view state:

1. `systemViews`
   - Developer-defined static views.
   - Automatically marked `isSystem: true`.
   - Render before saved backend views.
   - Cannot be edited or deleted through `DataGridViewSelect`, but can still be hidden.

2. active-view persistence
   - Controlled by `persistActiveView` and `persistKey`.
   - Default storage namespace: `docyrus:data-grid-view:appSlug:dataSourceSlug[:appId]`.
   - Initial resolution order: current in-memory value, stored value, backend `is_default`, then first available view.

3. hidden-view persistence
   - Managed through `hiddenViewIds`, `onViewHide`, and `onViewUnhide` in `gridViewSelectProps`.
   - Stored separately from the active-view key.
   - Useful when system views should exist but not always stay visible in the tab strip.

## Manual server-query translation from a saved view

If you keep custom row fetching, derive request params from the active view yourself.

```ts
const activeView = views.find(view => view.id === activeViewId);

const columns = buildColumnsFromView(fields, activeView);
const orderBy = (activeView?.sorting ?? []).map(sort => ({
  field: sort.id,
  direction: sort.desc ? 'desc' : 'asc'
}));
const filters = activeView?.filterQuery;

const params = {
  columns,
  orderBy: orderBy.length > 0 ? orderBy : undefined,
  filters: filters?.rules?.length ? filters : undefined,
  filterKeyword: keyword || undefined,
  limit: 50,
  offset: 0
};
```

Rules to preserve from the built-in hook behavior:

- Always include `id` in `columns`.
- When `columnOrder` is present, treat it as the ordered whitelist of visible field slugs.
- Otherwise use `columnVisibility` entries with `true` values.
- With no explicit saved-view visibility config, fetch all field slugs.
- If the active view groups by a field, make sure that field stays in `columns` even if the saved view hides it.
- If you merge app-specific overrides, apply them last so they win.

## Reserved columns versus backend field slugs

There are two different conventions:

- Local/manual views can include reserved table columns like `select` and `actions` inside `columnOrder`.
- Backend Docyrus-saved views only know real data-source field slugs.

That difference matters when you build a hybrid page. If you manually persist local views, reserved columns are safe. If you round-trip views through Docyrus backend APIs, assume only real field slugs are persisted and prepend reserved UI columns in the page code.

## Paging ownership

Saved views can carry:

- `pagingEnabled`
- `pagingMode`
- `pageSize`

`useDocyrusDataGrid` reads those values and forwards them into `useDataGrid`, and `gridProps` then carries the resolved paging mode.

If you build the page manually and want the active view to own paging:

- read the values from `activeView`
- pass `pagingMode` and `pageSize` into `useDataGrid`
- ensure your own row query or pagination model stays compatible with that page size

If you ignore these properties in manual mode, the view editor can save paging settings that never affect the rendered grid.

## Good advanced combinations

### System starter views + backend user views

Use `systemViews` for non-editable defaults like:

- All records
- My records
- Needs attention
- Recently updated

Then let users create their own backend-saved views beside them.

### Hook-owned metadata + app-owned data fetching

Use `useDocyrusDataViewSelect` when you want:

- backend view CRUD
- field metadata for the filter editor
- active-view persistence
- hidden-view persistence

but you still want the page's existing query hook, export pipeline, or analytics request builder to own row fetching.

## Debug checklist

- **Wrong initial view?** Check `persistActiveView`, `persistKey`, and whether a stale value is still in local storage.
- **Grouping not working?** Make sure the grouping field is included in fetched `columns` and remains visible enough for the row model.
- **Filter builder missing?** Pass `fields` into `DataGridViewSelect`.
- **View tabs empty during loading?** Pass `isLoading` so the selector can render its loading state.
- **Editor closes too early on save?** Pass `isSaving` when the host owns async create or update.
- **Backend returns incomplete rows?** You probably forgot `columns`.
