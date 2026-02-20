# Component Selection Guide

Decision trees and best practices for choosing the right component for each use case.

## Table of Contents

1. [Selection Process](#selection-process)
2. [Common Use Case Decision Trees](#common-use-case-decision-trees)
3. [Library-Specific Strengths](#library-specific-strengths)
4. [Default Choices](#default-choices)
5. [When to Use Each Library](#when-to-use-each-library)

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
   - docyrus (if Docyrus-specific data handling needed)
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
| Stat/metric card | **AwesomeCard** | docyrus | Default choice - hatched header design, perfect for metrics |
| Alternative card | Card | shadcn | Basic card for custom designs |
| Stat display | Stat | diceui | Dedicated stat display with formatting |
| Chart/graph | Chart + Recharts | shadcn | Built-in Recharts integration |
| Gauge/dial | Gauge | diceui | Circular progress indicator |
| Progress bar | Progress | shadcn | Linear progress indicator |
| Circular progress | Circular Progress | diceui | Ring-style progress |

### Navigation & Layout

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| App sidebar | **Sidebar** | animate-ui | Default choice - smooth animations, composable |
| Alternative sidebar | Navigation Menu | shadcn | For mega menus and dropdowns |
| Breadcrumbs | Breadcrumb | shadcn | Path navigation |
| Menu bar | Menubar | shadcn | Desktop-style persistent menu |
| Dropdown menu | Dropdown Menu | animate-ui | Animated transitions |
| Context menu | Context Menu | shadcn | Right-click menus |
| Tabs | Tabs | animate-ui | Animated tab transitions |

### Data Display & Tables

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Editable grid | Data Grid | docyrus | Virtualized, keyboard nav, cell editing |
| Read-only table | Table | shadcn | Simple, responsive tables |
| Advanced data table | Data Table | diceui | Filtering, sorting, pagination built-in |
| Table filters | Data Table Filter | docyrus | Multi-column filter bar |
| Value display | Value Renderers | docyrus | 44 renderer types for read-only display |
| Empty state | Empty | shadcn | No data placeholder |

### Forms & Inputs

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Dynamic forms | Form Fields | docyrus | 47 field types with auto-dispatch |
| Text input | Input | shadcn | Basic text input |
| Text area | Textarea | shadcn | Multi-line text |
| Select dropdown | Select | shadcn | Basic select |
| Combobox/autocomplete | Combobox | diceui | Search + select |
| Date picker | Date Time Picker | docyrus | Date + time combined |
| Date only | Calendar | shadcn | Date selection only |
| Time only | Time Picker | diceui | Time selection only |
| Phone number | Phone Input | diceui | International formatting |
| Color selection | Color Picker | diceui | Full spectrum + palette |
| File upload | File Upload | diceui | Drag-drop, preview, progress |
| Tags input | Tags Input | diceui | Multiple tag entry |
| Rating | Rating | diceui | Star/heart rating |
| Checkbox group | Checkbox Group | diceui | Multiple selections |
| Radio group | Radio Group | animate-ui | Single selection with animation |
| Switch | Switch | animate-ui | Toggle with animation |
| Slider | Slider | shadcn | Range selection |
| OTP input | Input OTP | shadcn | One-time password codes |

### Dialogs & Overlays

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Responsive dialog | Responsive Dialog | diceui | Auto-switches to drawer on mobile |
| Basic dialog | Dialog | animate-ui | Animated modal |
| Alert dialog | Alert Dialog | animate-ui | Confirmation prompts |
| Drawer | Drawer | shadcn | Side/bottom panel |
| Sheet | Sheet | animate-ui | Complementary content |
| Popover | Popover | animate-ui | Floating content |
| Tooltip | Tooltip | animate-ui | Hover hints |
| Hover card | Hover Card | animate-ui | Rich preview on hover |

### Specialized Components

| Need | Component | Library | Rationale |
|------|-----------|---------|-----------|
| Kanban board | Kanban | diceui | Drag-drop columns |
| Timeline | Timeline | diceui | Event/step display |
| Stepper | Stepper | diceui | Multi-step process |
| Tour/onboarding | Tour | diceui | Interactive tutorials |
| Query builder | Query Builder | docyrus | Docyrus query construction |
| Media player | Media Player | diceui | Video/audio playback |
| QR code | QR Code | diceui | QR generation |
| Cropper | Cropper | docyrus | Image cropping |
| Mentions | Mention | diceui | @mention functionality |
| Command palette | Command | shadcn | Keyboard shortcuts |

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

### docyrus (19 components)
**Best for**: Docyrus-specific data handling, forms, business logic

**Strengths**:
- Deep Docyrus platform integration
- 47 form field types
- 44 value renderer types
- Data source query builders

**Use when**: Working with Docyrus data sources, forms, or queries

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
| Dashboard card | AwesomeCard | docyrus |
| App navigation | Sidebar | animate-ui |
| Charts | Chart + Recharts | shadcn |
| Data table | Data Table | diceui |
| Forms | Form Fields | docyrus |
| File upload | File Upload | diceui |
| Dialogs | Responsive Dialog | diceui |

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

### Use docyrus when:
- Working directly with Docyrus data sources
- Building forms that map to Docyrus fields
- Displaying Docyrus record data in tables/cards
- Need Docyrus-specific components (query builder, activity panel)

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

### Data & Display
- Tables: docyrus Data Grid, diceui Data Table, shadcn Table
- Cards: docyrus AwesomeCard, shadcn Card
- Charts: shadcn Chart + Recharts
- Stats: diceui Stat, diceui Gauge, shadcn Progress

### Forms & Input
- Dynamic: docyrus Form Fields
- Text: shadcn Input, Textarea
- Selection: shadcn Select, diceui Combobox
- Dates: docyrus Date Time Picker, shadcn Calendar
- Files: diceui File Upload
- Special: diceui Phone Input, Color Picker, Tags Input

### Navigation
- Sidebar: animate-ui Sidebar
- Menus: animate-ui Dropdown Menu, shadcn Menubar
- Breadcrumbs: shadcn Breadcrumb
- Tabs: animate-ui Tabs

### Overlays
- Dialogs: diceui Responsive Dialog, animate-ui Dialog
- Popovers: animate-ui Popover, Tooltip, Hover Card
- Drawers: shadcn Drawer, animate-ui Sheet

### Specialized
- Kanban: diceui Kanban
- Timeline: diceui Timeline
- Stepper: diceui Stepper
- Query: docyrus Query Builder
