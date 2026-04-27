---
name: docyrus-record-detail-form-design
description: Build Docyrus React record forms, detail sheets, inspector panels, and inline-edit record UIs with `useDocyrusFormView`, `DynamicFormField`, `DynamicValue`, `EditableRecordDetail`, `EditableValue`, and value renderers. Use when asked to create or refactor create/edit/view pages, modal or sheet record forms, click-to-edit detail panels, metadata-driven field layouts, manual read-only record summaries, or field-type mapping logic that must choose correctly between form inputs, inline editors, and value-renderer fallbacks.
---

# Docyrus Record Detail Form Design

Build full Docyrus record create, edit, and detail experiences around shared field metadata.

## Choose the build mode

1. **Standard create/edit/view record screen** → use `useDocyrusFormView`.
   - Best when the page is a normal Docyrus form or detail surface.
   - Handles metadata loading, item loading, option hydration, field rendering, and submit behavior.
   - Read `references/hook-form-view.md`.

2. **Custom manual form or detail layout** → use `DynamicFormField`, `DynamicValue`, or `useDocyrusFieldComponent`.
   - Best when the page already owns form state, layout, or data fetching.
   - Read `references/manual-form-detail-patterns.md`.

3. **Inline editing, click-to-edit detail rows, or field-by-field editing** → use `EditableRecordDetail` and `EditableValue`.
   - Best when users should edit a record in place without switching to a full form.
   - Read `references/advanced-inline-edit-and-renderers.md`.

4. **Complex query/schema/API work** → also load `docyrus-api-dev` or `docyrus-app-dev-react`.
5. **Field-type mapping or unsupported-field decisions** → read `references/field-type-mapping-and-fallbacks.md`.

## Default workflow

1. Confirm `appSlug`, `dataSourceSlug`, `mode`, and `itemId` if the record already exists.
2. Decide whether the page should be hook-first or fully manual.
3. Keep editable fields and read-only renderers on the same field-type registry.
4. Hydrate enum, user, and relation options before declaring the page done.
5. Verify create, edit, view, reset, and save behavior.
6. If the page supports inline editing, verify companion-field saves for composite types such as money, phone, and status.

## Non-negotiables

- Prefer `useDocyrusFormView` for standard create/edit/view pages.
- Manual editable layouts should use `DynamicFormField`, specific form-field components, or `useDocyrusFieldComponent(..., 'form-field')`.
- Manual read-only layouts should use `DynamicValue`, specific value renderers, or `useDocyrusFieldComponent(..., 'value-renderer')`.
- Do not invent parallel per-field switch statements when the shared registry already solves the dispatch.
- `useDocyrusFormView` already keeps form fields, value renderers, and inline detail mode aligned through `useDocyrusFieldComponent`.
- `clickToEdit` on `useDocyrusFormView` view layouts routes through `EditableRecordDetail`; use it when you want view-first UX with inline saves.
- Manual Docyrus item fetching must always send `columns`. `useDocyrusFormView` derives them automatically, including companion columns.
- Composite fields can require companion keys on read and write: money, phone, status, avatar, and similar types must stay intact in manual save flows.
- If a field has no editable component, create-mode UX should usually skip it; edit/view UX should usually fall back to a value renderer unless you have a better explicit design.

## References

- `references/hook-form-view.md` — hook-first create, edit, view, layout, submit, and click-to-edit patterns.
- `references/manual-form-detail-patterns.md` — manual form and detail composition with field components, renderers, and shared field maps.
- `references/advanced-inline-edit-and-renderers.md` — `EditableRecordDetail`, `EditableValue`, renderer selection rules, and companion-field save patterns.
- `references/field-type-mapping-and-fallbacks.md` — shared field maps, fallback rules, and unsupported-field strategy by render mode.
