# Manual Form and Detail Patterns

## Use this path when

- The page already owns its form state.
- You want a completely custom layout.
- The record is already loaded elsewhere.
- You need precise control over when editable inputs versus read-only renderers appear.

## Manual editable form with `DynamicFormField`

```tsx
'use client';

import { useForm } from '@tanstack/react-form';
import { DynamicFormField, type IField, type EnumOption } from '@docyrus/ui/components/form-fields';

const fields: IField[] = [
  { id: '1', name: 'Full Name', slug: 'full_name', type: 'field-text' },
  { id: '2', name: 'Status', slug: 'status', type: 'field-select' }
];

const statusOptions: EnumOption[] = [
  { id: 'lead', slug: 'lead', name: 'Lead', color: '#64748b' },
  { id: 'customer', slug: 'customer', name: 'Customer', color: '#22c55e' }
];

export function ContactForm() {
  const form = useForm({
    defaultValues: {
      full_name: '',
      status: 'lead'
    },
    onSubmit: async ({ value }) => {
      void value;
    }
  });

  return (
    <form onSubmit={(event) => { event.preventDefault(); void form.handleSubmit(); }}>
      {fields.map((field) => (
        <DynamicFormField
          key={field.id}
          field={field}
          form={form}
          enumOptions={field.slug === 'status' ? statusOptions : undefined} />
      ))}
    </form>
  );
}
```

Use the specific form-field components directly when you want more explicit imports or field-specific props. Use `DynamicFormField` when the field list is metadata-driven.

## Manual read-only detail with `DynamicValue`

```tsx
import { DynamicValue } from '@docyrus/ui/components/value-renderers';

function ContactDetail({ fields, record, statusOptions }: {
  fields: IField[];
  record: Record<string, unknown>;
  statusOptions: EnumOption[];
}) {
  return (
    <div className="grid gap-4 md:grid-cols-2">
      {fields.map((field) => (
        <div key={field.id} className="space-y-1">
          <div className="text-sm font-medium">{field.name}</div>
          <DynamicValue
            field={field}
            value={record[field.slug]}
            record={record}
            enumOptions={field.slug === 'status' ? statusOptions : undefined} />
        </div>
      ))}
    </div>
  );
}
```

Use this when the page is read-only or when some fields are intentionally not editable.

## Shared registry pattern with `useDocyrusFieldComponent`

When you need total control, use the registry hook directly instead of writing a per-type switch.

```tsx
const FormField = useDocyrusFieldComponent(field.type, 'form-field');
const Value = useDocyrusFieldComponent(field.type, 'value-renderer');

if (isEditing && FormField) {
  return <FormField field={field} form={form} enumOptions={options} />;
}

return <Value field={field} value={value} record={record} enumOptions={options} />;
```

This is the safest manual pattern because it stays aligned with the Docyrus UI field maps.

## When to choose which primitive

- `DynamicFormField`: metadata-driven editable inputs.
- specific form-field component: explicit field type, custom props, or smaller import surface.
- `DynamicValue`: metadata-driven read-only display.
- specific value renderer: explicit display control for a known field type.
- `useDocyrusFieldComponent`: custom render orchestration that still stays on the shared registry.

## Manual option requirements

Manual pages must supply the option data that the renderer or field needs:

- select-like fields need `enumOptions`
- relation renderers usually need `record` context
- user and relation field editors often need hydrated options upstream
- composite renderers such as status, money, phone, and avatar depend on companion values in the record object

If those values are absent, the component may render a weaker fallback.

## Common manual pattern

Use editable inputs in edit mode and read-only renderers in view mode, but keep the same field list and the same metadata source. That keeps create, edit, and detail pages consistent even when the layout differs.

For inline editing and change tracking, also read `advanced-inline-edit-and-renderers.md`.
For field-type dispatch rules and unsupported-field fallback strategy, also read `field-type-mapping-and-fallbacks.md`.
