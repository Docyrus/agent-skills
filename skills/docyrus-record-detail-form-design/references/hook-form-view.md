# Hook-first Record Forms and Detail Views

## Use this path when

- The page is a standard Docyrus create, edit, or read-only record screen.
- You want field metadata, item loading, option hydration, and submit behavior handled in one place.
- You want the same field list to render as editable inputs in one mode and value renderers in another.

## Minimal create form

```tsx
'use client';

import { useDocyrusAuth } from '@docyrus/signin';
import { useDocyrusFormView } from '@docyrus/ui/library/hooks/use-docyrus-form-view';
import { Button } from '@docyrus/ui/primitives/ui/button';

export function CreateContactForm() {
  const { client } = useDocyrusAuth();

  if (!client) return null;

  const formView = useDocyrusFormView({
    client,
    appSlug: 'crm',
    dataSourceSlug: 'contacts',
    mode: 'create',
    gridColumns: 2,
    defaultValues: { status: 'lead' },
    fieldOrder: ['full_name', 'email', 'phone', 'status', 'notes'],
    fieldLayout: {
      notes: { colSpan: 'full' }
    }
  });

  return (
    <form
      className="space-y-4"
      onSubmit={async (event) => {
        event.preventDefault();
        await formView.submit();
      }}>
      {formView.renderLayout()}
      <div className="flex items-center gap-2">
        <Button type="submit" disabled={formView.isSubmitting || formView.isLoading}>Create</Button>
        <Button type="button" variant="outline" onClick={formView.reset}>Reset</Button>
      </div>
    </form>
  );
}
```

## Minimal edit and view patterns

- **edit**: pass `mode: 'edit'` and `itemId`.
- **view**: pass `mode: 'view'` and `itemId`.
- **prefetched record**: pass `item` to skip the item query.
- **generated collection**: pass `collection` so the hook uses `collection.get`, `collection.create`, and `collection.update`.

## Click-to-edit detail mode

In `view` mode, `clickToEdit: true` swaps the plain value layout for `EditableRecordDetail` while still using the normal Docyrus submit pipeline.

```tsx
const detailView = useDocyrusFormView({
  client,
  appSlug: 'crm',
  dataSourceSlug: 'contacts',
  itemId: contactId,
  mode: 'view',
  clickToEdit: true,
  fieldOrder: ['full_name', 'email', 'phone', 'status', 'notes']
});

return detailView.renderLayout();
```

Use this when the page should feel like a detail sheet first, but still allow inline field edits.

## High-value options

- `fieldOrder`: impose explicit field ordering.
- `fieldSlugs`: whitelist fields.
- `hiddenFieldSlugs`: hard-hide fields.
- `fieldLayout`: override hidden, required, readOnly, disabled, colSpan, label, description, and per-field props.
- `layout`: build nested `fieldset`, `tabpanel`, and `tab` sections.
- `mapField`: transform normalized field metadata or drop a field entirely.
- `includeReadOnlyFields`: keep read-only rows in the rendered layout.
- `unsupportedFieldBehavior`: choose whether unsupported editable fields are skipped or rendered as values.
- `resolveUserOptions`, `resolveRelationOptions`, `enumOptions`: control option hydration.
- `transformSubmit`: final payload transform before create/update.
- `onSubmit`: replace default submit behavior entirely.
- `onSubmitSuccess`, `onSubmitError`: react to mutation outcomes.

## How render mode is decided

The hook resolves components from the shared field-component registry:

- editable mode → `form-field`
- read-only mode → `value-renderer`

It does not keep a separate form/detail mapping system. That means `useDocyrusFormView`, `DynamicFormField`, `DynamicValue`, and data-grid field behavior stay aligned when the registry changes.

## Why this hook is usually the default

It handles three difficult layers together:

1. data-source metadata
2. current item loading
3. dynamic option hydration for enum, user, and relation fields

It also derives the required backend `columns`, including companion columns for composite fields.

## Important behavior

- `renderField(slug)` renders a single resolved field.
- `renderLayout()` renders the full layout grid and is usually the fastest path.
- `reset()` restores the latest committed baseline.
- successful `submit()` updates the clean baseline, so the page stops being dirty.
- in `view` mode, `submit()` returns current values and skips normal validation.

## Default recommendation

Start with `useDocyrusFormView` unless the page already has its own form state, its own item query lifecycle, or a heavily custom layout system that would fight the hook.

For deeper inline editing and renderer behavior, also read `advanced-inline-edit-and-renderers.md`.
For shared field-map behavior and unsupported-field defaults, also read `field-type-mapping-and-fallbacks.md`.
