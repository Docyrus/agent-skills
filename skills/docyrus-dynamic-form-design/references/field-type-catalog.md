# Field types

A form does not choose a field's type — the **data source does**. The renderer dispatches on the data-source field's own `type`; the `type` you write in `fieldConfig` is a mirror that keeps the document readable and drives the studio UI. Always copy the type reported by `docyrus studio list-fields`; never invent one to change how a field renders.

```bash
docyrus studio list-fields --appSlug crm --dataSourceSlug contact --json
```

## Palette types (41)

These are the types a form designer works with day to day, grouped as the studio palette groups them.

| Group | Types |
|-------|-------|
| Text & input | `field-text`, `field-textarea`, `field-email`, `field-url`, `field-phone`, `field-color` |
| Numbers | `field-number`, `field-money`, `field-percent`, `field-rating`, `field-duration`, `field-currency` |
| Selection | `field-select`, `field-multiSelect`, `field-tagSelect`, `field-radioGroup`, `field-enum`, `field-status`, `field-approvalStatus` |
| Date & time | `field-date`, `field-dateTime`, `field-time`, `field-dateRange` |
| Toggle | `field-switch`, `field-checkbox` |
| Visual & media | `field-icon`, `field-avatar`, `field-file`, `field-image` |
| Relation & user | `field-userSelect`, `field-userMultiSelect`, `field-relation`, `field-locationSelect` |
| Rich content | `field-htmlEditor`, `field-codeEditor`, `field-docEditor`, `field-emailEditor`, `field-adaptiveCard` |
| Data structure | `field-taskList`, `field-schemaRepeater`, `field-queryBuilder` |

## Other renderable types (27)

Legal on a data source but not offered in the palette — usually system- or schema-owned. Most render through the same dispatcher; put them on a form only when the data source actually has such a field:

`field-password`, `field-display`, `field-markdown`, `field-formula`, `field-relatedField`, `field-code`, `field-button`, `field-identity`, `field-autonumber`, `field-json`, `field-jsonSchema`, `field-jsonata`, `field-handlebars`, `field-dsql`, `field-inlineData`, `field-inlineForm`, `field-list`, `field-todo`, `field-conversationChannel`, `field-dynamic`, `field-schema`, `field-fileStorageFolder`, `field-systemEnum`, `field-systemBuffer`, `field-systemVector`, `field-systemTextArray`, `field-systemUuidArray`

**Three of those have no renderer at all** — `field-dsql`, `field-dynamic` and `field-schema` are registered in neither the form-field map nor the value-renderer map. Listing one produces an unsupported field: it is skipped, or shown as a raw value, depending on the host's `unsupportedFieldBehavior`. Do not put them on a form.

An unsupported or unknown type never crashes the form — it degrades to an empty slot or a read-only value row.

## Types with extra requirements

| Type(s) | What the data source must provide | Form-side note |
|---------|-----------------------------------|----------------|
| `field-select`, `field-multiSelect`, `field-tagSelect`, `field-radioGroup`, `field-enum`, `field-status`, `field-approvalStatus` | Enum options on the field (`enums`, or a tenant enum set) | The form does **not** carry option lists. To change options, edit the field's enums — `docyrus studio list-enums` / `update-enums`. |
| `field-userSelect`, `field-userMultiSelect` | — | Options come from tenant users at runtime. |
| `field-relation` | `relationDataSourceId` on the field | Without a target the picker has nothing to load. Fix it on the field, not the form. |
| `field-money` | A companion currency column (`__<slug>_currency`) | Include the money field; the companion is handled by the renderer. |
| `field-rating` | `maxRating` on the field (default 5, max 10) | Not settable per form. |
| `field-file`, `field-image`, `field-avatar` | — | The host app must supply upload handlers; a form cannot enable uploads on its own. |
| `field-display`, `field-formula`, `field-relatedField`, `field-identity`, `field-autonumber` | — | Read-only by type — the renderer forces it regardless of the form. They render as value rows; never mark them `required`. |

## Read-only and computed fields

A field is read-only when the data-source field says so, when the form's `fieldConfig` marks it, when the layout is rendered in `view` mode, or when its type is one of the five inherently read-only types above. Read-only fields:

- render as a value row instead of an input,
- are **excluded from the submitted payload**,
- are skipped by validation.

Because a saved layout switches the renderer to "include read-only fields", listing one is how you show a computed or audit column on the form. Omit it entirely to hide it.

## Choosing a column span

`columnSpan` is per node, relative to the grid it sits in (the form's `gridColumns`, or the section's own). Rules of thumb that keep forms readable:

- Long text (`field-textarea`, `field-htmlEditor`, `field-docEditor`, `field-codeEditor`, `field-emailEditor`) → full width.
- Structural fields (`field-taskList`, `field-schemaRepeater`, `field-queryBuilder`, `field-adaptiveCard`) → full width.
- Short scalars (dates, numbers, switches, selects) → 1 column.
- `field-locationSelect` and `field-dateRange` → 2 columns when the grid allows.
