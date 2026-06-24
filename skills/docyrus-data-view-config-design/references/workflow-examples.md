# Data View Worked Examples

Copy-adaptable payloads and the validation/test playbook. All assume `--appSlug crm --dataSourceSlug contacts`; replace column ids with real field slugs from `docyrus studio list-fields` and enum/relation values with UUIDs from `docyrus studio list-enums`.

Create with either:
```bash
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts --from-file ./view.json --json
# or update an existing one
docyrus studio update-data-view --appSlug crm --dataSourceSlug contacts --viewId <id> --from-file ./view.json --json
```

## Table of contents

- [1. Minimal default view](#1-minimal-default-view)
- [2. Filtered + sorted + column-curated view](#2-filtered--sorted--column-curated-view)
- [3. Grouped view with row height](#3-grouped-view-with-row-height)
- [4. Color-coded view](#4-color-coded-view)
- [5. Gallery (card) view](#5-gallery-card-view)
- [6. Paging + inline-editing view](#6-paging--inline-editing-view)
- [7. Form-bound view](#7-form-bound-view)
- [Validation checklist](#validation-checklist)
- [Test playbook](#test-playbook)

## 1. Minimal default view

Just a named default tab — inherits all grid defaults. Fastest path; flags only.
```bash
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts \
  --name "All Contacts" --icon "huge file-list-table" --isDefault true --sortOrder 1 --json
```

## 2. Filtered + sorted + column-curated view

"Active leads" — hides noise columns, fixes order, filters by status enum + numeric, sorts newest first. `<status-active-uuid>` is an enum row id from `list-enums`.
```json
{
  "name": "Active leads",
  "icon": "fas filter",
  "sort_order": 2,
  "columns": {
    "visibility": { "description": false, "last_modified_on": false },
    "order": ["name", "email", "status", "ltv", "account", "created_on"],
    "pinning": { "left": ["name"], "right": [] }
  },
  "filters": {
    "columnFilters": [],
    "filterQuery": {
      "combinator": "and",
      "rules": [
        { "field": "status", "operator": "=", "value": "<status-active-uuid>" },
        { "field": "ltv", "operator": ">", "value": 1000 }
      ]
    }
  },
  "sort": { "sorting": [{ "id": "created_on", "desc": true }] },
  "quick_filter_fields": ["<status-field-uuid>", "<account-field-uuid>"]
}
```

## 3. Grouped view with row height

Group rows by `status`, taller rows, multi-column sort.
```json
{
  "name": "By status",
  "sort_order": 3,
  "columns": {
    "order": ["name", "status", "ltv", "account"],
    "grouping": ["status"],
    "rowHeight": "tall",
    "displayMode": "table"
  },
  "sort": { "sorting": [{ "id": "status", "desc": false }, { "id": "ltv", "desc": true }] }
}
```

## 4. Color-coded view

Whole-row red when overdue; green `ltv` cell when high value. `formula` references column ids.
```json
{
  "name": "Risk board",
  "sort_order": 4,
  "color_rules": {
    "row":  [ { "formula": "status == 'overdue'", "color": "#fee2e2" } ],
    "cell": [ { "column": "ltv", "formula": "ltv > 100000", "color": "#dcfce7" } ]
  }
}
```

## 5. Gallery (card) view

Card layout instead of a table. `displayMode: "gallery"` is the switch.
```json
{
  "name": "Gallery",
  "icon": "fas grid",
  "sort_order": 5,
  "columns": {
    "displayMode": "gallery",
    "order": ["name", "status", "account"]
  }
}
```
Gallery slot bindings (`galleryCardConfig` / `galleryDisplayConfig`) are only honored by the `<DataGallery>` surface (`useDocyrusDataGallery`); add them under `columns` only when targeting that surface.

## 6. Paging + inline-editing view

Standard pagination at 50/page; inline editing on except two protected columns.
```json
{
  "name": "Editable grid",
  "sort_order": 6,
  "columns": {
    "pagingEnabled": true,
    "pagingMode": "standard",
    "pageSize": 50,
    "inlineEditingEnabled": true,
    "readOnlyColumns": ["created_on", "autonumber_id"]
  }
}
```

## 7. Form-bound view

Bind a saved form so this view's create/edit/view surfaces use that form's layout. `formId` lives **inside** the `columns` blob.
```json
{
  "name": "Intake",
  "sort_order": 7,
  "columns": { "formId": "<data-form-uuid>" }
}
```
Get the form id from `docyrus studio list-forms --appSlug crm --dataSourceSlug contacts --json`.

## Validation checklist

After create/update, run `docyrus studio get-data-view --appSlug crm --dataSourceSlug contacts --viewId <id> --json` and confirm:

| Aspect | What "correct" looks like |
|---|---|
| Scalars | `name`, `is_default`, `sort_order`, `icon`, `color`, `archived` match intent. |
| `columns.visibility` | Only columns you meant to hide are `false`; keys are real column ids. |
| `columns.order` | Every id resolves against `list-fields`; sequence is what you intended. |
| `columns.grouping` | Group-by column ids resolve; empty `[]` if no grouping. |
| `columns.rowHeight` / `displayMode` | One of the allowed enum values (`short\|medium\|tall\|extra-tall`, `table\|gallery`). |
| `columns` features | `pagingMode` ∈ `virtual-scroll\|standard`; `formId`/`readOnlyColumns`/`columnOptions` ids resolve. |
| `filters.filterQuery` | `combinator` is `and`/`or`; each rule's `field` resolves; enum/relation `value`s are UUIDs, not labels. |
| `sort.sorting` | Each `id` resolves; `desc` booleans correct; priority order intended. |
| `color_rules.row`/`cell` | Stored under `row`/`cell` (not `rowColorRules`/`cellColorRules`); `column` ids resolve. |
| `quick_filter_fields` | Top-level uuid[] of **field ids** (not slugs, not in a blob). |

Then `docyrus studio list-data-views ... --json` — confirm the view appears, `sort_order` ordering, and at most one `is_default`.

## Test playbook

1. **Round-trip the blobs.** Re-read the view and unpack each blob with the key map in `view-config-fields.md`. Confirm the visible columns / order / sort / filter equal your design. Pay special attention that no sub-key was silently ignored (a UI-name typo leaves the corresponding blob key absent on read-back).
2. **Filter sanity against real data.** Reproduce `filters` as a record query and confirm row selection:
   ```bash
   docyrus ds list crm contacts --columns "name,email,status,ltv" --filter "status=<status-active-uuid>" --json
   # or, for complex queries:
   docyrus dsql query "SELECT name, ltv FROM contacts WHERE status = '<status-active-uuid>' AND ltv > 1000 ORDER BY created_on DESC"
   ```
   The count/rows should match what the view's filter+sort would show.
3. **Default selection.** If you set `is_default`, confirm exactly one default exists for the data source (`list-data-views`); clear any stale previous default with `update-data-view --viewId <old> --isDefault false`.
4. **Clean up.** Remove throwaway views: `docyrus studio delete-data-view --appSlug crm --dataSourceSlug contacts --viewId <id> --json` (or `--archived true` via `update-data-view` to retire without deleting).
