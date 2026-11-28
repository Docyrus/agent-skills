# Conditional behavior: computed keys, actions, expressions

Two mechanisms, one expression language. Pick by intent:

| Intent | Use |
|--------|-----|
| "This field is hidden / required / labelled differently depending on the values" | **Computed key** on that field |
| "This field's value is derived from other values" | **`computedFormula`** on that field |
| "When *this* field changes, do things to *other* fields" | **`fieldActions`** on the changing field |
| "When the form loads / before it submits / after it saves, do things" | **`formActions`** at the layout root |
| "The record as a whole is invalid" | **`formCustomValidations`** at the layout root |

## Expression language

Expressions are **JSONata** strings, evaluated against the form's current values. Field slugs are bare identifiers:

```jsonata
status = "active"
amount > 1000 and currency = "USD"
$length(description) <= 500
owner != null
$contains(email, /^[^@]+@example\.com$/)
```

Anywhere an expression is accepted you may instead pass **Query Builder JSON** — `{ "combinator": "and", "rules": [ … ] }` — which is converted to JSONata at evaluation time. Prefer plain JSONata strings when authoring by hand; they are shorter and diff cleanly.

Inside a **field** `customValidations` rule the context is richer:

- `value` — that field's current value
- `values` — the whole value map, keyed by slug

A **form-level** rule sees the values directly as bare slugs.

> **These are JSON string values.** Escape them accordingly: `"` becomes `\"` and a regex backslash becomes `\\`. The pattern example above is written in a JSON document as
> `"expression": "$contains(value, /^[A-Z]{2}-\\d+$/)"`. An expression that fails to parse is skipped silently, so a mis-escaped rule looks exactly like a rule that always passes.

> A typo in a slug is not an error. It evaluates to `undefined`, which is falsy — so a `computedRequired` silently never fires, a `computedHidden` never hides, and a `formCustomValidations` rule fails on every submit. Check every identifier against `studio list-fields`.

## Computed keys

Set on a field node's `fieldConfig`. All are optional; `null` means "not set".

| Key | Returns | Effect |
|-----|---------|--------|
| `computedHidden` | boolean | `true` → the field is hidden (and skipped by validation) |
| `computedRequired` | boolean | `true` → the field is required |
| `computedLabel` | string | Replaces the field's label |
| `computedDescription` | string | Replaces the field's description |
| `computedFormula` | any | Written back as the field's value — the highest-priority value writer |

```jsonc
{
  "id": "n7",
  "componentType": "field-userSelect",
  "dataSourceFieldId": "approver",
  "fieldConfig": {
    "slug": "approver",
    "type": "field-userSelect",
    "computedHidden": "amount <= 1000",
    "computedRequired": "amount > 1000",
    "computedLabel": "amount > 10000 ? \"Executive approver\" : \"Approver\""
  }
}
```

A field driven by `computedFormula` should also be presented as read-only — otherwise a user edits a value that is immediately overwritten.

## Actions

Both `fieldActions` (on a field) and `formActions` (at the root) use the same block structure.

```jsonc
{
  "id": "act-1",
  "triggerType": "onFieldChange",        // field actions: always this
  "blocks": [
    {
      "id": "blk-1",
      "name": "Escalation routing",      // authoring label only
      "sortOrder": 0,                    // blocks run in this order
      "conditionalItems": [              // IF / ELSE IF — first truthy wins
        {
          "id": "cond-1",
          "condition": "priority = \"urgent\"",
          "actions": [
            { "method": "showField", "fieldSlug": "approver" },
            { "method": "setFieldRequired", "fieldSlug": "approver", "required": true }
          ]
        }
      ],
      "elseActions": [                   // no condition matched
        { "method": "hideField", "fieldSlug": "approver" },
        { "method": "clearFieldValue", "fieldSlug": "approver" }
      ],
      "unconditionalActions": [          // ALWAYS, after the branch
        { "method": "setFieldValue", "fieldSlug": "triaged", "value": "true" }
      ]
    }
  ]
}
```

**Evaluation order:** blocks in `sortOrder` → within a block, `conditionalItems` top-to-bottom with the first truthy condition winning → `elseActions` if none matched → `unconditionalActions` always (including when the block has no conditions at all, which is how you express "always do this").

### Step methods (8)

| `method` | Payload | Effect |
|----------|---------|--------|
| `setFieldValue` | `{ fieldSlug, value }` | Write a value |
| `setFieldValues` | `{ fields: [{ fieldSlug, value }] }` | Batch write — the only step without a top-level `fieldSlug` |
| `clearFieldValue` | `{ fieldSlug }` | Reset to empty |
| `showField` | `{ fieldSlug }` | Write `hidden: false` — this also overrides a static hidden set elsewhere; only a full override reset clears it |
| `hideField` | `{ fieldSlug }` | Hide |
| `setFieldRequired` | `{ fieldSlug, required: boolean }` | Toggle required |
| `setFieldDisabled` | `{ fieldSlug, disabled: boolean }` | Toggle disabled |
| `setFieldReadOnly` | `{ fieldSlug, readOnly: boolean }` | Toggle read-only |

The four non-value methods accumulate into per-field overrides layered over the static config; they never mutate the saved document. They **write** a value rather than clearing one, so `showField` beats a `fieldLayout.hidden` and `setFieldRequired … false` beats a `required` validation until the overrides are reset.

**Targets are slugs that must exist in this form.** A step pointing at a field the layout does not render does nothing.

**Values are written as authored.** The studio UI stores them as strings (`"4"`, `"true"`), so prefer expressions or values whose type matches the target field, and be explicit when a number or boolean is required.

### Form lifecycle triggers

Root `formActions[]`, one entry per trigger:

| `triggerType` | Fires | Use for |
|---------------|-------|---------|
| `onFormLoad` | Once, after initial data loads | Seed defaults, pre-hide fields |
| `onFormBeforeSubmit` | After validation passes, before the save call | Last-minute normalization |
| `onFormAfterSubmit` | After a successful save | Post-save state; the API response is added to the evaluation context under the key `$result` |

> **Reaching the save result in `onFormAfterSubmit`:** the response is placed on the context object under the literal key `$result`, not registered as a JSONata variable. A bare `$result` in an expression therefore resolves as an *unbound variable* (empty), not the response. Quote it as a path to read it:
> `` `$result`.id `` . Verify the value before relying on it — if the quoted form does not resolve either, treat the response as unavailable and move the logic to the app.

```jsonc
"formActions": [
  {
    "id": "fa-1",
    "triggerType": "onFormLoad",
    "blocks": [
      {
        "id": "fa-blk-1",
        "sortOrder": 0,
        "conditionalItems": [],
        "elseActions": [],
        "unconditionalActions": [
          { "method": "setFieldValue", "fieldSlug": "status", "value": "new" }
        ]
      }
    ]
  }
]
```

## Submit validation

Field rule (error under the field) — `customValidations` on `fieldConfig`:

```jsonc
"customValidations": [
  { "id": "cv-1", "expression": "$length(value) <= 500", "message": "Keep notes under 500 characters." },
  { "id": "cv-2", "expression": "value >= values.min_amount", "message": "Below the configured minimum." }
]
```

Form rule (banner above the form) — `formCustomValidations` at the root:

```jsonc
"formCustomValidations": [
  { "id": "fv-1", "expression": "end_date > start_date", "message": "End date must be after the start date." },
  { "id": "fv-2", "expression": "status != \"active\" or owner != null", "message": "Active records need an owner." }
]
```

Rules must return `true` to pass. Field rules run after `required`; form rules run only once every field rule passes. An expression that errors is skipped silently — test the JSONata separately if a rule seems not to fire.

> `customValidations` on a form field **replace** the data-source field's own rules for that form. If a field already carries validation at the schema level and you only want to add to it, restate the existing rules alongside the new ones.

## Choosing between a computed key and an action

They overlap for visibility and required-ness. Use:

- **Computed key** — the rule is a pure function of the values, it re-evaluates continuously, and it lives with the field it affects. Prefer this; it is declarative and cannot drift out of sync.
- **Action** — the change has to write *other* fields' values, or must happen once at a lifecycle moment. Actions fire on a trigger, so a field hidden by an action stays hidden until something fires again.

Do not express the same rule both ways: field-action overrides beat form-action overrides, and computed formulas beat both, so a duplicated rule is a debugging trap.
