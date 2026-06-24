---
name: docyrus-dashboard-design
description: Build Docyrus dashboards, analytics pages, KPI/metric views, and reporting screens with a fixed, opinionated component stack — Bklit UI for ALL charts/data-viz, AwesomeCard for ALL dashboard cards/panels, and AwesomeStats for ALL stat/metric/KPI cards. Use when creating or refactoring a dashboard, analytics view, reporting page, KPI grid, metric strip, overview/home screen, or any data-visualization layout in a Docyrus-backed React app. Triggers on "dashboard", "analytics page", "stat cards", "KPI cards", "metric cards", "charts", "data visualization", "reporting screen", "overview page", AwesomeCard, AwesomeStats, bklit, or chart composition. This skill OVERRIDES the default "Recharts / shadcn chart" and generic-card guidance from docyrus-app-dev-react.
---

# Docyrus Dashboard Design

Build dashboards, analytics, and reporting UIs for Docyrus-backed React apps with one **fixed component stack**. This skill is intentionally opinionated: when a dashboard surface is in scope, these three choices are **mandatory**, not suggestions.

| Surface | MANDATORY component | Library | Install |
|---------|---------------------|---------|---------|
| **Every chart / data-viz** | Bklit charts (`LineChart`, `BarChart`, `AreaChart`, `PieChart`, `RingChart`, `GaugeChart`, …) | Bklit UI (`@bklit` registry) | `npx shadcn@latest add @bklit/<slug>` |
| **Every dashboard card / panel** | `AwesomeCard` (+ `AwesomeCardHeader` / `Body` / `Value` / `Trend`) | docyrus-ui | `pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card` |
| **Every stat / KPI / metric card** | `AwesomeStats` (data-driven metric cards) | docyrus-ui | `pnpm dlx @docyrus/cli add @docyrus/ui-awesome-stats` |

> This **overrides** the default guidance in `docyrus-app-dev-react` ("Prefer Recharts / shadcn chart", generic `Card`). On dashboard/analytics/reporting work, do not reach for Recharts, the shadcn `chart` primitive, raw `<Card>`, or hand-rolled stat tiles. Use the stack above.

## The One Rule

**Do not invent dashboard primitives.** Charts come from Bklit, cards come from AwesomeCard, stat tiles come from AwesomeStats. If a need seems unmet, read the component's docs first — it is almost always already supported via props.

- ❌ `import { BarChart } from 'recharts'` → ✅ `npx shadcn@latest add @bklit/bar-chart` then `import { BarChart } from '@bklitui/ui/charts'`
- ❌ `import { ChartContainer } from '@/components/ui/chart'` (shadcn) → ✅ Bklit chart composition
- ❌ `<Card><CardHeader>…</Card>` for a dashboard panel → ✅ `<AwesomeCard><AwesomeCardHeader>…</AwesomeCard>`
- ❌ hand-built `<div>` grids of "value + label + arrow" KPI tiles → ✅ `<AwesomeStats items={…} layout={…} />`

## Decision Guide — which one?

| You are building… | Use |
|-------------------|-----|
| A grid/strip/scroll of KPI numbers (revenue, count, %, with optional trend + mini-chart) | **`AwesomeStats`** — it renders the whole set from a data array |
| A single titled panel holding a chart, table, list, or custom content | **`AwesomeCard`** as the wrapper, chart inside from **Bklit** |
| A single headline metric panel (one big number + trend), not a set | **`AwesomeCard`** with `AwesomeCardValue` + `AwesomeCardTrend` |
| Any visualization of series/parts/flow/geo/dial | **Bklit** chart (pick the slug from the catalog) |

Rule of thumb: **3+ comparable metrics → `AwesomeStats`. One metric or mixed content → `AwesomeCard`. Anything plotted → Bklit.**

## Workflow

1. **Detect the dashboard surface.** Dashboard / analytics / overview / KPI / reporting page → this skill applies.
2. **Check what's installed**, then install the missing pieces:
   ```bash
   npx shadcn@latest info --json                          # framework, aliases, registries
   pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card     # if not present
   pnpm dlx @docyrus/cli add @docyrus/ui-awesome-stats    # if not present
   npx shadcn@latest add @bklit/<slug>                     # per chart you use
   ```
   Confirm the `@bklit` registry is configured in `components.json` before adding charts (see the **bklit-ui** skill).
3. **Lay out the page** top-down: KPI strip (`AwesomeStats`) → chart panels (`AwesomeCard` wrapping Bklit charts) → tables/detail.
4. **Feed real data** via Docyrus collections/queries (use the `docyrus-app-dev-react` skill for `.list()` + `calculations`/`pivot`/`formulas`). Shape it into `AwesomeStats` items and Bklit `data` arrays.
5. **Theme & compose correctly** — Bklit composition + `chartCssVars`; AwesomeStats `format` for tenant-aware number/currency formatting.
6. **Verify** typecheck + a quick visual check.

## Charts — Bklit UI (mandatory for all data-viz)

Charts are installed as source from the `@bklit` shadcn registry and **composed** (root → `Grid` → series → axes → `ChartTooltip`).

```tsx
import { LineChart, Line, Grid, XAxis, ChartTooltip, chartCssVars } from '@bklitui/ui/charts';

<LineChart data={data} xDataKey="date">
  <Grid horizontal />
  <Line dataKey="users" stroke={chartCssVars.linePrimary} />
  <XAxis />
  <ChartTooltip />
</LineChart>
```

**For anything beyond a trivial chart — composition rules, theming tokens, animation, tooltips, the full chart catalog (area, bar, line, live-line, composed, scatter, candlestick, pie, ring, radar, gauge, funnel, sankey, choropleth), and install/registry setup — defer to the dedicated `bklit-ui` skill.** Read its `SKILL.md` and `rules/*.md` rather than guessing chart props. Each chart also has docs at `https://ui.bklit.com/docs/components/<slug>`.

Pick the chart by the story the data tells:

| Story | Chart slug |
|-------|-----------|
| Trend over time | `line-chart`, `area-chart` |
| Category comparison | `bar-chart` |
| Streaming/real-time | `live-line-chart` |
| Mixed bar+line | `composed-chart` |
| Part-to-whole | `pie-chart`, `ring-chart` |
| Single-value dial/KPI gauge | `gauge-chart`, `ring-chart` |
| Correlation/distribution | `scatter-chart` |
| Stage conversion | `funnel-chart` |
| Flow between nodes | `sankey-chart` |
| Multi-axis profile | `radar-chart` |
| Geo by value | `choropleth-chart` |

Gotchas confirmed from the live Bklit docs: import everything from `@bklitui/ui/charts`; **bar charts use `BarXAxis`** (line/area use `XAxis`); root props include `data`, `xDataKey`, `margin`, `animationDuration`, `status`. Always confirm a chart's exact sub-components on its doc page before composing.

> **Mini-charts inside stat cards are different:** `AwesomeStats` has its own built-in `miniChart` (sparkline/bar/area) — see below. Use Bklit for the page's primary, full-size charts.

## Cards — AwesomeCard (mandatory for all dashboard panels)

A card with a hatched-stripe header and an inset content area. Wrap **every** dashboard panel in it. Details: `references/awesome-card.md`. Import: `@docyrus/ui/components/awesome-card` (package) or `@/components/docyrus/awesome-card` (CLI-installed app source) — examples below use the app path.

```tsx
import {
  AwesomeCard, AwesomeCardHeader, AwesomeCardTitle, AwesomeCardIcon,
  AwesomeCardBody, AwesomeCardValue, AwesomeCardTrend,
} from '@/components/docyrus/awesome-card';

// Headline metric panel
<AwesomeCard patternStyle="stripes">
  <AwesomeCardHeader>
    <AwesomeCardTitle>Total Revenue</AwesomeCardTitle>
    <AwesomeCardIcon><Wallet className="size-4" /></AwesomeCardIcon>
  </AwesomeCardHeader>
  <AwesomeCardBody>
    <AwesomeCardValue>{formatCurrency(total, 'USD')}</AwesomeCardValue>
    <AwesomeCardTrend positive>+12.4% vs last month</AwesomeCardTrend>
  </AwesomeCardBody>
</AwesomeCard>

// Chart panel — AwesomeCard wraps a Bklit chart
<AwesomeCard patternStyle="dots">
  <AwesomeCardHeader>
    <AwesomeCardTitle>Revenue Trend</AwesomeCardTitle>
  </AwesomeCardHeader>
  <AwesomeCardBody>
    <AreaChart data={monthly} xDataKey="month"> … </AreaChart>
  </AwesomeCardBody>
</AwesomeCard>
```

Key props: `patternStyle` (`'stripes' | 'dots' | …`), `collapsible`, `expandable` (`expandTrigger`, `expandDirection`, `expandedWidth/Height`). Full prop list and slot reference in `references/awesome-card.md`.

## Stat cards — AwesomeStats (mandatory for all KPI/metric tiles)

Data-driven metric cards: pass an `items` array and a `layout`. Renders grid / flex (wrap or scroll) / tabs, with comparison deltas, built-in mini-charts, `Intl.NumberFormat` value formatting, drag-and-drop reordering, and per-card menus. Details: `references/awesome-stats.md`.

```tsx
import { AwesomeStats } from '@/components/docyrus/awesome-stats';
import type { AwesomeStatItem } from '@/components/docyrus/awesome-stats';

const items: AwesomeStatItem[] = [
  {
    id: 'revenue',
    eyebrow: 'USD',
    title: 'Gross Total',
    subtitle: '128 reports',
    icon: 'fal sack-dollar',            // DocyrusIcon string or ReactNode
    color: 'emerald',
    value: 184250,
    format: { style: 'currency', currency: 'USD', notation: 'compact', maximumFractionDigits: 1 },
    comparison: { previousValue: 164000, period: 'last-month', positiveDirection: 'up' },
    miniChart: { type: 'area', data: [12, 18, 14, 22, 19, 26], position: 'right' },
  },
  // …more metrics
];

<AwesomeStats
  items={items}
  layout={{ type: 'flex', behavior: 'scroll', cardWidth: 280, gap: '0.75rem' }}
  cardVariant="awesome"   // 'awesome' = AwesomeCard styling, 'default' = plain card
/>
```

Layouts: `{ type: 'grid', columns, maxCardWidth?, gap? }` · `{ type: 'flex', behavior: 'wrap' | 'scroll', cardWidth, gap? }` · `{ type: 'tabs', defaultTabId? }`. Full item/layout/comparison/format/menu reference in `references/awesome-stats.md`.

## Critical Rules

1. **Charts = Bklit, always.** No Recharts, no shadcn `chart`, no hand-rolled SVG on dashboards.
2. **Dashboard panels = AwesomeCard, always.** No raw `<Card>` for dashboard sections.
3. **KPI/metric tiles = AwesomeStats, always.** Don't hand-build value+trend grids.
4. **Install, don't reimplement.** Add from the registries; compose with props.
5. **Read the docs before guessing props.** Bklit → `bklit-ui` skill + `ui.bklit.com/docs`. AwesomeCard/AwesomeStats → the reference files here + `https://alpha-ui.docy.app/docs/web/components/<name>/llms.txt`.
6. **Tenant-aware formatting.** Use `AwesomeStats` `format` (Intl) and `@docyrus/app-utils` `numberUtils`/`dateUtils` — never hardcode locale, currency, or separators when tenant preferences should drive them. (See `docyrus-app-dev-react`.)
7. **Real data via Docyrus collections.** Compute metrics with `.list()` + `calculations` / `pivot` / `formulas`; shape into stat items and chart `data`. (See `docyrus-app-dev-react`.)
8. **Icons** follow the project order: hugeicons → fontawesome light → lucide.

## Relationship to other skills

- **`bklit-ui`** — the authoritative chart skill. Defer to it for all chart composition, theming, animation, tooltips, and the registry setup. This skill only mandates *that* you use Bklit and points you there.
- **`docyrus-app-dev-react`** — app architecture, auth, collections, queries, formatting. Use it for the data layer that feeds these dashboards. This skill overrides only its chart/card/stat component choices for dashboard surfaces.
- **`docyrus-data-grid-page-design`** — for the table/list parts of a screen (a dashboard often pairs a KPI strip + charts on top with a grid below).

## References

- `references/awesome-card.md` — full AwesomeCard prop & slot reference, patterns, collapsible/expandable usage.
- `references/awesome-stats.md` — full AwesomeStats item/layout/comparison/format/miniChart/menu reference and recipes.
- `references/dashboard-recipes.md` — end-to-end page layouts wiring all three together with Docyrus data.
- the **bklit-ui** skill — charts (composition, theming, animation, catalog).
