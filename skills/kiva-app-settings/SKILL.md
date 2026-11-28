---
name: kiva-app-settings
description: >-
  Design and use KIVA app-scoped and app-user-scoped settings -
  developer-defined, database-backed configuration documents attached to an app
  - through the `@kivacore/app-utils` config clients, the REST API, or `kiva
  curl`. Use when a frontend/app developer wants to persist and read settings
  for a KIVA app: an app-wide config shared by every user in the tenant
  (AppConfig - one document per app) or a private per-user config
  (UserAppConfig - one document per user per app). Examples: app feature flags
  / defaults an admin configures, default grid filters, onboarding/setup state,
  per-user theme, layout, sidebar collapsed state, dismissed hints. Covers
  choosing the scope, the document shape, reading/upserting/deleting with
  `createAppConfigClient` / `createUserAppConfigClient`, the REST contract
  (`GET`/`PUT`/`DELETE` `/v1/tenant/apps/:appId/config` and `.../user-config`),
  required OAuth2 scopes, and the CLI. Triggers on "app settings", "app
  config", "user config", "user preferences", "per-app settings", "per-user
  settings", "AppConfig", "UserAppConfig", "createAppConfigClient",
  "createUserAppConfigClient", "save app configuration", "persist user
  preferences for an app". For tenant-wide regional/formatting preferences see
  kiva-account-settings; for building the app UI see kiva-app-dev-react.
---

# KIVA App Settings

Two database-backed **settings documents** you can attach to a KIVA app. The
shape of what you store is **entirely yours** — each document holds a free-form
JSON object (`data`) that you design, plus a small numeric `status` flag. KIVA
just persists and returns it per app (and, for the user variant, per user).

| Surface | Scope | Cardinality | Who sees it | Use for |
|---|---|---|---|---|
| **App config** (`AppConfig`) | The whole app | **one** document per app | every user in the tenant | app-wide settings an admin/builder configures: feature flags, defaults, shared filters, integration settings |
| **User config** (`UserAppConfig`) | The app **and** the signed-in user | **one** document per user per app | only that user | per-user preferences: theme, layout, sidebar state, onboarding/dismissed hints, saved personal defaults |

Both are **upsert-only single documents** — there is no list, no query, and no
extra rows to manage. You read the one document, change it, write it back.

## Choosing the scope

- Shared across everyone, one source of truth for the app → **app config**.
  Reading is broadly allowed; **writing needs a tenant-admin-level token** (see
  scopes below), because one user's write changes it for all.
- Personal to the current user, different per person → **user config**. Any
  signed-in user can read and write **their own**; there is no way to read or
  write another user's document through the API.

If a setting is really a tenant-wide regional/formatting preference (language,
date/number/currency formats, working hours), that is **not** this — use
`kiva-account-settings` (`tenant-preferences`) instead.

## Document shape

Both documents return the same envelope of fields (user config adds `user_id`):

```ts
interface AppConfig {
  id: string;                        // "" when no document exists yet (see gotcha)
  data: Record<string, unknown>;     // YOUR settings object — design it freely
  status: number;                    // small integer flag you define; defaults to 1 on first write
  tenant_app_id: string;             // the app this belongs to
  created_by: string | null;
  created_on: string;                // ISO timestamp
  last_modified_by: string | null;
  last_modified_on: string | null;
}

interface UserAppConfig extends AppConfig {
  user_id: string;                   // always the signed-in user
}
```

Rules that hold for both:

- **`data` must be a JSON object** (`{...}`) — not an array, string, or number.
- **`data` is replaced wholesale on write**, not deep-merged. To change one key,
  read the current `data`, spread it, and write the merged object back.
- **`status`** is a free small integer (e.g. `1` = active, `0` = disabled). It
  defaults to `1` when the document is first created; on later writes it only
  changes if you send it.

## Primary path: `@kivacore/app-utils` clients

In a KIVA React/TypeScript app, use the config clients from
`@kivacore/app-utils`. Each is built from a `RestApiClient` (from
`@kivacore/api-client`) plus the app's id, and exposes `get` / `upsert` /
`remove`. They unwrap the API envelope for you and return the typed document.

```ts
import { createAppConfigClient, createUserAppConfigClient } from '@kivacore/app-utils'

const appConfig = createAppConfigClient(client, appId)   // shared, app-wide
const userConfig = createUserAppConfigClient(client, appId) // private, per-user
```

Client surface (identical for both):

```ts
appConfig.get(): Promise<AppConfig>                 // never throws on "empty" — returns id: ""
appConfig.upsert(body: { data?; status? }): Promise<AppConfig>  // create or update
appConfig.remove(): Promise<void>                   // deletes; throws 404 if nothing exists
```

### Design your own settings and use them

```ts
// 1. Design the settings you want to persist.
interface MyAppSettings {
  defaultView: 'board' | 'table'
  notificationsEnabled: boolean
  pinnedFilters: string[]
}

// 2. Read (with a typed cast over the free-form `data`).
const doc = await userConfig.get()
const settings = doc.data as Partial<MyAppSettings>
const view = settings.defaultView ?? 'table'   // apply your own defaults

// 3. Change ONE key — read-merge-write, because `data` is replaced wholesale.
await userConfig.upsert({ data: { ...doc.data, defaultView: 'board' } })

// 4. Reset to defaults.
await userConfig.remove()
```

App config is the same, just a different client — write app-wide defaults an
admin sets once and every user reads:

```ts
await appConfig.upsert({ data: { onboardingFlow: 'v2', maxUploadMb: 25 } })
```

### The "empty document" gotcha

`get()` **always resolves** — when no document exists yet it returns a blank one
with **`id: ""`**, `data: {}`, `status: 0`. It does **not** throw or return
`null`. Detect "not configured yet" by checking the id:

```ts
const doc = await appConfig.get()
if (!doc.id) {
  // no config saved yet — show setup UI / seed defaults
}
```

### Optional: shared caching via the inventory client

For an app that reads these documents in many places, pass a shared
`InventoryClient` so reads are cached and writes patch the cache in place
(no refetch):

```ts
const inventory = createInventoryClient({ client })
const appConfig = createAppConfigClient(client, appId, { inventory })
const userConfig = createUserAppConfigClient(client, appId, { inventory })
```

`upsert`/`remove` keep the inventory consistent automatically. Wire these
clients into your data layer (e.g. a TanStack Query `queryFn`) the same way as
the other `@kivacore/app-utils` runtime helpers — see **kiva-app-dev-react**
for the full app-runtime wiring pattern and how to obtain `client`/`appId`.

## Alternative path: raw REST / CLI

When you are not in a `@kivacore/app-utils` app (a different language, a script,
a quick check), call the endpoints directly. There is **no dedicated `kiva`
subcommand** for these — use `kiva curl` from the terminal:

```bash
# resolve your app's id if you only know the slug
kiva apps list --format json

# read / write / delete the app-wide config
kiva curl /v1/tenant/apps/<appId>/config --format json
kiva curl -X PUT /v1/tenant/apps/<appId>/config -d '{"data":{"maxUploadMb":25}}'
kiva curl -X DELETE /v1/tenant/apps/<appId>/config

# the current user's private config
kiva curl /v1/tenant/apps/<appId>/user-config --format json
kiva curl -X PUT /v1/tenant/apps/<appId>/user-config -d '{"data":{"theme":"dark"}}'
```

The response is an envelope whose `data` is the document — and the document
carries its **own** `data` key with your settings, so your values live at
**`response.data.data`**. Full HTTP contract (all six endpoints, request/response
bodies, status codes, OAuth2 scopes, error shapes) is in
[references/rest-api.md](references/rest-api.md).

## Key rules

- **Two scopes, one document each.** `config` = shared app-wide; `user-config` =
  private to the caller. Pick by "should everyone see the same value?".
- **`data` is your design** — any JSON **object**. Version it yourself (e.g. a
  `schemaVersion` key) if it will evolve.
- **Writes replace `data` wholesale.** Change one field with read-merge-write
  (`{ ...doc.data, key: value }`); a bare `upsert({ data: {...} })` drops any
  keys you omit.
- **`get()` never fails on empty** — check `id === ""` (`!doc.id`) to detect an
  unconfigured app; `remove()` **does** throw `404` when there is nothing to
  delete.
- **User config is self-only.** It always targets the signed-in user; you cannot
  address another user's document.
- **App-config writes need an admin-scoped token; user-config writes only need
  the user's own token.** See scopes in [references/rest-api.md](references/rest-api.md).
- **Settings values are at `response.data.data`** over raw REST (envelope `data`
  → document → its own `data`). The `@kivacore/app-utils` clients hide the outer
  envelope, so there you read `doc.data`.

## Related skills

- **kiva-app-dev-react** — building the app UI; how to obtain `client` and
  `appId` and wire the app-utils runtime (dates, numbers, data views, these config clients).
- **kiva-api-dev** — constructing a `RestApiClient` and OAuth2 auth for non-app callers.
- **kiva-account-settings** — tenant-wide regional & formatting preferences and the user's own profile (a different surface).
- **kiva-cli-app** — full `kiva` CLI reference (`curl`, `apps`, auth, environments).
