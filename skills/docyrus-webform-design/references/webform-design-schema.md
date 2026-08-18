# Webform design JSON specification

This document specifies the JSON produced by the project-owned form designer in
`apps/webforms/src/components/webform-builder/` and the related contracts derived from it.
It describes the implementation as it exists in this repository; it is not a generic
Docyrus form-schema proposal.

Machine-readable companion files:

- [`schemas/webform-builder.schema.json`](schemas/webform-builder.schema.json) — persisted `schema`
- [`schemas/webform-options.schema.json`](schemas/webform-options.schema.json) — persisted `options`
- [`schemas/webform-design-output.schema.json`](schemas/webform-design-output.schema.json) — documentation wrapper containing both documents
- [`schemas/webform-submission-envelope.schema.json`](schemas/webform-submission-envelope.schema.json) — static public POST envelope
- [`examples/webform-design.full.example.json`](examples/webform-design.full.example.json) — all 30 palette field types in one design

## 1. Contract overview

A saved webform has four related JSON contracts:

| Contract | Produced by | Purpose |
|---|---|---|
| `schema` | `schemaFromBuilderState()` | Ordered fields, field settings, enum choices, and per-field layout |
| `options` | `normalizeWebformOptions()` + `mergeBuilderStateIntoOptions()` | Layout variant, theme, title/copy, grid, and wizard/conversational settings |
| `json_schema` | `webformSchemaToJsonSchema()` | Draft 2020-12 description of the submitted `data` object; regenerated whenever `schema` is sent in an admin create/update payload |
| submission body | public app/embed runtime | `{ "data": { ... }, "turnstileToken"?: "..." }` posted to the public webform endpoint |

`schema` and `options` are separate properties on the webform API record. The combined
`webform-design-output.schema.json` exists only to make examples and external validation easier.

The in-memory `BuilderState` is **not** a persisted contract. It contains editor-only state such
as `selectedFieldId` and `viewMode`.

## 2. Canonical persisted design

A representative compact design is:

```json
{
  "schema": {
    "version": 1,
    "children": [
      {
        "type": "field-text",
        "field": {
          "id": "field-full-name",
          "name": "Full name",
          "slug": "full_name",
          "type": "field-text",
          "validations": ["required", "minLength:2", "maxLength:100"],
          "options": {
            "placeholder": "Ada Lovelace",
            "description": "Enter your legal name."
          }
        },
        "layout": {
          "colSpan": 1,
          "rowSpan": 1,
          "stepId": "step-contact"
        }
      },
      {
        "type": "field-select",
        "field": {
          "id": "field-priority",
          "name": "Priority",
          "slug": "priority",
          "type": "field-select"
        },
        "enumOptions": [
          { "id": "low", "name": "Low", "color": "#22c55e" },
          { "id": "high", "name": "High", "color": "#ef4444" }
        ],
        "layout": {
          "colSpan": 1,
          "rowSpan": 1,
          "stepId": "step-contact"
        }
      }
    ]
  },
  "options": {
    "layoutVariant": "classic-grid",
    "theme": {
      "accentColor": "#2563eb",
      "surfaceStyle": "soft"
    },
    "classicGrid": {
      "title": "Contact us",
      "description": "We usually reply within one business day.",
      "gridColumns": 2,
      "stepMode": "wizard",
      "showStepProgress": true,
      "submitLabel": "Send",
      "successMessage": "Your response has been received.",
      "steps": [
        {
          "id": "step-contact",
          "title": "Contact details",
          "description": "How we can reach you"
        }
      ]
    },
    "conversationalFlow": {
      "welcomeTitle": "Welcome",
      "welcomeDescription": "Let’s go through each question one at a time.",
      "progressStyle": "bar",
      "autoAdvance": false,
      "submitLabel": "Send",
      "successMessage": "Your response has been received."
    }
  }
}
```

## 3. `schema` contract

### 3.1 Root

```ts
interface WebformBuilderSchema {
  version: 1;
  children: WebformSchemaChild[];
}
```

- The serializer always writes `version: 1`.
- `children` order is rendering order.
- A normal studio save requires at least one serialized child.
- The serializer rebuilds the root from `BuilderState`; unknown root properties are not preserved.

### 3.2 Child

```ts
interface WebformSchemaChild {
  type: string;
  field: IField;
  enumOptions?: EnumOption[];
  layout: {
    colSpan: 1 | 2;
    rowSpan: 1 | 2;
    stepId?: string | null;
  };
}
```

| Property | Required in builder output | Meaning |
|---|---:|---|
| `type` | yes | Palette/component renderer key; normally equal to `field.type` |
| `field` | yes | Field identity, label, submission slug, type, validation, and settings |
| `enumOptions` | no | Choice values; omitted when absent or empty |
| `layout` | yes | Per-field placement; `colSpan` and `rowSpan` are always serialized |
| `layout.stepId` | no | Wizard assignment; omitted when the builder value is null/empty |

Semantic requirements:

1. `child.type` and `child.field.type` **should be equal**. The serializer does not enforce this.
2. `field.slug` **must be non-empty and unique within the form** for reliable rendering and submission. The inspector currently allows empty/duplicate slugs, so external validators should enforce this semantic rule.
3. `field.id`, step IDs, option IDs, and grid-row IDs should be stable and unique in their own scopes.
4. Enum/grid values are submitted by `id`, never by label.
5. A wizard `layout.stepId` should reference one of `options.classicGrid.steps[].id`. A missing value falls back to the first step at runtime.
6. `colSpan: 2` is clamped to the active grid width by the embed. `rowSpan` is used by the in-app renderer, but the current embed parser records it without applying a row-span style.

### 3.3 Load and round-trip behavior

`builderStateFromWebform()` loads a child when it has a `field` object and `field.slug` is a
string. Important behavior:

- A missing/empty `field.id` is replaced with a new UUID before the next save.
- Missing layout spans become `1`.
- Missing `stepId` becomes the first normalized step in builder state.
- The complete `field` object and `enumOptions` are retained.
- Unknown child-level properties, `child.options`, unknown root properties, and builder-only
  `customProps` are not emitted by `schemaFromBuilderState()`.
- Standalone platform children such as `KvButton` and `FieldTurnstile` are recognized by runtime
  helper code, but they are not available in the palette and are not part of the native builder
  output contract. A standalone child without `field` is dropped on a designer save.
- Imported children with a `field.slug` and an unknown `type` can round-trip because the internal
  type is a string. The machine schema therefore treats the type as extensible while documenting
  the 30 native palette values.

## 4. Field object

### 4.1 Core and inspector-managed properties

```ts
interface IField {
  id: string;
  name: string;
  slug: string;
  type: string;
  defaultValue?: string | null;
  validations?: string[] | null;
  readOnly?: boolean | null;
  options?: Record<string, unknown> | null;
  // Advanced properties are listed below.
}
```

The inspector stores properties in two different places:

| UI setting | JSON location | Notes |
|---|---|---|
| Field name | `field.name` | Visible label/title |
| Slug | `field.slug` | Submission key |
| Read only | `field.readOnly` | Honored by Docyrus UI fields; the lean embed does not currently disable inputs from this flag |
| Minimum value | `field.min` | Editor metadata for number/money/percent; actual JSON Schema bounds come from validation tokens |
| Maximum value | `field.max` | Same caveat as `min` |
| Maximum rating | `field.maxRating` | Rating renderer uses it; inspector range is 1–10 |
| Validations | `field.validations[]` | Token grammar in section 7 |
| Placeholder | `field.options.placeholder` | Text, textarea, email, and URL in the current inspector |
| Description | `field.options.description` | Helper text; Section renders it as body copy |
| Horizontal choices | `field.options.horizontal` | Tag Select and Radio Group |
| Linear scale bounds/labels | `field.options.scaleMin`, `scaleMax`, `minLabel`, `maxLabel` | Defaults 1, 5, empty, empty |
| Matrix rows | `field.options.rows[]` | Choice Grid and Checkbox Grid; `{ id, name }` |
| Internal field | `field.options.internal: true` | Hidden from public render/submission; remains available to studio reviewers |
| Column/row span | `child.layout` | Not stored inside `field` |
| Wizard step | `child.layout.stepId` | Not stored inside `field` |

UI defaults are not always serialized. For example, a new Number displays minimum `0` and maximum
`100` in the inspector, but those keys are absent until edited. A new Rating similarly behaves as
five stars without necessarily containing `maxRating: 5`.

### 4.2 Advanced properties preserved in `field`

The field type imported from Docyrus UI can also carry:

| Property | Shape | Purpose/support boundary |
|---|---|---|
| `enums` | `EnumOption[] | null` | Inline enum metadata separate from child `enumOptions`; the webform builder/renderers use child `enumOptions` |
| `relationDataSourceId` | `string | null` | Relation/user source identifier |
| `avatarMapping` | mapping object | Icon/color/image mapping |
| `nested`, `nestedByProp` | boolean/string | Nested option metadata |
| `itemMapping` | mapping object | Icon/color/image/description mapping |
| `computedHidden` | JSONata string, query-builder group, or null | Preserved; not evaluated by the project-owned embed/builder preview |
| `computedRequired` | JSONata/query-builder group/null | Preserved; not evaluated by the project-owned embed/builder preview |
| `computedLabel`, `computedDescription`, `computedFormula` | JSONata strings/null | Preserved; require a higher-level Docyrus form runtime to evaluate |
| `fieldActions[]` | `onFieldChange` block/action graph | Preserved; not executed by the project-owned public runtimes |
| `customValidations[]` | `{ id, expression, message }[]` | Preserved; public runtimes currently enforce only static `required` |

The machine schema defines the complete currently imported `IField`, enum-option, computed-condition,
validation, and field-action shapes. `field.additionalProperties` intentionally remains enabled so
new Docyrus field metadata can round-trip.

## 5. Enum options and grids

### 5.1 Enum option

```json
{
  "id": "high",
  "name": "High",
  "color": "#ef4444",
  "icon": "ArrowUp",
  "parent": "priority",
  "slug": "high",
  "sort_order": 3,
  "active": true,
  "is_final_option": false,
  "force_description": false,
  "force_followup_date": false,
  "description": "Requires prompt attention"
}
```

Only `id` and `name` are structurally required. The designer edits `id`, `name`, and `color`.

New Select, Tag Select, and Radio Group fields start with:

```json
[
  { "id": "option-1", "name": "Option 1", "color": "#3b82f6" },
  { "id": "option-2", "name": "Option 2", "color": "#22c55e" },
  { "id": "option-3", "name": "Option 3", "color": "#f59e0b" }
]
```

### 5.2 Matrix fields

Choice Grid and Checkbox Grid use:

- `field.options.rows[]` for row definitions
- `child.enumOptions[]` for column definitions

```json
{
  "type": "field-choiceGrid",
  "field": {
    "id": "grid-1",
    "name": "Satisfaction",
    "slug": "satisfaction",
    "type": "field-choiceGrid",
    "options": {
      "rows": [
        { "id": "service", "name": "Service" },
        { "id": "quality", "name": "Quality" }
      ]
    }
  },
  "enumOptions": [
    { "id": "poor", "name": "Poor" },
    { "id": "good", "name": "Good" }
  ],
  "layout": { "colSpan": 2, "rowSpan": 2 }
}
```

Values:

```json
{
  "choice_grid": {
    "service": "good",
    "quality": "poor"
  },
  "checkbox_grid": {
    "service": ["good", "fast"],
    "quality": ["reliable"]
  }
}
```

## 6. Native palette field catalog

The palette currently exposes **30** field types in eight categories.

| Category | Type | New-field type settings/defaults | Full Docyrus UI value | Lean embed value | Derived `json_schema` node |
|---|---|---|---|---|---|
| Text & Input | `field-text` | none; optional `options.placeholder` | string | string | `string` |
| Text & Input | `field-textarea` | none; optional placeholder | string | string | `string` |
| Text & Input | `field-email` | none; optional placeholder | string | string | `string`, `format: email` |
| Text & Input | `field-url` | none; optional placeholder | string | string | `string`, `format: uri` |
| Text & Input | `field-phone` | no type-specific inspector setting | number text plus companion country field | same | `string`; companion key is not represented |
| Numbers | `field-number` | inspector defaults min 0/max 100 | number/null | number/null | `number` |
| Numbers | `field-money` | inspector defaults min 0/max 100 | number/null plus companion currency field | number/null; no currency selector | `number`; companion key is not represented |
| Numbers | `field-percent` | inspector defaults min 0/max 100 | number/null | number/null | `number` |
| Numbers | `field-rating` | visual default 5; `field.maxRating` 1–10 | integer/null | integer/null | `integer` |
| Numbers | `field-linearScale` | `options.scaleMin: 1`, `scaleMax: 5`, empty end labels | integer | integer | unconstrained `{}` (current generator gap) |
| Numbers | `field-duration` | none | number of seconds | fallback string input | `number` |
| Numbers | `field-currency` | none | ISO currency code string | number/null input | `string` |
| Selection | `field-select` | three default options | one option ID string | one option ID string | enum string when options exist |
| Selection | `field-tagSelect` | three defaults; `options.horizontal` | option ID string array | option ID string array | array of enum strings |
| Selection | `field-radioGroup` | three defaults; `options.horizontal` | one option ID string | one option ID string | enum string |
| Selection | `field-choiceGrid` | two default rows, three default columns | `{ [rowId]: columnId }` | same | unconstrained `{}` (current generator gap) |
| Selection | `field-checkboxGrid` | two default rows, three default columns | `{ [rowId]: columnId[] }` | same | unconstrained `{}` (current generator gap) |
| Layout | `field-section` | default description `Optional section description` | no answer | no answer | property with unconstrained `{}` (current generator gap) |
| Date & Time | `field-date` | none | `YYYY-MM-DD` string | same | `string`, `format: date` |
| Date & Time | `field-dateTime` | none | local `YYYY-MM-DDTHH:mm` string | same | `string`, `format: date-time` |
| Date & Time | `field-time` | none | `HH:mm` string | same | `string`, `format: time` |
| Date & Time | `field-dateRange` | none | bracket string such as `[2026-07-01, 2026-07-31]` | fallback string input | `object` (current generator/runtime mismatch) |
| Toggle | `field-switch` | none | boolean | boolean | `boolean` |
| Toggle | `field-checkbox` | none | boolean | boolean | `boolean` |
| Visual & Media | `field-file` | none | stored-file object when an upload handler is supplied | fallback string input | array of objects (current generator/runtime mismatch) |
| Visual & Media | `field-image` | none; builder does not enable gallery mode | stored-file object; array only when gallery mode is supplied externally | fallback string input | array of objects (current default-runtime mismatch) |
| Relation & User | `field-userSelect` | optional enum list in inspector | user ID string | fallback string input | `string` |
| Relation & User | `field-userMultiSelect` | optional enum list in inspector | user ID string array | fallback string input | array of strings |
| Relation & User | `field-relation` | optional enum list in inspector | record ID string | fallback string input | `string` |
| Relation & User | `field-locationSelect` | none | location object | fallback string input | `object` |

### 6.1 Conversion groups

The inspector permits conversion only within these groups:

- Text: `field-text`, `field-textarea`, `field-email`, `field-url`, `field-phone`
- Number: `field-number`, `field-money`, `field-percent`, `field-rating`, `field-duration`, `field-currency`
- Choice: `field-select`, `field-radioGroup`, `field-tagSelect`

Conversion preserves `id`, `name`, `slug`, validations, and existing `options`; it applies the
new type's defaults and keeps existing enum options when the target has no defaults.

### 6.2 File and location shapes

A successfully uploaded file/image value has this shape:

```json
{
  "file_name": "brief.pdf",
  "source": "s3://bucket/path/brief.pdf",
  "file_size": 284913,
  "file_type": "application/pdf",
  "signed_url": "https://storage.example/brief.pdf?signature=...",
  "file_data": null
}
```

Without an upload callback, Docyrus UI temporarily stores the browser `File`; JSON serialization
of that value does not produce a usable uploaded-file contract. The lean embed has no upload
implementation and currently renders File/Image as fallback text inputs.

A location value can contain:

```json
{
  "address": "Levent, 34330 Beşiktaş/İstanbul",
  "description": "Head office",
  "details": "Floor 4",
  "latitude": 41.0825,
  "longitude": 29.0092,
  "lat": 41.0825,
  "lng": 29.0092,
  "placeId": "ChIJ..."
}
```

## 7. Validation tokens

`field.validations` is an ordered string array.

| Token | Inspector availability | Derived JSON Schema | Public runtime enforcement |
|---|---|---|---|
| `required` | all types | adds slug to root `required` unless `options.internal === true` | enforced |
| `minLength:<n>` | text, textarea, email, URL, phone group in validation panel | `minLength` on string nodes | not enforced by project-owned public runtimes |
| `maxLength:<n>` | same | `maxLength` | not enforced by project-owned public runtimes |
| `pattern:<regex>` | same | `pattern` | not enforced by project-owned public runtimes |
| `min:<n>` | number, money, percent, rating | `minimum` | not enforced by project-owned public runtimes |
| `max:<n>` | same | `maximum` | not enforced by project-owned public runtimes |

The validation panel's current code includes Phone in the text family despite the old inline comment
in the local layout reference. A Section can also be marked required because the validation tab is
shown for every child; do not do this, because Section has no answer and can make the form impossible
to complete.

`field.min`/`field.max` inspector values and `min:`/`max:` validation tokens are independent. Only
the tokens are copied into `json_schema`.

## 8. `options` contract

### 8.1 Normalized defaults

```json
{
  "layoutVariant": "classic-grid",
  "theme": {
    "accentColor": "#2563eb",
    "surfaceStyle": "soft"
  },
  "classicGrid": {
    "title": "",
    "description": "",
    "gridColumns": 2,
    "stepMode": "single-page",
    "showStepProgress": true,
    "submitLabel": "Submit",
    "successMessage": "Your response has been received.",
    "steps": [
      { "id": "step-1", "title": "Step 1" }
    ]
  },
  "conversationalFlow": {
    "welcomeTitle": "Welcome",
    "welcomeDescription": "Let’s go through each question one at a time.",
    "progressStyle": "bar",
    "autoAdvance": false,
    "submitLabel": "Submit",
    "successMessage": "Your response has been received."
  }
}
```

Both layout buckets are retained regardless of the active variant, so switching variants does not
normally discard the other variant's copy/settings.

### 8.2 Property reference

| Path | Type | Default/behavior |
|---|---|---|
| `layoutVariant` | `classic-grid \| conversational-flow` | Missing/unknown input resolves to `classic-grid` |
| `theme.accentColor` | CSS color string | `#2563eb` |
| `theme.surfaceStyle` | `soft \| solid` | `soft`; embed maps to `#f8fafc` vs `#ffffff` |
| `classicGrid.title` | string | Builder form title; falls back to webform record name while loading |
| `classicGrid.description` | string | Introductory copy |
| `classicGrid.gridColumns` | `1 \| 2 \| 3` | `2` |
| `classicGrid.stepMode` | `single-page \| wizard` | `single-page` |
| `classicGrid.showStepProgress` | boolean | `true` |
| `classicGrid.submitLabel` | string | `Submit` |
| `classicGrid.successMessage` | string | Default receipt message |
| `classicGrid.errorMessage` | string | Optional embed-only extension; not exposed by the studio UI |
| `classicGrid.steps[]` | `{ id, title, description? }[]` | At least one normalized step |
| `conversationalFlow.welcomeTitle` | string | `Welcome` |
| `conversationalFlow.welcomeDescription` | string | Default one-question-at-a-time copy |
| `conversationalFlow.progressStyle` | `bar \| minimal` | `bar` |
| `conversationalFlow.autoAdvance` | boolean | `false`; stored and read by configuration, but the current embed flow does not invoke it |
| `conversationalFlow.submitLabel` | string | `Submit` |
| `conversationalFlow.successMessage` | string | Default receipt message |
| `conversationalFlow.errorMessage` | string | Optional embed-only extension |
| `archived` | boolean | Optional soft-archive flag preserved as a free-form extension |

`WebformBuilderOptions` permits unknown properties. Normalization shallow-merges the root and
nested theme/layout buckets, so extension keys survive studio saves. The machine schema therefore
uses `additionalProperties: true` for options and its nested buckets.

## 9. Derived `json_schema`

Every admin create/update that includes `schema` passes it through:

```ts
webformSchemaToJsonSchema(payload.schema, { title: payload.name })
```

The result describes the **contents of submission `data`**, not the outer submission envelope:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Contact us",
  "type": "object",
  "properties": {
    "full_name": {
      "type": "string",
      "title": "Full name",
      "minLength": 2,
      "maxLength": 100
    },
    "priority": {
      "type": "string",
      "enum": ["low", "high"],
      "x-enumNames": ["Low", "High"],
      "title": "Priority"
    }
  },
  "required": ["full_name"],
  "additionalProperties": false
}
```

Generation rules:

1. Iterate `schema.children` in order.
2. Skip a child without a non-empty `field.slug`.
3. Skip only `field-button` and `field-display` as value-less types.
4. Choose `field.type`, falling back to `child.type`.
5. Add type/format/enum information according to the field catalog.
6. Copy supported validation tokens to JSON Schema constraints.
7. Copy non-empty `field.name` to `title`, non-empty `defaultValue` to `default`, and
   `readOnly: true` to `readOnly`.
8. Add a non-internal `required` field to the root `required` array.
9. Keep internal fields as optional properties because studio reviewers can write them later.
10. Set root `additionalProperties: false`.

Current compatibility gaps that consumers must account for:

- `field-linearScale`, both grid types, and `field-section` are not specially mapped and become `{}`.
- Section is included as a property even though it never submits a value.
- Date Range is described as an object but the full renderer emits a bracketed string.
- File and Image are described as arrays of objects; the builder's default renderers emit one
  stored-file object unless Image gallery mode is supplied externally.
- Phone's `__<slug>_country` and Money's `__<slug>_currency` companion properties are absent while
  the root forbids additional properties.
- The full Date Time renderer emits a local minute-precision string; strict JSON Schema
  `date-time` validation may require a timezone/seconds depending on the validator.

These are documentation of the current implementation, not recommended target behavior.

## 10. Submission contract

### 10.1 Endpoint and envelope

Secure public address:

```text
POST {API_BASE}/webforms/{formId}/{formToken}
```

Readable branded address:

```text
POST {API_BASE}/webforms/workspaces/{workspaceSlug}/{webformSlug}
```

Body:

```json
{
  "data": {
    "full_name": "Ada Lovelace",
    "priority": "high",
    "topics": ["product", "support"],
    "recommendation_score": 9,
    "satisfaction_by_area": {
      "service": "good",
      "quality": "excellent"
    }
  },
  "turnstileToken": "optional-token"
}
```

Apply [`webform-submission-envelope.schema.json`](schemas/webform-submission-envelope.schema.json)
to the outer body and the webform record's generated `json_schema` to `body.data`.

### 10.2 Data assembly

The public implementations copy values only for known parsed field slugs. Consequently:

- untouched fields are absent rather than serialized as null;
- unknown keys are dropped;
- `field.options.internal === true` fields are excluded from public render, validation, and submit;
- Section normally has no value and is absent;
- select/radio values are option IDs;
- tag select and user multi-select values are arrays;
- grid objects are keyed first by row ID, then by column ID(s).

Renderer differences:

- The lean embed explicitly merges Phone's `__<slug>_country` companion key into `data`.
- The in-app `buildPublicSubmissionPayload()` currently copies only real field slugs, so companion
  Phone/Money keys held by Docyrus UI controls are not merged.
- `FieldTurnstile` handling exists in schema decorators and the in-app payload helper, but the
  native builder does not create this child and the lean embed does not currently render a
  Turnstile control from `hasTurnstile` alone.

## 11. Builder-only state

For integrations that control `<FormBuilder value onChange>`, the in-memory shape is:

```ts
interface BuilderState {
  fields: Array<{
    id: string;                    // editor row identity
    componentType: string;
    fieldConfig: IField;
    enumOptions?: EnumOption[];
    customProps?: Record<string, unknown>; // not serialized
    colSpan?: 1 | 2;
    rowSpan?: 1 | 2;
    stepId?: string | null;
  }>;
  selectedFieldId: string | null; // not serialized
  viewMode: 'edit' | 'preview' | 'code'; // not serialized
  formTitle: string;              // options.classicGrid.title
  formDescription: string;        // options.classicGrid.description
  gridColumns: 1 | 2 | 3;         // options.classicGrid.gridColumns
  stepMode: 'single-page' | 'wizard'; // options.classicGrid.stepMode
  steps: Array<{ id: string; title: string; description?: string }>;
}
```

Do not persist this object as the webform `schema`. Use `schemaFromBuilderState()` and
`mergeBuilderStateIntoOptions()`.

## 12. Authoring and validation checklist

Before accepting or importing a design:

1. Validate `schema` and `options` against the machine-readable schemas.
2. Require non-empty unique field slugs.
3. Require unique field IDs, enum IDs per field, grid-row IDs per grid, and step IDs.
4. Verify `child.type === child.field.type`.
5. Verify every wizard `stepId` references an existing step.
6. Do not mark Section required.
7. Keep data-source-bound field slugs aligned with the destination data-source field slugs.
8. Treat `enumOptions[].id` and grid row IDs as persisted data keys; changing them changes the
   meaning of existing submissions.
9. Regenerate `json_schema` after every `schema` change. The app's admin API wrapper does this
   automatically when `schema` is included.
10. Test the intended renderer. The full in-app renderer and lean external embed intentionally
    have different support levels for complex fields.

## 13. Implementation sources

The contract was traced from:

- `apps/webforms/src/components/webform-builder/form-builder-types.ts`
- `apps/webforms/src/components/webform-builder/form-builder-context.tsx`
- `apps/webforms/src/components/webform-builder/registry/field-type-configs.ts`
- `apps/webforms/src/components/webform-builder/registry/property-schemas.ts`
- `apps/webforms/src/components/webform-builder/properties/*`
- `apps/webforms/src/lib/webforms-builder.ts`
- `apps/webforms/src/lib/webform-json-schema.ts`
- `apps/webforms/src/lib/webforms-public.ts`
- `apps/webforms/src/components/webforms/webform-custom-field.tsx`
- `packages/webform-runtime/src/index.ts`
- `packages/embed/src/webform-app.tsx`
- `packages/docyrus-ui/src/components/docyrus/form-fields/types/index.ts`
