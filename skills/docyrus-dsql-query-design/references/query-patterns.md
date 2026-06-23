# DSQL Query Patterns

## Basic patterns

### Simple select with ordering

```sql
select p.id, p.name, p.status
from base.project p
order by p.created_on desc
limit 25
```

### Filter by current user's own records

```sql
select t.id, t.subject, t.status
from base.task t
where t.record_owner = tenant.current_user_id()
order by t.created_on desc
limit 50
```

### Filter by team membership (show records owned by anyone in my teams)

```sql
select t.id, t.subject
from base.task t
where tenant.user_in_current_teams(t.record_owner)
order by t.created_on desc
```

### Filter by hierarchy scope (records owned by users in my org branch)

```sql
select t.id, t.subject
from base.task t
where tenant.user_in_current_hierarchy_scope(t.record_owner)
```

---

## Join patterns

### Join with `tenant.user` (resolve owner name + email)

```sql
select t.id, t.subject, u.name as owner_name, u.email as owner_email
from base.task t
left join tenant.user u on u.id = t.record_owner
order by t.created_on desc
limit 50
```

### Join with `tenant.enum` (resolve status label and color)

```sql
select t.id, t.subject, e.name as status_label, e.color
from base.task t
left join tenant.enum e on e.id = t.status
order by t.created_on desc
limit 50
```

### Join two data sources

```sql
select t.id, t.subject, p.name as project_name
from base.task t
left join base.project p on p.id = t.project
where t.subject is not null
order by t.created_on desc
limit 50
```

### Same source joined twice (use explicit aliases)

```sql
select c.id, c.name, m.name as manager_name
from crm.contact c
left join crm.contact m on m.id = c.manager
limit 50
```

---

## Aggregate patterns

### Count records by status

```sql
select e.name as status, count(t.id) as task_count
from base.task t
left join tenant.enum e on e.id = t.status
group by e.name
order by task_count desc
```

### Tasks per project with owner name

```sql
select p.name as project, u.name as owner, count(t.id) as task_count
from base.task t
left join base.project p on p.id = t.project
left join tenant.user u on u.id = p.record_owner
group by p.name, u.name
order by task_count desc
limit 25
```

### Sum a numeric field grouped by a category

```sql
select e.name as priority, sum(t.estimated_hours) as total_hours
from base.task t
left join tenant.enum e on e.id = t.priority
group by e.name
order by total_hours desc
```

---

## Date range patterns

### Records created in the last 30 days

```sql
select t.id, t.subject, t.created_on
from base.task t
where t.created_on >= now() - interval '30 days'
order by t.created_on desc
limit 100
```

### Truncated by week (weekly counts)

```sql
select date_trunc('week', t.created_on) as week, count(t.id) as count
from base.task t
where t.created_on >= now() - interval '90 days'
group by week
order by week desc
```

### Filter by a specific date range

```sql
select t.id, t.subject, t.created_on
from base.task t
where t.created_on between '2026-01-01' and '2026-06-30'
order by t.created_on desc
```

---

## CTE patterns

### Tasks with project-level counts

```sql
with task_counts as (
  select t.project, count(t.id) as task_count
  from base.task t
  group by t.project
)
select p.id, p.name, coalesce(task_counts.task_count, 0) as task_count
from base.project p
left join task_counts on task_counts.project = p.id
order by task_count desc
limit 25
```

### Multi-CTE: top assignees this month

```sql
with recent as (
  select t.record_owner, count(t.id) as completed
  from base.task t
  where t.status_slug = 'done'
    and t.updated_on >= date_trunc('month', now())
  group by t.record_owner
)
select u.name, r.completed
from recent r
left join tenant.user u on u.id = r.record_owner
order by r.completed desc
limit 10
```

---

## Custom data source patterns

### Query a simple data source by field slug

```sql
select s.id, s.name, s.email, s.phone
from crm.contact s
where s.email ilike '%@example.com'
order by s.created_on desc
limit 50
```

### Join custom source with base source

```sql
select d.id, d.name as deal_name, c.name as company
from crm.deal d
left join crm.contact c on c.id = d.company
order by d.created_on desc
limit 50
```

---

## AI-assisted patterns

### Natural language → generate only (review before running)

```bash
docyrus dsql generate "show me all overdue tasks with their owners and project names"
# Returns: { prompt, query }  — inspect the SQL, then run it manually
```

### Natural language → generate + run in one shot

```bash
docyrus dsql ask "how many tasks were completed per project last month?"
# Returns: { prompt, query, data: [...], meta: { count: N } }
```

### Override the DSQL generator agent

```bash
docyrus dsql ask "sales pipeline by stage" --agentId <custom-agent-id>
```

---

## Validation checklist

After writing a query, confirm:

- [ ] Every table uses `appSlug.dataSourceSlug` notation
- [ ] Every table has an explicit alias
- [ ] All columns are qualified with an alias in multi-table queries
- [ ] No bare `*` with multiple sources in scope
- [ ] No INSERT/UPDATE/DELETE/DDL
- [ ] Only approved PostgreSQL functions used (no schema-qualified calls)
- [ ] `tenant.*` pseudo-functions used for session-identity filters
- [ ] LIMIT is set (or the default 100 is acceptable)
- [ ] Schema was inspected — field slugs verified

---

## Error reference

| Error | Cause | Fix |
|---|---|---|
| `Unknown field "xyz"` | Field slug doesn't exist on the source | Fetch schema and check exact slugs |
| `Ambiguous column "name"` | Same column name in two joined sources, no alias | Qualify: `t.name`, `p.name` |
| `Unsupported statement` | Non-SELECT SQL (INSERT, UPDATE, etc.) | DSQL is read-only — use SELECT only |
| `Direct table access not allowed` | Physical table name used (`public.tenant_user`) | Use `tenant.user` logical source |
| `Function not supported: md5` | Unsupported Postgres function | Check the approved function list |
| `Query result exceeds max rows` | Result > 1000 rows | Add a LIMIT or aggregate |
| `ambiguous bare *` | `SELECT *` with multiple joined sources | Use `alias.*` or list columns |
| Schema returns empty `schema` string | Data source exists but is not DSQL-queryable | Use only queryable (non-external) sources |
