# DSQL Language Reference

DSQL (Docyrus Structured Query Language) is a read-only, PostgreSQL-compatible SQL dialect. Queries run against **logical data sources** — not physical Postgres tables.

## Core Model

| Concept | Detail |
|---|---|
| Table notation | `appSlug.dataSourceSlug` — e.g. `base.task`, `crm.contact`, `custom.inventory` |
| System user table | `tenant.user` — join on `u.id` for names and emails |
| System enum table | `tenant.enum` — join on enum field id for labels, color, icon |
| Inherited fields | Child sources (`base.task`, `base.event`) expose parent-source (`base.activity`) fields directly |
| Implicit alias | If no alias is given, DSQL uses the data-source slug; always set explicit aliases |

---

## Schema Discovery Endpoints

| Use case | Endpoint | CLI |
|---|---|---|
| All sources in an app | `GET /v1/dsql/schema/apps/:appSlug` | `docyrus dsql schema app <slug>` |
| One source by slugs | `GET /v1/dsql/schema/apps/:appSlug/data-sources/:dsSlug` | `docyrus dsql schema data-source <appSlug> <dsSlug>` |
| Sources by IDs | `GET /v1/dsql/schema/data-sources?ids=<uuid>,<uuid>` | `docyrus dsql schema data-sources --ids <id1,id2>` |

Each response item has `{ id, appSlug, slug, name, title, schema }` where `schema` is a compact `CREATE TABLE appSlug.dataSourceSlug (...)` string — read the field slugs and `-- references` join hints from here before writing joins.

---

## Supported SQL

### Clauses

```sql
SELECT ... FROM ... [JOIN ...] [WHERE ...] [GROUP BY ...] [HAVING ...] [ORDER BY ...] [LIMIT n] [OFFSET n]
```

Also:
- CTEs: `WITH cte AS (SELECT ...)`
- Nested subqueries
- Column and table aliases
- `alias.*` expansion (not bare `*` when multiple sources are joined)
- Standard aggregates

### What is rejected

- `INSERT`, `UPDATE`, `DELETE`, `MERGE`, `CALL`, `DO`
- DDL (`CREATE`, `ALTER`, `DROP`)
- `EXPLAIN`
- Multiple statements in one call
- Physical / internal Postgres table names
- Functions in `FROM` clause
- Data-modifying CTEs
- Unknown or ambiguous fields
- Schema-qualified functions (e.g. `pg_catalog.now()`)
- SQL parameters (`$1`, `:name`)
- External-provider data sources (v1)

---

## Table Resolution

- `appSlug.dataSourceSlug` → matches the logical data source
- Implicit alias = the data-source slug when no alias is given
- Same source used twice → give each instance an explicit alias
- Unqualified names are allowed only for CTE aliases and unambiguous single-source scopes

---

## Column Resolution

- `alias.fieldSlug` — preferred form
- `fieldSlug` — allowed only when unambiguous in the current scope
- `appSlug.dataSourceSlug.fieldSlug` — fully qualified form
- Ambiguous unqualified column → query rejected

---

## Row Limits

| Rule | Value |
|---|---|
| Default limit (no LIMIT clause) | 100 |
| Hard maximum | 1000 |
| User-written smaller limit | preserved |
| `LIMIT ALL` | subject to the hard cap |
| Query text max length | 100,000 characters |

---

## System Sources

### `tenant.user`

Logical user table for the tenant. Join on `u.id`.

```sql
select t.id, u.name, u.email
from base.task t
left join tenant.user u on u.id = t.record_owner
```

### `tenant.enum`

Logical enum table. Join on the enum field's id value.

```sql
select t.id, e.name as status_label, e.color
from base.task t
left join tenant.enum e on e.id = t.status
```

---

## Tenant Pseudo-Functions

These are rewritten to safe, session-aware SQL by DSQL at compile time. Use them for security-correct filtering — never substitute hard-coded UUIDs.

### Identity

| Function | Returns |
|---|---|
| `tenant.current_user_id()` | UUID of the authenticated user |
| `tenant.active_user_id()` | UUID of the active (may differ if impersonating) |
| `tenant.current_tenant_id()` | Tenant UUID |
| `tenant.current_role_id()` | Primary role UUID |
| `tenant.current_role_ids()` | Array of role UUIDs |
| `tenant.current_team_ids()` | Array of team UUIDs |
| `tenant.current_hierarchy_unit_id()` | Hierarchy unit UUID |
| `tenant.current_scope_user_ids()` | All user IDs in the current org scope |

### User Matching

```sql
tenant.is_current_user(userId)        -- user IS the current session user
tenant.active_user(userId)            -- user IS the active user
tenant.is_not_current_user(userId)
tenant.not_active_user(userId)
```

### Scope Filters (hierarchy-aware)

```sql
tenant.in_current_user_scope(userId)
tenant.user_in_current_scope(userId)
tenant.user_in_current_hierarchy_scope(userId)
tenant.not_in_current_user_scope(userId)
tenant.user_not_in_current_scope(userId)
tenant.user_not_in_current_hierarchy_scope(userId)
```

### Role Filters

```sql
tenant.in_role(userId, roleId)
tenant.user_in_role(userId, roleId)
tenant.not_in_role(userId, roleId)
tenant.user_not_in_role(userId, roleId)
tenant.in_current_roles(userId)          -- user has any of the current session's roles
tenant.user_in_current_roles(userId)
tenant.not_in_current_roles(userId)
tenant.user_not_in_current_roles(userId)
tenant.current_user_in_role(roleId)
```

### Team Filters

```sql
tenant.in_team(userId, teamId)
tenant.user_in_team(userId, teamId)
tenant.not_in_team(userId, teamId)
tenant.user_not_in_team(userId, teamId)
tenant.in_current_user_team(userId)      -- user is in any team the current user belongs to
tenant.in_active_user_team(userId)
tenant.user_in_current_teams(userId)
tenant.not_in_current_user_team(userId)
tenant.not_in_active_user_team(userId)
tenant.user_not_in_current_teams(userId)
tenant.current_user_in_team(teamId)
```

### Hierarchy Unit Filters

```sql
tenant.in_unit(userId, unitId)
tenant.user_in_unit(userId, unitId)
tenant.not_in_unit(userId, unitId)
tenant.user_not_in_unit(userId, unitId)
tenant.in_sub_unit(userId, unitId)           -- userId is in a sub-unit of unitId
tenant.user_in_sub_unit(userId, unitId)
tenant.user_in_sub_units(userId, unitId)
tenant.not_in_sub_unit(userId, unitId)
tenant.user_not_in_sub_unit(userId, unitId)
tenant.user_not_in_sub_units(userId, unitId)
tenant.current_user_in_unit(unitId)
```

---

## Supported PostgreSQL Functions

### String
`length`, `lower`, `upper`, `substr`, `replace`, `concat`, `trim`, `ltrim`, `rtrim`, `btrim`, `split_part`, `initcap`, `reverse`, `strpos`, `lpad`, `rpad`

### Number
`abs`, `ceil`, `floor`, `round`, `sqrt`, `power`, `mod`, `gcd`, `lcm`, `exp`, `ln`, `log`, `log10`, `log1p`, `pi`, `sign`, `width_bucket`, `trunc`, `greatest`, `least`

### Date and Time
`now`, `age`, `clock_timestamp`, `date_part`, `date_trunc`, `extract`, `isfinite`, `justify_days`, `justify_hours`, `make_date`, `make_time`, `make_timestamp`, `make_timestamptz`, `timeofday`, `to_timestamp`, `to_char`, `to_date`, `to_time`

### Utility
`coalesce`

### JSON
`jsonb_array_length`, `jsonb_extract_path`, `jsonb_extract_path_text`, `jsonb_object_keys`, `jsonb_build_object`, `json_build_object`, `array_to_json`, `row_to_json`

### Aggregates
`count`, `sum`, `avg`, `min`, `max`, `array_agg`, `json_agg`, `jsonb_agg`

> Schema-qualified functions (e.g. `pg_catalog.now()`, `public.get_hierarchy(id)`) are rejected. `tenant.*` pseudo-functions are the only exception.
