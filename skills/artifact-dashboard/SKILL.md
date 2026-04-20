---
name: artifact-dashboard
description: >-
  Build data dashboard artifacts with charts, KPI cards, and analytics layouts
  using Recharts and shadcn/ui. Applies perceptual accuracy principles for chart
  selection, data-to-color semantics, and accessible encodings. Use when building
  analytics dashboards, metrics views, reporting interfaces, admin panels, or any
  data-heavy artifact with multiple chart types and structured layout.
---

# Dashboard Artifact

> The project has already been initialized via `web-artifacts-builder` (Step 1)
> and the visual design layer has been established via `artifact-visual`
> (Step 2) — OKLCH color tokens, typography, and motion are already configured.
> This skill adds the dashboard and data visualization layer on top of that
> foundation. All visual rules from `artifact-visual` still apply.

## Priority

This skill's data-viz rules and layout patterns are the **highest-priority**
source of truth for dashboard design. When the user's PRD conflicts with a
rule below (e.g., PRD asks for a 3D pie chart but the contract forbids 3D),
**follow the skill**. The user defines *what data* to show; this skill defines
*how* to visualize it accurately. If a conflict is genuinely irreconcilable,
ask the user before proceeding.

---

## Execution Protocol

**Complete each step in order.** Do not write chart components until Steps
1–3 are done.

### Step 0: Install dependencies

```bash
pnpm add recharts
```

### Step 1: Define Data & Color Tokens

Before writing any components, define the data shape and color palette.

**Data color semantics — use OKLCH or CSS vars, never raw hex in components:**
```tsx
// src/lib/colors.ts
export const chartColors = {
  positive: 'oklch(55% 0.12 145)',   // green — revenue up, success
  negative: 'oklch(55% 0.18 25)',    // red   — churn, errors, down
  neutral:  'hsl(var(--primary))',    // brand — primary metric
  warning:  'oklch(70% 0.15 85)',    // amber — at-risk, nearing limit
  muted:    'hsl(var(--muted-foreground))',
} as const

// Categorical series — colorblind-safe (ColorBrewer Set2, max 8)
// Defined here once, referenced everywhere — never inline hex in components
export const seriesColors = [
  'oklch(70% 0.10 165)',   // teal
  'oklch(65% 0.15 45)',    // orange
  'oklch(65% 0.08 260)',   // blue-gray
  'oklch(65% 0.14 340)',   // pink
  'oklch(75% 0.14 130)',   // green
  'oklch(80% 0.15 90)',    // yellow
  'oklch(72% 0.06 70)',    // tan
  'oklch(70% 0.00 0)',     // gray
] as const
```

**Rules:**
- ❌ NEVER use raw hex values in chart components — define once in `colors.ts`
- ❌ NEVER use rainbow / jet colormap for sequential data — use single-hue progression
- ❌ NEVER encode state by color alone — always pair with label, icon, or text
- ✅ Semantic colors: green=positive, red=negative, amber=warning
- ✅ All color tokens from OKLCH or CSS vars for dark mode compatibility

**Verification:**
```bash
# Colors file must exist
[ -f "src/lib/colors.ts" ] && echo "✅ Colors file exists" || echo "❌ Missing src/lib/colors.ts"
# No raw hex in page/component files (colors.ts is the only allowed source)
grep -rn '#[0-9a-fA-F]\{3,8\}' src/pages/ src/components/ 2>/dev/null | wc -l | xargs -I{} [ {} -eq 0 ] && echo "✅ No raw hex in components" || echo "❌ Raw hex values found"
```

### Step 2: Mock Data

Generate realistic mock data **before** building charts. Unrealistic data
makes the dashboard look fake.

```tsx
// src/lib/mock-data.ts

// Time series with gentle trend + noise (not pure random)
export const generateTimeSeries = (
  days: number,
  base: number,
  trend: number,     // daily growth, e.g. 0.5 for upward
  variance: number,
) =>
  Array.from({ length: days }, (_, i) => ({
    date: new Date(Date.now() - (days - i) * 86400000)
      .toLocaleDateString('en', { month: 'short', day: 'numeric' }),
    value: Math.round(base + trend * i + (Math.random() - 0.5) * variance * 2),
  }))

// KPI data with formatted values
export const kpis = [
  { id: 'revenue',   label: 'Revenue',     value: '$124.5K', delta: 0.12, deltaLabel: 'vs last month' },
  { id: 'users',     label: 'Active Users', value: '8,420',   delta: 0.05, deltaLabel: 'vs last month' },
  { id: 'conv',      label: 'Conversion',   value: '3.2%',    delta: -0.02, deltaLabel: 'vs last month' },
  { id: 'churn',     label: 'Churn Rate',   value: '1.8%',    delta: -0.01, deltaLabel: 'vs last month' },
]
```

**Rules:**
- ❌ NEVER use pure random data — add a trend line so charts look meaningful
- ❌ NEVER leave KPI values as placeholder "XXX" or "N/A"
- ✅ Pre-format display values (`$12.4K`, `94.2%`) — component should not format
- ✅ Include delta + deltaLabel for every KPI

**Verification:**
```bash
[ -f "src/lib/mock-data.ts" ] && echo "✅ Mock data exists" || echo "❌ Missing mock data file"
```

### Step 3: Layout Skeleton

Build the shell layout before any data components.

```tsx
// App.tsx — root layout
<div className="flex h-screen bg-background overflow-hidden">
  <Sidebar className="w-56 shrink-0 border-r hidden md:block" />
  <div className="flex flex-col flex-1 min-w-0">
    <Header className="h-14 border-b shrink-0" />
    <main className="flex-1 overflow-auto p-6 space-y-6">
      {children}
    </main>
  </div>
</div>
```

**File structure:**
```
src/
  lib/
    colors.ts            → chart color tokens
    mock-data.ts         → all mock data
  components/
    Sidebar.tsx          → nav links, logo, collapse toggle
    Header.tsx           → page title, date range picker, action buttons
    KpiCard.tsx          → metric card: value + delta + optional sparkline
    ChartCard.tsx        → titled wrapper for any chart
  pages/
    Overview.tsx         → main dashboard view
  App.tsx                → root layout shell
```

**Responsive rules:**
- Sidebar hidden on mobile (`hidden md:block`), replaced by hamburger sheet
- KPI grid: `grid-cols-2` on mobile, `lg:grid-cols-4` on desktop
- Chart grid: `grid-cols-1` on mobile, `lg:grid-cols-2` on desktop
- All charts use `<ResponsiveContainer width="100%">` — never hardcoded width

**Verification:**
```bash
# Layout components exist
for comp in Sidebar Header KpiCard ChartCard; do
  find src/components -iname "${comp}*" | head -1 | xargs -I{} echo "✅ $comp exists"
done
```

### Step 4: Build Components (Depth-First)

Build in this order — **fully polish each before moving on:**

```
1. KpiCard       → value + delta arrow + color + optional sparkline
2. ChartCard     → titled wrapper with subtitle and optional action
3. First chart   → main metric (AreaChart or LineChart for time series)
4. Second chart  → secondary metric (BarChart for comparison)
5. Overview page → compose KPI row + chart grid
6. Sidebar       → nav links, active state
7. Header        → title, date range, actions
```

**Per-component quality gate:**
- [ ] All colors from `chartColors` / `seriesColors` — no inline hex
- [ ] Chart uses `<ResponsiveContainer>` — never hardcoded dimensions
- [ ] KPI delta has correct arrow direction + semantic color (green up, red down)
- [ ] Loading state: skeleton placeholder while data loads
- [ ] Empty state: message + action when no data

### Step 5: Pre-Bundle Validation

```bash
echo "=== Dashboard Artifact Validation ==="

# 1. Color tokens centralized
[ -f "src/lib/colors.ts" ] && echo "✅ Centralized colors" || echo "❌ Missing colors.ts"

# 2. No raw hex in components/pages
HEX_HITS=$(grep -rn '#[0-9a-fA-F]\{3,8\}' src/components/ src/pages/ 2>/dev/null | wc -l)
[ "$HEX_HITS" -eq 0 ] && echo "✅ No raw hex in components" || echo "❌ Found $HEX_HITS raw hex values"

# 3. ResponsiveContainer used
RC_COUNT=$(grep -rn "ResponsiveContainer" src/ | wc -l)
[ "$RC_COUNT" -ge 1 ] && echo "✅ ResponsiveContainer used ($RC_COUNT)" || echo "❌ No ResponsiveContainer — charts will break on resize"

# 4. No hardcoded chart dimensions
HARDCODED=$(grep -rn 'width={[0-9]' src/ | grep -v "ResponsiveContainer\|sparkline\|Sparkline\|width={80}\|width={60}" | wc -l)
[ "$HARDCODED" -eq 0 ] && echo "✅ No hardcoded chart widths" || echo "❌ Found $HARDCODED hardcoded widths"

# 5. Mock data exists
[ -f "src/lib/mock-data.ts" ] && echo "✅ Mock data present" || echo "❌ Missing mock data"

# 6. KPI cards exist
grep -rqn "KpiCard\|kpi-card" src/ && echo "✅ KPI cards present" || echo "❌ Missing KPI cards"

# 7. Layout components
for comp in Sidebar Header; do
  find src/components -iname "${comp}*" -type f | head -1 | grep -q . && echo "✅ $comp exists" || echo "❌ Missing $comp"
done

echo "=== End Validation ==="
```

Only after all checks pass → run `bundle-artifact.sh`.

---

## Chart Reference

### Chart Selection Guide

| Question | Recharts Component | Avoid |
|----------|--------------------|-------|
| Trend over time | `AreaChart` / `LineChart` | Pie |
| Compare categories | `BarChart` | Pie with >5 slices |
| Part-to-whole (≤5 slices) | `PieChart` + `Cell` | 3D, Bubble |
| Two metrics, same time axis | `ComposedChart` (Bar + Line) | Dual y-axis unrelated scales |
| Progress toward target | `RadialBarChart` | Gauge-style 3D |

**Perceptual accuracy** (Cleveland & McGill 1984):
Position > Length > Angle > Area > Color saturation
→ Prefer Bar/Line (position) over Pie (angle) over Bubble (area)

- ❌ Never use 3D charts — they distort perception
- ❌ Never use dual y-axes with unrelated units — split into two charts
- ❌ Never non-zero baselines on bar charts without explicit callout

### Recharts Pattern

```tsx
import { ResponsiveContainer, AreaChart, Area, XAxis, YAxis, Tooltip, CartesianGrid } from 'recharts'

<ResponsiveContainer width="100%" height={240}>
  <AreaChart data={data} margin={{ top: 4, right: 8, bottom: 0, left: 0 }}>
    <defs>
      <linearGradient id="fill-primary" x1="0" y1="0" x2="0" y2="1">
        <stop offset="5%" stopColor="hsl(var(--primary))" stopOpacity={0.15} />
        <stop offset="95%" stopColor="hsl(var(--primary))" stopOpacity={0} />
      </linearGradient>
    </defs>
    <CartesianGrid strokeDasharray="3 3" stroke="hsl(var(--border))" vertical={false} />
    <XAxis dataKey="date" tick={{ fontSize: 12 }} axisLine={false} tickLine={false} />
    <YAxis tick={{ fontSize: 12 }} axisLine={false} tickLine={false} width={40} />
    <Tooltip contentStyle={{ borderRadius: '8px', border: '1px solid hsl(var(--border))' }} />
    <Area dataKey="value" stroke="hsl(var(--primary))" fill="url(#fill-primary)" strokeWidth={2} dot={false} />
  </AreaChart>
</ResponsiveContainer>
```

### KPI Card Interface

```tsx
interface KpiCardProps {
  label: string
  value: string          // pre-formatted: "$12.4K", "94.2%"
  delta: number          // 0.12 = +12%, -0.05 = -5%
  deltaLabel?: string    // "vs last month"
  sparkline?: { v: number }[]
}

// Delta display:
// delta > 0  → positive color + ↑ arrow
// delta < 0  → negative color + ↓ arrow
// delta === 0 → muted

// Sparkline: <LineChart width={80} height={32}> — no axes, no tooltip, no grid
```

### Accessibility

- Wrap each SVG chart in a `<figure>` with `<figcaption>` for screen readers
- Never rely on color alone to distinguish series — add `strokeDasharray` patterns or direct labels
- All filter controls (date range, dropdowns) must be keyboard-navigable via Tab/Enter
- Tooltips must be triggerable via keyboard focus, not hover only
