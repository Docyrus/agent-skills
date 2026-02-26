---
name: docyrus-app-ui-design
description: Design and build production-grade UI components for Docyrus React applications using preferred component libraries. Use when creating dashboards, forms, data tables, layouts, or any UI elements. Triggers on tasks involving component selection, UI design, dashboard creation, layout design, shadcn, diceui, animate-ui, docyrus-ui, reui components, or requests to build user interfaces.
---

# Docyrus App UI Design

Build polished, accessible UI components for Docyrus React applications using a curated set of 140+ pre-built components from shadcn, diceui, animate-ui, docyrus-ui, and reui libraries.

## Component Library Preferences

**Primary component libraries (in order of preference):**

1. **shadcn** — 43 core components (buttons, forms, dialogs, tables, charts)
2. **diceui** — 42 advanced components (data grids, kanban, file upload, color picker)
3. **animate-ui** — 21 animated components (sidebar, dialogs, cards, menus)
4. **docyrus** — 32 Docyrus-specific components (awesome dialog, data grid, form fields, value renderers, editable record detail, gantt, notifications)
5. **reui** — 2 utility components (file upload, sortable)

**Total available**: 140 components

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
9. **Use EditableRecordDetail for inline editing** — When showing item detail views, use `EditableRecordDetail` with `EditableRecordDetailField` to allow users to edit fields inline without opening a full form. Add an **"Edit All"** button in the detail page header to switch to a full form editing experience.

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
    <EditableRecordDetail fields={fields} record={record} onSave={handleSave}>
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

<EditableRecordDetail fields={fields} record={record} onSave={handleSave} onCancel={handleCancel}>
  <div className="space-y-3">
    <EditableRecordDetailField slug="company_name" />
    <EditableRecordDetailField slug="status" />
    <EditableRecordDetailField slug="priority" />
    <EditableRecordDetailField slug="email" />
  </div>
</EditableRecordDetail>
{/* ActionBar appears automatically when fields are changed — shows "N fields changed" with Save/Cancel */}
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

All components follow the shadcn registry pattern:

```bash
# shadcn components
pnpm dlx shadcn@latest add button

# diceui components
pnpm dlx shadcn@latest add @diceui/data-table

# animate-ui components
pnpm dlx shadcn@latest add @animate-ui/sidebar

# docyrus components
pnpm dlx shadcn@latest add @docyrus/ui-awesome-card

# reui components
pnpm dlx shadcn@latest add @reui/file-upload-default
```

## Common Use Cases

| Use Case | Preferred Component | Library |
|----------|-------------------|---------|
| Item create forms | AwesomeDialog (sheet/modal/drawer) | docyrus |
| Item detail (small) | AwesomeDialog (sheet right) | docyrus |
| Item detail (large) | Dedicated page | — |
| Inline record editing | EditableRecordDetail | docyrus |
| Dashboard cards | AwesomeCard | docyrus |
| App navigation | Sidebar | animate-ui |
| Data tables | DataTable | diceui |
| Editable grids | Data Grid | docyrus |
| Forms | Form Fields + TanStack Form | docyrus |
| File upload | File Upload | diceui |
| Charts | Chart + Recharts | shadcn |
| Confirmation dialogs | Alert Dialog | animate-ui |
| Date picker | Date Time Picker | docyrus |
| Color picker | Color Picker | diceui |
| Kanban board | Kanban | diceui |
| Gantt chart | Gantt | docyrus |
| Timeline | Timeline | diceui |
| Notifications | NotificationStack | docyrus |
| Search | SearchInput | docyrus |
| Location input | PlaceAutocomplete | docyrus |
| Map display | Map | docyrus |
| Tree hierarchy | TreeView | docyrus |

## References

Read these files for detailed component information:

- **`references/preferred-components-catalog.md`** — Complete catalog of all 140 components with descriptions, install commands, and doc paths
- **`references/component-selection-guide.md`** — Decision trees and guidelines for choosing the right component for each use case
- **`references/icon-usage-guide.md`** — Icon library integration patterns and usage examples for hugeicons, fontawesome, and lucide

## Integration with docyrus-app-dev

This skill works alongside `docyrus-app-dev` for full-stack app development:

- **docyrus-app-dev** handles data fetching, collections, queries, auth
- **docyrus-app-ui-design** handles component selection, UI design, layout

When building a feature:
1. Use `docyrus-app-dev` to set up data queries and mutations
2. Use `docyrus-app-ui-design` to select and implement UI components
3. Combine them for complete, polished features
