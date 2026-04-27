# Advanced Inline Editing and Renderer Patterns

## Read this when

- The page should edit fields inline instead of switching to a dedicated form.
- You need change tracking or save/discard actions.
- You need to decide between `EditableRecordDetail`, `EditableValue`, `DynamicValue`, and raw form fields.
- A field type has companion values that must survive save flows.

## `EditableRecordDetail` for record-level inline editing

Use `EditableRecordDetail` when the whole record should be displayed as a detail surface, but each row can become editable.

```tsx
import {
  EditableRecordDetail,
  EditableRecordDetailField
} from '@docyrus/ui/components/editable-record-detail';

<EditableRecordDetail
  fields={fields}
  record={record}
  onSave={async (changes, values) => {
    void changes;
    void values;
  }}
  trackChanges>
  <EditableRecordDetailField slug="full_name" />
  <EditableRecordDetailField slug="email" />
  <EditableRecordDetailField slug="status" />
</EditableRecordDetail>
```

Use it when:

- the page is a detail panel or sheet
- users should edit a few fields in place
- you want a floating save/discard bar with changed-field summaries

## `EditableValue` for single-field inline editing

Use `EditableValue` when only one field should toggle between display and edit modes.

```tsx
<EditableValue
  field={field}
  value={record[field.slug]}
  record={record}
  enumOptions={enumOptions}
  onValueChange={(next) => saveOneField(field.slug, next)}
  onCompanionChange={(companion) => saveCompanionFields(companion)}
  changed={isChanged}
  trackChanges
/>
```

This is the right primitive for compact inline edits inside cards, summaries, or custom detail rows.

## Field behavior categories inside `EditableValue`

`EditableValue` is not a simple text editor. It dispatches behavior by field type:

- inline types save on blur or Enter
- instant-save types commit immediately on change
- explicit-save types show confirm/cancel actions
- popover types keep edit mode stable while focus moves into portaled content
- read-only types stay display-only

That is why `useDocyrusFieldComponent(..., 'editable-value')` always returns `EditableValue` instead of a field-type-specific component.

## Companion fields matter

Manual inline save flows must preserve companion keys for composite field types.

Common examples:

- money → value + `__slug_currency`
- phone → value + `__slug_country`
- status → value + `__slug_secondary`, `__slug_description`, `__slug_followup_date`
- avatar → main value plus mapped companion fields

If you save only the visible main field, you can silently corrupt or flatten the record state.

## Renderer selection rules

Use these defaults:

- full editable form → form-field component
- full read-only detail → value renderer
- one-off inline field → `EditableValue`
- view-first detail screen with record-level inline save → `EditableRecordDetail`

If a field has no editable form component, prefer a value-render fallback instead of inventing a broken partial editor.

## Value-renderer guidance

Value renderers are not just pretty labels. They understand field semantics:

- `StatusValue` reads status companion data like description and follow-up date
- `MoneyValue` formats amounts with currency context
- `UserValue` and `UserMultiValue` present users properly
- `RelationValue` renders relationship display values
- `RichTextValue` and `DocEditorValue` display stored rich content safely

Prefer the value renderer over custom text formatting whenever you are showing a Docyrus field type in read-only mode.

For the pure field-type dispatch matrix and unsupported-field defaults, also read `field-type-mapping-and-fallbacks.md`.

## Shared maps are the source of truth

The safest advanced pattern is to stay on the shared registries exposed by `useDocyrusFieldComponent`:

- `FORM_FIELD_MAP`
- `VALUE_RENDERER_MAP`
- `CELL_COMPONENT_MAP`

That keeps forms, detail views, data-grid cells, and inline editing behavior aligned.

## Good combinations

### Detail page with manual sections + inline fields

- layout shell: your own markup
- read-only rows: `DynamicValue`
- editable rows: `EditableValue`
- bulk record save experience: `EditableRecordDetail`

### Hook-first detail page with selective inline editing

- outer data workflow: `useDocyrusFormView`
- default layout: `renderLayout()` in `view` mode
- inline edit UX: `clickToEdit: true`

## Debug checklist

- **Field never becomes editable?** It may be a read-only field type or lack a registered form-field component.
- **Inline save loses currency/country/status metadata?** You forgot companion-field handling.
- **Renderer shows weak output?** Check whether `record` or `enumOptions` is incomplete.
- **Detail page edits but does not show change state?** Enable `trackChanges` and wire changed-state inputs correctly.
- **Custom manual switch statement is drifting from Docyrus UI behavior?** Replace it with `useDocyrusFieldComponent`, `DynamicFormField`, or `DynamicValue`.
