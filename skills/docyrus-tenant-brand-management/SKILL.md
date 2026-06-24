---
name: docyrus-tenant-brand-management
description: Create, read, update, and delete a tenant's brand identity records (`tenant_brand`) using the `docyrus account brands` CLI commands. Use when the user wants to manage a Docyrus brand — set or change the brand's colors, typography, spacing, logos/favicons, voice & messaging guidelines, slide/presentation defaults, chart styling, illustration rules, or content constraints; mark a default brand; archive a brand; or import branding from a website. Triggers on "create a brand", "set brand colors", "update the brand voice/typography", "add a logo to the brand", "make this the default brand", "archive a brand", "import brand from website", "brand guidelines", `docyrus account brands`, `account brands create`, `account brands update`, `tenant_brand`, or any tenant brand identity / brand-kit management task in Docyrus. For the full CLI command index see docyrus-cli-app; for platform concepts see docyrus-platform.
---

# Docyrus Tenant Brand Management

A **tenant brand** (`tenant_brand`) is a tenant-owned brand-identity record: the
visual system (colors, typography, spacing, components, logos), the verbal system
(voice, tone, terminology, content constraints), and presentation/chart/illustration
defaults that downstream features (documents, slides, AI generation) read from.

Manage brands with `docyrus account brands`. There is **no design-time vs runtime
split** — these commands directly CRUD the live records for the active tenant.

- A brand is identified by its UUID (`--brandId`). There is no brand slug.
- The only required field is `--name` (on create); everything else is optional.
- The record is large. For the **complete field catalog** (every flag, its
  `snake_case` column, type, and meaning) read
  [references/fields.md](references/fields.md). Keep this open while building a brand.

## Workflow

1. **Confirm auth + tenant.**
   ```bash
   docyrus auth who --json
   ```
   No session → stop and ask the user to run `docyrus auth login`. Brands are
   tenant-scoped, so confirm the active tenant is the intended one.

2. **List existing brands** before creating one — a tenant often already has a
   default brand to update rather than duplicate.
   ```bash
   docyrus account brands list --json
   ```

3. **Create or update.** Start with identity + the few fields the user cares
   about; the rest can be filled in later updates. See [Commands](#commands).

4. **Validate.** Re-read the brand and confirm the values landed.
   ```bash
   docyrus account brands get --brandId <uuid> --json
   ```

## Commands

Add `--json` for machine-readable output. Run `docyrus account brands <cmd> --help`
for the live flag list.

```bash
# List non-archived brands (default brand first)
docyrus account brands list --json

# Get one brand by id
docyrus account brands get --brandId <uuid> --json

# Create — only --name is required
docyrus account brands create --name "Acme" --json

# Create with common identity + visual fields
docyrus account brands create \
  --name "Acme" \
  --description "Acme corporate brand" \
  --websiteUrl "https://acme.com" \
  --isDefault true \
  --colorPrimary "#1d4ed8" \
  --colorSecondary "#9333ea" \
  --logoUrl "https://acme.com/logo.svg" \
  --fontFamilyPrimary "Inter" \
  --voiceIntent "bold,concise,friendly" \
  --json

# Update specific fields (partial; only sent keys change)
docyrus account brands update --brandId <uuid> \
  --colorPrimary "#0f766e" \
  --formalityLevel "casual" \
  --json

# Archive / restore (update-only flag)
docyrus account brands update --brandId <uuid> --archived true --json
docyrus account brands update --brandId <uuid> --archived false --json

# Delete (hard delete — returns { deleted: true, id })
docyrus account brands delete --brandId <uuid> --json

# Import branding from the brand's website (Firecrawl scrape, applied to the record)
docyrus account brands fetch-from-website --brandId <uuid> --json
```

### Structured (`json`/`list`) fields and bulk payloads

- **`list` flags** are comma-separated and sent as arrays:
  `--grammarDo "Use active voice,Lead with the verb"`.
- **`json` flags** take a JSON string parsed into an object/array:
  `--spacingPadding '{"sm":"8px","md":"16px"}'`,
  `--fonts '[{"family":"Inter","weights":[400,700]}]'`.
- For many fields at once, pass the whole record as JSON. Use the `snake_case`
  **column** names (see [references/fields.md](references/fields.md)):
  ```bash
  docyrus account brands create --from-file brand.json --json
  docyrus account brands update --brandId <uuid> --data '{"color_primary":"#1d4ed8","tone_by_context":{"support":"warm"}}' --json
  ```
  `--data` / `--from-file` accept **JSON only**. When a convenience flag and a
  `--data` key set the same field, the **flag wins**.

## Key rules

- **`--name` required on create**; the CLI rejects a create with no name before
  calling the API.
- **`--archived` is update-only.** Passing it to `create` fails (unknown flag).
  `list` hides archived brands; `get` still returns them.
- **One default per tenant.** `is_default` has a unique partial index — don't try
  to create a second default; update the intended brand instead.
- **`delete` is a hard delete** and returns `{ deleted: true, id }` (the API
  itself returns `{ data: { success: true } }`). There is no restore for a
  deleted brand — prefer `--archived true` when the user may want it back.
- **`fetch-from-website` requires the brand to have `website_url` set** and
  overwrites scraped fields (colors, typography, spacing, components, images,
  icons/animations/layout/personality) on the record.
- Flags are camelCase and map 1:1 onto the `snake_case` columns; the full mapping
  is in [references/fields.md](references/fields.md).

## Related skills

- **docyrus-cli-app** — full `docyrus` CLI command index and global conventions (`--json`, auth, environments).
- **docyrus-platform** — Docyrus platform concepts and building blocks.
