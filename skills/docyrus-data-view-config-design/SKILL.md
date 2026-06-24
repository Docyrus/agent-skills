---
name: docyrus-data-view-config-design
description: Design, configure, validate, and test Docyrus data views (saved grid/gallery views) with the `docyrus studio` data-view CLI commands. A data view is a named saved configuration of how one data source's records are presented — visible columns, order/pinning/grouping, row height, table/gallery display, sorting, filters (column filters + react-querybuilder query), quick-filter fields, row/cell color rules, paging, inline editing, and a bound form. Use when creating or editing a saved view / tab / list view, hiding or reordering columns, setting a default filtered/sorted view, grouping rows, adding color rules, configuring paging or inline editing, or binding a form to a view. Triggers on "create a data view", "saved view", "add a grid tab", "default view", "filter/sort a view", "hide columns", "group by", "color rules", "gallery view", `docyrus studio create-data-view`, `update-data-view`, `list-data-views`. For the schema use docyrus-data-source-design; for the React grid UI use docyrus-data-grid-page-design.
---

# Docyrus Data View Config Design

A **data view** is a saved, named configuration over a single data source's records. It does **not** change the schema or the data — it controls how a grid/gallery surface (TanStack Table + react-querybuilder) presents that data source: which columns show, in what order, how rows are filtered/sorted/grouped/colored, and how paging/inline-editing/forms behave. Views back the "tab strip" / view picker in Docyrus list pages (`useDocyrusDataViewSelect`).

Author views with `docyrus studio *-data-view`, then **validate** (read back) and **test** (confirm the saved JSON unpacks to the intended grid state). An unvalidated view is not done.

For the schema being viewed see **docyrus-data-source-design**; for the React grid that renders views see **docyrus-data-grid-page-design**; for full CLI flag reference see **docyrus-cli-app**.

## Critical model: opaque JSON blobs with packed sub-keys

The backend stores a view as a flat row (`tenant_app_config_data_view`) with four **opaque `jsonb` columns** — `columns`, `filters`, `sort`, `color_rules` — plus scalar columns (`name`, `description`, `icon`, `color`, `is_default`, `sort_order`, `quick_filter_fields`, `archived`, `tenant_app_id`). The backend does **not** validate the inner shape of the four jsonb blobs (the DTO only checks they are objects). The grid UI packs/unpacks them with specific sub-keys — **if you write the wrong sub-keys the view saves fine but renders as empty defaults.**

The single most important fact: **the stored sub-key names differ from the TanStack/UI property names.** e.g. visible-columns is stored as `columns.visibility` (not `columnVisibility`); color rules are `color_rules.row` / `color_rules.cell` (not `rowColorRules`). Get these exact key names from [references/view-config-fields.md](references/view-config-fields.md) — do not guess them.

```
columns      → { visibility, order, pinning, grouping, rowHeight, displayMode,
                 pagingEnabled, pagingMode, pageSize, inlineEditingEnabled,
                 readOnlyColumns, columnOptions, formId }
filters      → { columnFilters, filterQuery }
sort         → { sorting }
color_rules  → { row, cell }
```

## Workflow

1. **Confirm auth, app, and data source.** A view belongs to a data source inside an app.
   ```bash
   docyrus auth who --json
   docyrus apps list --json                                   # find appSlug
   docyrus studio list-fields --appSlug <app> --dataSourceSlug <ds> --json
   ```
   The `list-fields` output gives the **column ids / field slugs** you reference in `visibility`, `order`, `grouping`, `sorting`, filters, and color rules. For enum/select/status filter values you need enum **row UUIDs** — `docyrus studio list-enums --appSlug <app> --dataSourceSlug <ds> --fieldSlug <f> --json`. If there is no session, stop and ask the user to run `docyrus auth login`.

2. **Decide the view's intent.** Per view settle: name (tab label), which columns are visible and in what order, default sort, any filter (simple column filters and/or a query), grouping, row height, display mode (table or gallery), color rules, paging, inline editing, whether it is the default view. Sketch this for the user when ambiguous. **If the data source has a status/single-select field, default to status-based views — see [Status fields → design views per status](#status-fields--design-views-per-status).**

3. **Build the payload.** Prefer `--from-file ./view.json` for anything beyond a trivial view — the blobs are nested JSON and awkward inline. Use the per-blob convenience flags (`--columns`, `--filters`, `--sort`, `--colorRules`, `--quickFilterFields` take JSON strings; `--name`, `--description`, `--icon`, `--color`, `--isDefault`, `--sortOrder` are scalars) only for small views. See [references/view-config-fields.md](references/view-config-fields.md) for every key's exact name, type, and accepted values, and [references/workflow-examples.md](references/workflow-examples.md) for ready-to-adapt payloads.

4. **Create the view.** See [Create](#create-a-data-view).

5. **Validate.** Re-read with `get-data-view` and confirm every blob has the intended sub-keys with the right column ids / enum UUIDs. See [Validate](#validate).

6. **Test.** Confirm the saved JSON round-trips to the grid state you intended (and, if possible, that it filters/sorts real records correctly). See [Test](#test).

## Status fields → design views per status

**A status field (status / single-select / "stage"-style enum) is a strong signal to design status-based views.** The existence of such a field means the records move through stages, and the most useful tab strip on a list page is one that splits records by stage. When a data source has a status field, default to creating a set of status views (plus a default "All" view) unless the user wants otherwise.

How many views depends on how many enum options the status field has:

- **3–4 options → one view per option.** e.g. status `New`, `Active`, `Won`, `Lost` → four views `New` / `Active` / `Won` / `Lost`, each filtering `status = <enum uuid>`.
- **More than ~4 options → group related statuses into fewer, meaningful views.** One tab per value becomes an unusable strip. Group them by stage of life, e.g. `New`, `In Progress`, `On Hold` → an **"Open"** view; `Done`, `Cancelled` → a **"Closed"** view. Pick group names the user's domain would recognise.
- Usually also keep an **"All"** view (no status filter) and mark the most-used view (often "All" or "Open") as `is_default`.

Each status view is an ordinary view whose `filters` blob targets the status field **by enum row UUID, never the label** (read UUIDs from `docyrus studio list-enums ... --fieldSlug status --json`):

- **Single status:** one `filters.filterQuery` rule with `=` (or a `columnFilters` entry).
- **Grouped (multi-status) view:** a `filters.filterQuery` rule using `in` with an array of UUIDs.

Grouped "Open" view payload (filter only — merge with the rest of your view config):

```jsonc
{
  "name": "Open",
  "icon": "circle-dot",
  "sort_order": 2,
  "filters": {
    "columnFilters": [],
    "filterQuery": {
      "combinator": "and",
      "rules": [
        { "field": "status", "operator": "in",
          "value": ["<new-uuid>", "<in-progress-uuid>", "<on-hold-uuid>"] }
      ]
    }
  }
}
```

These status views are exactly what backs the view-type / status tab strip planned during project planning — keep one view per visible tab.

## Studio data-view command cheat-sheet

Selectors — pass exactly one of each pair; the CLI resolves the other and routes to `/v1/apps/<appSlug>/data-sources/<dataSourceSlug>/views`:
`--appId | --appSlug`, `--dataSourceId | --dataSourceSlug`. Mutations also accept `--data '<json>'` or `--from-file ./view.json` (JSON only); explicit per-blob flags are **merged over** the file/data payload. Append `--json` for machine output.

```bash
docyrus studio list-data-views   --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-data-view     --appSlug crm --dataSourceSlug contacts --viewId <id> --json
docyrus studio create-data-view  --appSlug crm --dataSourceSlug contacts --from-file ./view.json --json
docyrus studio update-data-view  --appSlug crm --dataSourceSlug contacts --viewId <id> --from-file ./view.json --json
docyrus studio delete-data-view  --appSlug crm --dataSourceSlug contacts --viewId <id> --json
```

- `list-data-views` / `create-data-view` accept an optional `--tenantAppId` (maps to `tenant_app_id`) to scope a view to a different app than the data source's owner. Most views omit it.
- `update-data-view` additionally accepts `--archived` (soft-delete / hide from the tab strip). `archived` is **not** the same as deleting — prefer `--archived true` over `delete-data-view` when you just want to retire a view.
- `create-data-view` requires `name`. Everything else is optional; an omitted blob means "grid defaults" for that aspect.

### Create a data view

Minimal (just a named view — inherits all grid defaults):
```bash
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts \
  --name "All Contacts" --isDefault true --sortOrder 1 --json
```

Configured (recommended via file — see [references/workflow-examples.md](references/workflow-examples.md) for full examples):
```bash
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts \
  --from-file ./active-leads-view.json --json
```

## Critical rules

- **Stored sub-keys ≠ UI property names.** Visible columns → `columns.visibility`, order → `columns.order`, pinning → `columns.pinning`, grouping → `columns.grouping`, color rules → `color_rules.row` / `color_rules.cell`, sort → `sort.sorting`, filters → `filters.columnFilters` / `filters.filterQuery`. Wrong key = silently ignored (view renders as defaults). Full list in [references/view-config-fields.md](references/view-config-fields.md).
- **Blobs are not validated server-side.** The DTO only asserts each of `columns`/`filters`/`sort`/`color_rules` is an object. Malformed inner JSON is accepted and then ignored by the UI — so **always validate by reading back and checking the unpacked shape**; the create call succeeding proves nothing about correctness.
- **Column ids are field slugs** (plus built-ins like `id`, `name`, `autonumber_id`). Get them from `list-fields`. A typo'd column id in `visibility`/`order`/`sorting` is dropped silently.
- **Enum/relation filter values are UUIDs**, never labels (same rule as records). Read them from `list-enums` / the related data source before writing a `filters` blob that targets a select/status/relation field.
- **`columns.visibility` is an allow/deny map, not a column list.** It is `{ "<colId>": true|false }`; columns absent from the map use their default visibility. To *hide* a column set it `false`; to force-show set `true`. Use `columns.order` to control sequence.
- **Many features live inside the `columns` blob**, not their own column: paging (`pagingEnabled`/`pagingMode`/`pageSize`), inline editing (`inlineEditingEnabled`/`readOnlyColumns`), per-column options (`columnOptions`), and the bound form (`formId`). This is so the backend needs no schema change — see [references/view-config-fields.md](references/view-config-fields.md#the-columns-blob).
- **`is_default` is per data source.** Marking a view default does not auto-unset other views server-side; if exactly-one-default matters, clear the previous default's `is_default` yourself. The UI fallback chain is: persisted localStorage selection → `is_default` view → first view.
- **`quick_filter_fields` is a top-level `uuid[]` column** (field ids), not part of any blob.
- **Validate then test, every time.** Read back the view and confirm the unpacked shape; delete throwaway test views you create.

## Validate

After authoring, confirm the saved view is exactly what you intended:

1. `docyrus studio get-data-view --appSlug <app> --dataSourceSlug <ds> --viewId <id> --json` — confirm scalars (`name`, `is_default`, `sort_order`, `icon`, `color`, `archived`) and that each blob holds the **expected sub-keys** with valid column ids / enum UUIDs.
2. Cross-check every column id in `columns.visibility`/`order`/`grouping`, every `sort.sorting[].id`, every filter field, and every `color_rules` formula reference against `list-fields` output — drop anything that does not resolve.
3. `docyrus studio list-data-views ... --json` — confirm the view appears, ordering (`sort_order`), and that at most one `is_default` is set if that was the intent.

Per-field "what correct looks like" checklist is in [references/workflow-examples.md](references/workflow-examples.md#validation-checklist).

## Test

Prove the view behaves as intended:

1. **Round-trip check** — re-read the view and unpack each blob using the key map in [references/view-config-fields.md](references/view-config-fields.md); confirm it yields the visible columns / order / sort / filter you designed.
2. **Filter sanity** — run the equivalent query against real records to confirm the filter selects the right rows, e.g. `docyrus ds list <app> <ds> --columns "..." --filter "..."` (see docyrus-cli-app), or `docyrus dsql` for complex filters. The view's `filters.filterQuery` mirrors react-querybuilder; the column filters mirror TanStack `columnFilters`.
3. **Clean up** — `docyrus studio delete-data-view ... --viewId <id>` for throwaway views (or `--archived true` to retire without deleting).

Full test playbook is in [references/workflow-examples.md](references/workflow-examples.md#test-playbook).

## References

- **[references/view-config-fields.md](references/view-config-fields.md)** — The authoritative field reference: every scalar column, the exact packed sub-keys of `columns` / `filters` / `sort` / `color_rules`, their types, accepted values (display modes, row heights, paging modes), the UI-name↔stored-key mapping, and gotchas. **Read this before writing any non-trivial view payload.**
- **[references/workflow-examples.md](references/workflow-examples.md)** — End-to-end worked examples (default view, filtered+sorted+grouped view, color-coded view, gallery view, paging + inline-editing view, form-bound view) as copy-adaptable JSON payloads, plus the validation checklist and test playbook.
