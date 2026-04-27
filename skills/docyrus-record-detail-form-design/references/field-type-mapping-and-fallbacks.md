# Field-Type Mapping and Unsupported-Field Strategy

## Read this when

- You need to decide which component stack should render a Docyrus field.
- A field type has no editable component yet.
- You are building manual metadata-driven UIs and want safe fallback behavior.
- You need to extend Docyrus UI for a new field type without breaking forms, details, and inline editing.

## Single source of truth

Docyrus field rendering should flow through the shared field maps exposed by `useDocyrusFieldComponent`:

- `FORM_FIELD_MAP`
- `VALUE_RENDERER_MAP`
- `CELL_COMPONENT_MAP`

Use these indirectly through:

- `DynamicFormField`
- `DynamicValue`
- `useDocyrusFormView`
- `useDocyrusFieldComponent`

Do **not** create a parallel local switch statement unless there is a very strong reason.

## Render-context lookup matrix

`useDocyrusFieldComponent(field.type, kind)` supports five render kinds:

| kind | Typical use | Unknown / unsupported fallback |
|------|-------------|--------------------------------|
| `form-field` | editable forms | `null` |
| `value-renderer` | read-only detail | `TextValue` |
| `data-grid-cell-variant` | grid cells | `ShortTextCell` |
| `editable-value` | single-field inline edit | `EditableValue` |
| `tanstack-column-def` | dynamic table columns | builder with short-text cell fallback |

This fallback behavior is the baseline for every manual page and hook-first page.

## What each dispatcher does

### `DynamicFormField`

- resolves `form-field`
- returns `null` when the field type has no registered editable component
- best for metadata-driven editable forms

### `DynamicValue`

- resolves `value-renderer`
- always returns a renderable component because the fallback is `TextValue`
- best for metadata-driven read-only layouts

### `EditableValue`

- is the inline-edit dispatcher
- returned for the `editable-value` kind regardless of field type
- internally decides whether the field behaves like inline edit, instant save, explicit save, popover edit, or read-only display

### `useDocyrusFormView`

- uses `form-field` in editable mode
- uses `value-renderer` in read-only mode
- handles unsupported editable fields differently by mode:
  - create mode default: `unsupportedFieldBehavior = 'skip'`
  - edit/view mode default: `unsupportedFieldBehavior = 'value'`

That default is usually correct because create screens should not tease unusable inputs, while edit/view screens can still show existing values safely.

## Recommended unsupported-field strategy

### Create mode

Prefer skipping unsupported editable fields unless the page explicitly designs a read-only preview row.

Why:

- users expect every visible field in a create form to be fillable
- showing a read-only renderer inside create mode can be confusing
- the hook default already follows this rule

### Edit mode

Prefer value-render fallback when the field cannot be edited.

Why:

- users can still review the field
- the record detail stays complete
- you avoid broken partial editors

### View mode

Always prefer value renderers.

That is the natural mode for detail pages, approval reviews, summaries, and read-only sheets.

## Safe manual pattern

When building a manual dynamic page, use this decision order:

```tsx
const FormField = useDocyrusFieldComponent(field.type, 'form-field');
const Value = useDocyrusFieldComponent(field.type, 'value-renderer');

if (mode === 'view') {
  return <Value field={field} value={value} record={record} enumOptions={options} />;
}

if (FormField) {
  return <FormField field={field} form={form} enumOptions={options} />;
}

return mode === 'create'
  ? null
  : <Value field={field} value={value} record={record} enumOptions={options} />;
```

This keeps manual pages aligned with hook-first Docyrus behavior.

## When a field type needs more than mapping

Some field types require more than just selecting a component:

- enum/select-like fields need `enumOptions`
- user and relation editors need hydrated option lists
- status fields may need companion values like description and follow-up date
- money and phone fields need companion currency/country keys
- avatar-like fields can require companion image or color keys

A field can be “supported” in the map but still render weakly if its supporting data is missing.

## Adding support for a new field type

When extending Docyrus UI, check all relevant layers:

1. add the editable component to `FORM_FIELD_MAP` if the type is editable
2. add the read-only renderer to `VALUE_RENDERER_MAP`
3. add the grid cell component or fallback strategy if the type should appear richly in grids
4. update any companion read/write logic if the field stores secondary keys
5. verify `useDocyrusFormView`, `DynamicFormField`, `DynamicValue`, and `EditableValue` behavior

## Debug checklist

- **Field disappears in manual edit mode?** `form-field` likely resolved to `null`.
- **Field shows raw text instead of richer UI?** You are hitting the `value-renderer` fallback or missing supporting props.
- **Create form unexpectedly shows read-only rows?** Your unsupported strategy is too permissive for create mode.
- **Inline field never edits?** The field may be effectively read-only in `EditableValue` or missing required option data.
- **New field type works in forms but not in details or grids?** You updated one map but not the others.
