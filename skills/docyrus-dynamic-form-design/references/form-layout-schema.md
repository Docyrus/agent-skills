# Saved form layout — authoritative contract

The `layout` property of a form row (`docyrus studio get-form` → `layout`) is a single JSON object. It is read by `convertDocyrusFormLayout`, which translates it into the props `useDocyrusFormView` consumes. Everything below describes what that converter actually accepts — keys it does not read are inert, no matter how reasonable they look.

Two formats are supported. **Author the build-studio format**; the legacy one is documented only so you can recognize and migrate it.

- **Build-studio (current)** — selected when the root has a `fields` array.
- **Legacy `KvFormView`** — selected when the root has a `children` array. See [Legacy format](#legacy-kvformview-format).

---

## Root object

| Key | Type | Default | Effect |
|-----|------|---------|--------|
| `fields` | `Node[]` | — | **Required.** Ordered field and section nodes. Its presence selects the build-studio format. |
| `gridColumns` | `1 \| 2 \| 3 \| 4` | `1` | Column count of the form grid. **A missing or non-numeric value clamps to 1**, so always set it explicitly; values >4 clamp to 4. A legacy `options.columnCount` is also read and **outranks** `gridColumns` when both are present — never write both. |
| `labelAlign` | `'top' \| 'left'` | `'top'` | `'left'` renders a horizontal label column. |
| `labelWidth` | `'sm' \| 'md' \| 'lg'` | `'md'` | Width of that label column. Only meaningful with `labelAlign: 'left'`. A legacy numeric value is bucketed (`<8` → sm, `≤14` → md, else lg). |
| `fieldSize` | `'sm' \| 'md' \| 'lg'` | `'md'` | Input density for every field. Legacy `xs` → `sm`, `xl` → `lg`. |
| `fieldVariant` | `'outline' \| 'filled'` | `'outline'` | Input style for every field. |
| `formActions` | `FormAction[]` | — | Lifecycle automations. Runs automatically — the host app does not wire anything. See [actions-and-expressions.md](actions-and-expressions.md). |
| `formCustomValidations` | `FormCustomValidationRule[]` | — | Submit-time cross-field rules, surfaced as a banner. |

Anything else at the root is ignored.

> **Read-only fields:** the mere presence of a saved layout flips the renderer's `includeReadOnlyFields` to `true` — a saved form shows read-only fields as value rows rather than dropping them. Omit a field from `fields` if it should not appear at all.

---

## Field node

```jsonc
{
  "id": "n1",                          // node id — free-form, keep it stable across edits
  "componentType": "field-text",       // palette identity; mirror fieldConfig.type
  "dataSourceFieldId": "full_name",    // data-source field SLUG or ID
  "columnSpan": 2,
  "hidden": false,
  "fieldConfig": {
    "slug": "full_name",               // the binding that wins
    "type": "field-text",
    "validations": ["required"],
    "computedHidden": "status = \"closed\"",
    "computedRequired": null,
    "computedLabel": null,
    "computedDescription": null,
    "computedFormula": null,
    "customValidations": [],
    "fieldActions": []
  }
}
```

### Binding resolution (in order)

1. `fieldConfig.slug` — used when it matches a real field slug on the data source.
2. `dataSourceFieldId` — accepted as either a slug or a field **id** (ids are mapped back to slugs using the data source schema).
3. `fieldConfig.slug` again, unvalidated.
4. `id` — mapped as an id → slug.

**A node that resolves to nothing is skipped entirely.** No error, no placeholder — the field is simply absent from the rendered form. This is the single most common cause of a "form is missing fields" report.

Write both `fieldConfig.slug` and `dataSourceFieldId` with the same slug. It is redundant on purpose: it survives either resolution path and keeps the document readable.

### Field node keys

| Key | Type | Effect |
|-----|------|--------|
| `id` | `string` | Node identity. Not the field id. |
| `componentType` | `string` | Palette identity, e.g. `field-text`. Not read by the renderer (it dispatches on the data-source field's own type) but keep it correct — the studio UI reads it. |
| `dataSourceFieldId` | `string` | Slug or id binding. |
| `fieldConfig` | `object` | Per-form overrides for this field. See below. |
| `columnSpan` | `number` | Columns to occupy. In a grid of **more than one** column, `≥` the column count → full width; `≤1` → 1; >4 → 4. In a **1-column** grid there is no full-width promotion — a `columnSpan` of 2 stays 2 and overflows, so leave it out. Also accepted at `fieldConfig.columnSpan` or `options.columnSpan`. An array is accepted; its first element is used. |
| `hidden` | `boolean` | `true` removes the node from the form completely — not rendered, not validated, not submitted. Also read from `fieldConfig.hidden` / `options.hidden`. For a *conditional* hide use `computedHidden` instead. |

### `fieldConfig` keys the renderer reads

| Key | Type | Effect |
|-----|------|--------|
| `slug` | `string` | Binding (see above). |
| `type` | `string` | Field type mirror; see [field-type-catalog.md](field-type-catalog.md). |
| `validations` | `string[]` | Token list enforced at submit for this form: `required`, `minLength:N`, `maxLength:N`, `pattern:RE`, `min:N`, `max:N`. **Overrides** the data-source field's own tokens — see [Validation](#validation). |
| `computedHidden` | JSONata string or QB JSON | `true` → field hidden. |
| `computedRequired` | JSONata string or QB JSON | `true` → field required. |
| `computedLabel` | JSONata string | Result replaces the field label. |
| `computedDescription` | JSONata string | Result replaces the field description. |
| `computedFormula` | JSONata string | Result is written back as the field's value (highest-priority value writer). |
| `customValidations` | `CustomValidationRule[]` | Submit-time JSONata rules. **Replaces** the data-source field's own rules when present. |
| `fieldActions` | `FieldAction[]` | `onFieldChange` automations. |

In the build-studio format every one of these keys is also accepted under a node-level `options` object (`fieldConfig` wins when both exist). The **legacy** format reads only `columnSpan`, `validations` and the five `computed*` keys from `options` — `options.fieldActions` and `options.customValidations` are dropped there.

> **Computed-key marker strings:** the legacy builder stored a mode marker (`"formula"` / `"template"`) alongside the expression. A bare `"formula"` or `"template"` string is rejected as a value — never write one as an expression.

---

## Section node (panel)

A node is read as a **section** when it has nested children **and no field binding** (`dataSourceFieldId` and `fieldConfig` both absent).

```jsonc
{
  "id": "sec-contact",
  "componentType": "panel",
  "label": "Contact details",
  "columnSpan": 2,
  "gridColumns": 1,
  "fields": [ /* field nodes — nesting works recursively */ ]
}
```

| Key | Type | Effect |
|-----|------|--------|
| `fields` \| `children` \| `items` | `Node[]` | Members. Resolved in that precedence order. |
| `label` \| `title` \| `options.label` | `string` | Panel heading. Omit for an untitled panel. |
| `gridColumns` \| `columnCount` \| `options.columnCount` | `1..4` | The panel's **own** inner grid, and the column count its children's `columnSpan` is measured against. Defaults to the parent's count. |
| `columnSpan` \| `options.columnSpan` | `number` | Panel width inside the parent grid. `≥` parent columns → full width. |
| `id` | `string` | Panel id. Auto-generated (`section_<n>`) when missing. |

Sections render as bordered fieldsets. Nested sections are supported.

> **Tabs are not available in the build-studio format.** A tabbed layout (`tabpanel` / `tab`) can only be expressed in the legacy `KvFormView` format via `options.tabView: true`. If a user asks for tabs on a saved form, say so and offer sections instead.

---

## Validation

What the runtime enforces at submit, per field, in this order — first failure wins:

1. **`required`** — from `fieldConfig.validations`, the data-source field's own tokens, `computedRequired`, or an action override. Empty string, whitespace-only string, empty array and null all count as missing; `false` and `0` are real values and pass. **Always enforced.**
2. **Token constraints** — `minLength:N` / `maxLength:N` (string length or array item count), `pattern:RE` (strings; raw unanchored regex, everything after the first colon, so the pattern may contain colons), `min:N` / `max:N` (numbers, plus numeric-typed fields whose input hands back a string). **Enforced only when the host opts in** — see below.
3. **Field `customValidations`** — JSONata expressions that must evaluate to `true`. **Always enforced.**

Then, once every field passes: **root `formCustomValidations`**, rendered as a banner. Always enforced.

### Token enforcement is opt-in

The form-view hooks take `validationTokens?: 'off' | 'form' | 'all'`, default **`'off'`**:

| Mode | Enforces |
|------|----------|
| `'off'` | nothing beyond `required` — tokens are stored and shown but never block a submit |
| `'form'` | the tokens **this form** declares for a field (`fieldConfig.validations`) |
| `'all'` | form tokens where present, otherwise the data-source field's own tokens |

The default exists because data-source fields have carried tokens for years without them ever being enforced; switching them on globally would start rejecting records that already violate a stale bound. A form you author today should still declare its tokens — they are the clearest expression of intent, the builder's preview enforces them, and they go live the moment the host sets `'form'`. When a constraint must hold regardless of the host's setting, also write it as a `customValidations` rule.

Rules that keep tokens predictable once enforcement is on:

- An **empty optional value is never failed by a token** — only `required` inspects emptiness. `["minLength:5"]` on a blank optional field passes.
- A token that does not fit the value's shape is **skipped, not failed**: `pattern` on a number, `min` on free text, `minLength` on a boolean.
- **Unknown tokens and malformed bounds are ignored** (`minLength:abc`, `weird:1`), so a stale token cannot brick a form.
- An **unparseable `pattern` is ignored at runtime** — a broken regex can never block a user's submit. (The builder's preview does report it, as a design-time signal.)
- A form's `validations` **replace** the data-source field's tokens for that form, exactly like `customValidations`. Restate the schema-level tokens you want to keep. (In `'form'` mode only the form's own list is ever enforced, so a field the form leaves untokened is unconstrained even if the data source declares bounds.)

Use a `customValidations` rule when a token cannot express the rule — anything conditional, cross-field, or needing its own message:

| Intent | Token | Custom validation |
|--------|-------|-------------------|
| Length / range / format of one value | ✅ prefer the token | — |
| Depends on another field | — | `value >= values.min_amount` |
| Only applies in some states | — | `values.status != "final" or value != null` |
| Needs a specific message | — | any expression + your `message` |
| Date ordering | — | form-level: `end_date > start_date` |

A rule is `{ "id": "<uuid>", "expression": "<jsonata>", "message": "<shown on failure>" }`. Inside a field rule, `value` is that field's value and `values` is the whole record; a form-level rule addresses fields as bare slugs.

--------|---------------|------------------------------|
| Minimum length | `minLength:5` | `$length(value) >= 5` |
| Maximum length | `maxLength:80` | `$length(value) <= 80` |
| Pattern | `pattern:^[A-Z]{2}-\d+$` | `$contains(value, /^[A-Z]{2}-\d+$/)` |
| Minimum value | `min:10` | `value >= 10` |
| Maximum value | `max:100` | `value <= 100` |
| Range | `min:1` + `max:5` | `value >= 1 and value <= 5` |

A rule is `{ "id": "<uuid>", "expression": "<jsonata>", "message": "<shown on failure>" }`. Inside a field rule, `value` is that field's value and `values` is the whole record; a form-level rule addresses fields as bare slugs.

---

## Worked example

Two-column form, one full-width field, one single-column panel, a conditional requirement, and a submit rule:

```json
{
  "gridColumns": 2,
  "labelAlign": "top",
  "fieldSize": "md",
  "fieldVariant": "outline",
  "fields": [
    {
      "id": "n1",
      "componentType": "field-text",
      "dataSourceFieldId": "name",
      "columnSpan": 2,
      "fieldConfig": { "slug": "name", "type": "field-text", "validations": ["required"] }
    },
    {
      "id": "n2",
      "componentType": "field-select",
      "dataSourceFieldId": "status",
      "fieldConfig": { "slug": "status", "type": "field-select", "validations": ["required"] }
    },
    {
      "id": "n3",
      "componentType": "field-userSelect",
      "dataSourceFieldId": "owner",
      "fieldConfig": {
        "slug": "owner",
        "type": "field-userSelect",
        "computedRequired": "status = \"active\""
      }
    },
    {
      "id": "sec-notes",
      "componentType": "panel",
      "label": "Notes",
      "columnSpan": 2,
      "gridColumns": 1,
      "fields": [
        {
          "id": "n4",
          "componentType": "field-textarea",
          "dataSourceFieldId": "description",
          "fieldConfig": {
            "slug": "description",
            "type": "field-textarea",
            "customValidations": [
              { "id": "cv-1", "expression": "$length(value) <= 500", "message": "Keep notes under 500 characters." }
            ]
          }
        }
      ]
    }
  ],
  "formCustomValidations": [
    { "id": "fv-1", "expression": "status != \"active\" or owner != null", "message": "Active records need an owner." }
  ]
}
```

---

## Legacy `KvFormView` format

Recognize it by a root `children` array (instead of `fields`) and `options.columnCount` (instead of `gridColumns`). It still renders; do not rewrite one unless asked.

```jsonc
{
  "type": "KvFormView",
  "options": { "columnCount": 2, "tabView": false },
  "children": [
    { "id": "<fieldId>", "type": "field-text", "options": { "columnSpan": 2, "validations": ["required"] } },
    { "id": "panel_1", "type": "panel", "options": { "label": "Details", "columnCount": 1 }, "children": [ /* … */ ] }
  ]
}
```

Differences that matter when migrating:

- Field nodes bind by `id`, which is resolved as a **slug first and a field id second** — so a legacy node whose `id` happens to be a valid slug still works. Container nodes are identified by having `children`.
- Per-field settings live under `options`, not `fieldConfig`.
- `options.hideLabel: true` on a panel suppresses its title (the build-studio format simply omits `label`).
- `options.tabView: true` turns a panel into a tab strip — the only way to get tabs. Set at the **root** `options`, it wraps the entire form in one tab strip whose top-level children become the tabs.
- Form-level styling uses `options.fieldLabelAlign` / `options.fieldLabelWidth` (numeric grid units, bucketed on read).
- Legacy layouts written by the old builder carry no `formActions` / `formCustomValidations`, but the reader is format-agnostic: if you add them at the root of a `children` document they are honored exactly as in the build-studio format.

To migrate: map each `children` node to a `fields` node, resolve every field id to its slug, move `options.*` into `fieldConfig.*`, and set `gridColumns` from `options.columnCount`.
