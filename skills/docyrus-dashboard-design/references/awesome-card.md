# AwesomeCard Reference

`AwesomeCard` is the **mandatory wrapper for every dashboard card/panel** in Docyrus dashboards. It renders a card with a hatched/patterned header strip and an inset, bordered content area — ideal for metrics, charts, lists, and grouped content.

- **Install:** `pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card`
- **Import:** `@docyrus/ui/components/awesome-card` (package consumer). In CLI-installed Docyrus apps the source lands at `@/components/docyrus/awesome-card` — use whichever resolves in the target project.
- **Docs:** `https://alpha-ui.docy.app/docs/web/components/awesome-card/llms.txt`

## Exports

| Export | Role |
|--------|------|
| `AwesomeCard` | Root container (pattern background, optional collapse/expand) |
| `AwesomeCardHeader` | Header strip; holds title + icon; becomes the collapse toggle when `collapsible` |
| `AwesomeCardTitle` | Muted title text (brightens on hover) |
| `AwesomeCardIcon` | Right-aligned icon slot |
| `AwesomeCardBody` | Inset bordered content area (animates open/closed when collapsible) |
| `AwesomeCardExpandedBody` | Alternate content shown when the card is expanded (paired with `expandable`) |
| `AwesomeCardValue` | Large bold tabular-nums number for headline metrics |
| `AwesomeCardTrend` | Small delta/caption line; `positive` toggles green/red |
| type `ExpandDirection` | `'rt' | 'rb' | 'lt' | 'lb'` |
| type `PatternStyle` | re-exported from `@/lib/pattern-styles` |

## `AwesomeCard` props

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `pattern` | `boolean` | `true` | Toggle the patterned header background |
| `patternStyle` | `PatternStyle` | `'stripes'` | e.g. `'stripes'`, `'dots'` (from `@/lib/pattern-styles`) |
| `collapsible` | `boolean` | `false` | Header becomes a toggle; body animates open/closed |
| `collapsed` | `boolean` | `false` | Initial collapsed state (uncontrolled) |
| `chevronPosition` | `'left' \| 'right'` | `'left'` | Where the collapse chevron sits |
| `expandable` | `boolean` | `false` | Card can expand to an overlay (GSAP animated) |
| `expandTrigger` | `'hover' \| 'click'` | `'click'` | Click shows a maximize button; hover expands on hover |
| `expandDirection` | `ExpandDirection` | `'rb'` | Anchor corner for the expansion |
| `expandedWidth` | `number \| string` | `480` | Expanded overlay width |
| `expandedHeight` | `number \| string` | `320` | Expanded overlay height |
| `expanded` | `boolean` | — | Controlled expanded state |
| `onExpandedChange` | `(expanded: boolean) => void` | — | Controlled expansion callback |
| `expandSpeed` | `number` | `0.35` | Animation duration (s) |
| `className` | `string` | — | Extra classes on the root |
| `children` | `ReactNode` | — | Header + body slots |

Sub-components (`Header`, `Title`, `Icon`, `Body`, `ExpandedBody`, `Value`, `Trend`) take `className` + `children`; `AwesomeCardTrend` also takes `positive?: boolean` (omit for a neutral muted caption).

Honors `prefers-reduced-motion`; expanded cards close on outside click, `Escape`, or window resize.

## Patterns

### Headline metric panel

```tsx
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
```

### Chart panel (wraps a Bklit chart)

```tsx
<AwesomeCard patternStyle="dots">
  <AwesomeCardHeader>
    <AwesomeCardTitle>Revenue Trend</AwesomeCardTitle>
    <AwesomeCardIcon><DocyrusIcon icon="fas chart-area" /></AwesomeCardIcon>
  </AwesomeCardHeader>
  <AwesomeCardBody>
    {/* Bklit chart — never Recharts */}
    <AreaChart data={monthly} xDataKey="month">
      <Grid horizontal />
      <Area dataKey="revenue" />
      <XAxis />
      <ChartTooltip />
    </AreaChart>
  </AwesomeCardBody>
</AwesomeCard>
```

### Collapsible section

```tsx
<AwesomeCard collapsible chevronPosition="left" patternStyle="stripes">
  <AwesomeCardHeader>
    <AwesomeCardTitle>Breakdown by Region</AwesomeCardTitle>
  </AwesomeCardHeader>
  <AwesomeCardBody>{/* table or chart */}</AwesomeCardBody>
</AwesomeCard>
```

### Expandable card (compact preview → detail overlay)

```tsx
<AwesomeCard expandable expandTrigger="click" expandDirection="rb" expandedWidth={560} expandedHeight={400}>
  <AwesomeCardHeader>
    <AwesomeCardTitle>Pipeline</AwesomeCardTitle>
  </AwesomeCardHeader>
  <AwesomeCardBody>{/* compact summary */}</AwesomeCardBody>
  <AwesomeCardExpandedBody>{/* full detail / large Bklit chart */}</AwesomeCardExpandedBody>
</AwesomeCard>
```

## Rules

- Use `AwesomeCard` for **every** dashboard panel — do not fall back to the raw shadcn `<Card>` on dashboard surfaces.
- Charts inside the body must be **Bklit** charts.
- For a *set* of comparable KPI tiles, prefer `AwesomeStats` (which can itself render in AwesomeCard style via `cardVariant="awesome"`) over many hand-assembled `AwesomeCard` value panels.
