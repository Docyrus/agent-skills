---
name: docyrus-dynamic-form-design
description: Design, save, and validate a Docyrus dynamic form — the record create/edit/view layout that is saved against a data source and rendered by `useDocyrusFormView` — using the `docyrus studio` form CLI commands. Use when the user wants to model a form for a data source (a "record form", "create/edit form", "detail layout", "form view"), lay fields out in a grid or sections, mark fields required, add conditional show/hide or auto-fill rules, add submit-time validation, or list/read/update/delete the forms saved on a data source. Covers the persisted layout contract (`gridColumns`, field nodes, panel sections, form-level actions and validations), the field-type catalog, the validation and JSONata expression surfaces, and the CRUD commands. Triggers on "create a form for <data source>", "edit form layout", "make field X required", "hide field Y unless…", "auto-fill field Z", "default form", "studio create-form / update-form / list-forms", or any dynamic-form authoring task. For the fields themselves use docyrus-data-source-design; for a public unauthenticated intake form use docyrus-webform-design.
---

# Docyrus Dynamic Form Design

A **dynamic form** is a saved layout attached to one data source. It decides which of the data source's fields appear on the record create / edit / view screen, in what order and grid, grouped into which panels, which are required, and what runs when a value changes or the form is submitted. It is stored as a `layout` JSON document on a form row and rendered by `useDocyrusFormView` (or `useDynamicFormView` for a backend-free host).

The host app fetches the form and hands its `layout` to the renderer, which translates it into the field list, grid, per-field overrides, and automations. **The document must therefore be self-contained** — nothing outside it tells the renderer what the form should do.

**Read this first, then author the JSON against [references/form-layout-schema.md](references/form-layout-schema.md).**

## The three rules that prevent most broken forms

1. **A form references fields; it never defines them.** Every field node must bind to an existing field on the data source, by `fieldConfig.slug` (preferred) or `dataSourceFieldId`. A binding that resolves to nothing is **silently dropped** from the rendered form. List the real slugs first with `studio list-fields`. To add a field, use **docyrus-data-source-design** — not this skill.
2. **Fields you omit do not render.** A saved layout is a whitelist: the form shows exactly the fields it lists, in its own order. Omitting a required-in-the-database column produces a form that cannot be submitted successfully.
3. **`required` and JSONata rules always run; the other tokens are opt-in.** `minLength:` / `maxLength:` / `pattern:` / `min:` / `max:` are enforced only where the host app turns them on (`validationTokens: 'form' | 'all'`, default `'off'`). Write them — they are the right place for single-field constraints and they are enforced the moment a host opts in — but when a rule **must** hold today, put it in `customValidations` as well. See [Validation](#validation-what-actually-runs).

## Workflow

1. **Resolve the target and read the real field slugs.**
   ```bash
   docyrus whoami --json
   docyrus apps list --json
   docyrus studio list-fields --appSlug crm --dataSourceSlug contact --json
   ```
2. **Read what already exists** — never design blind against a data source that already has forms.
   ```bash
   docyrus studio list-forms --appSlug crm --dataSourceSlug contact --json
   docyrus studio get-form --appSlug crm --dataSourceSlug contact --formId <id> --json
   ```
3. **Design the layout document.** Follow [references/form-layout-schema.md](references/form-layout-schema.md); pick field types from [references/field-type-catalog.md](references/field-type-catalog.md); express conditional behavior with [references/actions-and-expressions.md](references/actions-and-expressions.md). Start from [references/examples/form-layout.full.example.json](references/examples/form-layout.full.example.json) and delete what the user does not need.
4. **Validate the JSON** against [references/schemas/form-layout.schema.json](references/schemas/form-layout.schema.json), then run the checklist in [Validate](#validate-before-you-save).
5. **Save it** with `create-form` (new) or `update-form` (existing) — see [CRUD commands](#crud-commands). Keep the layout in a file: pass it as `--layout "$(cat form.layout.json)"`, or put the whole record (name, title, layout, …) in one file and use `--from-file`. Do not hand-type a layout into a shell flag.
6. **Read it back** with `get-form` and confirm the layout round-tripped and every slug still resolves.

## Layout document in one screen

The saved `layout` is a single JSON object. Field order is the array order; sections are nodes that carry nested `fields`.

```jsonc
{
  "gridColumns": 2,                     // 1–4. Non-numeric or missing → 1. Always set it.
  "labelAlign": "top",                  // "top" | "left"
  "labelWidth": "md",                   // "sm" | "md" | "lg"  (only with labelAlign "left")
  "fieldSize": "md",                    // "sm" | "md" | "lg"
  "fieldVariant": "outline",            // "outline" | "filled"
  "fields": [
    {
      "id": "n1",
      "componentType": "field-text",
      "dataSourceFieldId": "full_name", // slug or field id
      "columnSpan": 2,                  // ≥ gridColumns → full width (grids of 2+ only)
      "fieldConfig": {
        "slug": "full_name",            // the binding that actually matters
        "type": "field-text",
        "validations": ["required"]
      }
    },
    {
      "id": "sec-contact",              // section = nested `fields`, no binding
      "componentType": "panel",
      "label": "Contact",
      "columnSpan": 2,                  // panel width in the form grid
      "gridColumns": 1,                 // the panel's own inner grid
      "fields": [ /* field nodes */ ]
    }
  ],
  "formActions": [],                    // lifecycle automations (see references)
  "formCustomValidations": []           // submit-time cross-field rules
}
```

The complete key-by-key contract — including every accepted alias, the exact binding-resolution order, and what each key does at render time — is [references/form-layout-schema.md](references/form-layout-schema.md).

## Validation: what actually runs

Field errors render under the field; form errors render as a banner.

| Rule | Where it lives | Applies to | Runs at submit |
|------|----------------|-----------|----------------|
| `required` | `fieldConfig.validations: ["required"]` | any value — empty string, empty array and null all count as missing | **always** |
| `computedRequired` | `fieldConfig.computedRequired` (JSONata / QB JSON) | conditional required | **always** |
| Field `customValidations` | `fieldConfig.customValidations[]` (JSONata → `true`) | anything, incl. other fields via `values` | **always** |
| Form `formCustomValidations` | layout root (JSONata → `true`) | cross-field rules; runs after every field rule passes | **always** |
| `minLength:N` / `maxLength:N` | `fieldConfig.validations` tokens | strings **and** arrays (character count / item count) | when enabled |
| `pattern:RE` | `fieldConfig.validations` token | strings; raw unanchored regex, everything after the first colon | when enabled |
| `min:N` / `max:N` | `fieldConfig.validations` tokens | numbers, and numeric-typed fields whose input returns a string | when enabled |

**"When enabled"** means the host app passes `validationTokens: 'form'` (enforce what the form declares) or `'all'` (also enforce the data source's own tokens) to the form-view hook. The default is `'off'`, which keeps a stale bound on an old data-source field from blocking records that already violate it. A form's tokens are always stored and always shown by the builder's preview — the switch only governs runtime enforcement. If you cannot confirm the host has opted in and a constraint is not optional, mirror it in `customValidations`.

Order per field: `required` → tokens → custom validations; the first failure wins. An **empty optional value is never failed by a token** — only `required` looks at emptiness. A token that does not fit the value's shape (a `pattern` on a number, a `min` on text) is skipped, and unknown tokens or malformed bounds are ignored.

Use a custom validation when a token cannot express the rule:

```jsonc
"customValidations": [
  { "id": "cv-1", "expression": "value >= values.min_amount", "message": "Below the configured minimum." }
]
```

## Conditional behavior

Two mechanisms, both JSONata (or Query Builder JSON) over the form's values:

- **Computed keys** — declarative, per field: `computedHidden`, `computedRequired`, `computedLabel`, `computedDescription`, `computedFormula` (writes the field's value).
- **Actions** — imperative blocks: `fieldActions` on a field (fires when *that* field changes) and `formActions` at the root (`onFormLoad` / `onFormBeforeSubmit` / `onFormAfterSubmit`). A block runs its `conditionalItems` top-to-bottom, first truthy wins, else `elseActions`, then `unconditionalActions` always.

Prefer a computed key when the rule is "this field's state depends on the values"; use an action when one change must write *other* fields. Full semantics, the 8 step methods, and worked expressions: [references/actions-and-expressions.md](references/actions-and-expressions.md).

## CRUD commands

All five are `docyrus studio` subcommands scoped to a data source. Selectors: `--appId | --appSlug` and `--dataSourceId | --dataSourceSlug`; the form itself is addressed by `--formId` (forms have **no slug**). Add `--json` for machine-readable output. Write commands accept individual flags **or** `--data` / `--from-file` JSON; flags merge over the JSON.

```bash
# List every form saved on a data source (find the default and any existing ids)
docyrus studio list-forms --appSlug crm --dataSourceSlug contact --json

# Read one form, including its full layout document
docyrus studio get-form --appSlug crm --dataSourceSlug contact --formId <formId> --json

# Create a form. Keep the layout in a file — it is too big for a shell flag.
docyrus studio create-form --appSlug crm --dataSourceSlug contact \
  --name "Contact form" --title "Contact" --isDefault true \
  --layout "$(cat contact-form.layout.json)" --json

# Same, passing the whole record as one payload file
docyrus studio create-form --appSlug crm --dataSourceSlug contact \
  --from-file contact-form.json --json

# Update — send the FULL layout you want stored; it replaces, it does not deep-merge
docyrus studio update-form --appSlug crm --dataSourceSlug contact --formId <formId> \
  --layout "$(cat contact-form.layout.json)" --json

# Rename / re-flag without touching the layout
docyrus studio update-form --appSlug crm --dataSourceSlug contact --formId <formId> \
  --name "Contact form (v2)" --isDefault true --json

# Delete
docyrus studio delete-form --appSlug crm --dataSourceSlug contact --formId <formId> --json
```

Record fields writable on create/update: `--name`, `--title`, `--description`, `--subtopic`, `--color`, `--icon`, `--layout`, `--isDefault`, `--status`; `update-form` additionally takes `--archived`. A `--from-file` payload uses those same names as camelCase JSON keys (`{ "name": …, "layout": { … } }`), and individual flags merge over it.

`--status` is a numeric status code. Rather than guessing it, read an existing form on the tenant with `list-forms` and copy the value a working form uses.

> ⚠️ `update-form --layout` **replaces** the stored document. To change one field, `get-form` first, edit the returned layout, and send the whole thing back.

> ⚠️ `--isDefault true` marks the form as the data source's default — the one apps pick when no specific form is requested. Setting it on a second form is how you switch defaults; check `list-forms` first and confirm with the user before moving a default that already exists.

## Validate before you save

Run through this list — each item maps to a failure that is silent at save time and visible only when a user opens the form:

1. **Every binding resolves.** Each `fieldConfig.slug` appears in `studio list-fields` output. Unresolvable nodes vanish.
2. **No duplicate slug** across the layout. A field listed twice renders twice against one value.
3. **`gridColumns` is a number 1–4.** A missing or string value collapses the form to one column.
4. **Every `columnSpan` ≤ `gridColumns`** (or intentionally full-width).
5. **Sections have no binding** — a node with nested `fields` plus a `dataSourceFieldId`/`fieldConfig` is read as a *field*, and its children are lost.
6. **Non-`required` constraints are `customValidations`**, not tokens (see the table above).
7. **Every expression references real slugs.** A typo evaluates to `undefined` — usually falsy, so a `computedRequired` silently never fires and a `formCustomValidations` rule blocks every submit.
8. **Action `fieldSlug` targets exist in the layout.** A step pointing at a field the form does not render does nothing.
9. **Read back with `get-form`** and diff against what you sent.

## Reference material

| File | Read it when |
|------|--------------|
| [references/form-layout-schema.md](references/form-layout-schema.md) | Authoring or editing any layout JSON — the authoritative key-by-key contract, binding resolution, sections, and the legacy format |
| [references/field-type-catalog.md](references/field-type-catalog.md) | Choosing `type` / `componentType` per field, or wiring enum-backed and relation fields |
| [references/actions-and-expressions.md](references/actions-and-expressions.md) | Adding conditional visibility, auto-fill, lifecycle automation, or submit validation |
| [references/schemas/form-layout.schema.json](references/schemas/form-layout.schema.json) | Machine-validating a layout document before saving |
| [references/examples/form-layout.full.example.json](references/examples/form-layout.full.example.json) | A complete example exercising sections, spans, computed keys, actions, and validations |
| [references/examples/form-layout.minimal.example.json](references/examples/form-layout.minimal.example.json) | The smallest correct form — a starting point for simple asks |
