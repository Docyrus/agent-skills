# Dashboard Recipes

End-to-end patterns wiring the mandatory stack — **AwesomeStats** (KPI tiles) + **AwesomeCard** (panels) + **Bklit** (charts) — onto Docyrus data. For the data layer (collections, `.list()`, `calculations`, `pivot`, `formulas`, tenant formatting) see the `docyrus-app-dev-react` skill; for chart composition/theming see the `bklit-ui` skill.

## Page skeleton

```tsx
import { AwesomeStats } from '@/components/docyrus/awesome-stats';
import {
  AwesomeCard, AwesomeCardHeader, AwesomeCardTitle, AwesomeCardIcon, AwesomeCardBody,
} from '@/components/docyrus/awesome-card';
import {
  AreaChart, Area, BarChart, Bar, Grid, XAxis, BarXAxis, ChartTooltip, chartCssVars,
} from '@bklitui/ui/charts';

function Dashboard() {
  const stats = useStatItems();          // AwesomeStatItem[] from Docyrus data
  const trend = useRevenueTrend();        // [{ month, revenue }]
  const byRegion = useRevenueByRegion();  // [{ region, revenue }]

  return (
    <div className="flex flex-col gap-4 p-6">
      {/* 1 — KPI strip: AwesomeStats */}
      <AwesomeStats
        items={stats}
        layout={{ type: 'flex', behavior: 'scroll', cardWidth: 280, gap: '0.75rem' }}
        cardVariant="awesome"
      />

      {/* 2 — Chart panels: AwesomeCard wrapping Bklit */}
      <div className="grid grid-cols-1 gap-4 lg:grid-cols-2">
        <AwesomeCard patternStyle="stripes">
          <AwesomeCardHeader>
            <AwesomeCardTitle>Revenue Trend</AwesomeCardTitle>
            <AwesomeCardIcon><DocyrusIcon icon="fas chart-area" /></AwesomeCardIcon>
          </AwesomeCardHeader>
          <AwesomeCardBody>
            <AreaChart data={trend} xDataKey="month">
              <Grid horizontal />
              <Area dataKey="revenue" stroke={chartCssVars.linePrimary} />
              <XAxis /><ChartTooltip />
            </AreaChart>
          </AwesomeCardBody>
        </AwesomeCard>

        <AwesomeCard patternStyle="dots">
          <AwesomeCardHeader>
            <AwesomeCardTitle>Revenue by Region</AwesomeCardTitle>
          </AwesomeCardHeader>
          <AwesomeCardBody>
            {/* Bar charts use BarXAxis, not XAxis */}
            <BarChart data={byRegion} xDataKey="region">
              <Grid horizontal />
              <Bar dataKey="revenue" />
              <BarXAxis /><ChartTooltip />
            </BarChart>
          </AwesomeCardBody>
        </AwesomeCard>
      </div>

      {/* 3 — Tables/detail below (see docyrus-data-grid-page-design) */}
    </div>
  );
}
```

## Shaping Docyrus data into stat items

Compute metrics with collection queries, then map into `AwesomeStatItem[]`. Use `id` for `count`, real slugs for `sum`/`avg`/etc.

```tsx
function useStatItems(): AwesomeStatItem[] {
  const { list } = useBaseInvoiceCollection();
  const { data } = useQuery({
    queryKey: ['invoices', 'kpi'],
    queryFn: () => list({
      columns: ['gross_total', 'status', 'currency', 'report_date'],
      calculations: [{ key: 'gross_total', type: 'sum' }, { key: 'id', type: 'count' }],
      limit: 1000,
    }),
  });

  return useMemo(() => buildKpiItems(data), [data]);
  // buildKpiItems → [{ id, title, value, format: { style:'currency', currency }, comparison, miniChart }]
}
```

Drive `format.currency` / `format.locale` and any date formatting from tenant preferences (`@docyrus/app-utils` `numberUtils` / `dateUtils`) — don't hardcode.

## Layout cheat-sheet

| Need | Choice |
|------|--------|
| Many KPIs, horizontal scroll | `AwesomeStats` flex `scroll` |
| Fixed KPI board | `AwesomeStats` grid `columns: 4` |
| One big metric + trend | `AwesomeCard` + `AwesomeCardValue` + `AwesomeCardTrend` (mini-chart via AwesomeStats `miniChart` if part of a set) |
| Time trend | `AwesomeCard` → Bklit `area-chart` / `line-chart` |
| Category comparison | `AwesomeCard` → Bklit `bar-chart` |
| Part-to-whole | `AwesomeCard` → Bklit `pie-chart` / `ring-chart` |
| Single dial KPI | `AwesomeCard` → Bklit `gauge-chart` / `ring-chart` |
| User-reorderable board | `AwesomeStats sortable` + persist order to `UserAppConfig` |

## Install checklist

```bash
npx shadcn@latest info --json                          # verify aliases + @bklit registry
pnpm dlx @docyrus/cli add @docyrus/ui-awesome-card
pnpm dlx @docyrus/cli add @docyrus/ui-awesome-stats
npx shadcn@latest add @bklit/area-chart @bklit/bar-chart    # + any other chart slugs used
```

## Anti-patterns (do not do on dashboards)

- ❌ `import { … } from 'recharts'` or `@/components/ui/chart` → use Bklit.
- ❌ Raw `<Card>` panels → use `AwesomeCard`.
- ❌ Hand-built KPI tile grids → use `AwesomeStats`.
- ❌ Hardcoded currency/locale/separators → use `format` + `@docyrus/app-utils`.
