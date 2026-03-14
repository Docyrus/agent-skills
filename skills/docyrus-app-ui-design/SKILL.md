---
name: docyrus-app-ui-design
description: Design and build production-grade UI components for Docyrus React applications using preferred component libraries. Use when creating dashboards, forms, data tables, layouts, or any UI elements. Triggers on tasks involving component selection, UI design, dashboard creation, layout design, shadcn, diceui, animate-ui, docyrus-ui, reui components, or requests to build user interfaces.
---

# Docyrus App UI Design

Build polished, accessible UI components for Docyrus React applications using a curated set of 159 pre-built components from shadcn, diceui, animate-ui, docyrus-ui, and reui libraries.

## Component Library Preferences

**Primary component libraries (in order of preference):**

1. **shadcn** — 43 core components (buttons, forms, dialogs, tables, charts)
2. **diceui** — 41 advanced components (data grids, kanban, file upload, color picker)
3. **animate-ui** — 21 animated components (sidebar, dialogs, cards, menus)
4. **docyrus** — 51 Docyrus-specific components (awesome dialog, awesome stats, data grid, pivot grid, form fields, value renderers, editable record detail, gantt, scheduling, chat, AI agents, pricing, sharing, email composer, log activity form, schema repeater, stepper)
5. **reui** — 2 utility components (file upload, sortable)

**Total available**: 158 components

## Critical Design Rules

1. **Always check preferred components first** — Before implementing any new component, search the preferred components reference to find an existing match.
2. **Use AwesomeCard for dashboards** — Unless the user specifically requests a different card design, use `@docyrus/ui-awesome-card` for dashboard stat cards and metrics.
3. **Use animate-ui Sidebar for layouts** — Unless the user requests a different layout, use `@animate-ui/sidebar` for app navigation.
4. **Prefer Recharts for charting** — Use Recharts as the first choice for all data visualization needs. shadcn Chart components are built on Recharts.
5. **Icon library preference order**:
   - **First choice**: hugeicons
   - **Second choice**: fontawesome light
   - **Third choice**: lucide-icons
6. **Use AwesomeDialog for item create forms** — Use the AwesomeDialog system for all item creation and editing forms. Choose the container type based on form complexity:
   - **Small/simple forms** → `container="sheet"` with `side="right"`
   - **Long/complex forms** → `container="modal"` or `container="drawer"`
7. **Item detail page strategy** — Choose the right container based on item complexity:
   - **Large items** (e.g. projects, workspaces) → Create a dedicated **new page** with full layout
   - **Small items** (e.g. tasks, comments, contacts) → Use **AwesomeDialog** with `container="sheet"` and `side="right"` as a right drawer
8. **Always use TanStack Form + Docyrus form system** — All forms must use TanStack Form with the Docyrus `DynamicFormField` system and `@docyrus/ui-form-fields`. Never use plain HTML forms or React Hook Form directly.
9. **Use EditableRecordDetail for inline editing** — When showing item detail views, use `EditableRecordDetail` with `EditableRecordDetailField` to allow users to edit fields inline without opening a full form. **Always enable `trackChanges`** to highlight changed fields and show a floating ActionBar with Save/Cancel. Add an **"Edit All"** button in the detail page header to switch to a full form editing experience.
10. **Always enable `trackChanges` for editable components** — When using `EditableRecordDetail` or `DataGrid` with cell editing, always pass `trackChanges` prop. This highlights modified cells/fields, shows pending change counts, and provides Save/Cancel actions via a floating ActionBar.

## Quick Start Patterns

### Item Create Form with AwesomeDialog (Small Form → Sheet)

```tsx
import {
  AwesomeDialog, AwesomeDialogHeader, AwesomeDialogBody, AwesomeDialogFooter
} from '@docyrus/ui/components/awesome-dialog'

<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right" size="default">
  <AwesomeDialogHeader title="Create Task" icon="far-plus" />
  <AwesomeDialogBody>
    <form.Field name="title">{(field) => <TextFormField field={field} label="Title" />}</form.Field>
    <form.Field name="status">{(field) => <SelectFormField field={field} label="Status" />}</form.Field>
  </AwesomeDialogBody>
  <AwesomeDialogFooter>
    <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleSubmit}>Create</Button>
  </AwesomeDialogFooter>
</AwesomeDialog>
```

### Item Create Form with AwesomeDialog (Long Form → Modal)

```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="modal" size="lg" fullscreenable>
  <AwesomeDialogHeader title="Create Project" icon="far-folder-plus" />
  <AwesomeDialogBody>
    {/* Long form with many fields — body scrolls, header/footer stay fixed */}
  </AwesomeDialogBody>
  <AwesomeDialogFooter>
    <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleSubmit}>Create Project</Button>
  </AwesomeDialogFooter>
</AwesomeDialog>
```

### Item Detail in AwesomeDialog (Small Item → Right Drawer)

```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right" size="lg" fullscreenable>
  <AwesomeDialogHeader
    title="Task Detail"
    description="Review and edit task fields inline"
    headerButtons={<Button variant="outline" size="sm" onClick={switchToFullForm}>Edit All</Button>}
  />
  <AwesomeDialogBody>
    <EditableRecordDetail fields={fields} record={record} onSave={handleSave} trackChanges>
      <EditableRecordDetailField slug="title" />
      <EditableRecordDetailField slug="status" />
      <EditableRecordDetailField slug="assignee" />
      <EditableRecordDetailField slug="due_date" />
    </EditableRecordDetail>
  </AwesomeDialogBody>
</AwesomeDialog>
```

### Inline Editing with EditableRecordDetail

```tsx
import {
  EditableRecordDetail, EditableRecordDetailField
} from '@docyrus/ui/components/editable-record-detail'

<EditableRecordDetail
  fields={fields}
  record={record}
  onSave={handleSave}
  onCancel={handleCancel}
  trackChanges  // Always enable — highlights changed fields and shows floating ActionBar with Save/Cancel
>
  <div className="space-y-3">
    <EditableRecordDetailField slug="company_name" />
    <EditableRecordDetailField slug="status" />
    <EditableRecordDetailField slug="priority" />
    <EditableRecordDetailField slug="email" />
  </div>
</EditableRecordDetail>
{/* With trackChanges: changed fields get amber highlight, ActionBar shows "N fields changed" with Save/Cancel */}
```

### Dashboard with AwesomeCard

```tsx
import { AwesomeCard } from '@/components/ui/awesome-card'
import { HugeIcon } from '@/components/ui/icons'

<AwesomeCard
  title="Total Revenue"
  value="$124,500"
  icon={<HugeIcon name="dollar-circle" />}
  trend={{ value: 12.5, direction: 'up' }}
/>
```

### App Layout with animate-ui Sidebar

```tsx
import { Sidebar, SidebarContent, SidebarHeader } from '@/components/ui/sidebar'

<Sidebar>
  <SidebarHeader>
    <AppLogo />
  </SidebarHeader>
  <SidebarContent>
    <NavItems />
  </SidebarContent>
</Sidebar>
```

### Data Table with diceui

```tsx
import { DataTable } from '@/components/ui/data-table'

<DataTable
  columns={columns}
  data={projects}
  enableFiltering
  enableSorting
  enablePagination
/>
```

### Charts with shadcn + Recharts

```tsx
import { ChartContainer, ChartTooltip } from '@/components/ui/chart'
import { LineChart, Line, XAxis, YAxis } from 'recharts'

<ChartContainer>
  <LineChart data={chartData}>
    <XAxis dataKey="month" />
    <YAxis />
    <ChartTooltip />
    <Line type="monotone" dataKey="revenue" />
  </LineChart>
</ChartContainer>
```

### Quick Record Create with CreateRecordDialog

```tsx
import { CreateRecordDialog } from '@docyrus/ui/components/create-record-dialog'

<CreateRecordDialog
  headerIcon={<PlusIcon />}
  headerLabel="New Task"
  headerFields={[
    { key: 'project', label: 'Project', options: projectOptions }
  ]}
  subjectPlaceholder="Task name..."
  descriptionPlaceholder="Add description..."
  footerFields={[
    { key: 'assignee', label: 'Assignee', options: userOptions },
    { key: 'priority', label: 'Priority', options: priorityOptions }
  ]}
  enableCreateMore
  onSubmit={handleSubmit}
  isPending={isPending}
>
  <Button variant="outline" size="sm"><PlusIcon /> New Task</Button>
</CreateRecordDialog>
```

### Team Chat with TeamChatChannel

```tsx
import { TeamChatChannel } from '@docyrus/ui/components/team-chat-channel'

<TeamChatChannel
  posts={posts}
  currentUser={currentUser}
  users={users}
  channelName="General"
  onCreatePost={handleCreatePost}
  onDeletePost={handleDeletePost}
  onToggleReaction={handleToggleReaction}
  onUploadFile={handleUploadFile}
  onLoadReplies={handleLoadReplies}
  maxHeight={600}
/>
```

### Resource Scheduler

```tsx
import { ResourceSchedulerPanel } from '@docyrus/ui/components/resource-scheduler-panel'

<ResourceSchedulerPanel
  resources={resources}
  events={events}
  onEventClick={handleEventClick}
  onEventMove={handleEventMove}
  onSlotClick={handleSlotClick}
  showTodayIndicator
  height={500}
/>
```

### Dashboard Stats with AwesomeStats

```tsx
import { AwesomeStats } from '@docyrus/ui/components/awesome-stats'

const stats = [
  {
    id: 'revenue',
    title: 'Total Revenue',
    value: 124500,
    format: { style: 'currency', currency: 'USD', notation: 'compact' },
    icon: 'fal chart-line',
    color: 'blue',
    comparison: { previousValue: 110000 },
    miniChart: { type: 'sparkline', data: [80, 95, 110, 105, 120, 125] }
  },
  // ... more stats
]

// Grid layout (dashboard)
<AwesomeStats items={stats} layout="grid" columns={4} />

// Tabs layout (single card with animated transitions)
<AwesomeStats items={stats} layout="tabs" />

// With AwesomeCard styling
<AwesomeStats items={stats} layout="grid" cardVariant="awesome" />
```

### Pivot Grid for Analytics

```tsx
import { usePivotGrid, PivotGridView, PivotGridToolbar } from '@docyrus/ui/components/pivot-grid'

const { pivotState, ...pivotProps } = usePivotGrid({
  data: salesData,
  dimensions: [
    { id: 'region', label: 'Region', getValue: (row) => row.region },
    { id: 'product', label: 'Product', getValue: (row) => row.product },
  ],
  measures: [
    { id: 'revenue', label: 'Revenue', aggregate: 'sum', getValue: (row) => row.revenue, formatValue: (v) => `$${v.toLocaleString()}` },
  ],
})

<PivotGridToolbar {...pivotProps} />
<PivotGridView {...pivotProps} />
```

### Log Activity Form (CRM)

```tsx
import { LogActivityForm } from '@docyrus/ui/components/log-activity-form'

<LogActivityForm
  defaultActivityType="call"
  mentionUsers={users}
  events={calendarEvents}
  statusOptions={statusOptions}
  onSubmit={handleSubmit}
  isSubmitting={isPending}
/>
```

### AI Agent with DocyrusAgent

```tsx
import { DocyrusAgent } from '@docyrus/ui/components/docyrus-agent'

<DocyrusAgent
  mode="chat"
  agent={{ name: 'Assistant', description: 'AI helper' }}
  messages={messages}
  chatStatus={status}
  onSendMessage={handleSendMessage}
  onStopGeneration={handleStop}
  allowAttachments
  suggestions={['Summarize this', 'Create a report']}
/>
```

### Pricing Engine

```tsx
import { PricingEnginePanel } from '@docyrus/ui/components/pricing-engine-panel'

<PricingEnginePanel
  title="Quote #1234"
  enableVat
  enableLineDiscount
  enableGlobalDiscount
  defaultVatRate={20}
  vatRates={[0, 10, 20]}
  productCatalog={products}
  onSave={handleSave}
  onTotalsChange={handleTotalsChange}
/>
```

### Mega Select for Rich Picking

```tsx
import { MegaSelect } from '@docyrus/ui/components/mega-select'

<MegaSelect
  items={templateItems}
  categories={categories}
  columns={3}
  size="large"
  searchable
  onChoose={(id, item) => handleChoose(item)}
/>
```

## Component Selection Strategy

When the user requests a UI component:

1. **Search the reference** — Check `references/preferred-components-catalog.md` for existing components by name, category, or functionality
2. **Match by use case** — If multiple options exist, prefer:
   - shadcn for basic/common components
   - diceui for advanced/specialized components
   - animate-ui for components requiring animation/transitions
   - docyrus for Docyrus-specific data handling
3. **Install the component** — Use the registry install command from the catalog
4. **Reference the docs** — Point to the component's doc file for detailed usage

## Installation Pattern

Components follow the shadcn registry pattern. Docyrus components use the `@docyrus/cli`:

```bash
# shadcn components
pnpm dlx shadcn@latest add button

# diceui components
pnpm dlx shadcn@latest add @diceui/data-table

# animate-ui components
pnpm dlx shadcn@latest add @animate-ui/sidebar

# docyrus components (use @docyrus/cli)
pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card

# reui components
pnpm dlx shadcn@latest add @reui/file-upload-default
```

## Common Use Cases

| Use Case | Preferred Component | Library |
|----------|-------------------|---------|
| Item create forms | AwesomeDialog (sheet/modal/drawer) | docyrus |
| Quick record create | CreateRecordDialog | docyrus |
| Item detail (small) | AwesomeDialog (sheet right) | docyrus |
| Item detail (large) | Dedicated page | — |
| Inline record editing | EditableRecordDetail | docyrus |
| Dashboard cards | AwesomeCard | docyrus |
| App navigation | Sidebar | animate-ui |
| Data tables | DataTable | diceui |
| Editable grids | Data Grid | docyrus |
| Grid saved views | DataGridViewSelect | docyrus |
| Forms | Form Fields + TanStack Form | docyrus |
| Rich item selector | MegaSelect | docyrus |
| Radio card selection | RadioGroup (card variant) | docyrus |
| File upload | File Upload | diceui |
| Charts | Chart + Recharts | shadcn |
| Confirmation dialogs | Alert Dialog | animate-ui |
| Date picker | Date Time Picker | docyrus |
| Color picker | Color Picker | diceui |
| Kanban board | Kanban | docyrus |
| Gantt chart | Gantt | docyrus |
| Resource scheduling | ResourceSchedulerPanel | docyrus |
| Appointment booking | TimeSlotScheduler | docyrus |
| Timeline | Timeline | diceui |
| Team chat | TeamChatChannel | docyrus |
| Email composing | EmailComposer | docyrus |
| Markdown editing | SimpleMarkdownEditor | docyrus |
| Contact activities | ContactActivityPanel | docyrus |
| AI agent interface | DocyrusAgent | docyrus |
| Pricing / quoting | PricingEnginePanel | docyrus |
| Record sharing | RecordSharing | docyrus |
| Notifications | NotificationStack | docyrus |
| Search | SearchInput | docyrus |
| Location input | PlaceAutocomplete | docyrus |
| Map display | Map | docyrus |
| Tree hierarchy | TreeView | docyrus |
| Dashboard stats | AwesomeStats | docyrus |
| Pivot table / analytics | PivotGrid | docyrus |
| Date range selection | DateTimeRangePicker | docyrus |
| Activity logging (CRM) | LogActivityForm | docyrus |
| Dynamic repeating rows | SchemaRepeater | docyrus |
| Multi-step wizard | Stepper | docyrus |

## References

Read these files for detailed component information:

- **`references/preferred-components-catalog.md`** — Complete catalog of all 160 components with descriptions, install commands, and doc paths (docyrus docs at `https://alpha-ui.docy.app/docs/web/components/<name>/llms.txt`)
- **`references/component-selection-guide.md`** — Decision trees and guidelines for choosing the right component for each use case
- **`references/icon-usage-guide.md`** — Icon library integration patterns and usage examples for hugeicons, fontawesome, and lucide

### Docyrus UI Guide Documentation

Fetch these llms.txt endpoints for detailed guidance:

- **[Components Overview](https://alpha-ui.docy.app/docs/web/components/llms.txt)** — Production-ready UI components built with React, TypeScript, Tailwind CSS v4, and CVA variants
- **[Installation](https://alpha-ui.docy.app/docs/web/guide/installation/llms.txt)** — How to install and set up Docyrus UI components in your project
- **[Theming](https://alpha-ui.docy.app/docs/web/guide/theming/llms.txt)** — Customize the look and feel with CSS variables and the theme generator
- **[Distributions](https://alpha-ui.docy.app/docs/web/guide/distributions/llms.txt)** — Component distributions and core libraries that power Docyrus UI web components
- **[Examples & Recipes](https://alpha-ui.docy.app/docs/web/guide/examples/llms.txt)** — Practical multi-component patterns and recipes for common UI scenarios
- **[Troubleshooting](https://alpha-ui.docy.app/docs/web/guide/troubleshooting/llms.txt)** — Common issues and solutions when using Docyrus UI web components
- **[Releases](https://alpha-ui.docy.app/docs/web/guide/releases/llms.txt)** — Latest releases and updates for Docyrus UI web components

## Integration with docyrus-app-dev

This skill works alongside `docyrus-app-dev` for full-stack app development:

- **docyrus-app-dev** handles data fetching, collections, queries, auth
- **docyrus-app-ui-design** handles component selection, UI design, layout

When building a feature:
1. Use `docyrus-app-dev` to set up data queries and mutations
2. Use `docyrus-app-ui-design` to select and implement UI components
3. Combine them for complete, polished features
