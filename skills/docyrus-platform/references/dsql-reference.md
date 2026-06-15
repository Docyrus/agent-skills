# DSQL Reference — Capabilities, Limitations & Pseudo-Functions

DSQL (Docyrus Structured Query Language) is a **read-only, PostgreSQL-compatible SQL dialect** that runs over *logical* data-source tables instead of physical tables. Callers write `SELECT` queries against logical names like `base.contact`, and the engine resolves them to the correct tenant schema, applies row-level access control, and enforces a strict allow-list of statements and functions.

This reference documents exactly what DSQL supports and rejects. It is derived from the engine implementation (`LogicalSqlQueryRunner`, `DsqlSchema`, `DsqlAccessPlanner`), not from generic Postgres behavior — where the two differ, this file is authoritative. Use it when generating DSQL (e.g. for the *DSQL Generator* agent), consuming the DSQL REST endpoints, or explaining the capability.

> **Surfaces that run DSQL:**
> - **REST API**: `PUT /v1/dsql/query` with `{ "query": "select …" }`; schema discovery via `GET /v1/dsql/schema/apps/:appSlug`, `GET /v1/dsql/schema/apps/:appSlug/data-sources/:dataSourceSlug`, and `GET /v1/dsql/schema/data-sources?ids=…`.
> - **CLI**: `docyrus dsql query "…"`, `docyrus dsql schema app|data-source|data-sources …`.
> - **Agent tools**: `queryDatabase` + the schema-discovery tools used by the DSQL Generator agent.
> - **Natural language**: `docyrus dsql generate "<question>"` returns a generated DSQL query, and `docyrus dsql ask "<question>"` generates and runs it — both delegate to the base DSQL generator agent, which follows every rule in this document.
>
> Always discover the schema **first** — never guess table or field slugs.

---

## 1. Logical table naming

Tables are addressed as **`appSlug.dataSourceSlug`** (schema-qualified logical names). Unqualified table names are rejected unless they are a CTE alias.

```sql
select * from base.task t
select * from crm.contact c
select * from custom.my_source s
```

| Logical namespace | What it holds |
|---|---|
| `<appSlug>.<dataSourceSlug>` | Business data sources for an app (e.g. `base.task`, `crm.lead`). Subject to access control. |
| `tenant.user` | The logical users table. Join target for owner/assignee names and emails. Its primary key is exposed as `id` (the physical `user_id` is hidden). |
| `tenant.enum` | The logical enum/option table. Join target to resolve enum labels, colors, icons, and metadata. |

Physical resolution (informational — you never write these): global datasets map to the `dataset` schema, `SYSTEM` data sources to `public`, everything else to the tenant's private schema.

**External data sources cannot be queried** — any source backed by an external SQL provider is rejected with `External data source '…' is not supported by the SQL query endpoint`.

### Child / inherited sources

Some data sources inherit fields from a parent ("base") source. You query the **child** logical source directly; when you reference an inherited (`base`) field, the engine auto-injects a `LEFT JOIN` to the base table on `base.id = child.id`. You do not write that join yourself.

- A child whose base is a **simple**-type source is not queryable (`Simple base data source '…' is not supported`).

---

## 2. Field referencing

Reference fields by **logical field slug**, in one of three forms (resolved in this order):

| Form | Example | Notes |
|---|---|---|
| Unqualified | `email` | Resolved across all in-scope sources; **errors if ambiguous** across joined tables. |
| Alias-qualified | `c.email` | Preferred whenever more than one source is in scope. |
| Fully-qualified | `crm.contact.email` | Explicit app + source + field. |

- **`id`** is always available on every logical source (a synthetic UUID is provided if the source has no natural id). For `tenant.user`, `id` is the user's id.
- Simple-source custom fields are stored internally in a JSONB `data` column but are exposed as **regular columns by slug** — query `s.amount`, not `s.data->'amount'`.
- **Virtual / computed field types are NOT queryable** and are omitted from the schema: `field-list`, `field-display`, `field-formula`, `field-taskList`, `field-button`. Referencing one throws `Field '…' is not supported by the SQL query endpoint`.
- Internal columns are hidden and not selectable: `tenant_id`, `tenant_data_source_id`, `cursor_date`, `sort_order`, `style`, `docyment`, `parent_data_source_id`, `parent_record_id`, `editor_view_id`, `followers`, `mentions` (and the raw `data` column on simple sources).

### Schema discovery output

The `schema` endpoints/tools return, per data source: `id`, `appSlug`, `slug`, `name`, `title`, and a token-efficient `schema` — a `create table appSlug.dataSourceSlug (…)` DDL string. Each column line carries the field slug, its database type, and reference hints (`references tenant.user` / `references tenant.enum` / `references appSlug.dataSourceSlug`), an inline agent-description comment when set, and `id=name` pairs for enum fields. Read these hints to build correct joins.

---

## 3. Supported SQL surface

DSQL accepts **exactly one read-only `SELECT` statement**. Within that statement you may use:

- `SELECT` (including `DISTINCT`), `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`, `OFFSET`
- **Joins**: `INNER`, `LEFT`, `RIGHT`, `FULL`, `CROSS`, plus implicit joins via `WHERE`
- **CTEs** (`WITH …`) — read-only only
- **Subqueries** in `SELECT`, `WHERE`, and `FROM` (a `FROM` subquery **must** be aliased)
- **Set operations**: `UNION` / `UNION ALL` / `INTERSECT` / `EXCEPT` (each side is itself a validated `SELECT`)
- **Aggregates** and `GROUP BY` / `HAVING`
- `ORDER BY` may reference a `SELECT`-list alias
- Type casts with `::type` (see §6)

`SELECT *` is allowed but must be unambiguous: with more than one source in `FROM`, qualify it (`t.*`), and `*` may only appear in the `SELECT` list.

---

## 4. Hard restrictions

Rejected at parse/compile time (before any execution):

- **Only `SELECT`.** Every non-select statement type is blocked: `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `CALL`, `DO`, all `CREATE*`/`ALTER*`/`DROP*`, `TRUNCATE`, `COPY`, `GRANT`, `INDEX`, `VACUUM`, `SET`/`SHOW`, `NOTIFY`/`LISTEN`, `BEGIN`/`COMMIT` (transactions), and `EXPLAIN`.
- **Exactly one statement** — no multiple semicolon-separated statements.
- **No `SELECT INTO`.**
- **No data-modifying CTEs** (a `WITH` branch that is anything other than a `SELECT`).
- **No functions/`LATERAL` in `FROM`** (`Functions in FROM are not supported`). Set-returning functions as a table source are not allowed.
- **No unqualified physical/table names** — only `appSlug.dataSourceSlug` logical names (or CTE aliases).
- **No `pg_*` functions** and no access to physical tenant schemas or `public` tables directly.
- **Read-only execution**: the query runs inside a `readOnly` transaction with a **15-second statement timeout**.

---

## 5. Supported functions (allow-list)

Only the functions below may be called. Names are matched case-insensitively. Anything else throws `Function '…' is not supported`.

**String**
`length`, `lower`, `upper`, `substr`, `replace`, `concat`, `trim`, `ltrim`, `rtrim`, `btrim`, `split_part`, `initcap`, `reverse`, `strpos`, `lpad`, `rpad`

**Number**
`abs`, `ceil`, `floor`, `round`, `sqrt`, `power`, `mod`, `gcd`, `lcm`, `exp`, `ln`, `log`, `log10`, `log1p`, `pi`, `sign`, `width_bucket`, `trunc`, `greatest`, `least`

**Date / time**
`now`, `age`, `clock_timestamp`, `date_part`, `date_trunc`, `extract`, `isfinite`, `justify_days`, `justify_hours`, `make_date`, `make_time`, `make_timestamp`, `make_timestamptz`, `timeofday`, `to_timestamp`, `to_char`, `to_date`, `to_time`

**JSON / JSONB**
`jsonb_array_length`, `jsonb_extract_path`, `jsonb_extract_path_text`, `jsonb_object_keys`, `jsonb_build_object`, `json_build_object`, `array_to_json`, `row_to_json`

**Utility**
`coalesce`

**Aggregates** (callable as aggregates only)
`count`, `sum`, `avg`, `min`, `max`, `array_agg`, `json_agg`, `jsonb_agg`

**Allowed bare literals**: `current_date`, `current_time`, `current_timestamp`.

### Not supported (common surprises)

- **Window functions**: dedicated window functions (`row_number`, `rank`, `dense_rank`, `lead`, `lag`, `ntile`, `first_value`, …) are **not** on the allow-list and are rejected. Do not rely on `OVER (…)` windowing — it is outside the supported surface.
- **Explicitly blocked**: `current_setting`, `set_config`, `version`, `pg_sleep`, `pg_read_file`, `pg_read_binary_file`, `inet_server_addr`, `inet_server_port`, and **everything prefixed `pg_`**.
- Anything not in the lists above (e.g. `string_agg`, `regexp_replace`, `position`, `left`/`right`, `nullif`, `case`-as-function, `generate_series`) — if you need it, restructure the query or post-process the result.
- Schema-qualified function calls are only allowed as `pg_catalog.<name>` (and still must resolve to an allow-listed name).

---

## 6. Type casts

`::type` casts are restricted to this set (scalar and `[]` array variants):

`int`, `int2`, `int4`, `int8`, `bigint`, `real`, `float`, `float4`, `float8`, `numeric`, `decimal`, `double`, `money`, `timestamp`, `timestamptz`, `date`, `time`, `interval`, `bool`, `boolean`, `uuid`, `text`

---

## 7. `tenant.*` pseudo-functions

These are **DSQL-only** helpers (not real Postgres functions) that inject the current user's identity and organizational scope into your query. They are translated to SQL at compile time. Use them in `WHERE`/`SELECT` to write permission- and org-aware queries. Each takes 0 or 1–2 UUID arguments as shown; an argument-count mismatch throws `tenant.X expects N argument(s)`.

### Identity & context (no args)

| Function | Returns |
|---|---|
| `tenant.current_user_id()` | Current user's UUID |
| `tenant.active_user_id()` | Alias of `current_user_id()` |
| `tenant.current_tenant_id()` | Current tenant's UUID |
| `tenant.current_role_id()` | Current user's primary role UUID (or NULL) |
| `tenant.current_role_ids()` | UUID[] of all the current user's role ids |
| `tenant.current_team_ids()` | UUID[] of the current user's team ids |
| `tenant.current_hierarchy_unit_id()` | Current user's org-unit UUID (or NULL) |
| `tenant.current_scope_user_ids()` | UUID[] of users within the current user's visibility scope |

### User matching (1 arg → boolean)

| Function | True when |
|---|---|
| `tenant.is_current_user(userId)` / `tenant.active_user(userId)` | `userId` is the current user |
| `tenant.is_not_current_user(userId)` / `tenant.not_active_user(userId)` | `userId` is not the current user (or is NULL) |

### Scope membership (1 arg → boolean)

| Function | True when |
|---|---|
| `tenant.in_current_user_scope(userId)` / `tenant.user_in_current_scope(userId)` / `tenant.user_in_current_hierarchy_scope(userId)` | `userId` is inside the current user's scope |
| `tenant.not_in_current_user_scope(userId)` / `tenant.user_not_in_current_scope(userId)` / `tenant.user_not_in_current_hierarchy_scope(userId)` | negation of the above |

### Role membership

| Function | Args | True when |
|---|---|---|
| `tenant.in_role(userId, roleId)` / `tenant.user_in_role(userId, roleId)` | 2 | `userId` has role `roleId` |
| `tenant.not_in_role(userId, roleId)` / `tenant.user_not_in_role(userId, roleId)` | 2 | negation |
| `tenant.in_current_roles(userId)` / `tenant.user_in_current_roles(userId)` | 1 | `userId` shares a role with the current user |
| `tenant.not_in_current_roles(userId)` / `tenant.user_not_in_current_roles(userId)` | 1 | negation |
| `tenant.current_user_in_role(roleId)` | 1 | the current user has role `roleId` |

### Team membership

| Function | Args | True when |
|---|---|---|
| `tenant.in_team(userId, teamId)` / `tenant.user_in_team(userId, teamId)` | 2 | `userId` is in team `teamId` |
| `tenant.not_in_team(userId, teamId)` / `tenant.user_not_in_team(userId, teamId)` | 2 | negation |
| `tenant.in_current_user_team(userId)` / `tenant.in_active_user_team(userId)` / `tenant.user_in_current_teams(userId)` | 1 | `userId` shares a team with the current user |
| `tenant.not_in_current_user_team(userId)` / `tenant.not_in_active_user_team(userId)` / `tenant.user_not_in_current_teams(userId)` | 1 | negation |
| `tenant.current_user_in_team(teamId)` | 1 | the current user is in team `teamId` |

### Hierarchy-unit membership

| Function | Args | True when |
|---|---|---|
| `tenant.in_unit(userId, unitId)` / `tenant.user_in_unit(userId, unitId)` | 2 | `userId` is in unit `unitId` (exact unit) |
| `tenant.not_in_unit(userId, unitId)` / `tenant.user_not_in_unit(userId, unitId)` | 2 | negation |
| `tenant.in_sub_unit(userId, unitId)` / `tenant.user_in_sub_unit(userId, unitId)` / `tenant.user_in_sub_units(userId, unitId)` | 2 | `userId` is in `unitId` or any descendant unit |
| `tenant.not_in_sub_unit(userId, unitId)` / `tenant.user_not_in_sub_unit(userId, unitId)` / `tenant.user_not_in_sub_units(userId, unitId)` | 2 | negation |
| `tenant.current_user_in_unit(unitId)` | 1 | the current user is in unit `unitId` |

---

## 8. Limits & execution

| Limit | Value |
|---|---|
| Max query length | **100,000 characters** |
| Default `LIMIT` (when none given) | **100 rows** |
| Max `LIMIT` — interactive/user context | **1,000 rows** |
| Max `LIMIT` — API client (app token) context | **100 rows** |
| Statement timeout | **15 seconds**, read-only transaction |
| `PUT /v1/dsql/query` throttle | **60 requests / minute** |

- If you omit `LIMIT`, the engine wraps the query and applies the default. If you request more than the cap, it is silently clamped to the cap. `LIMIT ALL` or a non-literal limit falls back to the default. Negative `LIMIT` is rejected.
- For totals/aggregates, do the aggregation in SQL (`count`, `sum`, …) rather than fetching rows and counting client-side.
- `PUT /v1/dsql/query` returns `{ data: [...rows], meta: { count } }`.

---

## 9. Access control & scopes

DSQL enforces the same row-level security as the rest of the platform — a caller only ever sees rows they are permitted to see.

- **Business sources** (any app slug other than `core`/`tenant`) require a view ACL for the current user's roles; otherwise the query fails with **403 `Access denied to logical data source '…'`**. Role-restriction queries (hidden-level restrictions) and child-relation filters are injected automatically.
- **`PRIVATE` data-access** sources require a `record_owner` field and limit visibility to the owner and their hierarchy subordinates (gated by the source's unit-peer-access level).
- **`core`/`tenant` sources** skip source ACL but still apply tenant scoping.

**OAuth scopes (when called with an app/API token):**

- `DS.Read.All` or `DS.ReadWrite.All` → all data sources.
- Per-source: `DS.Read.{Slug}` or `DS.ReadWrite.{Slug}`, where `{Slug}` is the PascalCase of `appSlug_dataSourceSlug` (e.g. `base.contact` → `DS.Read.BaseContact`).
- Delegated user-context tokens (`openid`/`profile`/`email`/`offline_access`) are not subject to the per-source scope gate.
- A missing scope throws an insufficient-scope error listing the scope(s) required.

---

## 10. Error catalog & recovery

All DSQL errors are `QueryBuilderError`s (HTTP 400 unless noted). Common ones and how to fix:

| Message | Cause | Fix |
|---|---|---|
| `Only SELECT statements are supported` | DML/DDL or non-select statement | Use a single `SELECT`. |
| `Exactly one SELECT statement is supported` | Multiple statements | Send one statement; combine with `UNION` if needed. |
| `Query exceeds the maximum length of 100000 characters` | Query too long | Shorten / parameterize. |
| `Unqualified table 'X' is not a logical data source or CTE alias` | Bare table name | Use `appSlug.dataSourceSlug` or define a CTE. |
| `Unknown logical data source 'app.source'` | Wrong slug / not queryable | Re-fetch the app schema and copy exact slugs. |
| `External data source '…' is not supported` | Source backed by external provider | Not queryable via DSQL. |
| `Unknown field 'slug'` / `Unknown field 'slug' on 'app.source'` | Wrong field slug | Check schema; qualify with alias. |
| `Field '…' is not supported by the SQL query endpoint` | Virtual field (`field-formula`/`field-list`/…) | Not queryable; derive it in SQL or omit. |
| `Function 'X' is not supported` | Function not on allow-list (or `pg_*`) | Replace with an allow-listed function or post-process. |
| `tenant.X expects N argument(s)` | Wrong pseudo-function arity | Match the signature in §7. |
| `Subqueries in FROM must have an alias` | Unaliased `FROM (SELECT …)` | Add an alias. |
| `Bare * is ambiguous with multiple FROM sources` | `SELECT *` across joins | Qualify: `t.*`. |
| `Duplicate table alias 'X'` | Reused alias | Give each source a distinct alias. |
| `Access denied to logical data source '…'` (403) | No view ACL | The caller lacks permission for that source. |

When a query fails: read the message, fix against the schema and the rules above, and retry. If it suggests a wrong source/field, go back to schema discovery.

---

## 11. Patterns

```sql
-- Resolve owner name/email via tenant.user
select t.id, t.title, u.full_name as owner
from base.task t
left join tenant.user u on u.id = t.record_owner
order by t.created_on desc
limit 50;

-- Resolve a status enum label/color via tenant.enum
select c.id, c.name, e.name as status_label, e.color
from crm.contact c
left join tenant.enum e on e.id = c.status;

-- Status breakdown, aggregated in SQL
select e.name as status, count(*) as total
from base.task t
left join tenant.enum e on e.id = t.status
group by e.name
order by total desc;

-- "My records" using identity pseudo-functions
select t.id, t.title
from base.task t
where tenant.is_current_user(t.record_owner)
limit 100;

-- Records owned by anyone in the current user's hierarchy scope
select t.id, t.title, t.record_owner
from base.task t
where tenant.in_current_user_scope(t.record_owner);
```

---

## 12. Differences from standard PostgreSQL (gotchas)

- You query **logical** names (`appSlug.dataSourceSlug`), never physical tables. Bare/unqualified table names are rejected.
- The function surface is a **strict allow-list** — many everyday Postgres functions (window functions, `string_agg`, `regexp_*`, `nullif`, `left`/`right`, `position`, `generate_series`) are unavailable.
- `id` is guaranteed on every source; the physical `user_id` on `tenant.user` is hidden behind `id`.
- Virtual/computed Docyrus fields don't exist as columns.
- Every query is implicitly row-level-secured and clamped by a `LIMIT` cap; "see everything" is not possible.
- `tenant.*` are pseudo-functions, not real functions — they only work inside DSQL.
