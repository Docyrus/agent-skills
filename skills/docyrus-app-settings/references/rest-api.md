# App Settings — REST API contract

Complete HTTP contract for the two app-settings surfaces. Use this when calling
the endpoints directly (a non-`@docyrus/app-utils` client, another language, a
script, or `docyrus curl`). The `@docyrus/app-utils` clients
(`createAppConfigClient` / `createUserAppConfigClient`) wrap all of this.

## Contents

- [Common conventions](#common-conventions)
- [Response envelope & the nested `data`](#response-envelope--the-nested-data)
- [App config — shared, one per app](#app-config--shared-one-per-app)
- [User config — private, one per user per app](#user-config--private-one-per-user-per-app)
- [Errors](#errors)
- [`docyrus curl` examples](#docyrus-curl-examples)

## Common conventions

- **Base:** paths are versioned as `/v1/...`. `:appId` is the app's id (UUID).
- **Auth:** every endpoint requires an **OAuth2 bearer access token**
  (`Authorization: Bearer <token>`) with the scopes listed per endpoint.
- **Content type:** send `Content-Type: application/json` on `PUT`.
- **Body (write):** both `PUT` endpoints accept the same body — every field optional:

  | Field | Type | Notes |
  |---|---|---|
  | `data` | JSON **object** | Your settings. Must be an object (not array/scalar). **Replaces** the stored object wholesale — omitted keys are dropped. |
  | `status` | number | Small integer flag you define. Defaults to `1` when the document is first created; on update, unchanged unless sent. |

- **Upsert semantics:** `PUT` creates the document if it does not exist, else
  updates it. There is exactly one document per app (per user, for user-config).

## Response envelope & the nested `data`

Success responses are wrapped: `{ "success": true, "data": <document> }`. The
**document itself** carries a `data` field holding your settings — so your values
are at **`response.data.data`**:

```jsonc
{
  "success": true,
  "data": {                     // the document
    "id": "0192f3c1-...",
    "data": {                   // <-- YOUR settings live here
      "theme": "dark",
      "pinnedFilters": ["open", "mine"]
    },
    "status": 1,
    "tenant_app_id": "0192abcd-...",
    "created_by": "0192user-...",
    "created_on": "2026-01-04T10:11:12.000Z",
    "last_modified_by": "0192user-...",
    "last_modified_on": "2026-01-05T09:00:00.000Z"
  }
}
```

**Empty read:** when no document exists yet, `GET` still returns `200` with a
blank document — `id: ""`, `data: {}`, `status: 0`. It does **not** `404`. Detect
"not configured" by an empty `id`.

## App config — shared, one per app

One document shared by every user in the tenant. Base path `/v1/tenant/apps/:appId/config`.

| Method | Path | Purpose | Required scope (any of) | Success |
|---|---|---|---|---|
| `GET` | `/v1/tenant/apps/:appId/config` | Read the app config | `Tenant.Read`, `Tenant.ReadWrite` | `200` `{ data: <document> }` (blank doc if none) |
| `PUT` | `/v1/tenant/apps/:appId/config` | Create or update it | `Tenant.ReadWrite` | `200` `{ data: <document> }` |
| `DELETE` | `/v1/tenant/apps/:appId/config` | Delete it | `Tenant.ReadWrite` | `200` `{ data: { success: true } }` |

Writing app config effectively requires a **tenant-admin-level token** (the
`Tenant.ReadWrite` scope) — one write changes the value for all users.

## User config — private, one per user per app

One document per **signed-in user** per app. It always targets the caller — there
is no user selector, and no way to read or write another user's document. Base
path `/v1/tenant/apps/:appId/user-config`.

| Method | Path | Purpose | Required scope (any of) | Success |
|---|---|---|---|---|
| `GET` | `/v1/tenant/apps/:appId/user-config` | Read the caller's config | `Users.Read.All`, `Users.ReadWrite.All` | `200` `{ data: <document> }` (blank doc if none) |
| `PUT` | `/v1/tenant/apps/:appId/user-config` | Create or update the caller's config | `Users.ReadWrite.All` | `200` `{ data: <document> }` |
| `DELETE` | `/v1/tenant/apps/:appId/user-config` | Delete the caller's config | `Users.ReadWrite.All` | `200` `{ data: { success: true } }` |

The document is identical to app config plus a `user_id` field (always the
caller's id).

## Errors

Errors use `{ "success": false, ... }` with the matching HTTP status:

| Status | When |
|---|---|
| `400` | `data` is not a JSON object, or the body is otherwise invalid |
| `401` | Missing / invalid / expired bearer token |
| `403` | Token lacks the required scope (e.g. writing app config without `Tenant.ReadWrite`) |
| `404` | `DELETE` when no document exists (unlike `GET`, delete does not treat "missing" as success) |

`GET` never `404`s for a missing document — it returns a blank one (`id: ""`).

## `docyrus curl` examples

`docyrus curl` sends authenticated, path-only requests through the active
session. There is no dedicated app-settings subcommand — this is the CLI path.

```bash
# find the app id from its slug
docyrus apps list --format json

# --- app config (shared) ---
docyrus curl /v1/tenant/apps/<appId>/config --format json
docyrus curl -X PUT /v1/tenant/apps/<appId>/config \
  -H 'Content-Type: application/json' \
  -d '{"data":{"onboardingFlow":"v2","maxUploadMb":25},"status":1}'
docyrus curl -X DELETE /v1/tenant/apps/<appId>/config

# --- user config (private to you) ---
docyrus curl /v1/tenant/apps/<appId>/user-config --format json
docyrus curl -X PUT /v1/tenant/apps/<appId>/user-config \
  -H 'Content-Type: application/json' \
  -d '{"data":{"theme":"dark","sidebarCollapsed":true}}'
docyrus curl -X DELETE /v1/tenant/apps/<appId>/user-config
```

Remember: to change one key without dropping the rest, read the current
document first and send the merged `data` back — `PUT` replaces `data` wholesale.
