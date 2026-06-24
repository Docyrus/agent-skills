# Data View Config Field Reference

Authoritative reference for every field of a Docyrus data view and the exact packed JSON shapes. The view is stored in `tenant_app_config_data_view` and exposed/consumed through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/views`.

## Table of contents

- [Scalar (top-level) fields](#scalar-top-level-fields)
- [The four opaque blobs at a glance](#the-four-opaque-blobs-at-a-glance)
- [The `columns` blob](#the-columns-blob)
- [The `filters` blob](#the-filters-blob)
- [The `sort` blob](#the-sort-blob)
- [The `color_rules` blob](#the-color_rules-blob)
- [UI name ↔ stored key map](#ui-name--stored-key-map)
- [Gotchas](#gotchas)

## Scalar (top-level) fields

These are real columns on the row / DTO. CLI flag → payload key in parentheses.

| Payload key | CLI flag | Type | Notes |
|---|---|---|---|
| `name` | `--name` | `string` | **Required on create.** The tab/label shown in the view picker. |
| `description` | `--description` | `string \| null` | Optional. |
| `icon` | `--icon` | `string \| null` | Docyrus icon id (e.g. `"huge file-list-table"`, `"fas table"`, or an emoji `"📋"`). Rendered in the tab. |
| `color` | `--color` | `string \| null` | View accent color (hex or token). |
| `is_default` | `--isDefault` | `boolean` | Marks the initial active view. **Per data source; not auto-exclusive** (see gotchas). |
| `sort_order` | `--sortOrder` | `number` (smallint) | Tab ordering; lower first. |
| `quick_filter_fields` | `--quickFilterFields` | `string[]` (uuid[]) | Field **ids** surfaced as quick-filter chips above the grid. Top-level array column, NOT inside any blob. |
| `tenant_app_id` | `--tenantAppId` | `string \| null` (uuid) | Scope the view to an app other than the data source owner. Usually omitted. |
| `archived` | `--archived` (update only) | `boolean` | Soft-delete / hide from tab strip. Retire a view with `--archived true` rather than deleting it. |
| `columns` | `--columns` | `object \| null` (jsonb) | Opaque presentation blob — see below. |
| `filters` | `--filters` | `object \| null` (jsonb) | Opaque filter blob — see below. |
| `sort` | `--sort` | `object \| null` (jsonb) | Opaque sort blob — see below. |
| `color_rules` | `--colorRules` | `object \| null` (jsonb) | Opaque conditional-color blob — see below. |

Read-only / server-managed: `id`, `tenant_id`, `tenant_data_source_id`, `ownership` (defaults `CUSTOM`), `created_by`/`created_on`/`last_modified_by`/`last_modified_on`.

The per-blob convenience flags (`--columns`, `--filters`, `--sort`, `--colorRules`, `--quickFilterFields`) take a **JSON string**; the CLI parses it and merges over any `--data`/`--from-file` payload. For non-trivial blobs prefer `--from-file ./view.json`.

## The four opaque blobs at a glance

The backend validates only that each blob is an object; it never inspects the inner keys. The grid UI (`useDocyrusDataViewSelect` mappers) is the real schema. Stored shape:

```jsonc
{
  "name": "Active leads",
  "columns": { "visibility": {…}, "order": […], "pinning": {…}, "grouping": […],
               "rowHeight": "medium", "displayMode": "table",
               "pagingEnabled": true, "pagingMode": "standard", "pageSize": 50,
               "inlineEditingEnabled": false, "readOnlyColumns": […],
               "columnOptions": {…}, "formId": "<uuid>" },
  "filters": { "columnFilters": […], "filterQuery": {…} },
  "sort":    { "sorting": [{ "id": "created_on", "desc": true }] },
  "color_rules": { "row": […], "cell": […] }
}
```

## The `columns` blob

Controls everything about column presentation **plus** several grid features that ride in this blob to avoid a backend schema change. All keys optional; omit any you don't set.

| Key | Type | Meaning |
|---|---|---|
| `visibility` | `Record<string, boolean>` | Per-column show/hide map keyed by **column id** (field slug). `false` hides; `true` force-shows; absent → column default. **Not** a list of visible columns. |
| `order` | `string[]` | Left-to-right column order, by column id. Ids omitted fall back to natural order after the listed ones. |
| `pinning` | `{ left: string[], right: string[] }` | Pinned (frozen) columns. TanStack `ColumnPinningState`. Default `{ left: [], right: [] }`. |
| `grouping` | `string[]` | Row-grouping columns (group-by), by column id. Empty `[]` = no grouping. |
| `rowHeight` | `"short" \| "medium" \| "tall" \| "extra-tall"` | Row density. |
| `displayMode` | `"table" \| "gallery"` | Grid vs. gallery/card layout. |
| `pagingEnabled` | `boolean` | When true, grid paginates and shows a paging footer. |
| `pagingMode` | `"virtual-scroll" \| "standard"` | Layout when paging is on. Default `virtual-scroll`. |
| `pageSize` | `number` | Page size when paging is on. |
| `inlineEditingEnabled` | `boolean` | Master switch for inline cell editing. When off, all cells read-only. |
| `readOnlyColumns` | `string[]` | Per-column read-only override (only meaningful when inline editing is on). Note: `id`, `autonumber_id`, and metadata-locked fields are auto-protected regardless. |
| `columnOptions` | `Record<string, { showAutonumber?: boolean }>` | Per-column runtime display overrides keyed by column id (e.g. show a relation's `autonumber_id` above the name). |
| `formId` | `string` (uuid) | Id of a saved Docyrus form (`DataForm.id`) bound to this view, so create/edit/view surfaces render that form's layout. Empty → host falls back to a default/auto form. |

Gallery-only extras (`galleryCardConfig`, `galleryDisplayConfig`) are part of the `SavedDataGridView` type and, when used by the gallery surface, also travel under the `columns` blob as flat serializable records. The standard grid view mapper does not round-trip them; only set them when targeting the gallery (`useDocyrusDataGallery`) surface.

## The `filters` blob

| Key | Type | Meaning |
|---|---|---|
| `columnFilters` | `Array<{ id: string, value: unknown }>` | TanStack `ColumnFiltersState` — simple per-column filters (the column-header filter UI). `id` is the column id; `value` shape depends on the column's filter editor. |
| `filterQuery` | `RuleGroupType` (react-querybuilder) | The advanced filter builder query. Shape: `{ combinator: "and" \| "or", rules: Array<Rule \| RuleGroupType>, not?: boolean }` where a `Rule` is `{ field: string, operator: string, value: unknown }`. `field` is the column id / field slug; for enum/select/status/relation fields, `value` is the enum row **UUID** (or array of UUIDs), never the label. |

A view may set either, both, or neither. `columnFilters` and `filterQuery` are combined by the consuming grid.

Example `filterQuery`:
```json
{
  "combinator": "and",
  "rules": [
    { "field": "status", "operator": "=", "value": "5f3a…-enum-uuid" },
    { "field": "ltv", "operator": ">", "value": 10000 }
  ]
}
```

## The `sort` blob

| Key | Type | Meaning |
|---|---|---|
| `sorting` | `Array<{ id: string, desc: boolean }>` | TanStack `SortingState` — ordered list of sort columns. `id` = column id, `desc: true` for descending. Multi-column sort = multiple entries in priority order. |

```json
{ "sorting": [ { "id": "priority", "desc": true }, { "id": "created_on", "desc": false } ] }
```

## The `color_rules` blob

Conditional formatting. Note the stored keys are `row` / `cell` (not `rowColorRules` / `cellColorRules`).

| Key | Type | Meaning |
|---|---|---|
| `row` | `Array<{ formula: string, color: string }>` | Row color rules. `formula` is a boolean expression over the row; when it evaluates true the whole row takes `color`. |
| `cell` | `Array<{ column: string, formula: string, color: string }>` | Cell color rules. Same as row but scoped to a single `column` (column id). |

`color` is a hex/token string; `formula` uses the platform's formula syntax referencing column ids.

```json
{
  "row":  [ { "formula": "status == 'overdue'", "color": "#fee2e2" } ],
  "cell": [ { "column": "ltv", "formula": "ltv > 100000", "color": "#dcfce7" } ]
}
```

## UI name ↔ stored key map

When translating from a `SavedDataGridView` (what UI code / docs talk about) to the stored payload, apply this mapping. **Writing the UI name as the stored key is the #1 mistake — it saves but is ignored.**

| `SavedDataGridView` (UI) | Stored location |
|---|---|
| `columnVisibility` | `columns.visibility` |
| `columnOrder` | `columns.order` |
| `columnPinning` | `columns.pinning` |
| `grouping` | `columns.grouping` |
| `rowHeight` | `columns.rowHeight` |
| `displayMode` | `columns.displayMode` |
| `pagingEnabled` / `pagingMode` / `pageSize` | `columns.pagingEnabled` / `columns.pagingMode` / `columns.pageSize` |
| `inlineEditingEnabled` / `readOnlyColumns` | `columns.inlineEditingEnabled` / `columns.readOnlyColumns` |
| `columnOptions` | `columns.columnOptions` |
| `formId` | `columns.formId` |
| `columnFilters` | `filters.columnFilters` |
| `filterQuery` | `filters.filterQuery` |
| `sorting` | `sort.sorting` |
| `rowColorRules` | `color_rules.row` |
| `cellColorRules` | `color_rules.cell` |
| `isDefault` | `is_default` (top-level scalar) |
| `icon` | `icon` (top-level scalar) |

## Gotchas

- **Sub-key names differ from UI prop names** (see map above). Wrong key = silently dropped; view renders as defaults.
- **No server-side blob validation.** A 200/201 only means "each blob is an object." Always read back and verify the unpacked shape.
- **Column ids = field slugs** (+ built-ins `id`, `name`, `autonumber_id`, …) from `studio list-fields`. Typos are ignored silently.
- **Enum/select/status/relation filter values are UUIDs**, not labels — read from `studio list-enums` / the related data source.
- **`visibility` is a partial map**, not a whitelist. Only columns you explicitly set `false` are hidden; everything else uses its default.
- **`is_default` is not exclusive server-side.** Setting one view default does not unset others — clear the prior default yourself if exactly one default matters.
- **`pinning` defaults to `{ left: [], right: [] }`** when unpacked; if you set it, include both arrays.
- **`quick_filter_fields` is a top-level uuid[]** (field ids), separate from the blobs.
- **`archived` ≠ delete.** Archiving hides the view from the tab strip but keeps it; `delete-data-view` removes it.
