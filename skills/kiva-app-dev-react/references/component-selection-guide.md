# Component Selection Guide

Decision trees and best practices for choosing the right component for each use case.

## Table of Contents

1. [UX Design Patterns](#ux-design-patterns)
2. [Selection Process](#selection-process)
3. [Common Use Case Decision Trees](#common-use-case-decision-trees)
4. [Library-Specific Strengths](#library-specific-strengths)
5. [Default Choices](#default-choices)
6. [When to Use Each Library](#when-to-use-each-library)

---

## UX Design Patterns

These patterns are **mandatory** and must be followed in all KIVA applications.

### Pattern 1: Item Create Forms → AwesomeDialog

All item creation and editing forms use the **AwesomeDialog** system. Choose the container type based on form complexity:

```
Form complexity → Container choice:

Small/simple form (3-6 fields)     → container="sheet" side="right"
Long/complex form (7+ fields)      → container="modal" size="lg"
Mobile-first form                  → container="drawer" side="bottom"
```

**AwesomeDialog** props reference:
- `container`: `'modal'` | `'sheet'` | `'drawer'` — determines the dialog presentation
- `side`: `'left'` | `'right'` | `'top'` | `'bottom'` — positioning for sheet/drawer
- `size`: `'sm'` | `'default'` | `'lg'` | `'xl'` | `'full'` — size preset
- `pattern`: boolean — show decorative pattern background (default: true)
- `patternStyle`: `'stripes'` | `'dots'` | `'grid'` | `'crosshatch'` | `'zigzag'`
- `fullscreenable`: boolean — allow fullscreen toggle
- `minimizable`: boolean — allow minimize (requires GlobalDialogProvider)
- `resizable`: boolean — allow resize handles
- `dialogId`: string — unique ID for global dialog tracking

**Sub-components**: `AwesomeDialogHeader`, `AwesomeDialogBody`, `AwesomeDialogFooter`, `AwesomeDialogToolbar`

**Example — Small create form (task):**
```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right">
  <AwesomeDialogHeader title="Create Task" icon="far-plus" />
  <AwesomeDialogBody>{/* TanStack Form fields */}</AwesomeDialogBody>
  <AwesomeDialogFooter>
    <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleSubmit}>Create</Button>
  </AwesomeDialogFooter>
</AwesomeDialog>
```

**Example — Large create form (project):**
```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="modal" size="lg" fullscreenable>
  <AwesomeDialogHeader title="Create Project" icon="far-folder-plus" />
  <AwesomeDialogBody>{/* Many form fields — body scrolls */}</AwesomeDialogBody>
  <AwesomeDialogFooter>
    <Button variant="outline" onClick={() => setOpen(false)}>Cancel</Button>
    <Button onClick={handleSubmit}>Create Project</Button>
  </AwesomeDialogFooter>
</AwesomeDialog>
```

### Pattern 2: Item Detail Pages → New Page vs AwesomeDialog

Choose the right container based on item complexity:

```
Item complexity → Detail view choice:

Large items (projects, workspaces, reports)  → Dedicated new page (full route)
Small items (tasks, contacts, comments)      → AwesomeDialog with container="sheet" side="right"
```

**Decision tree:**
```
Does the item have:
  - Multiple tabs/sections?
  - Sub-items or child lists?
  - Complex layout with sidebar?
  → YES: Create a dedicated page route

  - Simple field list?
  - Quick view/edit pattern?
  - Opened from a list or table?
  → YES: Use AwesomeDialog right drawer
```

**Example — Task detail in AwesomeDialog:**
```tsx
<AwesomeDialog open={open} onOpenChange={setOpen} container="sheet" side="right" size="lg" fullscreenable>
  <AwesomeDialogHeader
    title={task.title}
    description={`${task.status} · ${task.assignee}`}
    headerButtons={
      <Button variant="outline" size="sm" onClick={switchToFullForm}>Edit All</Button>
    }
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

### Pattern 3: Form System → TanStack Form + KIVA Form Fields

**All forms must use** TanStack Form with the KIVA form field system. Never use plain HTML forms or React Hook Form directly.

**Key components:**
- `DynamicFormField` — Auto-dispatches to the correct field type based on `IField.type`
- 47+ field types: text, number, email, url, phone, date, dateTime, time, select, multiSelect, status, relation, file, image, code, docEditor, and more
- Each field type has a dedicated `*FormField` component (e.g., `TextFormField`, `SelectFormField`, `DateFormField`)

**Form field pattern:**
```tsx
import { useForm } from '@tanstack/react-form'
import { TextFormField, SelectFormField, DateFormField } from '@kivacore/ui/components/form-fields'

const form = useForm({ defaultValues: { title: '', status: '', dueDate: '' } })

<form.Field name="title">
  {(field) => <TextFormField field={field} label="Title" />}
</form.Field>
<form.Field name="status">
  {(field) => <SelectFormField field={field} label="Status" options={statusOptions} />}
</form.Field>
```

### Pattern 4: Inline Editing → EditableRecordDetail

Use `EditableRecordDetail` for detail views where users can edit individual fields inline without opening a full form page. **Always enable `trackChanges`** to highlight changed fields and show a floating ActionBar. Always include an **"Edit All"** button in the header to switch to a full form editing experience.

**Key components:**
- `EditableRecordDetail` — Provider/wrapper that manages field state, change tracking, and save/cancel
- `EditableRecordDetailField` — Individual field that reads config from context, renders label + inline-editable value
- `EditableValue` — Lower-level single-field inline editor (used internally by EditableRecordDetailField)
- `useEditableRecordDetail()` — Hook to access form, values, changes, and save/cancel from within the provider

**How it works (with `trackChanges` enabled):**
1. Fields render as read-only `DynamicValue` display
2. Click a field → switches to `DynamicFormField` editor inline
3. Changed fields get highlighted with amber background
4. Floating `ActionBar` appears showing "N fields changed" with Save/Cancel buttons
5. Save commits only changed fields; Cancel reverts all changes

**Important:** Always pass `trackChanges` prop when using `EditableRecordDetail` or `DataGrid` with cell editing. Without it, users have no visual feedback about which fields they've modified and no centralized Save/Cancel flow.

**Field change tracking types:**
```tsx
interface RecordDetailField {
  field: IField           // Field configuration (name, slug, type)
  enumOptions?: EnumOption[] // Options for select-based fields
  readOnly?: boolean      // Per-field read-only override
  appSlug?: string        // For dynamic enum loading
  dataSourceSlug?: string // For dynamic enum loading
}

interface FieldChange {
  fieldSlug: string
  fieldName: string
  originalValue: unknown
  newValue: unknown
}
```

**Example — Detail view with inline editing and "Edit All" button:**
```tsx
<AwesomeDialogHeader
  title="Task Detail"
  headerButtons={<Button variant="outline" size="sm" onClick={switchToFullForm}>Edit All</Button>}
/>
<AwesomeDialogBody>
  <EditableRecordDetail fields={fields} record={record} onSave={handleSave} onCancel={handleCancel} trackChanges>
    <div className="space-y-3">
      <h4 className="border-b pb-1.5 text-xs font-semibold uppercase tracking-wider text-muted-foreground">
        General Info
      </h4>
      <EditableRecordDetailField slug="title" />
      <EditableRecordDetailField slug="status" />
      <EditableRecordDetailField slug="priority" />
      <EditableRecordDetailField slug="assignee" />
      <EditableRecordDetailField slug="due_date" />
    </div>
  </EditableRecordDetail>
</AwesomeDialogBody>
```

---

## Selection Process

Follow this process when selecting a component:

```
1. Check if there's a default preference for the use case
   ↓
2. If no default, search for matching components by:
   - Name match (exact or similar)
   - Functionality match
   - Category match (form, data, navigation, etc.)
   ↓
3. If multiple matches found, apply library priority:
   - kiva (if KIVA-specific data handling needed)
   - shadcn (for basic/common components)
   - diceui (for advanced/specialized features)
   - animate-ui (when animation/transitions are important)
   - reui (for specific utility needs)
   ↓
4. Verify the component meets requirements by checking docs
   ↓
5. Install and implement
```

---

## Common Use Case Decision Trees

### Dashboard & Data Visualization

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Stat/metric card | **AwesomeCard** | kiva | Default choice - hatched header design, perfect for metrics |
| Stat dashboard (multi-card) | **AwesomeStats** | kiva | Grid/flex/tabs layouts, mini-charts, comparisons, drag-reorder |
| Alternative card | Card | shadcn | Basic card for custom designs |
| Stat display | Stat | diceui | Dedicated stat display with formatting |
| Pivot table / analytics | **PivotGrid** | kiva | 3-level hierarchies, subtotals, drilldown, export (CSV/Excel/PDF) |
| Chart/graph | Chart + Recharts | shadcn | Built-in Recharts integration |
| Gauge/dial | Gauge | diceui | Circular progress indicator |
| Progress bar | Progress | shadcn | Linear progress indicator |
| Circular progress | Circular Progress | diceui | Ring-style progress |

### Navigation & Layout

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Dynamic forms (default) | **Form Fields + TanStack Form** | kiva | 47 field types with auto-dispatch — **always use this** |
| Inline record editing | **EditableRecordDetail** | kiva | Click-to-edit fields with change tracking + ActionBar |
| Single field inline edit | EditableValue | kiva | Lower-level click-to-edit for individual values |
| Text input | Input | shadcn | Basic text input (use via TextFormField in forms) |
| Text area | Textarea | shadcn | Multi-line text (use via TextareaFormField in forms) |
| Select dropdown | Select | shadcn | Basic select (use via SelectFormField in forms) |
| Combobox/autocomplete | Combobox | diceui | Search + select |
| Date picker | Date Time Picker | kiva | Date + time combined |
| Date range picker | **Date Time Range Picker** | kiva | Start/end date+time pair with size variants |
| Date only | Calendar | shadcn | Date selection only |
| Time only | Time Picker | diceui | Time selection only |
| Phone number | Phone Input | diceui | International formatting |
| Color selection | Color Picker | diceui | Full spectrum + palette |
| File upload | File Upload | diceui | Drag-drop, preview, progress |
| Tags input | Tags Input | diceui | Multiple tag entry |
| Rating | Rating | diceui | Star/heart rating |
| Checkbox group | Checkbox Group | diceui | Multiple selections |
| Radio group (animated) | Radio Group | animate-ui | Single selection with animation |
| Radio group (card/grid) | **Radio Group** | kiva | Card variant with icons, descriptions, grid columns |
| Switch | Switch | animate-ui | Toggle with animation |
| Slider | Slider | shadcn | Range selection |
| OTP input | Input OTP | shadcn | One-time password codes |

### Data Display & Tables

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Editable grid | Data Grid | kiva | Virtualized, keyboard nav, cell editing — enable `trackChanges` for edit tracking |
| Grid saved views | **Data Grid View Select** | kiva | Saved view management with sort/filter/columns — persist with `DataViews` from `@kivacore/app-utils` |
| Read-only table | Table | shadcn | Simple, responsive tables |
| Advanced data table | Data Table | diceui | Filtering, sorting, pagination built-in |
| Table filters | Data Table Filter | kiva | Multi-column filter bar |
| Value display | Value Renderers | kiva | 44 renderer types for read-only display |
| Empty state | Empty | shadcn | No data placeholder |

When using **Data Grid View Select**, back its `views`, `onViewCreate`, `onViewSave`, `onViewDelete`, `onViewHide`, and `onViewUnhide` flows with `createDataViewClient(client, appId)` from `@kivacore/app-utils`. Always pass the TanStack `table` instance, and pass `fields` when you want the editor's built-in filter builder enabled.

### Forms & Inputs

**Always use TanStack Form + KIVA form fields.** The `DynamicFormField` component auto-dispatches to the correct field type.

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Dynamic forms (default) | **Form Fields + TanStack Form** | kiva | 47 field types with auto-dispatch — **always use this** |
| Inline record editing | **EditableRecordDetail** | kiva | Click-to-edit fields with change tracking + ActionBar |
| Single field inline edit | EditableValue | kiva | Lower-level click-to-edit for individual values |
| Text input | Input | shadcn | Basic text input (use via TextFormField in forms) |
| Text area | Textarea | shadcn | Multi-line text (use via TextareaFormField in forms) |
| Select dropdown | Select | shadcn | Basic select (use via SelectFormField in forms) |
| Combobox/autocomplete | Combobox | diceui | Search + select |
| Date picker | Date Time Picker | kiva | Date + time combined |
| Date range picker | **Date Time Range Picker** | kiva | Start/end date+time pair with size variants |
| Date only | Calendar | shadcn | Date selection only |
| Time only | Time Picker | diceui | Time selection only |
| Phone number | Phone Input | diceui | International formatting |
| Color selection | Color Picker | diceui | Full spectrum + palette |
| File upload | File Upload | diceui | Drag-drop, preview, progress |
| Tags input | Tags Input | diceui | Multiple tag entry |
| Rating | Rating | diceui | Star/heart rating |
| Checkbox group | Checkbox Group | diceui | Multiple selections |
| Radio group (animated) | Radio Group | animate-ui | Single selection with animation |
| Radio group (card/grid) | **Radio Group** | kiva | Card variant with icons, descriptions, grid columns |
| Switch | Switch | animate-ui | Toggle with animation |
| Slider | Slider | shadcn | Range selection |
| OTP input | Input OTP | shadcn | One-time password codes |

### Dialogs & Overlays

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Item create form (small) | **AwesomeDialog** (sheet) | kiva | Default — sheet right for quick forms |
| Item create form (large) | **AwesomeDialog** (modal) | kiva | Default — modal for complex forms |
| Item detail (small item) | **AwesomeDialog** (sheet right) | kiva | Default — right drawer for task/contact detail |
| Minimizable/global dialogs | **AwesomeDialog** + GlobalDialogProvider | kiva | Taskbar-style dialog management |
| Responsive dialog | Responsive Dialog | diceui | Auto-switches to drawer on mobile |
| Alert/confirmation dialog | Alert Dialog | animate-ui | Confirmation prompts |
| Basic dialog | Dialog | animate-ui | Animated modal (non-form use cases) |
| Drawer | Drawer | shadcn | Simple side/bottom panel |
| Sheet | Sheet | animate-ui | Complementary content |
| Popover | Popover | animate-ui | Floating content |
| Tooltip | Tooltip | animate-ui | Hover hints |
| Hover card | Hover Card | animate-ui | Rich preview on hover |

### Specialized Components

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Kanban board | **Kanban** | kiva | Drag-drop columns with dnd-kit, keyboard nav |
| Gantt chart | **Gantt** | kiva | Project timeline scheduling |
| Resource scheduling | **Resource Scheduler Panel** | kiva | Horizontal timeline with drag-drop events |
| Appointment booking | **Time Slot Scheduler** | kiva | Columns/month views, capacity, timezone |
| Timeline | Timeline | diceui | Event/step display |
| Stepper / wizard | **Stepper** | kiva | 6 variants, horizontal/vertical, animated connectors |
| Tour/onboarding | Tour | diceui | Interactive tutorials |
| Query builder | Query Builder | kiva | KIVA query construction |
| Notifications | **NotificationStack** | kiva | Stacked notification cards |
| Notification panel | **NotificationsPanel** | kiva | Full notification management |
| Search input | **SearchInput** | kiva | Dedicated search with clear |
| Location input | **PlaceAutocomplete** | kiva | Address search + selection |
| Map display | **Map** | kiva | Geographic data (Leaflet) |
| Tree hierarchy | **TreeView** | kiva | Nested data display |
| Image editing | **ImageEditor** | kiva | Crop, adjust, transform |
| Media player | Media Player | diceui | Video/audio playback |
| QR code | QR Code | diceui | QR generation |
| Cropper / Image cropping | **ImageEditor** | kiva | Crop, adjust, transform |
| Mentions | Mention | diceui | @mention functionality |
| Command palette | Command | shadcn | Keyboard shortcuts |
| Confirmation action | **ConfirmationButton** | kiva | Button with confirm dialog |
| Activity logging (CRM) | **Log Activity Form** | kiva | Calls, emails, meetings, tasks, status updates with Plate editor |
| Dynamic repeating rows | **Schema Repeater** | kiva | Structured data list with customizable input types per column |
| Rich item selector | **Mega Select** | kiva | Grid picker with categories, search, detail panel |
| Quick record create | **Create Record Dialog** | kiva | Popover dialog with subject, mentions, selectors |
| Email composing | **Email Composer** | kiva | To/Cc/Bcc, formatting toolbar, attachments |
| Markdown editing | **Simple Markdown Editor** | kiva | Lightweight MD editor with toolbar/stats |
| Team chat | **Team Chat Channel** | kiva | Posts, threads, reactions, mentions, attachments |
| Contact activities | **Contact Activity Panel** | kiva | Activity timeline with calls, emails, tasks, chat |
| AI agent chat | **KIVA Agent** | kiva | Chat/action-panel/trigger modes for AI agents |
| Pricing/quoting | **Pricing Engine Panel** | kiva | Line items, VAT, discounts, currency, totals |
| Record sharing | **Record Sharing** | kiva | Permission-based sharing with users/teams/roles |

---

## Library-Specific Strengths

### shadcn (43 components)
**Best for**: Core UI primitives, forms, basic layout

**Strengths**:
- Well-tested, accessible components
- Great documentation
- Broad browser support
- Foundation for other libraries

**Use when**: You need a standard, reliable component without special features

### diceui (42 components)
**Best for**: Advanced interactions, specialized data components

**Strengths**:
- Rich feature sets (filtering, sorting, virtualization)
- Advanced input types (phone, color, mask)
- Drag-drop capabilities
- Data visualization components

**Use when**: You need advanced functionality beyond basic components

### animate-ui (21 components)
**Best for**: Animated transitions, polished UX

**Strengths**:
- Smooth, professional animations
- Enhanced visual feedback
- Composable animated patterns

**Use when**: Animation/transitions are important to the UX

### kiva (51 components)
**Best for**: KIVA-specific data handling, forms, dialogs, inline editing, scheduling, chat, AI agents, and business logic

**Strengths**:
- Deep KIVACORE platform integration
- AwesomeDialog system for item creation and detail views
- EditableRecordDetail for inline field editing with change tracking
- 47 form field types with TanStack Form integration
- 44 value renderer types
- Data source query builders
- Scheduling: Gantt, Resource Scheduler, Time Slot Scheduler, Calendar
- Communication: Team Chat Channel, Email Composer, Comments Panel
- AI integration: KIVA Agent (chat, action panel, trigger modes)
- Business: Pricing Engine, Record Sharing, Contact Activity
- Selection: Mega Select, Create Record Dialog
- Kanban board, notifications, maps, markdown editor, and more

**Use when**: Working with KIVA data sources, building item create/detail flows, forms, scheduling, chat, AI features, or queries

### reui (2 components)
**Best for**: Specific utility needs

**Strengths**:
- Focused implementations
- Lightweight alternatives

**Use when**: The specific component matches your exact need

---

## Default Choices

These are the **recommended defaults** unless the user specifies otherwise:

| Use Case | Default Component | Library |
|----------|------------------|---------|
| Item create form | **AwesomeDialog** (sheet/modal) | kiva |
| Quick record create | **Create Record Dialog** | kiva |
| Item detail (small) | **AwesomeDialog** (sheet right) | kiva |
| Inline record editing | **EditableRecordDetail** | kiva |
| Dashboard card | AwesomeCard | kiva |
| App navigation | Sidebar | animate-ui |
| Charts | Chart + Recharts | shadcn |
| Data table | Data Table | diceui |
| Data grid saved views | **Data Grid View Select** + `DataViews` | kiva + `@kivacore/app-utils` |
| Forms | **Form Fields + TanStack Form** | kiva |
| File upload | File Upload | diceui |
| Confirmation dialogs | Alert Dialog | animate-ui |
| Gantt / project scheduling | Gantt | kiva |
| Resource scheduling | **Resource Scheduler Panel** | kiva |
| Appointment booking | **Time Slot Scheduler** | kiva |
| Team chat | **Team Chat Channel** | kiva |
| Email composing | **Email Composer** | kiva |
| AI agent interface | **KIVA Agent** | kiva |
| Kanban board | **Kanban** | kiva |
| Notifications | NotificationStack | kiva |
| Pricing / quoting | **Pricing Engine Panel** | kiva |
| Record sharing | **Record Sharing** | kiva |
| Stat dashboards | **AwesomeStats** | kiva |
| Pivot table / analytics | **PivotGrid** | kiva |
| Activity logging (CRM) | **Log Activity Form** | kiva |
| Multi-step wizard | **Stepper** | kiva |

---

## When to Use Each Library

### Use shadcn when:
- Building basic forms with standard inputs
- Creating simple layouts with cards, buttons, badges
- Need reliable, accessible primitives
- No special animation or advanced features required

### Use diceui when:
- Building complex data tables with sorting/filtering
- Need advanced input types (phone, color, tags)
- Implementing drag-drop (kanban, sortable lists)
- Creating rich data visualizations (gauges, timelines)

### Use animate-ui when:
- Animation/transitions are important to the design
- Building navigation with smooth interactions
- Creating polished dialog/overlay experiences
- Need animated feedback for user actions

### Use kiva when:
- Building item create forms (AwesomeDialog, Create Record Dialog)
- Building item detail views (AwesomeDialog + EditableRecordDetail)
- Working directly with KIVA data sources
- Building forms that map to KIVA fields (TanStack Form + DynamicFormField)
- Displaying KIVA record data in tables/cards
- Need inline editing with change tracking
- Building scheduling UIs (Resource Scheduler, Time Slot Scheduler, Gantt, Calendar)
- Building communication features (Team Chat, Email Composer, Comments Panel)
- Integrating AI agents (KIVA Agent with chat/action modes)
- Need pricing/quoting (Pricing Engine Panel)
- Need record sharing with permissions (Record Sharing)
- Need rich item selectors (Mega Select, Kanban)
- Need KIVA-specific components (query builder, activity panel, notifications)

### Use reui when:
- The specific component (file upload or sortable) matches your exact need
- Want a lightweight alternative to diceui versions

---

## Icon Selection Guide

**Priority order for icons**:

1. **hugeicons** (first choice)
   - Modern, consistent style
   - Large icon set
   - Use for: All general-purpose icons

2. **fontawesome light** (second choice)
   - Professional appearance
   - Familiar to users
   - Use for: When hugeicons doesn't have the needed icon

3. **lucide-icons** (third choice)
   - Clean, minimalist style
   - Use for: Fallback when neither hugeicons nor fontawesome have the icon

**Example**:
```tsx
import { HugeIconDollarCircle } from '@/components/icons/hugeicons'
import { FaLightChartLine } from '@/components/icons/fontawesome'
import { LucideActivity } from 'lucide-react'

// Prefer hugeicons
<HugeIconDollarCircle className="h-5 w-5" />

// Use fontawesome if not in hugeicons
<FaLightChartLine className="h-5 w-5" />

// Use lucide as fallback
<LucideActivity className="h-5 w-5" />
```

---

## Quick Reference: Component Categories

### Dialogs & Item Flows
- Item create forms: **kiva AwesomeDialog** (sheet for small, modal for large)
- Quick record create: **kiva Create Record Dialog** (popover with subject/mentions)
- Item detail (small): **kiva AwesomeDialog** (sheet right)
- Item detail (large): Dedicated page route
- Inline editing: **kiva EditableRecordDetail**
- Confirmation: animate-ui Alert Dialog
- Responsive: diceui Responsive Dialog

### Data & Display
- Tables: kiva Data Grid, diceui Data Table, shadcn Table
- Pivot table: kiva PivotGrid (hierarchies, subtotals, drilldown, export)
- Grid saved views: kiva Data Grid View Select + `DataViews` from `@kivacore/app-utils`
- Cards: kiva AwesomeCard, shadcn Card
- Stat dashboards: kiva AwesomeStats (grid/flex/tabs, mini-charts, comparisons)
- Charts: shadcn Chart + Recharts
- Stats: diceui Stat, diceui Gauge, shadcn Progress
- Gantt: kiva Gantt
- Tree: kiva TreeView

### Forms & Input
- Dynamic forms: **kiva Form Fields + TanStack Form** (always use)
- Inline editing: kiva EditableRecordDetail, EditableValue
- Text: shadcn Input, Textarea
- Markdown: kiva Simple Markdown Editor
- Selection: shadcn Select, diceui Combobox
- Rich selection: kiva Mega Select (grid picker with categories)
- Radio cards: kiva Radio Group (card variant with icons/descriptions)
- Dates: kiva Date Time Picker, kiva Date Time Range Picker, shadcn Calendar
- Files: diceui File Upload
- Special: diceui Phone Input, Color Picker, Tags Input

### Navigation
- Sidebar: animate-ui Sidebar
- Menus: animate-ui Dropdown Menu, shadcn Menubar
- Breadcrumbs: shadcn Breadcrumb
- Tabs: animate-ui Tabs

### Overlays
- Item forms/details: **kiva AwesomeDialog** (preferred)
- Popovers: animate-ui Popover, Tooltip, Hover Card
- Drawers: shadcn Drawer, animate-ui Sheet

### Communication
- Team chat: kiva Team Chat Channel (threads, reactions, mentions)
- Email composing: kiva Email Composer (To/Cc/Bcc, toolbar, attachments)
- Comments: kiva Comments Panel (threaded conversations)
- Contact activities: kiva Contact Activity Panel (calls, meetings, emails)
- Activity logging: kiva Log Activity Form (calls, emails, meetings, tasks, statuses)

### Scheduling
- Project timelines: kiva Gantt
- Resource scheduling: kiva Resource Scheduler Panel (horizontal timeline)
- Appointment booking: kiva Time Slot Scheduler (columns/month views)
- Calendar events: kiva Calendar (month/week/day views)

### AI & Agents
- AI chat: kiva KIVA Agent (chat mode)
- AI actions: kiva KIVA Agent (action-panel mode)
- AI trigger: kiva KIVA Agent (floating trigger button)

### Business Logic
- Pricing/quoting: kiva Pricing Engine Panel (line items, VAT, discounts)
- Record sharing: kiva Record Sharing (permissions, users/teams/roles)

### Specialized
- Kanban: kiva Kanban (drag-drop columns)
- Timeline: diceui Timeline
- Stepper / wizard: kiva Stepper (6 variants, horizontal/vertical, animated)
- Query: kiva Query Builder
- Notifications: kiva NotificationStack
- Maps: kiva Map
- Search: kiva SearchInput
- Repeating rows: kiva SchemaRepeater (dynamic structured data lists)
