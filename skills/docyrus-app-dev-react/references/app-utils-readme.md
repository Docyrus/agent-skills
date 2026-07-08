<!--
Mirror of the published `@docyrus/app-utils` package README (npm: `@docyrus/app-utils`,
source: `docyrus-devkit/packages/app-utils`). Kept here as the authoritative runtime-utils
reference for this skill (tenant preferences, date/number formatting, app/user config,
data views/forms, data-source metadata, and the Inventory Cache). If the package README
changes, refresh this file.
-->

# @docyrus/app-utils

Framework-agnostic utility functions for frontend applications using Docyrus as backend. Provides date/number formatting, template compilation (Handlebars + JSONata), and a comprehensive set of expression helpers — all configured from tenant preferences.

## Features

- **Tenant Preferences**: Fetch tenant configuration from the Docyrus API
- **Date Formatting**: PHP-style date formatting with timezone support (`Y-m-d`, `H:i:s`, etc.)
- **Number Formatting**: Locale-aware number formatting with custom separators and precision
- **Number to Words**: Convert numbers to words in Turkish and English with currency labels
- **Duration Formatting**: Format seconds as `HH:MM`, decimal hours, or human-readable words
- **App Config**: Get, upsert, and delete a per-app JSON configuration (1:1 per app)
- **User App Config**: Get, upsert, and delete a per-user JSON configuration scoped to an app (1:1 per user per app)
- **Data Views**: CRUD for saved data source view configurations (columns, filters, sort, color rules)
- **Data Forms**: CRUD for dynamic form designs attached to a data source (layout JSON, title, icon, color, default flag, soft-archive)
- **Data Source Metadata**: Read data source metadata across apps, by app, by ID, or by slug with optional expansions like `fields`, plus field & enum-option helpers (`getFields`, `getFieldById`, `getFieldBySlug`, `getEnumsByFieldId`, `getEnumById`) and relation helpers for parent/child data-source & field selectors (`listParentDataSources`, `listChildDataSources`, `listParentDataSourceFields`, `listChildDataSourceFields`)
- **Inventory Cache**: Tenant-wide in-memory cache of apps, data sources (with embedded views/forms/fields), users, brands, tenant preferences, and per-app config — with request de-dup, TTL, direct cache patches shared across all inventory-backed clients, and a `load()` warm-up that emits progress events for a post-sign-in progress bar
- **Brands**: CRUD for tenant brand records (colors, typography, spacing, logos, voice & messaging, slide / chart styling), plus a website-scrape RPC that fills the brand from a configured URL
- **User Identity**: Persist decoded OIDC ID-token payloads from Microsoft (Graph) and Google after a frontend OAuth2 flow — writes both `tenant_user.identity_*` and the matching `tenant_connection_user` row
- **TanStack Query Integration** (optional, via `@docyrus/app-utils/query`): Prebuilt `queryOptions` factories and a query-key registry for data source metadata, with reactive caching and per-resource invalidation helpers
- **Envelope Unwrapping**: `envelopeUnwrapInterceptor` (a `RestApiClient` interceptor) and `unwrapEnvelope` (a per-call helper) for handling Docyrus' `{ success, data }` API envelope — both shape-strict so domain payloads with their own `data` column stay safe
- **Template Engine**: Handlebars compilation with async support and inline JSONata formulas
- **Formula Evaluation**: JSONata expression evaluation with 70+ built-in helper functions
- **JSONata Helpers**: Date arithmetic (date-fns), string manipulation, number formatting, equality checks, and more
- **TypeScript**: Full type definitions included
- **Framework-Agnostic**: No Vue, React, or Angular dependency — works everywhere

## Installation

```bash
npm install @docyrus/app-utils
```

```bash
pnpm add @docyrus/app-utils
```

### Peer Dependencies

| Package | Version | Required |
|---------|---------|----------|
| `@docyrus/api-client` | >= 0.1.0 | Yes (for `getTenantPreferences`) |
| `handlebars` | >= 4.7.0 | Yes (for template engine) |
| `jsonata` | >= 2.0.0 | Yes (for formula evaluation) |
| `date-fns` | >= 3.0.0 | Yes (for JSONata date helpers) |
| `@tanstack/query-core` | >= 5.17.0 | Optional (only for `@docyrus/app-utils/query`) |

## Quick Start

```ts
import {
  getTenantPreferences,
  createDateUtils,
  createNumberUtils,
  createTemplateEngine,
  createDataSourceClient
} from '@docyrus/app-utils';

// 1. Fetch tenant preferences
const preferences = await getTenantPreferences(apiClient);

// 2. Create utility instances
const dateUtils = createDateUtils({
  preferences,
  userTimezone: currentUser.timeZone?.id // e.g. 'Europe/Istanbul'
});

const numberUtils = createNumberUtils({ preferences });
const dataSources = createDataSourceClient(apiClient);

// 3. Create template engine (optional)
const engine = createTemplateEngine({
  dateUtils,
  numberUtils,
  user: currentUser,
  extraJsonataBindings: { hasRole }
});

// 4. Use utilities
dateUtils.formatDate('2024-01-15');           // '15.01.2024' (tenant format)
numberUtils.formatNumber(1234.5);             // '1.234,50'   (tenant locale)
await engine.compileFormula('$sum(items.price)', data);

// 5. Read app and data source metadata
const apps = await dataSources.listApps();
const sources = await dataSources.list({ expand: 'fields' });
const contacts = await dataSources.getBySlug('crm', 'contacts', {
  expand: 'fields'
});

// 6. Use metadata
const appSlugs = apps.map(app => app.slug);
const fieldSlugs = contacts.fields?.map(field => field.slug) ?? [];
const requiredFields = contacts.fields?.filter(field => field.required) ?? [];
```

## Envelope handling

Docyrus REST endpoints wrap their payload in a `{ success, data }` envelope (optionally with `meta` for pagination, or `error` / `message` for controller-level errors). `@docyrus/app-utils` provides two ways to handle that — both are tolerant of pre-stripped envelopes (i.e. idempotent), and both refuse to falsely unwrap domain payloads that carry their own `data` field (`AppConfig`, `UserAppConfig`, records with a `data` column).

### `envelopeUnwrapInterceptor` — global, conservative

A `RestApiClient` response interceptor. After it runs, `client.get/post/put/delete` returns the inner payload directly (so generated TanStack DB collections, hand-written calls, and app-utils clients all see bare values). **Required** when using `@docyrus/tanstack-db-generator` collections — the generator emits methods that don't read `.data` themselves.

```ts
import { RestApiClient } from '@docyrus/api-client';
import { envelopeUnwrapInterceptor } from '@docyrus/app-utils';

const client = new RestApiClient({ baseURL: 'https://api.example.com' });
client.addInterceptor(envelopeUnwrapInterceptor);

// Now plain envelopes unwrap automatically:
const views = await client.get<DataView[]>('/v1/apps/crm/data-sources/contacts/views');
//      ^ DataView[] (from { success, data: [...] })
```

It's deliberately **shape-strict and lossless** — it strips only when the envelope's only top-level keys are `data` and optionally `success`. Envelopes that carry sidecar information are left intact so callers can read them:

| Response shape | Behaviour | Why |
|---|---|---|
| `{ data }` | unwrap → `data` | Bare envelope. |
| `{ success, data }` | unwrap → `data` | Most endpoints. |
| `{ data, meta }` / `{ success, data, meta }` | left alone | Paginated lists — `meta.count`/`meta.total` needed by data grids. |
| `{ success: false, data, error, message }` | left alone | Controller-level error envelope — error info preserved. |
| `{ id, data, status, tenant_app_id, … }` | left alone | Domain record (e.g. `AppConfig`) carrying its own `data` column. |

For paginated endpoints, read both the array and the page total off the envelope:

```ts
const { data: rows, meta } = await client.get<{
  data: Record[];
  meta?: { count?: number; total?: number };
}>('/v1/apps/crm/data-sources/contacts/items');
```

### `unwrapEnvelope<T>(value)` — per-call, aggressive

A small helper for ad-hoc unwrap at known sites. It's **wider** than the interceptor — it also strips envelopes carrying `meta` / `error` / `message` because the caller has explicitly typed the return as `T` and isn't surfacing sidecar fields. This is exactly what app-utils' built-in clients (`createDataSourceClient`, `createDataViewClient`, `createDataFormClient`, `createAppConfigClient`, `createUserAppConfigClient`) use internally, so they keep working whether or not you've registered the interceptor globally.

```ts
import { unwrapEnvelope } from '@docyrus/app-utils';

const records = unwrapEnvelope<Record[]>(
  await client.get('/v1/apps/crm/data-sources/contacts/items')
);
// → Record[] (meta intentionally discarded)
```

Domain payloads with a `data` column are still safe — the closed-set check (`{ data, success, meta, error, message }`) rejects records that carry off-set keys like `id` / `status`.

## App Config & Data Views

### `createAppConfigClient(client, appId, options?)`

Manages the single JSON configuration object for an app (1:1 relationship — no config IDs).

```ts
import { createAppConfigClient } from '@docyrus/app-utils';

const config = createAppConfigClient(apiClient, appId);
```

Pass an `InventoryClient` to read `get` from the inventory's per-app config cache and patch it after `upsert` / `remove` (`setAppConfig`). When the inventory has cached `null` (no config exists), `get` falls through to the API so you still receive a real `404`:

```ts
const config = createAppConfigClient(apiClient, appId, { inventory });
```

#### `config.get()`

Returns the app's configuration. Throws `404` if none exists.

```ts
const appConfig = await config.get();
// { id: '...', data: { theme: 'dark', ... }, status: 1, tenant_app_id: '...', ... }
```

#### `config.upsert(body)`

Creates the config if it doesn't exist, or updates it if it does.

```ts
const updated = await config.upsert({
  data: { theme: 'dark', sidebar: { collapsed: false } },
  status: 1
});
```

#### `config.remove()`

Hard deletes the config (irreversible).

```ts
await config.remove();
```

---

### `createUserAppConfigClient(client, appId, options?)`

Manages the per-user JSON configuration object for an app (1:1 per user per app). Useful for storing user preferences, UI state, or personalisation that should persist per user independently of the shared app config.

```ts
import { createUserAppConfigClient } from '@docyrus/app-utils';

const userConfig = createUserAppConfigClient(apiClient, appId);
```

Pass an `InventoryClient` to read `get` from the inventory's per-app user-config cache and patch it after `upsert` / `remove` (`setUserAppConfig`), mirroring `createAppConfigClient`:

```ts
const userConfig = createUserAppConfigClient(apiClient, appId, { inventory });
```

#### `userConfig.get()`

Returns the current user's configuration for the app. Throws `404` if none exists.

```ts
const config = await userConfig.get();
// { id: '...', data: { theme: 'dark', ... }, status: 1, tenant_app_id: '...', user_id: '...', ... }
```

#### `userConfig.upsert(body)`

Creates the config if it doesn't exist, or updates it if it does.

```ts
const updated = await userConfig.upsert({
  data: { theme: 'dark', sidebarCollapsed: true },
  status: 1
});
```

#### `userConfig.remove()`

Hard deletes the user config (irreversible).

```ts
await userConfig.remove();
```

---

### `createDataViewClient(client, appSlug, dataSourceSlug, options?)`

Manages saved view configurations for a specific data source (1:many). Views define columns, filters, sorting, color rules, and quick-filter shortcuts.

```ts
import { createDataViewClient } from '@docyrus/app-utils';

const dataViews = createDataViewClient(apiClient, 'my-app', 'my-data-source');
```

Pass an `InventoryClient` to read `list` / `get` from the cached data source's embedded `views` array, and patch the cache in place after every mutation:

```ts
const dataViews = createDataViewClient(apiClient, 'crm', 'invoices', { inventory });

await dataViews.list();                              // reads from inventory cache (no network)
await dataViews.create({ name: 'Overdue' });         // POST, then inventory.upsertDataView(result)
await dataViews.update(id, { is_default: true });    // PUT,  then inventory.upsertDataView(result)
await dataViews.update(id, { archived: true });      // PUT,  then inventory.removeDataView(id)
await dataViews.remove(id);                          // DELETE, then inventory.removeDataView(id)
```

#### `dataViews.list(params?)`

Returns all non-archived views. Optionally pass `appId` to filter views configured for a specific app (different from the data source's owning app).

```ts
const views = await dataViews.list();
const forSpecificApp = await dataViews.list({ appId: 'other-app-uuid' });
```

#### `dataViews.get(viewId)`

Returns a single non-archived view by ID.

```ts
const view = await dataViews.get('view-uuid');
```

#### `dataViews.create(body)`

Creates a new data view. `name` is required. The data source is determined by the `dataSourceSlug` passed to the client factory.

```ts
const view = await dataViews.create({
  name: 'High Priority',
  columns: { visible: ['title', 'priority', 'assignee'] },
  filters: { priority: 'high' },
  sort: { field: 'due_date', direction: 'asc' },
  color: '#E74C3C',
  icon: 'alert-triangle',
  is_default: false,
  sort_order: 2
});
```

#### `dataViews.update(viewId, body)`

Partially updates a view. Use `archived: true` to soft-delete.

```ts
await dataViews.update('view-uuid', { name: 'Renamed View' });
await dataViews.update('view-uuid', { archived: true }); // soft-delete
```

#### `dataViews.remove(viewId)`

Hard deletes a view (irreversible).

```ts
await dataViews.remove('view-uuid');
```

---

### `createDataFormClient(client, appSlug, dataSourceSlug, options?)`

Manages dynamic form designs attached to a data source (1:many). Forms describe the editable layout used to view and edit records of a data source — title, icon, color, the layout JSON itself (sections / fields / tabs), default selection, and soft-archive state.

```ts
import { createDataFormClient } from '@docyrus/app-utils';

const dataForms = createDataFormClient(apiClient, 'crm', 'contacts');
```

Pass an `InventoryClient` to read `list` / `get` from the cached data source's embedded `forms` array, and patch the cache in place after every mutation:

```ts
const dataForms = createDataFormClient(apiClient, 'crm', 'contacts', { inventory });

await dataForms.list();                              // reads from inventory cache (no network)
await dataForms.create({ name: 'Compact' });         // POST, then inventory.upsertDataForm(result)
await dataForms.update(id, { title: 'Customer' });   // PUT,  then inventory.upsertDataForm(result)
await dataForms.update(id, { archived: true });      // PUT,  then inventory.removeDataForm(id)
await dataForms.remove(id);                          // DELETE, then inventory.removeDataForm(id)
```

Note: embedded forms carry the layout/display fields needed for rendering, but **not** `description`, `status`, or audit timestamps. If you need those, create a non-inventory-backed `createDataFormClient` and call `get(id)` against the standalone endpoint.

#### `dataForms.list()`

Returns all non-archived forms for the data source, ordered by `created_on` ascending.

```ts
const forms = await dataForms.list();
```

#### `dataForms.get(formId)`

Returns a single non-archived form by ID. Throws `404` if the form does not exist or is archived.

```ts
const form = await dataForms.get('form-uuid');
```

#### `dataForms.create(body)`

Creates a new form. `name` is required; everything else is optional. The data source is determined by the `dataSourceSlug` passed to the client factory — `tenant_data_source_id`, tenant scope, and the `created_by` audit fields are populated server-side.

```ts
const form = await dataForms.create({
  name: 'Compact contact form',
  title: 'Contact',
  icon: 'user',
  color: '#4F46E5',
  layout: {
    sections: [
      { id: 'main', fields: ['name', 'email', 'phone'] }
    ]
  },
  is_default: false,
  status: 1
});
```

#### `dataForms.update(formId, body)`

Partially updates a form. Only the fields present in the body are written; omitted fields are preserved. `layout` is replaced wholesale when provided. Use `archived: true` to soft-archive (the row is then excluded from list/get).

```ts
await dataForms.update('form-uuid', {
  title: 'Customer Contact',
  icon: 'address-book',
  layout: { sections: [/* updated sections */] }
});

await dataForms.update('form-uuid', { archived: true }); // soft-delete
```

#### `dataForms.remove(formId)`

Hard deletes a form (irreversible). Use `update(formId, { archived: true })` instead if you want a recoverable soft-delete.

```ts
await dataForms.remove('form-uuid');
```

---

### `createBrandClient(client, options?)`

Manages tenant brand records — the visual identity, voice / messaging, and slide / chart styling configuration used to render brand-aware UI, presentation templates, and AI-generated content. Records are tenant-scoped automatically; cross-tenant access is impossible.

```ts
import { createBrandClient } from '@docyrus/app-utils';

const brands = createBrandClient(apiClient);
```

Pass an `InventoryClient` to read `list` / `get` from the cached brand list and patch the cache after every mutation (`create` / `update` → `upsertBrand`, `remove` / `update({ archived: true })` → `removeBrand`, `fetchFromWebsite` → merges the returned partial):

```ts
const brands = createBrandClient(apiClient, { inventory });
```

#### `brands.list()`

Returns all non-archived brands for the active tenant, with the default brand first and then by creation date ascending.

```ts
const brands = await brands.list();
```

#### `brands.get(brandId)`

Returns a single brand by ID. Throws `404` if the brand does not exist or belongs to another tenant.

```ts
const brand = await brands.get('brand-uuid');
```

#### `brands.create(body)`

Creates a new brand. Only `name` is required; everything else is optional and defaults to `null` (or the column default). `tenant_id` and the `created_by` / `last_modified_by` audit fields are populated server-side.

```ts
const brand = await brands.create({
  name: 'Acme Corp',
  description: 'Primary marketing brand',
  website_url: 'https://acme.com',
  color_primary: '#FF6600',
  color_secondary: '#1E1E1E',
  font_family_primary: 'Inter',
  is_default: true
});
```

> Note: the API does not enforce a single default brand per tenant — the caller is responsible for clearing `is_default` on the previous default when promoting a new one.

#### `brands.update(brandId, body)`

Partially updates a brand. Only the fields present in the body are written; omitted fields are preserved. Setting a field to `null` clears it. Use `archived: true` to soft-archive (archived rows are excluded from `list()`).

```ts
await brands.update('brand-uuid', {
  color_primary: '#0044CC',
  logo_url: 'https://cdn.example.com/logos/acme-v2.svg',
  is_default: false
});

await brands.update('brand-uuid', { archived: true }); // soft-archive
```

#### `brands.remove(brandId)`

Hard deletes the brand (irreversible). Use `update(brandId, { archived: true })` for a recoverable soft delete.

```ts
await brands.remove('brand-uuid');
```

#### `brands.fetchFromWebsite(brandId)`

Scrapes the brand's configured `website_url` with Firecrawl and merges the extracted branding (colors, typography, spacing, components, images, voice / personality, fonts, icons, animations, layout) into the brand record. Only fields detected by the scraper are overwritten — existing values for missing fields are preserved.

This is a **synchronous** RPC — the request blocks until Firecrawl returns, so expect several seconds of latency. The response is intentionally a flat top-level object (not the standard `{ data }` envelope) so callers can preview the raw scraper output alongside the persisted summary.

```ts
const { brand, scrapedBranding } = await brands.fetchFromWebsite('brand-uuid');
```

Throws `400` if the brand has no `website_url` configured, `404` if the brand doesn't exist, or `500` if Firecrawl is unconfigured or returns no branding data.

---

### `createUserIdentityClient(client)`

Persists the **decoded** OIDC ID-token payload from Microsoft (Graph) or Google for the authenticated user. Designed to be called from the frontend right after a successful OAuth2 flow (e.g. `msal-browser`, `google-identity-services`, `nimbus-auth-js`) — pass the verified ID-token claims and the backend writes the payload into `tenant_user.identity_microsoft` / `identity_google` **and** upserts the matching `tenant_connection_user` row so the provider connection is recorded alongside the identity claims.

These endpoints are hidden from Swagger, which is exactly why this helper exists — typed wrappers make them safer to call than hand-rolled `fetch`. They require the `User.ReadWrite` (or `Users.ReadWrite.All`) scope.

```ts
import { createUserIdentityClient } from '@docyrus/app-utils';

const userIdentity = createUserIdentityClient(apiClient);
```

#### `userIdentity.saveMicrosoft(payload)`

Persists a Microsoft / Entra ID-token payload. `sub` is required; all other well-known claims (`oid`, `tid`, `email`, `name`, `preferred_username`) are typed but optional, and any additional claims (`aud`, `iss`, `iat`, `exp`, …) pass through verbatim.

```ts
await userIdentity.saveMicrosoft({
  sub: '6zIRb2daYqUXVlg-xu4dA3nLsJekGkVhZxIg2GLpn2I',
  oid: 'f0ea18f8-cf7d-4695-8d9e-d9674127b343',
  tid: 'a2b0309e-37c1-486d-bdbd-4d91b7d25cd5',
  email: 'anil.beyazoglu@docy.team',
  name: 'Anıl Beyazoğlu',
  preferred_username: 'anil.beyazoglu@docy.team',
  iss: 'https://login.microsoftonline.com/.../v2.0',
  aud: '0380d712-0e97-431f-b343-76604e1cfcd1',
  iat: 1754578886,
  exp: 1754582786
});
```

#### `userIdentity.saveGoogle(payload)`

Persists a Google ID-token payload. `sub` is required; `email` and `name` are typed but optional, and any additional claims pass through verbatim.

```ts
await userIdentity.saveGoogle({
  sub: '110169484474386276334',
  email: 'anil@example.com',
  name: 'Anıl Beyazoğlu'
});
```

#### Behaviour notes

- Both calls return `void` — the server responds with `{ success: true }` and nothing useful to surface.
- Pass the **decoded ID-token payload only**, not the raw OAuth2 access / refresh tokens. The endpoints intentionally do **not** touch the `access_token` / `refresh_token` columns on the connection row.
- The connection record uses the well-known provider slug (`microsoft` / `google`), so consumers like the chat-platform webhook can match users via `identity_microsoft->>'tid'`, `'oid'`, `'sub'`, `'preferred_username'`, `'email'` and `identity_google->>'sub'`, `'email'`.
- Throws `404` if the active product has no provider-auth row for the slug, or `422` if `sub` is missing.

---

### `createDataSourceClient(client, options?)`

Reads Docyrus app and data source metadata across apps or for a specific app. This is useful when you need schema-aware UI behavior such as loading available apps, reading field definitions, or resolving a data source by slug.

```ts
import { createDataSourceClient } from '@docyrus/app-utils';

const dataSources = createDataSourceClient(apiClient);
```

#### Backing the client with an inventory cache

Pass an `InventoryClient` (see [`createInventoryClient`](#createinventoryclientclient-options)) to read every list/get method through the inventory cache instead of hitting the network on every call:

```ts
import { createInventoryClient, createDataSourceClient } from '@docyrus/app-utils';

const inventory = createInventoryClient(apiClient);
const dataSources = createDataSourceClient(apiClient, { inventory });

// First call populates the inventory cache (one network request).
// Subsequent calls — including from other clients sharing the same inventory — are cache hits.
await dataSources.listApps();
await dataSources.listByAppSlug('crm');
await dataSources.getBySlug('crm', 'contacts');
```

Cache resolution rules:
- If `params.expand` is **omitted**, the cached entry (fetched with the inventory's `dataSourcesExpand`, default `'views,forms,fields'`) is used — every returned data source already carries its embedded `views`, `forms`, and `fields` (with `enums`).
- If `params.expand` is **provided**, the request bypasses the cache and hits the API directly — the cached entry may not include the requested expansions.
- `getById` / `getBySlug` fall back to the API when the lookup misses the cache (e.g. an entry that was created after the cache was populated).

#### Expand options

The metadata endpoints support comma-separated expansions via `expand`.

Available values:
- `children`
- `fields`
- `forms`
- `acl`
- `templates`
- `client-configuration`
- `misc`

Example:

```ts
const sources = await dataSources.list({
  expand: 'fields,forms'
});
```

#### `dataSources.listApps()`

Lists apps available to the current tenant.

```ts
const apps = await dataSources.listApps();
// [{ id: 'uuid', name: 'CRM', slug: 'crm', logo_url: '...', status: 'active' }]
```

#### `dataSources.list(params?)`

Lists all data sources the current tenant can access across all apps.

```ts
const all = await dataSources.list();
const allWithFields = await dataSources.list({ expand: 'fields' });
```

#### `dataSources.listByAppSlug(appSlug, params?)`

Lists all data sources for a specific app slug.

```ts
const crmSources = await dataSources.listByAppSlug('crm');
const crmWithFields = await dataSources.listByAppSlug('crm', {
  expand: 'fields,acl'
});
```

#### `dataSources.listByAppId(appId, params?)`

Lists all data sources for a specific app ID.

```ts
const appSources = await dataSources.listByAppId('app-uuid');
const appSourcesWithFields = await dataSources.listByAppId('app-uuid', {
  expand: 'fields'
});
```

#### `dataSources.getById(dataSourceId, params?)`

Returns a single data source metadata object by data source ID.

```ts
const contacts = await dataSources.getById('data-source-uuid');
const contactsWithFields = await dataSources.getById('data-source-uuid', {
  expand: 'fields'
});
```

#### `dataSources.getBySlug(appSlug, dataSourceSlug, params?)`

Returns a single data source metadata object by app slug and data source slug.

```ts
const contacts = await dataSources.getBySlug('crm', 'contacts');
const contactsWithFields = await dataSources.getBySlug('crm', 'contacts', {
  expand: 'fields,forms'
});
```

#### Response shape

Basic responses include fields such as:

```ts
{
  id: 'uuid',
  name: 'Contacts',
  slug: 'contacts',
  appSlug: 'crm'
}
```

When `expand=fields` is used, the response can include `fields` metadata as well.

#### Field & enum helpers

Convenience readers that resolve a data source's fields and their enum options. When the client is [backed by an inventory](#backing-the-client-with-an-inventory-cache), they read straight from the cache (fields + enums are embedded via `dataSourcesExpand`, so **no extra network call**); otherwise they fetch the data source once with `expand=fields`.

```ts
const fields = await dataSources.getFields('data-source-uuid');
const status = await dataSources.getFieldBySlug('data-source-uuid', 'status');
const field  = await dataSources.getFieldById('data-source-uuid', 'field-uuid');

const options = await dataSources.getEnumsByFieldId('data-source-uuid', 'status-field-uuid');
const won     = await dataSources.getEnumById('data-source-uuid', 'status-field-uuid', 'enum-uuid');
```

| Method | Returns |
|--------|---------|
| `getFields(dataSourceId)` | `DataSourceField[]` — `[]` when the data source has none |
| `getFieldById(dataSourceId, fieldId)` | `DataSourceField \| undefined` |
| `getFieldBySlug(dataSourceId, fieldSlug)` | `DataSourceField \| undefined` |
| `getEnumsByFieldId(dataSourceId, fieldId)` | `DataSourceEnumOption[]` — `[]` when the field has none or isn't found |
| `getEnumById(dataSourceId, fieldId, enumId)` | `DataSourceEnumOption \| undefined` |

#### Relation helpers (parent / child selectors)

Docyrus models a relationship with a `relation` field on the **child** whose `relation_data_source_id` points at the **parent**. These helpers resolve those edges so you can build parent/child data-source and field pickers:

```ts
// Given a data source, what does it link to (parents) and what links to it (children)?
const parents  = await dataSources.listParentDataSources('contacts-ds-id'); // e.g. [Accounts]
const children = await dataSources.listChildDataSources('accounts-ds-id');  // e.g. [Contacts, Deals]

// Flattened fields for a field selector on the related side:
const parentFields = await dataSources.listParentDataSourceFields('contacts-ds-id');
const childFields  = await dataSources.listChildDataSourceFields('accounts-ds-id');
```

| Method | Returns |
|--------|---------|
| `listParentDataSources(dataSourceId)` | `DataSource[]` — the `relation` targets (parents), distinct, self-links excluded |
| `listChildDataSources(dataSourceId)` | `DataSource[]` — data sources whose `relation` field targets this one (children) |
| `listParentDataSourceFields(dataSourceId)` | `DataSourceField[]` — parents' fields, flattened (ids are unique, no dupes) |
| `listChildDataSourceFields(dataSourceId)` | `DataSourceField[]` — children's fields, flattened |

These are built from a **memoized relation index** over the cached data sources: with an inventory wired in, the index is computed once and reused until the cache changes — so calls are effectively O(1) lookups with no network. Without an inventory they fall back to one `GET .../data-sources?expand=fields` per call (rebuilding the index each time), so **pass an inventory** for the "fast" path. Requires field metadata in the cache (the inventory's default `dataSourcesExpand` includes `fields`). Need the fields grouped by their owning data source? Use `listParentDataSources` / `listChildDataSources` and read each result's `.fields`.

---

### `createInventoryClient(client, options?)`

A tenant-wide inventory cache of the data a Docyrus app needs at startup — installed apps, data sources (with their embedded views/forms/fields), tenant users, brands, tenant preferences, and per-app config — with **no app or data source scope** required. Useful for global pickers, sync caches, admin dashboards, post-sign-in warm-up, or any UI that needs to enumerate everything the tenant can see. Each resource is cached independently (same TTL, request de-dup) and shared across every inventory-backed client (`createDataSourceClient`, `createDataViewClient`, `createDataFormClient`, `createBrandClient`, `createAppConfigClient`, `createUserAppConfigClient`) when you pass **one** instance to all of them.

```ts
import { createInventoryClient } from '@docyrus/app-utils';

const inventory = createInventoryClient(apiClient);
```

#### Recommended usage: one shared inventory, warmed after sign-in

The inventory pays off when you create it **once** and pass that single instance
to every client, then warm it a single time right after sign-in. From then on
the whole app reads metadata from memory instead of the network.

**1. Create one instance at bootstrap and wire every client to it.** Put this in
a module the rest of the app imports (or a DI container / React context) so
there is exactly one cache:

```ts
// services/docyrus.ts
import {
  createInventoryClient,
  createDataSourceClient,
  createDataViewClient,
  createDataFormClient,
  createBrandClient,
  createAppConfigClient,
  createUserAppConfigClient
} from '@docyrus/app-utils';

const appId = import.meta.env.VITE_APP_ID;

// The one cache. Every client below shares it.
export const inventory = createInventoryClient(apiClient);

export const dataSources = createDataSourceClient(apiClient, { inventory });
export const brands = createBrandClient(apiClient, { inventory });
export const appConfig = createAppConfigClient(apiClient, appId, { inventory });
export const userAppConfig = createUserAppConfigClient(apiClient, appId, { inventory });

// View / form clients are per data source — construct them on demand, always
// passing the same `inventory`:
export const viewsFor = (appSlug: string, dsSlug: string) =>
  createDataViewClient(apiClient, appSlug, dsSlug, { inventory });
export const formsFor = (appSlug: string, dsSlug: string) =>
  createDataFormClient(apiClient, appSlug, dsSlug, { inventory });
```

**2. Warm the cache once after sign-in, behind a progress bar.** Call `load()`
as soon as you have an authenticated client and before routing to the home
page. With `VITE_APP_ID` set, this also warms the current app's config and
user-config (see [`load()`](#inventoryloadoptions)):

```ts
// Runs after @docyrus/signin reports an authenticated session.
async function bootstrapAfterSignIn(setProgress: (p: { label: string; value: number }) => void) {
  try {
    await inventory.load({
      onProgress: ({ message, ratio, phase }) => {
        setProgress({ label: message, value: Math.round(ratio * 100) });
        if (phase === 'complete') setProgress({ label: 'Ready', value: 100 });
      }
    });
  } catch (error) {
    // A step failed (network, or a missing scope — see below). Surface it and
    // let the user retry; the app can still run, reads just fall back to the API.
    reportBootstrapError(error);
  }

  navigateToHome();
}
```

**3. Read everywhere through the wired clients — every call is now a cache hit:**

```ts
const sources = await dataSources.list();                 // from cache
const statusField = await dataSources.getFieldBySlug(dsId, 'status');
const options = await dataSources.getEnumsByFieldId(dsId, statusFieldId);
const cfg = await appConfig.get();                        // from cache (warmed via VITE_APP_ID)
const tenantBrands = await brands.list();                 // from cache
```

**Handling limited scopes.** `listUsers()` needs the `Users.Read.All` scope; if
the signed-in user may lack it, drop that step so `load()` doesn't reject:

```ts
await inventory.load({
  include: ['apps', 'data-sources', 'brands', 'preferences'], // no 'users'
  onProgress
});
```

**Keeping the cache correct over a session.** Mutations made through the wired
clients patch the cache automatically. For external changes, drop the stale
slice; on **tenant switch** call `refresh()` (or rebuild the inventory) so the
new tenant's data replaces the old:

```ts
await inventory.refresh();          // after a tenant switch: drop all, re-fetch apps + data sources
inventory.invalidateBrands();       // e.g. brands changed elsewhere — next read re-fetches
inventory.invalidateAppConfig();    // drop every cached app config
```

> Because the cache is in-memory per instance, a full page reload starts cold —
> that is exactly why the post-sign-in `load()` exists. On a tenant switch that
> does **not** reload, `refresh()` is required to avoid serving the previous
> tenant's metadata.

#### One fetch, everything cached

`listDataSources` is the single source of truth. It calls `GET /v1/apps/data-sources?expand=views,forms,fields` once and caches every data source together with its embedded views, forms (with `actions`), and fields (with enum options). `listDataViews` and `listDataForms` are **derived projections** over that cache — they trigger no extra HTTP calls.

| Option              | Default                 | Description                                                                                            |
|---------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| `ttlMs`             | `300_000` (5 min)       | Cache time-to-live. Pass `Infinity` to keep entries until `refresh()` / `invalidate*()`.              |
| `dataSourcesExpand` | `'views,forms,fields'`  | Comma-separated `expand`. Override to include extras (e.g. `'views,forms,fields,acl'`) — keep `views`, `forms`, `fields` so derived views / form clients still work. |

The embedded form payload (camelCased, reduced) is **normalized on cache write** to the canonical `DataForm` shape — `isDefault` → `is_default`, missing `tenant_data_source_id` is backfilled from the parent data source, `archived` defaults to `false`. Embedded extras (`ownership`, `ownerProductId`, `customized`) are preserved as optional fields on `DataForm`.

Alongside data sources, the inventory caches five more resources, each fetched by a single request and cached independently:

| Method | Endpoint | Cached as | Notes |
|--------|----------|-----------|-------|
| `listUsers()` | `GET /v1/users` | `User[]` | Requires the `Users.Read.All` scope. |
| `listBrands()` | `GET /v1/tenant/brands` | `Brand[]` | Patchable via `upsertBrand` / `removeBrand`. |
| `getPreferences()` | `GET /v1/tenant/preferences` | `TenantPreferences` | |
| `getAppConfig(appId)` | `GET /v1/tenant/apps/{appId}/config` | `AppConfig \| null` per app id | `null` (a `404`) is cached too. |
| `getUserAppConfig(appId)` | `GET /v1/tenant/apps/{appId}/user-config` | `UserAppConfig \| null` per app id | `null` (a `404`) is cached too. |

```ts
const inventory = createInventoryClient(apiClient, {
  ttlMs: 10 * 60 * 1000,
  dataSourcesExpand: 'views,forms,fields,acl'
});

// Cache management
await inventory.refresh();             // drop every cache and re-fetch apps + data sources
inventory.invalidateApps();             // selectively drop apps
inventory.invalidateDataSources();      // selectively drop data sources (and the derived views/forms)
inventory.invalidateUsers();
inventory.invalidateBrands();
inventory.invalidatePreferences();
inventory.invalidateAppConfig(appId);       // one app, or omit appId to drop all app configs
inventory.invalidateUserAppConfig(appId);   // one app, or omit appId to drop all user configs

// Direct cache patches (used by the inventory-backed clients after mutations)
inventory.upsertDataView(view);         // insert or replace by id, routed via view.tenant_data_source_id
inventory.removeDataView(viewId);
inventory.upsertDataForm(form);
inventory.removeDataForm(formId);
inventory.upsertBrand(brand);           // insert or replace by id (no-op until listBrands ran)
inventory.removeBrand(brandId);
inventory.setAppConfig(appId, config);      // set (or clear with null)
inventory.setUserAppConfig(appId, config);
```

Filters passed to `listDataViews({ appId, dataSourceId })` / `listDataForms({ dataSourceId })` are applied client-side over the cached data sources — they never refetch.

#### `inventory.load(options?)`

Warms the inventory in one sequence and emits progress events per step. Designed to run once right after sign-in, behind a progress dialog, so that every later read (from the inventory itself and from any inventory-backed client) is a cache hit. By default it warms **apps, data sources, users, brands, and preferences**, plus the **current app's config + user-config** when the `VITE_APP_ID` env var is set (see `configAppIds` below). Resolves with an `InventoryLoadResult`; rejects — after emitting an `error` event — if a step fails.

```ts
const { apps, dataSources, users, brands, preferences, appConfigs, userAppConfigs } =
  await inventory.load({
    onProgress: ({ step, appId, message, current, total, ratio, phase }) => {
      // Drive a determinate progress bar. `phase` is 'start' | 'success' | 'error' | 'complete'.
      // Map `step` (see step ids below, or `null` on 'complete') to your own i18n if you
      // don't want the default English `message`. `appId` is set on config steps.
      setProgress({ label: message, value: Math.round(ratio * 100) });
    }
  });
// → navigate to home; all inventory-backed reads are now cache hits
```

Default event sequence (`total` is `5`): a `start`/`success` pair for each of `apps`, `data-sources`, `users`, `brands`, `preferences` (with `ratio` stepping `0 → 0.2 → … → 1`), then a terminal `{ step: null, phase: 'complete', ratio: 1 }`.

| Option | Default | Description |
|--------|---------|-------------|
| `onProgress` | — | Callback fired for each lifecycle event (`start`/`success` per step, `error` on failure, one terminal `complete`). |
| `force` | `true` | Invalidate the to-be-loaded caches first so the load always issues fresh network calls. Set to `false` to reuse still-fresh entries (steps then resolve instantly but events are still emitted). |
| `include` | `['apps', 'data-sources', 'users', 'brands', 'preferences']` | Which tenant-wide steps to run, in order. Narrow it when the user lacks a scope — e.g. drop `'users'` (needs `Users.Read.All`) — or when you only need a subset. |
| `configAppIds` | `VITE_APP_ID` (if set) else `[]` | App ids whose config **and** user-config to warm, appended as two extra steps per id (`step: 'app-config'` / `'user-app-config'`, each carrying `appId`). When omitted, defaults to the current app from `VITE_APP_ID` (read from `import.meta.env` or `process.env`); pass an explicit `[]` to skip config warm-up. Missing configs cache as `null` — a `404` is not an error. |

Step ids (`InventoryLoadStepId`): `'apps' | 'data-sources' | 'users' | 'brands' | 'preferences' | 'app-config' | 'user-app-config'`.

```ts
// Warm everything the current app's home page needs, including its config:
await inventory.load({ configAppIds: [currentAppId], onProgress });

// A viewer without the users scope:
await inventory.load({ include: ['apps', 'data-sources', 'brands', 'preferences'], onProgress });
```

#### `inventory.listApps()`

Lists every app the current tenant can access. Backed by `GET /v1/apps`. Cached separately from data sources.

```ts
const apps = await inventory.listApps();
// [{ id: 'uuid', name: 'CRM', slug: 'crm', logo_url: '...', status: 'active' }]
```

#### `inventory.listDataSources()`

Lists every data source the current tenant can access across all apps, with `views`, `forms`, and `fields` (with `enums`) embedded inline. Triggers exactly **one** network call per cache window.

```ts
const sources = await inventory.listDataSources();
// sources[0].views — DataView[]
// sources[0].forms — DataForm[]   (normalized from the embedded camelCased shape)
// sources[0].fields[i].enums — DataSourceEnumOption[] for select-type fields
```

#### `inventory.listUsers()` / `listBrands()` / `getPreferences()`

Tenant-wide singletons, each backed by one request and cached independently (same TTL + request de-dup as the data-source cache).

```ts
const users = await inventory.listUsers();        // GET /v1/users  (needs Users.Read.All)
const brands = await inventory.listBrands();       // GET /v1/tenant/brands
const preferences = await inventory.getPreferences(); // GET /v1/tenant/preferences
```

#### `inventory.getAppConfig(appId)` / `getUserAppConfig(appId)`

Per-app config, cached in a map keyed by app id. Resolve to `null` when no config exists — a `404` is treated as a valid empty state and the `null` is cached, so repeat calls don't re-hit the API.

```ts
const config = await inventory.getAppConfig(appId);          // AppConfig | null
const userConfig = await inventory.getUserAppConfig(appId);  // UserAppConfig | null
```

#### `inventory.listDataViews(params?)`

Flattens every cached data source's `views` array into a single list. No network call. `params.appId` matches against `tenant_app_id`; `params.dataSourceId` matches against the data source's `id`.

```ts
const allViews = await inventory.listDataViews();

const crmViews = await inventory.listDataViews({
  appId: 'fbeebd52-98ff-11ed-93e6-37fc9e45fc08'
});

const viewsForSpecificDataSources = await inventory.listDataViews({
  dataSourceId: ['019c48d0-506a-7ad1-9a4d-20acbc82f300', '010e9ce2-ca44-11ed-bda6-6b0c6905cf5e']
});
```

Both filters accept either a single id or an array.

#### `inventory.listDataForms(params?)`

Same flattening for forms.

```ts
const allForms = await inventory.listDataForms();
const formsForApp = await inventory.listDataForms({ appId: ['fbeebd52-...'] });
```

---

### `createDataSourceQueries(client)` — TanStack Query integration

Returns prebuilt [`queryOptions`](https://tanstack.com/query/latest/docs/framework/react/reference/queryOptions)-style factories and a query-key registry, so you get caching, deduplication, and reactive auto-invalidation for free. Works with `@tanstack/react-query`, `@tanstack/vue-query`, `@tanstack/solid-query`, etc. — only `@tanstack/query-core` is required at runtime.

Imported from a separate entrypoint to keep the optional `@tanstack/query-core` peer dep out of the main bundle:

```ts
import { createDataSourceQueries } from '@docyrus/app-utils/query';

const queries = createDataSourceQueries(apiClient);
```

#### Usage with React

```tsx
import { useQuery, useQueryClient } from '@tanstack/react-query';
import { createDataSourceQueries } from '@docyrus/app-utils/query';

const queries = createDataSourceQueries(apiClient);

function ContactsTable() {
  const { data: contacts } = useQuery(
    queries.getBySlugOptions('crm', 'contacts', { expand: 'fields,forms' })
  );

  const queryClient = useQueryClient();
  const handleSchemaChange = () => queries.invalidateBySlug(queryClient, 'crm', 'contacts');

  // ...
}
```

#### Available `*Options` factories

Each factory mirrors the corresponding `createDataSourceClient` method and produces a `{ queryKey, queryFn }` object you pass straight to `useQuery` (or `useSuspenseQuery`, `prefetchQuery`, etc.):

| Factory | Wraps |
|---------|-------|
| `listAppsOptions()` | `dataSources.listApps()` |
| `listOptions(params?)` | `dataSources.list(params)` |
| `listByAppSlugOptions(appSlug, params?)` | `dataSources.listByAppSlug(...)` |
| `listByAppIdOptions(appId, params?)` | `dataSources.listByAppId(...)` |
| `getByIdOptions(dataSourceId, params?)` | `dataSources.getById(...)` |
| `getBySlugOptions(appSlug, dataSourceSlug, params?)` | `dataSources.getBySlug(...)` |

Per-request options (`staleTime`, `enabled`, `select`, `placeholderData`, …) are passed at the call site:

```ts
useQuery({
  ...queries.listOptions({ expand: 'fields' }),
  staleTime: 5 * 60 * 1000,
  enabled: isAuthenticated
});
```

#### Invalidation helpers

All take a `QueryClient` so consumers can call them anywhere (event handlers, mutation `onSuccess`, etc.).

| Helper | Invalidates |
|--------|-------------|
| `invalidateAll(qc)` | Every cached data-source query (apps + lists + details) |
| `invalidateApps(qc)` | Cached app list |
| `invalidateLists(qc)` | All `list*` variants (across apps and params) |
| `invalidateDetails(qc)` | All `getById` / `getBySlug` results |
| `invalidateById(qc, dataSourceId)` | Every cached detail for a single data source ID (any `expand`) |
| `invalidateBySlug(qc, appSlug, dataSourceSlug)` | Every cached detail for a single app+slug pair (any `expand`) |
| `invalidateForAppSlug(qc, appSlug)` | All list queries scoped to an app slug |
| `invalidateForAppId(qc, appId)` | All list queries scoped to an app ID |

```ts
import { useMutation, useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

const updateSchema = useMutation({
  mutationFn: (changes) => api.patchSchema(changes),
  onSuccess: () => queries.invalidateBySlug(queryClient, 'crm', 'contacts')
});
```

#### `dataSourceKeys` registry

The same query keys used internally are exported as `dataSourceKeys` (and as `queries.keys`). Use them when you need to read or set cache entries directly:

```ts
import { dataSourceKeys } from '@docyrus/app-utils/query';

queryClient.setQueryData(
  dataSourceKeys.bySlug('crm', 'contacts', { expand: 'fields' }),
  optimisticData
);
```

The key shape is stable: every key starts with `['docyrus', 'data-sources', ...]`, so `queryClient.invalidateQueries({ queryKey: dataSourceKeys.all() })` clears everything this module owns.

---

## API Reference

### `getTenantPreferences(client)`

Fetches tenant preferences from the Docyrus API.

```ts
import { getTenantPreferences } from '@docyrus/app-utils';

const preferences = await getTenantPreferences(apiClient);
// { date_format: 'd.m.Y', decimal_separator: ',', thousand_separator: '.', ... }
```

**Parameters:**
- `client` — A configured `RestApiClient` from `@docyrus/api-client`

**Returns:** `Promise<TenantPreferences>`

---

### `createDateUtils(config)`

Creates date formatting utilities configured from tenant preferences.

```ts
import { createDateUtils } from '@docyrus/app-utils';

const dateUtils = createDateUtils({
  preferences,
  userTimezone: 'Europe/Istanbul' // defaults to 'UTC'
});
```

#### `dateUtils.formatDate(date, options?)`

Formats a date using the tenant's `date_format` (default `Y-m-d`).

```ts
dateUtils.formatDate('2024-06-15');                         // uses tenant format
dateUtils.formatDate(new Date(), { format: 'd/m/Y' });     // custom format
dateUtils.formatDate('2024-06-15', { timezone: 'US/Eastern' }); // custom timezone
```

#### `dateUtils.formatDateTime(date, options?)`

Formats a datetime using the tenant's `date_time_format` (default `Y-m-d H:i:s`). Handles timezone-aware parsing — appends `Z` to strings without a timezone suffix.

```ts
dateUtils.formatDateTime('2024-06-15T14:30:00');     // parses as UTC
dateUtils.formatDateTime('2024-06-15T14:30:00+03:00'); // respects offset
```

#### `dateUtils.formatDateLong(date, options?)`

Formats a date using the tenant's `long_date_format` (default `Y-m-d H:i:s`).

#### `dateUtils.toUserTimezone(date)`

Converts a date to the user's timezone without formatting.

```ts
const localDate = dateUtils.toUserTimezone('2024-06-15T12:00:00Z');
```

**Format strings** use PHP date format syntax (via `php-date-formatter`):
| Token | Output | Example |
|-------|--------|---------|
| `Y` | 4-digit year | `2024` |
| `m` | Month (01-12) | `06` |
| `d` | Day (01-31) | `15` |
| `H` | Hours 24h (00-23) | `14` |
| `i` | Minutes (00-59) | `30` |
| `s` | Seconds (00-59) | `00` |
| `D` | Short day name | `Sat` |
| `l` | Full day name | `Saturday` |
| `F` | Full month name | `June` |

---

### `createNumberUtils(config)`

Creates number formatting utilities configured from tenant preferences.

```ts
import { createNumberUtils } from '@docyrus/app-utils';

const numberUtils = createNumberUtils({ preferences });
```

#### `numberUtils.formatNumber(value, options?)`

Formats a number using the tenant's locale, separators, and precision.

```ts
numberUtils.formatNumber(1234567.89);
// With tenant settings: thousand_separator='.', decimal_separator=',', decimal_precision=2
// → '1.234.567,89'

// Override per call:
numberUtils.formatNumber(1234.5, {
  decimalPrecision: 3,
  decimalSeparator: '.',
  thousandSeparator: ','
});
// → '1,234.500'
```

**Formatting logic:**
- If both `thousandSeparator` and `decimalSeparator` are set → manual formatting with regex
- Otherwise → `toLocaleString()` with the tenant's `locale`
- Pass `thousandSeparator: ''` to disable grouping

---

### `formatNumberToWords(value, options?)`

Converts a number to words with currency labels. Standalone function (no config needed).

```ts
import { formatNumberToWords } from '@docyrus/app-utils';

formatNumberToWords(1234.56);
// → 'BİNİKİYÜZOTUZDÖRT TÜRK LİRASI ELLİALTI KURUŞ'

formatNumberToWords(1234.56, { lang: 'EN', currency: 'USD' });
// → 'ONE THOUSAND TWO HUNDRED THIRTY FOUR DOLAR FIFTY SIX CENT'

formatNumberToWords(1234.56, { lang: 'TR', currency: 'EUR', useSpaces: true });
// → 'BİN İKİ YÜZ OTUZ DÖRT EURO ELLİ ALTI CENT'
```

**Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `lang` | `'TR' \| 'EN'` | `'TR'` | Word language |
| `currency` | `'TRY' \| 'EUR' \| 'USD' \| 'GBP' \| 'JPY'` | `'TRY'` | Currency labels |
| `useSpaces` | `boolean` | `false` | Add spaces between word groups |

---

### Duration Formatters

Standalone functions for formatting durations (in seconds).

```ts
import {
  formatDurationAsTime,
  formatDurationAsHours,
  formatDurationAsWords
} from '@docyrus/app-utils';

formatDurationAsTime(5461);        // → '01:31'
formatDurationAsHours(5461);       // → '1.52'
formatDurationAsHours(5461, 1);    // → '1.5'
formatDurationAsWords(5461);       // → '1 hr 31 mins'
formatDurationAsWords(45);         // → '0 mins'
```

---

### `createTemplateEngine(config?)`

Creates a template engine combining Handlebars (with async support) and JSONata formula evaluation.

```ts
import { createTemplateEngine } from '@docyrus/app-utils';

const engine = createTemplateEngine({
  dateUtils,                         // from createDateUtils()
  numberUtils,                       // from createNumberUtils()
  user: currentUser,                 // injected as metadata.user in formulas
  extraJsonataBindings: { hasRole }  // additional JSONata bindings
});
```

Each `createTemplateEngine` call creates an **isolated Handlebars instance**, so multiple engines (e.g., different tenants) won't conflict.

#### `engine.compileTpl(templateString)`

Preprocesses and compiles a Handlebars template. Returns an async function.

```ts
const tpl = engine.compileTpl('Hello {{name}}, total: {{formatNumber amount}}');
const html = await tpl({ name: 'John', amount: 1234.5 });
// → 'Hello John, total: 1.234,50'
```

**Template preprocessing:**
1. `{{formula $expr}}` → wraps expression in quotes for the formula helper
2. `{{#if $expr}}` → converts to `{{#if (formula '$expr')}}` for JSONata evaluation
3. `{{<content}}` → strips HTML tags from inline content

#### `engine.compileFormula(expression, data?)`

Evaluates a JSONata expression with all helpers bound. Injects `metadata.user` from config.

```ts
const total = await engine.compileFormula('$sum(items.price)', { items });
const greeting = await engine.compileFormula('"Hello " & name', { name: 'World' });
```

#### `engine.jsonataHelpers`

Direct access to the full bindings object (all built-in helpers + `extraJsonataBindings`).

---

### Handlebars Helpers

The following helpers are registered automatically when using `compileTpl`:

#### Formatting Helpers

| Helper | Usage | Description |
|--------|-------|-------------|
| `formatDate` | `{{formatDate dateField format="d/m/Y"}}` | Format date (uses tenant `date_format` by default) |
| `formatDateTime` | `{{formatDateTime dateField}}` | Format datetime (uses tenant `date_time_format`) |
| `formatNumber` | `{{formatNumber amount decimalPrecision=2}}` | Format number with tenant settings |
| `formatNumberToWords` | `{{formatNumberToWords amount lang="TR" currency="TRY"}}` | Number to words |
| `formatDurationAsTime` | `{{formatDurationAsTime seconds}}` | Duration as `HH:MM` |
| `formatDurationAsWords` | `{{formatDurationAsWords seconds}}` | Duration as `X hrs Y mins` |
| `formatDurationAsHours` | `{{formatDurationAsHours seconds decimalPrecision=1}}` | Duration as decimal hours |

#### Logic & Data Helpers

| Helper | Usage | Description |
|--------|-------|-------------|
| `formula` | `{{formula "$sum(items.price)"}}` or `{{#formula}}$expr{{/formula}}` | Evaluate JSONata expression |
| `path` | `{{#path "$.items[0]"}}...{{/path}}` | JSONPath query — sets context to result |
| `repeat` | `{{#repeat 5}}...{{/repeat}}` | Repeat block N times |
| `sum` | `{{sum items "price"}}` or `{{sum 1 2 3}}` | Sum array field or values |
| `json` | `{{json data}}` | Pretty-print JSON |

---

### JSONata Helpers

All helpers are available inside `compileFormula` and `{{formula}}` blocks. They are also exported as `jsonataHelpers` for direct use.

```ts
import { jsonataHelpers } from '@docyrus/app-utils';
```

#### Date Functions (date-fns, null-safe wrapped)

All date functions accept strings or Date objects. Returns `null` for null/empty input.

**Formatting:**
`formatDate`, `formatDistance`, `formatDistanceStrict`, `formatDistanceToNow`, `formatDistanceToNowStrict`, `formatDuration`, `formatISO`, `formatRelative`

**Arithmetic:**
`addYears`, `addMonths`, `addWeeks`, `addDays`, `addHours`, `addMinutes`, `addSeconds`, `addMilliseconds`, `subYears`, `subMonths`, `subWeeks`, `subDays`, `subHours`, `subMinutes`, `subSeconds`, `subMilliseconds`

**Comparison:**
`isAfter`, `isBefore`, `isDatesEqual`, `isPast`

**Difference:**
`differenceInYears`, `differenceInMonths`, `differenceInWeeks`, `differenceInDays`, `differenceInHours`, `differenceInMinutes`, `differenceInSeconds`, `differenceInBusinessDays`, `differenceInBusinessDaysCustom`

**Period boundaries:**
`startOfDay`, `startOfMonth`, `startOfWeek`, `startOfToday`, `startOfQuarter`, `endOfDay`, `endOfMonth`, `endOfWeek`, `endOfToday`, `endOfQuarter`

#### `differenceInBusinessDaysCustom(startDate, endDate, options?)`

Calculates business days with configurable work hours and lunch breaks.

```jsonata
$differenceInBusinessDaysCustom(startDate, endDate, {
  "workStartHour": 9,
  "workEndHour": 18,
  "lunchBreakHours": 1,
  "lunchStartHour": 12.5,
  "lunchEndHour": 13.5,
  "includePartialDays": true
})
```

#### Number / Format Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `formatDecimal` | `(number, precision?, decSep?, thousandSep?)` | Format decimal with separators |
| `formatMoney` | `(number, precision?, decSep?, thousandSep?, currency?, position?)` | Format as currency |
| `percentage` | `(value, total, decimals)` | Calculate percentage |
| `truncate` | `(text, limit)` | Truncate with `...` |
| `join` | `(array, separator)` | Join array to string |
| `formatDurationInSeconds` | `(seconds)` | Human-readable duration via date-fns |

#### String Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `sha256` | `(message)` | SHA-256 hash (async) |
| `ascii` | `(str)` | Strip accents and non-ASCII characters |
| `slug` | `(str, separator?)` | URL-friendly slug |
| `extractNameFromEmail` | `(email, fallback?)` | Extract name from email (e.g. `john.doe@...` → `John Doe`) |
| `filterEmpty` | `(array)` | Remove null/undefined/empty from array |
| `padLeft` | `(value, length, char?)` | Left-pad string |
| `padRight` | `(value, length, char?)` | Right-pad string |
| `ifNull` | `(value, alternative)` | Null coalescing |
| `objectToString` | `(object)` | Convert object to `key:value` string |
| `startsWith` | `(haystack, needle)` | String starts with |
| `endsWith` | `(haystack, needle)` | String ends with |
| `includes` | `(haystack, needle)` | String includes (null-safe) |

#### Equality / DB Helpers

| Function | Signature | Description |
|----------|-----------|-------------|
| `isEqual` | `(a, b)` | Deep equality — compares by `id`, Date, JSON, or strict |
| `isEqualOrContained` | `(needle, haystack)` | Check if value is equal to or contained in array |
| `isNotEqualOrContained` | `(needle, haystack)` | Negation of `isEqualOrContained` |
| `getDbValue` | `(value)` | Extract `id` from object or first item of array |

## TypeScript

All types are exported:

```ts
import type {
  TenantPreferences,
  DateUtils,
  DateUtilsConfig,
  FormatOptions,
  NumberUtils,
  NumberUtilsConfig,
  NumberFormatOptions,
  NumberToWordsOptions,
  NumberWordLang,
  CurrencyCode,
  TemplateEngine,
  TemplateEngineConfig,
  AppConfig,
  UpsertAppConfigBody,
  AppConfigClient,
  UserAppConfig,
  UpsertUserAppConfigBody,
  UserAppConfigClient,
  DataView,
  CreateDataViewBody,
  UpdateDataViewBody,
  ListDataViewsParams,
  DataViewClient,
  DataForm,
  CreateDataFormBody,
  UpdateDataFormBody,
  DataFormClient,
  DataSourceExpand,
  DataSourceField,
  DataSource,
  DataSourceApp,
  DataSourceListParams,
  DataSourceClient,
  Brand,
  BrandColorScheme,
  BrandActiveVoicePreference,
  BrandDirectness,
  BrandEmojiPolicy,
  BrandFormalityLevel,
  BrandMetricEmphasis,
  BrandSlideDensity,
  BrandIllustrationStyle,
  CreateBrandBody,
  UpdateBrandBody,
  FetchBrandFromWebsiteResponse,
  BrandClient,
  MicrosoftIdentityPayload,
  GoogleIdentityPayload,
  UserIdentityClient,
  AppUtilsConfig
} from '@docyrus/app-utils';
```

## License

MIT
