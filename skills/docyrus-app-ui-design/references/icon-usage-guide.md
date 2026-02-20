# Icon Usage Guide

Comprehensive guide for using icon libraries in Docyrus applications: hugeicons, fontawesome light, and lucide-icons.

## Table of Contents

1. [Icon Library Preferences](#icon-library-preferences)
2. [Installation](#installation)
3. [Usage Patterns](#usage-patterns)
4. [Selection Strategy](#selection-strategy)
5. [Common Icon Mappings](#common-icon-mappings)
6. [Sizing & Styling](#sizing--styling)
7. [Best Practices](#best-practices)

---

## Icon Library Preferences

**Priority order**:

1. **hugeicons** (first choice)
   - Modern, comprehensive icon set
   - Consistent design language
   - Wide coverage of common use cases

2. **fontawesome light** (second choice)
   - Professional, familiar appearance
   - Extensive icon collection
   - Well-established library

3. **lucide-icons** (third choice)
   - Clean, minimalist style
   - Fallback option
   - Good coverage of basic icons

**Selection rule**: Always try hugeicons first. If the icon doesn't exist, check fontawesome light. Use lucide-icons only as a last resort.

---

## Installation

### hugeicons

```bash
npm install @hugeicons/react
# or
pnpm add @hugeicons/react
```

**Import pattern**:
```tsx
import { IconName } from '@hugeicons/react'
```

### fontawesome light

```bash
npm install @fortawesome/fontawesome-svg-core
npm install @fortawesome/free-solid-svg-icons
npm install @fortawesome/react-fontawesome
# For light icons (pro):
npm install @fortawesome/pro-light-svg-icons
```

**Import pattern**:
```tsx
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome'
import { faIconName } from '@fortawesome/pro-light-svg-icons'
```

### lucide-icons

```bash
npm install lucide-react
# or
pnpm add lucide-react
```

**Import pattern**:
```tsx
import { IconName } from 'lucide-react'
```

---

## Usage Patterns

### hugeicons (Preferred)

```tsx
import { Home01, User01, Settings01, ChartLineData01 } from '@hugeicons/react'

// Basic usage
<Home01 />

// With className
<User01 className="h-5 w-5 text-muted-foreground" />

// In a button
<Button>
  <Settings01 className="mr-2 h-4 w-4" />
  Settings
</Button>

// In AwesomeCard
<AwesomeCard
  icon={<ChartLineData01 className="h-8 w-8" />}
  title="Revenue"
  value="$124,500"
/>
```

### fontawesome light (Second Choice)

```tsx
import { FontAwesomeIcon } from '@fortawesome/react-fontawesome'
import { faHome, faUser, faCog, faChartLine } from '@fortawesome/pro-light-svg-icons'

// Basic usage
<FontAwesomeIcon icon={faHome} />

// With className
<FontAwesomeIcon icon={faUser} className="h-5 w-5 text-muted-foreground" />

// In a button
<Button>
  <FontAwesomeIcon icon={faCog} className="mr-2 h-4 w-4" />
  Settings
</Button>
```

### lucide-icons (Fallback)

```tsx
import { Home, User, Settings, TrendingUp } from 'lucide-react'

// Basic usage
<Home />

// With className
<User className="h-5 w-5 text-muted-foreground" />

// In a button
<Button>
  <Settings className="mr-2 h-4 w-4" />
  Settings
</Button>
```

---

## Selection Strategy

### Decision Process

```
1. Determine what icon you need (e.g., "home", "user", "chart")
   ↓
2. Check hugeicons documentation/search
   - Found? ✓ Use it
   - Not found? → Continue to step 3
   ↓
3. Check fontawesome light documentation/search
   - Found? ✓ Use it
   - Not found? → Continue to step 4
   ↓
4. Check lucide-icons documentation/search
   - Found? ✓ Use it
   - Not found? → Use closest alternative from any library
```

### Example Search Flow

**Scenario**: Need a "dashboard" icon

1. **Check hugeicons**: `Dashboard01` ✓ **Use this**
2. ~~Check fontawesome~~ (already found)
3. ~~Check lucide~~ (already found)

**Scenario**: Need a "bell" icon

1. **Check hugeicons**: `Notification01` ✓ **Use this**
2. ~~Check fontawesome~~ (already found)
3. ~~Check lucide~~ (already found)

**Scenario**: Need a specific icon not in hugeicons

1. **Check hugeicons**: Not found ✗
2. **Check fontawesome**: `faBell` ✓ **Use this**
3. ~~Check lucide~~ (already found)

---

## Common Icon Mappings

| Use Case | hugeicons (1st) | fontawesome (2nd) | lucide (3rd) |
|----------|-----------------|-------------------|--------------|
| Home | `Home01` | `faHome` | `Home` |
| User/Profile | `User01` | `faUser` | `User` |
| Settings | `Settings01` | `faCog` | `Settings` |
| Search | `Search01` | `faSearch` | `Search` |
| Notifications | `Notification01` | `faBell` | `Bell` |
| Dashboard | `Dashboard01` | `faTachometer` | `LayoutDashboard` |
| Chart/Analytics | `ChartLineData01` | `faChartLine` | `TrendingUp` |
| Calendar | `Calendar01` | `faCalendar` | `Calendar` |
| File | `File01` | `faFile` | `File` |
| Folder | `Folder01` | `faFolder` | `Folder` |
| Download | `Download01` | `faDownload` | `Download` |
| Upload | `Upload01` | `faUpload` | `Upload` |
| Edit | `Edit01` | `faEdit` | `Edit` |
| Delete | `Delete01` | `faTrash` | `Trash2` |
| Plus/Add | `Add01` | `faPlus` | `Plus` |
| Close/X | `Cancel01` | `faTimes` | `X` |
| Check | `CheckmarkCircle01` | `faCheck` | `Check` |
| Arrow Right | `ArrowRight01` | `faArrowRight` | `ArrowRight` |
| Arrow Left | `ArrowLeft01` | `faArrowLeft` | `ArrowLeft` |
| Menu | `Menu01` | `faBars` | `Menu` |
| More (3 dots) | `MoreVertical` | `faEllipsisV` | `MoreVertical` |
| Info | `InformationCircle` | `faInfoCircle` | `Info` |
| Warning | `Alert01` | `faExclamationTriangle` | `AlertTriangle` |
| Error | `Cancel01` | `faTimesCircle` | `XCircle` |
| Success | `CheckmarkCircle01` | `faCheckCircle` | `CheckCircle` |
| Lock | `Lock01` | `faLock` | `Lock` |
| Unlock | `Unlock01` | `faUnlock` | `Unlock` |
| Eye/View | `View01` | `faEye` | `Eye` |
| Eye Off/Hide | `ViewOff01` | `faEyeSlash` | `EyeOff` |
| Heart | `Heart01` | `faHeart` | `Heart` |
| Star | `Star01` | `faStar` | `Star` |
| Filter | `Filter01` | `faFilter` | `Filter` |
| Sort | `SortVertical` | `faSort` | `ArrowUpDown` |
| Refresh | `Reload` | `faSync` | `RotateCw` |

---

## Sizing & Styling

### Standard Sizes

```tsx
// Extra small (navigation items, tight spaces)
<Icon className="h-3 w-3" />

// Small (buttons, inline with text)
<Icon className="h-4 w-4" />

// Medium (default for most UI elements)
<Icon className="h-5 w-5" />

// Large (emphasized actions, cards)
<Icon className="h-6 w-6" />

// Extra large (dashboard cards, hero sections)
<Icon className="h-8 w-8" />

// Hero (feature highlights)
<Icon className="h-12 w-12" />
```

### Color & Style

```tsx
// Inherit text color
<Icon className="h-5 w-5" />

// Muted/secondary
<Icon className="h-5 w-5 text-muted-foreground" />

// Primary color
<Icon className="h-5 w-5 text-primary" />

// Destructive/danger
<Icon className="h-5 w-5 text-destructive" />

// Success
<Icon className="h-5 w-5 text-green-600" />

// Warning
<Icon className="h-5 w-5 text-amber-600" />

// With opacity
<Icon className="h-5 w-5 text-foreground/50" />
```

### Common Patterns

**Icon with text (button)**:
```tsx
<Button>
  <Icon className="mr-2 h-4 w-4" />
  Button Text
</Button>
```

**Icon-only button**:
```tsx
<Button variant="ghost" size="icon">
  <Icon className="h-5 w-5" />
  <span className="sr-only">Accessible label</span>
</Button>
```

**Icon in input**:
```tsx
<div className="relative">
  <Icon className="absolute left-3 top-3 h-4 w-4 text-muted-foreground" />
  <Input className="pl-9" />
</div>
```

**Icon in card header**:
```tsx
<AwesomeCard
  icon={<Icon className="h-8 w-8 text-primary" />}
  title="Title"
  value="Value"
/>
```

---

## Best Practices

### 1. Consistency
- Use the same icon library throughout a feature/page
- Don't mix hugeicons and fontawesome for similar concepts
- Maintain consistent sizing within the same context

### 2. Accessibility
- Always provide accessible labels for icon-only buttons:
  ```tsx
  <Button variant="ghost" size="icon" aria-label="Settings">
    <Settings01 className="h-5 w-5" />
  </Button>
  ```

- Use `<span className="sr-only">` for screen readers:
  ```tsx
  <Button variant="ghost" size="icon">
    <Settings01 className="h-5 w-5" />
    <span className="sr-only">Open settings</span>
  </Button>
  ```

### 3. Performance
- Import only the icons you need (tree-shaking):
  ```tsx
  // Good
  import { Home01, User01 } from '@hugeicons/react'

  // Avoid
  import * as HugeIcons from '@hugeicons/react'
  ```

### 4. Semantic Usage
- Use appropriate icons for actions:
  - Delete: Trash/Delete icon
  - Edit: Pencil/Edit icon
  - View: Eye icon
  - Download: Download arrow
- Maintain icon semantics across the app

### 5. Spacing
- Use consistent spacing with Tailwind classes:
  ```tsx
  // Icon before text
  <Icon className="mr-2 h-4 w-4" />

  // Icon after text
  <Icon className="ml-2 h-4 w-4" />

  // Icon above text (flex column)
  <Icon className="mb-2 h-6 w-6" />
  ```

---

## Library-Specific Tips

### hugeicons
- Naming convention: `IconName01`, `IconName02` for variants
- Often includes outlined and filled versions
- Check for numbered variants (01, 02, 03) for style options

### fontawesome light
- Pro license required for light icons
- Use `@fortawesome/free-solid-svg-icons` for free versions
- Prefix: `fa` + camelCase name (e.g., `faHome`, `faUserCircle`)

### lucide-icons
- PascalCase naming (e.g., `Home`, `UserCircle`)
- Minimal, consistent stroke width
- Great for clean, modern designs

---

## Quick Reference: Priority Flow

```
Need an icon?
    ↓
Search hugeicons
    ↓
Found? → Use it ✓
    ↓
Not found?
    ↓
Search fontawesome light
    ↓
Found? → Use it ✓
    ↓
Not found?
    ↓
Search lucide-icons
    ↓
Found? → Use it ✓
    ↓
Not found?
    ↓
Use closest semantic match from any library
```

---

## Example: Dashboard Page

```tsx
import {
  Dashboard01,
  ChartLineData01,
  User01,
  ShoppingCart01
} from '@hugeicons/react'
import { AwesomeCard } from '@/components/ui/awesome-card'

export function DashboardPage() {
  return (
    <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-4">
      <AwesomeCard
        icon={<ChartLineData01 className="h-8 w-8" />}
        title="Total Revenue"
        value="$124,500"
      />
      <AwesomeCard
        icon={<User01 className="h-8 w-8" />}
        title="Active Users"
        value="1,234"
      />
      <AwesomeCard
        icon={<ShoppingCart01 className="h-8 w-8" />}
        title="Orders"
        value="456"
      />
      <AwesomeCard
        icon={<Dashboard01 className="h-8 w-8" />}
        title="Conversion Rate"
        value="3.2%"
      />
    </div>
  )
}
```

All icons from hugeicons (first choice) ✓
