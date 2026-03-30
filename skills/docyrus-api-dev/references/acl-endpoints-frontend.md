# ACL Endpoints for Frontend Developers

Base path: `/api/v1/users/acl`

All ACL endpoints require the normal authenticated API session.

These endpoints may be hidden from generated Swagger/OpenAPI output because the backend currently marks them with `@ApiExcludeEndpoint()`. Treat this document as the frontend integration source of truth and call these routes directly with `RestApiClient` or `useDocyrusClient()`.

## Important identifier rules

- Role assignments and ACL role relations are stored using `tenant_role.uid`.
- Returned nested role objects expose both `id` and `uid`, and both values map to the role UID.
- For role operations, backend can resolve incoming `roleId` values against both `tenant_role.uid` and `tenant_role.id`, but frontend apps should prefer role `uid` values from API responses.
- For user-role writes and role-query `roleIds`, send role UUIDs and prefer UID values.
- `tenant_role_query.query` is a JSON object matching the app's filter-query structure. Send raw JSON, not stringified JSON.

## Enum values

### Role ownership

- `APP`
- `CUSTOM`
- `PRODUCT`
- `SYSTEM`
- `USER`

### Role query restriction level

- `hidden`
- `read-only`
- `not-deletable`

## Endpoint groups

### 1) Record ACL endpoints

These manage record-level shares, not role CRUD.

| Method | Path | Purpose |
| :----- | :--- | :------ |
| `GET` | `/v1/users/acl?dataSourceId={uuid}&recordId={uuid}` | Fetch direct and effective ACL rows for a record |
| `POST` | `/v1/users/acl/share` | Upsert record share rows |
| `DELETE` | `/v1/users/acl/share` | Revoke matching share rows |
| `PUT` | `/v1/users/acl/owner` | Transfer record ownership |

#### Share payload notes

- `principalType` must be one of: `user`, `team`, `role`, `tenant`, `public`.
- `permissions` is the backend ACL bitmask value.
- `expiresAt` is optional and must be a valid ISO date if provided.

#### Record ACL response shapes

`GET /v1/users/acl` returns:

```json
{
  "direct": [
    {
      "id": "uuid",
      "principal_type": "user",
      "principal_id": "uuid",
      "permissions": 7,
      "expires_at": null,
      "created_by": "uuid-or-null",
      "created_on": "2026-03-29T20:10:00.000Z"
    }
  ],
  "effective": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "permissions": 7,
      "source_principal_type": "role",
      "source_principal_id": "uuid"
    }
  ]
}
```

### 2) Role endpoints

| Method | Path | Purpose |
| :----- | :--- | :------ |
| `GET` | `/v1/users/acl/roles` | List tenant roles |
| `GET` | `/v1/users/acl/roles/:roleId` | Get one role |
| `POST` | `/v1/users/acl/roles` | Create role |
| `PATCH` | `/v1/users/acl/roles/:roleId` | Partial update role |
| `DELETE` | `/v1/users/acl/roles/:roleId` | Hard delete role |

#### Role create/update rules

- `slug` is required on create and must be unique within the tenant.
- Create defaults:
  - `ownership = "CUSTOM"`
  - `privileges = ""`
  - `status = 1`
- Role responses include both `id` and `uid`, both pointing to the role UID.

#### Role delete side effects

Deleting a role also cleans up dependent ACL state:

- removes `tenant_user_role` assignments for that role
- sets `tenant_user.primary_role` to `null` where applicable
- removes linked `tenant_acl_rule` rows
- removes linked `tenant_acl_field_rule` rows
- removes the role from `tenant_role_query` role arrays
- hard deletes role-query rows that become empty after role removal

Frontend implication: after role deletion, refresh role lists, user-role lists, role-query lists, and any UI showing primary-role labels.

### 3) User-role endpoints

| Method | Path | Purpose |
| :----- | :--- | :------ |
| `GET` | `/v1/users/acl/user-roles` | List assignments across the tenant |
| `GET` | `/v1/users/acl/users/:userId/roles` | List assignments for one user |
| `POST` | `/v1/users/acl/users/:userId/roles` | Add roles to a user |
| `PUT` | `/v1/users/acl/users/:userId/roles` | Replace the full role set for a user |
| `DELETE` | `/v1/users/acl/users/:userId/roles/:roleId` | Remove one role assignment |

#### User-role behavior

- `GET /v1/users/acl/user-roles` accepts optional `userId` and `roleId` filters.
- If `roleId` is supplied, backend resolves it to the canonical role UID first.
- `POST /users/:userId/roles` is additive.
- `POST /users/:userId/roles` safely ignores duplicate assignments.
- `PUT /users/:userId/roles` is a full replacement operation.
- Sending `roleIds: []` to `PUT /users/:userId/roles` clears all additional roles for that user.

### 4) Role-query endpoints

Role queries are role-based filtering rules attached to roles and optionally scoped to a specific data source.

| Method | Path | Purpose |
| :----- | :--- | :------ |
| `GET` | `/v1/users/acl/role-queries` | List role queries |
| `GET` | `/v1/users/acl/role-queries/:roleQueryId` | Get one role query |
| `POST` | `/v1/users/acl/role-queries` | Create role query |
| `PATCH` | `/v1/users/acl/role-queries/:roleQueryId` | Partial update role query |
| `DELETE` | `/v1/users/acl/role-queries/:roleQueryId` | Hard delete role query |

#### Role-query rules

- `roleIds` is required on create and must contain at least one UUID.
- Stored role IDs are normalized to role UIDs.
- `query` is required on create and must be a JSON object.
- `filterChildRelations` defaults to `false`.
- `restrictionLevel` defaults to `hidden`.
- If `dataSourceId` is provided, backend derives `tenantAppId` automatically.
- If `dataSourceId` changes during update, backend recalculates `tenantAppId`.

## Common request examples

### Sync a user's complete role set

```json
{
  "roleIds": ["role-uid-uuid-1", "role-uid-uuid-2"]
}
```

### Create a role query

```json
{
  "name": "Hide archived records",
  "dataSourceId": "data-source-uuid",
  "roleIds": ["role-uid-uuid"],
  "query": {
    "condition": "and",
    "filters": []
  },
  "filterChildRelations": false,
  "restrictionLevel": "hidden"
}
```

### Share a record

```json
{
  "dataSourceId": "uuid",
  "recordId": "uuid",
  "items": [
    {
      "principalType": "user",
      "principalId": "uuid",
      "permissions": 7,
      "expiresAt": "2026-12-31T00:00:00.000Z"
    }
  ]
}
```

## Error expectations

Common backend error patterns:

- `400 Bad Request`
  - malformed UUID in path, query, or body
  - missing required DTO fields
  - invalid enum values
  - invalid boolean or date format in share payloads
- `404 Not Found`
  - role not found
  - role query not found
  - tenant user not found
  - tenant data source not found
  - one or more submitted role IDs could not be resolved
- `409 Conflict`
  - role slug already exists in the tenant
- `500 Internal Server Error`
  - unexpected persistence failure

## Frontend integration recommendations

- Use direct `RestApiClient` calls or `useDocyrusClient()` for ACL work; these routes may not be present in generated OpenAPI or collection layers.
- Prefer role `uid` values from API responses for future writes and filters.
- Treat `PUT /users/:userId/roles` as the canonical full-sync endpoint.
- Treat `POST /users/:userId/roles` as an additive convenience endpoint.
- Send role-query `query` values as raw JSON objects.
- Omit `tenantAppId` when sending a role query scoped by `dataSourceId`; backend derives it.
- After deleting a role, invalidate and refetch role lists, user-role lists, role-query lists, and any dependent role-label UI.

## Suggested TypeScript interfaces

```ts
export interface IAclRole {
  activitySummaryReportQueryId: string | null;
  createdBy: string | null;
  createdOn: string | null;
  databaseId: string | null;
  disableLogin: number | null;
  id: string;
  lastModifiedBy: string | null;
  lastModifiedOn: string | null;
  name: string;
  ownership: "APP" | "CUSTOM" | "PRODUCT" | "SYSTEM" | "USER";
  privileges: string;
  slug: string;
  status: number | null;
  tenantAppId: string | null;
  uid: string;
}

export interface IAclUserRoleAssignment {
  createdOn: string | null;
  id: string;
  role: {
    databaseId: string | null;
    id: string;
    name: string;
    slug: string;
    uid: string;
  };
  roleId: string;
  status: number | null;
  userId: string;
}

export interface IAclRoleQuery {
  createdBy: string | null;
  createdOn: string | null;
  dataSourceId: string | null;
  filterChildRelations: boolean;
  id: string;
  lastModifiedBy: string | null;
  lastModifiedOn: string | null;
  name: string | null;
  query: Record<string, unknown> | null;
  restrictionLevel: "hidden" | "read-only" | "not-deletable";
  roleIds: string[];
  tenantAppId: string | null;
}

export interface IAclRecordShare {
  id: string;
  principal_type: "user" | "team" | "role" | "tenant" | "public";
  principal_id: string;
  permissions: number;
  expires_at: string | null;
  created_by: string | null;
  created_on: string | null;
}

export interface IAclEffectiveUserAccess {
  id: string;
  user_id: string;
  permissions: number;
  source_principal_type: "user" | "team" | "role" | "tenant" | "public";
  source_principal_id: string;
}
```
