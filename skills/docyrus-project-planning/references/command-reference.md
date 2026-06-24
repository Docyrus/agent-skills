# Project Plan CLI Command Reference

All commands run as `docyrus project-plan <command> [options]`. Append `--json` for structured output (suppressed by default in human TTY mode).

---

## init

Initialise a new project plan. Creates the plan graph, the standard Docyrus backbone phases, and an empty `diagrams/` folder (where you author `*.mermaid` files) in one shot. Safe to call on an already-initialised project — prints a notice and exits without modifying anything.

```bash
docyrus project-plan init
```

Backbone phases created (with `order`): `Data Sources` (1), `Navigation & Layout` (2), `Dashboards` (90), `Templates` (91), `Access Control` (92), `Automations` (93), `AI Tools (Docy)` (94). Orders 10–89 are intentionally left free so you can insert topic-by-topic UI implementation phases between the shell and the tail phases.

Returns: `{ alreadyInitialized, graphPath, markdownPath, diagramsDir, phaseCount }`

---

## ensure

Internal command — guarantee the graph file exists. Called automatically by every write command before it modifies the graph. Use `init` for intentional project setup; `ensure` is for automation scripts that need to be idempotent without the opinionated phase scaffolding.

```bash
docyrus project-plan ensure
```

Returns: `{ graphPath, markdownPath, graph }`

---

## show

Print the full plan hierarchy: phases with their tasks, features with their tasks, and any linked local subtodos.

```bash
docyrus project-plan show
```

Returns: `{ graph, hierarchy, graphPath, markdownPath }`

---

## check

Validate the graph for integrity errors (orphaned task references, duplicate IDs, etc.).

```bash
docyrus project-plan check
```

Returns: `{ ok, errors[] }` where each error has `code`, `message`, and optional `target`.

---

## config

Show all resolved file paths for the current project root, plus entity counts.

```bash
docyrus project-plan config
```

Returns paths for: `projectPlanDir`, `graphPath`, `markdownPath`, `diagramsDir`, plus `phaseCount`, `featureCount`, `taskCount`.

---

## summary

Aggregate statistics: task distribution by status, assignee, and type; per-phase and per-feature progress percentages.

```bash
docyrus project-plan summary
```

Returns: `{ projectVersion, totalPhases, totalFeatures, totalTasks, tasksByStatus, tasksByAssignee, tasksByType, phaseProgress[], featureProgress[] }`

---

## upsert-phase

Create or update a phase. If `--id` matches an existing phase, updates it; otherwise creates a new phase.

```bash
docyrus project-plan upsert-phase \
  --title "Data Model" \
  --summary "Create all data sources and fields" \
  --order 2
```

| Flag | Required | Description |
|------|----------|-------------|
| `--title` | yes | Phase title |
| `--id` | no | Existing phase ID to update |
| `--slug` | no | URL-safe slug (auto-derived from title if omitted) |
| `--summary` | no | Short description |
| `--order` | no | Display order (lower = first) |

Returns the upserted phase with its stable `id`.

---

## upsert-feature

Create or update a feature. Use `--featureGroupId` to link multiple versions of the same feature.

```bash
docyrus project-plan upsert-feature \
  --title "Invoice data source" \
  --summary "Tracks customer invoices with line items and payment status" \
  --order 1
```

```bash
# Bump to version 2 (redesign) within the same feature group
docyrus project-plan upsert-feature \
  --title "Invoice data source" \
  --version 2 \
  --featureGroupId fg-<hash> \
  --summary "Redesigned with multi-currency support"
```

| Flag | Required | Description |
|------|----------|-------------|
| `--title` | yes | Feature title |
| `--featureId` | no | Existing feature ID to update |
| `--slug` | no | Auto-derived from title if omitted |
| `--summary` | no | Short description |
| `--version` | no | Version number (default: 1) |
| `--featureGroupId` | no | Group ID linking versions (auto-generated on v1 create) |
| `--order` | no | Display order |

Returns the upserted feature with `id`, `featureGroupId`, and `version`.

---

## upsert-task

Create or update a task. Every task belongs to exactly one feature and one phase.

```bash
docyrus project-plan upsert-task \
  --featureId feature-<hash> \
  --phaseId phase-<hash> \
  --title "Create Invoice data source with fields and enums" \
  --type new-implementation \
  --assignee agent \
  --acceptanceCriteria '["Data source exists with all required fields","Enum options created for status field","Schema validates with docyrus studio"]'
```

| Flag | Required | Description |
|------|----------|-------------|
| `--featureId` | yes | Owning feature ID |
| `--title` | yes | Task title |
| `--type` | yes | `new-implementation` / `bug-fix` / `work` / `api-test` / `browser-automation-test` |
| `--assignee` | yes | `agent` (automated) or `user` (requires human action) |
| `--taskId` | no | Existing task ID to update |
| `--phaseId` | no | Phase ID (defaults to first phase if omitted) |
| `--summary` | no | Detailed task description |
| `--status` | no | Initial status (default: `planned`) |
| `--acceptanceCriteria` | no | JSON string array of criteria |
| `--order` | no | Display order |

Returns the upserted task.

---

## list-phases

Slim phase list with per-phase progress. Token-efficient — use instead of `show` when you only need phase state.

```bash
docyrus project-plan list-phases
```

Returns: `{ phases[], total }` where each phase has `id`, `title`, `slug`, `status`, `taskCount`, `doneTaskCount`, `order`.

---

## list-features

Slim feature list with per-feature progress and version info.

```bash
docyrus project-plan list-features
```

Returns: `{ features[], total }` where each feature has `id`, `title`, `slug`, `version`, `featureGroupId`, `status`, `taskCount`, `doneTaskCount`.

---

## list-tasks

Filtered, slim task list. Use `--limit` to get the next N tasks to work on.

```bash
# Next 5 planned tasks
docyrus project-plan list-tasks --status planned --limit 5

# All tasks in a phase
docyrus project-plan list-tasks --phaseId phase-<hash>

# All tasks for a feature (with summaries)
docyrus project-plan list-tasks --featureId feature-<hash> --includeSummary true
```

| Flag | Description |
|------|-------------|
| `--phaseId` | Filter by phase |
| `--featureId` | Filter by feature |
| `--status` | Filter by status (`planned` / `in_progress` / `blocked` / `done`) |
| `--limit` | Max tasks to return |
| `--includeSummary` | Include task summary text (default false) |

Returns: `{ tasks[], total, returned }`

---

## find-tasks

Full-text and attribute search across all tasks.

```bash
docyrus project-plan find-tasks --titleContains "Invoice" --status planned
docyrus project-plan find-tasks --type api-test --assignee agent
docyrus project-plan find-tasks --taskId task-<hash>
```

| Flag | Description |
|------|-------------|
| `--title` | Exact title match |
| `--titleContains` | Case-insensitive substring |
| `--summaryContains` | Case-insensitive substring in summary |
| `--status` | Status filter |
| `--type` | Task type filter |
| `--assignee` | Assignee filter |
| `--featureId` | Feature filter |
| `--phaseId` | Phase filter |
| `--taskId` | Exact ID filter |
| `--limit` | Max results |

Returns: `{ tasks[], total, returned }`

---

## get-task

Single task with full detail and any linked local subtodos.

```bash
docyrus project-plan get-task --taskId task-<hash>
```

Returns the full task plus `linkedTodos[]`.

---

## set-task-status

Update a task's status. The main progress-tracking command.

```bash
docyrus project-plan set-task-status --taskId task-<hash> --status in_progress
docyrus project-plan set-task-status --taskId task-<hash> --status done
```

Valid transitions: `planned` → `in_progress` → `done` (or `blocked` at any point).

---

## set-order

Set the display order of any entity.

```bash
docyrus project-plan set-order --phaseId phase-<hash> --order 3
docyrus project-plan set-order --featureId feature-<hash> --order 2
docyrus project-plan set-order --taskId task-<hash> --order 1
```

Exactly one of `--phaseId`, `--featureId`, or `--taskId` is required.

---

## create-linked-todo

Create a `.pi/todos` subtask linked to an `agent`-assigned canonical task. Used by the coding agent to track sub-steps within a single task.

```bash
docyrus project-plan create-linked-todo \
  --taskId task-<hash> \
  --title "Create enums for status field" \
  --body "Use docyrus studio create-enums..."
```

Only works for `assignee: agent` tasks. Returns the created todo with its local file path.

---

## upsert-from-plan

Sync a `/plan` artifact markdown file (from a pi planning session) into the canonical graph. Creates or updates features and tasks parsed from the plan's sections.

```bash
docyrus project-plan upsert-from-plan \
  --artifactPath .docyrus/plans/20240115-invoice-module.md \
  --task "Design the Invoice module"
```

---

## upsert-from-architect

Sync an `/architect` artifact directory into the graph. Reads `PLAN.md` and `data-sources.plan.json` from the directory.

```bash
docyrus project-plan upsert-from-architect \
  --artifactDir .docyrus/plans/20240115-invoice-architect/ \
  --brief "Invoice module architecture"
```

---

## list-diagrams

List the author-owned Mermaid diagrams in the project plan's `diagrams/` folder. Diagrams are **not** generated by the CLI — you write `*.mermaid` files into the folder yourself (with the Write/file tools), and the dashboard renders whatever is there.

```bash
# List diagram file names + titles
docyrus project-plan list-diagrams

# Include each diagram's Mermaid source in the JSON output
docyrus project-plan list-diagrams --includeContent true
```

| Flag | Description |
|------|-------------|
| `--includeContent` | Include each diagram's Mermaid source in the output |

Returns: `{ diagramsDir, total, diagrams[] }` where each diagram has `{ name, title, fileName, path }` (plus `content` when `--includeContent` is set).

Each diagram's title comes from a leading `%% title: My Title` comment if present, otherwise from the humanized file name. Files are listed in file-name order.
