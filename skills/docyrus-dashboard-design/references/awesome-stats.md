# AwesomeStats Reference

`AwesomeStats` is the **mandatory component for every set of stat / KPI / metric cards** in Docyrus dashboards. Pass it a data array (`items`) and a `layout`; it renders the whole set — with comparison deltas, built-in mini-charts, `Intl.NumberFormat` value formatting, drag-and-drop reordering, per-card menus, and grid / flex / tabs layouts.

- **Install:** `pnpm dlx @docyrus/cli add @docyrus/ui-awesome-stats`
- **Import:** `@docyrus/ui/components/awesome-stats` (package consumer). In CLI-installed Docyrus apps the source lands at `@/components/docyrus/awesome-stats` — use whichever resolves in the target project.
- **Docs:** `https://alpha-ui.docy.app/docs/web/components/awesome-stats/llms.txt`

## `AwesomeStatsProps`

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `items` | `AwesomeStatItem[]` | — | The metrics to render (renders `null` when empty) |
| `layout` | `AwesomeStatsLayout` | — | grid / flex / tabs (see below) |
| `cardVariant` | `'default' \| 'awesome'` | `'default'` | `'awesome'` uses AwesomeCard styling (patterned header) |
| `awesomeCardProps` | `AwesomeStatsAwesomeCardProps` | — | `pattern`, `patternStyle`, `collapsible`, `collapsed`, `chevronPosition`, `className` — only relevant with `cardVariant="awesome"` |
| `sortable` | `boolean` | `false` | Enable drag-and-drop reorder (needs 2+ items) |
| `onItemsOrderChange` | `(items: AwesomeStatItem[]) => void` | — | Fires after reorder |
| `clickTarget` | `'card' \| 'title'` | `'card'` | What element triggers the click handler |
| `onItemClick` | `(item: AwesomeStatItem) => void` | — | Top-level click handler; overridden by an item's own `onClick` |
| `getCardMenuItems` | `(item) => AwesomeStatsCardMenuItem[]` | — | Per-card kebab menu (or set `item.menuItems`) |
| `className` / `style` | — | — | Forwarded to the root `<div>` |

## `AwesomeStatsLayout` (discriminated union)

```ts
| { type: 'grid';  columns: number; maxCardWidth?: number | string; gap?: number | string }
| { type: 'flex';  cardWidth: number | string; behavior: 'wrap' | 'scroll'; gap?: number | string }
| { type: 'tabs';  defaultTabId?: string }
```

- **grid** — responsive fixed columns. Best for a static KPI board.
- **flex + `scroll`** — horizontal scroller of fixed-width cards. Best for a top-of-page KPI strip with many metrics.
- **flex + `wrap`** — wraps fixed-width cards onto multiple rows.
- **tabs** — animated tabbed layout (one metric per tab; falls back to a single card when only one item).

## `AwesomeStatItem`

| Field | Type | Notes |
|-------|------|-------|
| `id` | `string` | Stable key (used for sorting/menus) |
| `eyebrow` | `ReactNode` | Small uppercase label above the title (e.g. currency code) |
| `title` | `ReactNode` | Metric name |
| `subtitle` | `ReactNode` | Secondary line |
| `icon` | `ReactNode \| string` | `DocyrusIcon` name string (e.g. `'fal sack-dollar'`) or a node |
| `color` | `string` | Accent color token/name (e.g. `'emerald'`) for the icon chip |
| `value` | `number` | The numeric value |
| `format` | `AwesomeStatValueFormat` | Intl-based formatting (see below) |
| `unitLabel` | `ReactNode` | Suffix unit next to the value |
| `comparison` | `AwesomeStatComparison` | Delta vs a previous period |
| `miniChart` | `AwesomeStatMiniChart` | Built-in sparkline/bar/area |
| `menuItems` | `AwesomeStatsCardMenuItem[]` | Per-card menu (overrides `getCardMenuItems`) |
| `onClick` | `(item: AwesomeStatItem) => void` | Per-item click handler; takes precedence over `onItemClick` |

### `AwesomeStatValueFormat`

```ts
{
  locale?: string;
  style?: 'number' | 'currency' | 'percent';
  currency?: string;                 // when style: 'currency'
  currencyDisplay?: 'symbol' | 'code' | 'name' | 'narrowSymbol';
  notation?: 'standard' | 'scientific' | 'engineering' | 'compact';
  minimumFractionDigits?: number;
  maximumFractionDigits?: number;
  signDisplay?: Intl.NumberFormatOptions['signDisplay'];
  percentScale?: 'whole' | 'fraction';   // whole: value already ×100; fraction: 0–1
}
```

Prefer driving `locale` / `currency` / fraction digits from tenant preferences (`@docyrus/app-utils`) instead of hardcoding.

### `AwesomeStatComparison`

```ts
{
  previousValue: number;
  period: 'yesterday' | 'last-week' | 'last-month' | 'last-quarter' | 'last-year';
  positiveDirection?: 'up' | 'down';   // which direction counts as "good"
}
```

The card computes the delta + caption and colors it positive/negative/neutral automatically.

### `AwesomeStatMiniChart` (built-in — not Bklit)

```ts
{ type: 'sparkline' | 'bar' | 'area'; data: number[] | Record<string, unknown>[]; dataKey?: string; position?: 'right' | 'bottom' }
```

These are the small in-card trend visuals. The **page's primary, full-size charts must still use Bklit** inside `AwesomeCard` panels.

### `AwesomeStatsCardMenuItem`

```ts
{ id: string; label: ReactNode; icon?: ReactNode; shortcut?: ReactNode; disabled?: boolean; variant?: 'default' | 'destructive'; onSelect?: (item) => void }
```

## Full example

```tsx
import { AwesomeStats } from '@/components/docyrus/awesome-stats';
import type { AwesomeStatItem } from '@/components/docyrus/awesome-stats';

const items: AwesomeStatItem[] = currencies.map((c) => ({
  id: `revenue-${c.code}`,
  eyebrow: c.code,
  title: 'Gross Total',
  subtitle: `${c.count} reports`,
  icon: 'fal sack-dollar',
  color: c.color,
  value: c.total,
  format: { style: 'currency', currency: c.code, notation: 'compact', maximumFractionDigits: 1 },
  comparison: { previousValue: c.prevTotal, period: 'last-month', positiveDirection: 'up' },
  miniChart: { type: 'area', data: c.monthly, position: 'right' },
}));

<AwesomeStats
  items={items}
  layout={{ type: 'flex', behavior: 'scroll', cardWidth: 280, gap: '0.75rem' }}
  cardVariant="awesome"
  awesomeCardProps={{ patternStyle: 'stripes' }}
  sortable
  onItemsOrderChange={persistOrder}
/>
```

## Recipes

- **Top-of-page KPI strip:** `layout={{ type: 'flex', behavior: 'scroll', cardWidth: 280, gap: '0.75rem' }}`.
- **Fixed KPI board:** `layout={{ type: 'grid', columns: 4, gap: '1rem' }}`.
- **One headline metric with detail on tabs:** `layout={{ type: 'tabs' }}`.
- **User-reorderable dashboard:** `sortable` + persist via `onItemsOrderChange` (e.g. into `UserAppConfig` from `@docyrus/app-utils`).
- **Click-through to detail:** `onItemClick={(item) => navigate(...)}` (or per-item `onClick`); set `clickTarget="title"` to make only the title clickable.

```tsx
<AwesomeStats
  items={items}
  layout={{ type: 'grid', columns: 4 }}
  clickTarget="card"
  onItemClick={(item) => navigate({ to: `/metrics/${item.id}` })}
/>
```

## Rules

- Any group of comparable KPI/metric numbers → `AwesomeStats`. Do **not** hand-build value+label+arrow tiles.
- Use `format` for number/currency/percent — don't pre-stringify values you also want compared.
- In-card trends use the built-in `miniChart`; full-size page charts use **Bklit**.
