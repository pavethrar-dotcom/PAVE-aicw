# AICW PROJECT — MASTER PROMPT
## Build a Fully Interactive Sales & Profitability Dashboard in a Single HTML File

> **How to use this prompt:** paste everything from "ROLE" to the end of "ACCEPTANCE CHECKLIST" into your AI builder (Claude, GPT, Gemini, Cursor, v0, etc.). The dataset file to attach is `Sample_-_Superstore_3c8af909.csv` (9,994 rows × 21 columns). All numbers below are the *verified ground truth* from that file — the dashboard must reproduce them, so they also act as a QA baseline.

---

## 1. ROLE

You are a **Senior BI Developer + Data Visualization Designer** with 10+ years building executive dashboards in Power BI / Tableau, now shipping production-grade **single-file HTML dashboards** using Chart.js (or Apache ECharts) and vanilla JavaScript.

You think in **business metrics first, charts second**. Every visual must answer a decision a Sales Director or CFO would actually make. You do not produce decorative charts, and you never invent numbers that are not in the data.

---

## 2. OBJECTIVE

Build **"AICW Sales & Profitability Command Center"** — a fully interactive, self-contained HTML dashboard analyzing 4 years (Jan 2014 – Dec 2017) of US retail order-line data for the **AICW Project**.

The dashboard must let a business user answer, in under 60 seconds and with ≥3 clicks or fewer:

1. How much did we sell and earn, and is the business growing profitably?
2. **Where** (region / state / city) is performance strongest and weakest?
3. **What** (category / sub-category / product) makes money — and what destroys it?
4. **Who** (segment / customer) drives revenue versus margin?
5. **Why** is margin leaking — specifically, what is the discount-to-profit relationship?
6. What should we **do next week**? (ranked, quantified actions)

**Primary audience:** VP Sales, Regional Sales Managers, CFO / Finance Business Partner, Category Managers.
**Tone:** executive, dense-but-readable, decision-oriented. No fluff.

---

## 3. DATASET — SCHEMA, TYPES & SEMANTICS

Grain: **one row = one product line on one order** (order-line level, not order level). File is Latin-1 encoded (contains `0x85`/`0xA0` bytes) — decode with `latin-1` / `cp1252`.

| # | Column | Raw Type | Analytical Type | Semantic Definition | Notes / Cleaning Rule |
|---|--------|----------|-----------------|---------------------|-----------------------|
| 1 | Row ID | int | Identifier | Surrogate line key, 1–9994 | Unique; not used in any metric. Drop from UI. |
| 2 | Order ID | text | Identifier (dim) | Order number, e.g. `CA-2016-152156` | 5,009 distinct orders. Multiple lines share one Order ID. Embeds order year in the string (do NOT use for date logic). |
| 3 | Order Date | text `M/D/YYYY` | **Date** | Date order was placed | Parse with format `%m/%d/%Y` (US). Range **2014-01-03 → 2017-12-30**. |
| 4 | Ship Date | text `M/D/YYYY` | **Date** | Date order shipped | Always ≥ Order Date. Derived metric: `Lead Time (days)`. |
| 5 | Ship Mode | text | Dimension (4 values) | Fulfilment service level | `Standard Class` (5,968), `Second Class` (1,945), `First Class` (1,538), `Same Day` (543). |
| 6 | Customer ID | text | Identifier (dim) | Customer key `XX-#####` | 793 distinct. |
| 7 | Customer Name | text | Dimension | Person name | 793 distinct (1:1 with Customer ID). |
| 8 | Segment | text | Dimension (3 values) | Customer type | `Consumer` (5,191), `Corporate` (3,020), `Home Office` (1,783). |
| 9 | Country | text | Dimension | Country | **Constant: "United States"** — filter it out of the UI, it has zero variance. |
| 10 | City | text | Dimension (geography) | Ship-to city | 531 distinct. High cardinality → searchable dropdown. |
| 11 | State | text | Dimension (geography) | Ship-to state | 49 distinct (no DC). |
| 12 | Postal Code | int | Dimension (geo) | ZIP | 631 distinct. Keep as **string** (leading zeros) if shown; used as tooltip detail only. |
| 13 | Region | text | Dimension (4 values) | Sales region | `West`, `East`, `Central`, `South`. |
| 14 | Product ID | text | Identifier (dim) | SKU key `CAT-SUB-#####` | 1,862 distinct. |
| 15 | Category | text | Dimension (3 values) | Top product class | `Office Supplies`, `Furniture`, `Technology`. |
| 16 | Sub-Category | text | Dimension (17 values) | Product class | 17 distinct; the **primary profit-leak drill target**. |
| 17 | Product Name | text | Dimension | Full product description | **1,850 distinct** (the 1,862 SKU IDs map 1:many onto names); long strings, may contain commas inside quotes. |
| 18 | **Sales** | float | **Measure (additive)** | Gross revenue for the line, **net of discount** (USD) | Total **$2,297,200.86**. No zeros/negatives. |
| 19 | **Quantity** | int | **Measure (additive)** | Units sold on the line | Total **37,873**. Min 1. |
| 20 | **Discount** | float | **Measure (rate, non-additive)** | Discount rate applied, 0.00–0.80 | 5,196 lines (52%) discounted; overall mean **15.6%**; max **80%**. **Never sum — always average or weight by sales.** |
| 21 | **Profit** | float | **Measure (additive)** | Gross profit for the line (USD); negative = loss | Total **$286,397.02**. 1,871 lines (18.7%) are negative. |

### 3.1 Data quality audit (verified — no fixes required)
- **No nulls** in any column (0 / 9,994).
- **No duplicate rows**; `Row ID` is a clean unique key.
- **No non-positive** Sales, Quantity, or Profit-records with zero revenue.
- Discount is bounded **0–0.80**, mechanically applied in 5-point steps (0, 0.1, 0.15, 0.2, 0.3, 0.4, 0.45, 0.5, 0.6, 0.7, 0.8) — treat as a discrete band, not a continuous curve.
- `Country` is a single value → remove from filters.
- Dates parse with zero failures once `%m/%d/%Y` is forced (do **not** let a library guess day-first, or 3,000+ rows silently shift months).
- Lead time is 0–7 days, mean **3.96 days** — no impossible/negative values.

### 3.2 Derived fields you MUST compute at load time
| Derived field | Formula | Why it matters |
|---|---|---|
| `Order Year` | `year(Order Date)` | 2014, 2015, 2016, 2017 trend. |
| `Order Month` | `YYYY-MM` | Monthly trend line / seasonality. |
| `Quarter` | `YYYY-Qn` | Q4 = 38.2% of revenue. |
| `Profit Margin %` | `SUM(Profit) / SUM(Sales)` | **Always recompute as a ratio-of-sums** at the current filter grain. Never average row-level margins — it produces false double-digit margins. |
| `Lead Time (days)` | `Ship Date − Order Date` | Ops/CSAT proxy by Ship Mode. |
| `Discount Band` | `0%`, `1–10%`, `11–20%`, `21–30%`, `31–40%`, `41%+` | The core "why margin leaks" axis. |
| `Line Status` | `Profit ≥ 0 ? "Profitable" : "Loss"` | Loss concentration analysis. |
| `Avg Selling Price` | `Sales / Quantity` | Discount/price-mix sanity. |
| `Revenue per Order` | `SUM(Sales) / COUNT(DISTINCT Order ID)` | AOV = **$458.61**. |

---

## 4. KPI SPECIFICATION (headline cards)

Render **8 KPI cards** in a responsive grid (4 cols desktop / 2 tablet / 1 mobile). Each card: label, big value, small delta or context line, and a sparkline (last 8 quarters) where noted. **All cards must react live to every slicer.**

| # | KPI | Formula | Verified Value | Format | Sub-line |
|---|-----|---------|----------------|--------|----------|
| 1 | **Total Sales** | `SUM(Sales)` | **$2,297,201** | `$2.30M` + full value on hover | Sparkline by quarter; `+20.4% YoY` (2017 vs 2016) |
| 2 | **Total Profit** | `SUM(Profit)` | **$286,397** | `$286.4K` | Sparkline by quarter |
| 3 | **Profit Margin** | `SUM(Profit)/SUM(Sales)` | **12.47%** | `12.5%` | Benchmarked against **20% target** — show gap |
| 4 | **Total Orders** | `COUNT(DISTINCT Order ID)` | **5,009** | `5,009` | Sparkline of order count by quarter |
| 5 | **Avg Order Value** | `SUM(Sales)/COUNT(DISTINCT Order ID)` | **$458.61** | `$458.61` | With avg **2.0 lines/order** |
| 6 | **Units Sold** | `SUM(Quantity)` | **37,873** | `37.9K` | Profit per unit **$7.56** |
| 7 | **Avg Discount** | `AVG(Discount)` (line-weighted) | **15.6%** | `15.6%` | Also show **sales-weighted** discount |
| 8 | **Loss Exposure** | `SUM(Profit WHERE Profit<0)` | **−$156,131** | `−$156.1K` in red | `1,022 of 5,009 orders (20.4%)` lost money |

**Bonus strip (secondary metrics, smaller type):** distinct Customers **793** · Products **1,862** · States **49** · Avg Lead Time **3.96 days** · Revenue from discounted lines **52.6%**.

**Behaviour rules**
- Formatting: `$` with thousands separators; ¥/€ never; margin to 1 decimal; compact notation (K/M) for card values, exact values in tooltip.
- Delta chips: green ▲ / red ▼ vs **same period last year**; when comparison is unavailable (2014), show `—`.
- While filters are applied, show a **"Filtered · N lines"** badge and a **Reset all** control.

---

## 5. VISUALIZATIONS (8 required — each must be interactive)

Every chart: hover tooltip with at least 2 measures, click-to-filter (cross-filtering), legend toggle, and a `⤢` expand-to-modal view. Titles must state the insight, not just the field name (e.g. *"Tables lose $17.7K — the only sub-category with negative margin"*).

### V1 — Sales & Profit Trend with Margin Overlay *(combo: bar + line, dual axis)*
- **Fields:** `Order Month` (x), `SUM(Sales)` bars, `SUM(Profit)` bars (stacked or clustered toggle), `Profit Margin %` line on right axis.
- **Interactions:** Month/Quarter/Year grain toggle (segmented control); brush-drag on x-axis to zoom; click a month → cross-filter the whole page.
- **Purpose:** Show growth is real but margin is flat. **2014 $484K → 2017 $733K while margin stays 12–13%.** Annotate the Q4 spike band.
- **Annotation:** Shade Sep–Nov (37.5% of annual revenue) as "peak season".

### V2 — Category & Sub-Category Profitability (Waterfall + ranked bar) — **the centerpiece**
- **Left / top:** **Waterfall** of `SUM(Profit)` by Category → Sub-Category, sorted descending from $55.6K (Copiers) to −$17.7K (Tables).
- **Right / bottom:** Horizontal diverging bar chart of `Profit` per Sub-Category, colored green/red, with `Margin %` as a data label and `Avg Discount %` on hover.
- **Interactions:** Click a sub-category → filter all visuals; toggle metric between `Profit`, `Sales`, `Margin %`.
- **Must surface:** Tables **−$17,725 (−8.6% margin, 26.1% avg discount)**, Bookcases **−$3,473 (−3.0%)**, Supplies **−$1,189** are the only three losers. Copiers **+$55.6K (37.2%)**, Paper **+$34.1K (43.4%)**, Phones **+$44.5K** are the winners.

### V3 — Discount Impact: Margin Erosion Curve *(column + line, the "why")*
- **Fields:** `Discount Band` (x, ordered), `SUM(Sales)` columns, `Profit Margin %` line, `SUM(Profit)` as a secondary series or KPI annotation.
- **Verified curve (must match):**

| Band | Sales | Profit | Margin |
|---|---|---|---|
| 0% | $1,087,908 | $320,988 | **+29.5%** |
| 1–10% | $54,369 | $9,029 | +16.6% |
| 11–20% | $792,153 | $91,756 | +11.6% |
| 21–30% | $103,227 | −$10,369 | **−10.0%** |
| 31–40% | $130,911 | −$25,448 | −19.4% |
| 41%+ | $128,632 | −$99,559 | **−77.4%** |

- **Insight to state explicitly:** margin turns negative **at ~21% discount**; every dollar of discount above 20% is value-destructive. Full-price lines earn 29.5% vs a 12.5% blended margin. Discount↔Profit correlation **−0.22**.
- **Interaction:** hover any band → tooltip with line count and % of revenue; click band → filter to those lines.

### V4 — Geographic Performance (Region + State)
- **Primary:** 4-region card/bullet set with `Sales` bar, `Profit` bar, and `Margin %` bullet vs target — West **$725K / $108K / 14.9%**, East **$679K / $91.5K / 13.5%**, South **$392K / $46.7K / 11.9%**, Central **$501K / $39.7K / 7.9%**.
- **Secondary:** **US state choropleth-style heat map** (SVG, no external map service required — hand-drawn states or a tile-grid "hex map"). Color = `Profit Margin %`, diverging red→green centered at 0. Tooltip = Sales, Profit, Margin, Orders.
- **Must surface the paradox:** California is the **largest** state ($457.7K, 16.7% margin) while **Texas is the largest loss** (−$25.7K, −15.1% margin) on $170K of sales; Ohio −$17.0K (−21.7%), Pennsylvania −$15.6K (−13.4%), Illinois −$12.6K, North Carolina −$7.5K. Central region's low margin is a **discount problem, not a demand problem** (avg discount 24.0% vs West 10.9%).
- **Interaction:** click region → filters states and all downstream visuals; click state → drill to top cities + customers in a side drawer.

### V5 — Segment & Customer Mix *(donut + scatter)*
- **Donut:** `Sales` share by `Segment` — Consumer **50.6%**, Corporate **30.7%**, Home Office **18.7%**; inner label = blended margin per segment (Consumer 11.5%, Corporate 13.0%, Home Office 14.0%).
- **Scatter (bubble):** x = `Sales`, y = `Profit Margin %`, size = `Orders`, one bubble per **Customer** (top 100 by sales), colored by Segment. Quadrant lines at median sales and 0% margin — label quadrants "Stars", "Volume traps", "Niche profit", "Drag".
- **Must surface:** Home Office is the **smallest but most profitable** segment; Consumer is largest but thinnest margin. **Sean Miller (Home Office) is the #1 customer by revenue at $25,043 and still lost $1,981** — the single best teaching example of "revenue ≠ profit". Top-10 customers = only **6.7%** of revenue (low concentration).
- **Interaction:** brush the scatter to select customers → KPI cards recompute; click bubble → customer profile modal (orders, sales, profit, avg discount, category mix).

### V6 — Product Profit Leaders & Detractors *(ranked horizontal bar, top 10 / bottom 10 toggle)*
- **Fields:** `Product Name`, `SUM(Profit)`, `SUM(Sales)`, `AVG(Discount)`.
- **Must surface bottom:** Cubify CubeX 3D Printer Double Head **−$8,880 at a 53% avg discount**, Lexmark MX611dhe **−$4,590 (40%)**, Cubify Triple Head **−$3,840 (50%)**, Chromcraft Bull-Nose Conference Table **−$2,876**, Cisco TelePresence EX90 **−$1,811 (50%)**.
- **Insight:** the biggest losers are **high-ticket Technology/Furniture SKUs sold at 40–53% discount** — a pricing-governance failure, not a product-quality issue.
- **Interaction:** searchable product list; click → line-level detail table filtered to that SKU.

### V7 — Operations: Ship Mode vs Speed, Volume & Profit *(grouped bar + scatter)*
- **Grouped bars:** `Ship Mode` vs `Sales` and `Profit`; labels for avg **lead time**: Same Day **0.04d**, First Class **2.18d**, Second Class **3.24d**, Standard Class **5.01d**.
- **Scatter:** x = avg lead time, y = `Profit Margin %`, size = `Sales`.
- **Insight:** Standard Class carries **59%** of revenue at a healthy margin — premium shipping is not the profit problem; discounting is. Same Day has the **lowest** sales but acceptable margin, so it is a candidate for a paid-upgrade push.
- **Interaction:** toggle metric; click mode → filter.

### V8 — Drill-Through Detail Table *(the audit trail)*
- **Columns:** Order Date, Order ID, Customer, Segment, Region, State, City, Category, Sub-Category, Product, Quantity, Sales, Discount, Profit, Margin %, Lead Time.
- **Features:** sortable on every column, multi-column sort, free-text search, 15/50/100 rows per page, sticky header, conditional formatting (Profit red/green, Discount amber ≥20%, Margin red <0%), CSV export of the **current filtered view**.
- **Rule:** the table must always reflect every active slicer and chart cross-filter — it is the ground truth users check the cards against.

---

## 6. INTERACTIVE SLICERS & FILTERS (global control bar)

Sticky top bar, wrapping responsively. All filters are **AND-combined** across dimensions, **OR-within** a multi-select dimension.

| Control | Type | Options | Default | Behaviour |
|---|---|---|---|---|
| **Date Range** | Dual-handle range slider **+** preset chips | 2014-01-03 → 2017-12-30 | Full range | Presets: `YTD`, `Last 12M`, `2017`, `2016`, `2015`, `2014`, `All`. Slider drives V1 brush and vice-versa. |
| **Region** | Multi-select chips | West, East, Central, South | All | Chip count badge; click to isolate (alt-click to solo). |
| **Segment** | Multi-select dropdown w/ checkboxes | Consumer, Corporate, Home Office | All | |
| **Category** | Multi-select (cascades from Sub-Category) | Furniture, Office Supplies, Technology | All | Selecting a Category narrows Sub-Category options. |
| **Sub-Category** | Searchable multi-select (17) | all 17 | All | Must include "Select all / Clear". |
| **State** | Searchable multi-select (49) | all 49 | All | Cascades from Region. |
| **Ship Mode** | Segmented toggle | 4 modes | All | |
| **Discount Band** | Range slider on `Discount` (0–80%) | continuous | 0–80% | Shows selected band label; snaps to 5-pt steps. |
| **Profit Status** | Toggle | All / Profitable only / Loss only | All | Enables instant "show me the bleeding" view. |
| **Search** | Global text input | Product, Customer, Order ID | empty | Debounced 250 ms; highlights matches. |
| **Reset All** | Button | — | — | Clears every filter, restores defaults, re-renders all 8 visuals. |

**Cross-filtering contract**
- Clicking a mark in any chart applies that dimension value as a *transient* filter; it must appear as a removable chip (`Texas ✕`) in the active-filter ribbon.
- Self-filtering is disabled (a chart never filters itself out of existence).
- Empty-result state: show a friendly "No data for this combination" panel with a **Clear filters** button — never a blank white chart or `NaN`.
- Every filter change updates **all** KPIs, **all** charts, the insight panel, and the detail table within **≤150 ms** (pre-aggregate in memory; no re-parsing CSV on interaction).

---

## 7. INSIGHT PANEL ("What the data is telling you")

A right-hand or full-width card grid of **6 auto-generated, quantified insight bullets**. Each bullet = *finding → evidence → recommended action → expected impact*. These must be **computed from the current filter context**, not hard-coded, but the following six are the canonical full-dataset insights and must be present by default:

1. **Pricing is the margin ceiling.** Zero-discount lines earn **29.5%** margin; anything above **~21%** discount is loss-making; the 41%+ band loses **$99.6K**. *Action:* hard-floor approval at 20% discount, exec sign-off above 25%. *Impact:* recovering even half of the 41%+ band's loss adds **≈$50K profit** with no new revenue.
2. **Furniture is a revenue trap.** **$742K of sales (32% of total) yields only $18.5K profit (2.5% margin)**, dragged by Tables (−$17.7K) and Bookcases (−$3.5K) at 21–26% average discounts. *Action:* reprice/limit Tables, shift assortment mix toward Chairs (+$26.6K) and Furnishings.
3. **Central region buys a lot and earns little.** $501K sales but **7.9% margin** and a **24.0% average discount** — nearly double West's 10.9%. *Action:* region-specific discount governance; audit Central's deal desk. *Impact:* lifting Central to East's 13.5% margin is worth **≈$28K**.
4. **Texas, Ohio, Pennsylvania, Illinois are margin-negative at scale.** Combined **−$70.9K profit on $445K sales**. *Action:* state-level pricing review and stop-loss on loss-making SKUs in these states.
5. **20.4% of orders lose money — and it's the big-ticket orders.** **1,022 orders** are unprofitable; the worst SKUs are $8K–$23K printers/conference tables discounted 40–53%. *Action:* deal-desk review for any order above $5K with >25% discount.
6. **Growth is real but inefficient.** Sales grew **+20.4% in 2017** ($609K → $733K) yet margin stayed **~12–13%** — expansion is being bought with discount. *Action:* tie 2018 targets to **profitable revenue**, not gross sales; the fastest lever is discount discipline (25% of revenue sits in >20% discount bands and produces **−$135K**).
7. **Seasonality is extreme and plannable.** **Q4 = 38.2%** of revenue (Sep–Nov = 37.5%); January/February are the trough (50%/31% of an average month). *Action:* inventory + staffing ramp in August; use Jan–Feb for clearance of slow, low-margin SKUs — not additional discounting.

Format each insight as a colored card (red = risk, green = opportunity, amber = watch), with a small `ⓘ` source note ("basis: 9,994 lines, full period"). Add a **"Copy insights"** button (plain-text export) for pasting into a deck or email.

---

## 8. DESIGN SYSTEM

**Aesthetic:** modern executive analytics — calm, high information density, generous whitespace, no gradients-on-gradients, no 3D, no pie-chart abuse. Think Stripe/Vercel dashboards.

- **Layout:** CSS Grid, 12-column, max-width 1600px, 24px gutters. Order: sticky header (title, date range, reset) → KPI row (8 cards) → Insight strip → main grid (V1 full width; V2 + V3 half/half; V4 full; V5 + V7 half/half; V6 full) → detail table (V8) → footer.
- **Palette (accessible, colorblind-safe):**
  - Background `#F7F8FA`; cards `#FFFFFF`; borders `#E4E7EC`.
  - Ink `#101828`; secondary text `#667085`; muted `#98A2B3`.
  - Primary/brand **AICW blue `#1D4ED8`**; secondary `#0E9384`.
  - Semantic: positive `#067647`, negative `#B42318`, warning `#B54708`, info `#175CD3`.
  - Category palette (consistent everywhere): Furniture `#7F56D9`, Office Supplies `#0E9384`, Technology `#1D4ED8`.
  - Never encode meaning by color alone — pair with labels, icons, or +/− signs.
- **Typography:** Inter / system-ui; KPI value 32–36px 600wt, tabular numerals (`font-variant-numeric: tabular-nums`); card title 13px uppercase 600wt letter-spaced; body 13px; table 12.5px. Line-height 1.4.
- **Cards:** 12px radius, 1px border, subtle shadow `0 1px 2px rgba(16,24,40,.06)`, 20px padding, section eyebrow + title + optional `⤢` and `⋯` (export PNG / CSV).
- **Charts:** no chart-junk; horizontal gridlines only, `#EAECF0`; axis labels 11px `#667085`; tooltips = dark card `#101828` with white text and 2–4 metric rows; currency/percentage formatted identically to the cards.
- **States:** skeleton shimmer on first paint (<400ms); hover elevation; focus ring `2px #1D4ED8` for keyboard users; empty state illustration + CTA.
- **Responsive:** ≥1440px 4 KPI cols; 1024–1439 4 cols tighter; 768–1023 2 cols, charts stack; <768 single column, filters collapse into a bottom sheet, table becomes horizontally scrollable with frozen first column.
- **Motion:** 200ms ease transitions on filter changes; respect `prefers-reduced-motion`.

---

## 9. TECHNICAL REQUIREMENTS

1. **One deliverable file:** `aicw-sales-dashboard.html` — must open by double-click with **no build step, no server, no npm**.
2. **Libraries via CDN:** Chart.js v4 **or** ECharts v5 (pick one, be consistent); optional `chartjs-plugin-datalabels`. No frameworks required (vanilla JS is preferred); Tailwind CDN is acceptable but hand-written CSS is preferred for offline resilience.
3. **Data embedding:** embed the 9,994 rows **pre-aggregated** where possible (or the full JSON array if size stays under ~4 MB). Data must be a JS constant `const RAW_DATA = [...]` — **no `fetch()` of a local CSV** (breaks on `file://` due to CORS). Underscore-prefixed keys to keep the payload small.
4. **Architecture:** a single `state` object holding all filter values → one `applyFilters()` → one `renderAll()`. Chart instances stored in a registry and `update()`d, not destroyed/recreated, on filter change. Pre-compute aggregates on load.
5. **Correctness rules (non-negotiable):**
   - Margin and discount are **ratios of sums / weighted averages**, never averages of row-level ratios.
   - `COUNT(DISTINCT Order ID)` for order counts — never `COUNT(rows)` (which would give 9,994 instead of 5,009).
   - Dates parsed with explicit `%m/%d/%Y`; never rely on locale auto-detection.
   - Latin-1 decoding for product names (preserve `’`, `&`, `é`); escape HTML when injecting into the DOM.
6. **Numerical formatting helper:** one `fmt.money()`, `fmt.pct()`, `fmt.int()`, `fmt.compact()` used everywhere so the same figure never appears as `$1,087,908.47` in one place and `1087908` in another.
7. **Accessibility:** WCAG 2.1 AA contrast; every chart has `role="img"` + `aria-label` summarizing the trend; keyboard-navigable filters; visible focus; table has `<caption>` and `<th scope>`.
8. **Performance:** initial interactive in **< 2s**; filter response < 150ms; no memory leaks; total file comfortable under ~8 MB.
9. **No external data calls, no tracking, no login, no backend.** Fully offline-capable after first load (CDN cached).
10. **Print:** a `@media print` stylesheet rendering a clean one-page-per-section executive pack (charts without hover chrome, KPI grid preserved).
11. **Code quality:** commented sections, named constants, no `console.log` left behind, no dead code.

---

## 10. ACCEPTANCE CHECKLIST (self-verify before you answer)

Reproduce these exact values on the **unfiltered** dashboard. If any mismatch, your aggregation logic is wrong — fix it before finishing.

- [ ] Total Sales = **$2,297,201** (±$1)
- [ ] Total Profit = **$286,397** (±$1)
- [ ] Profit Margin = **12.47%**
- [ ] Orders = **5,009** · Customers = **793** · Products = **1,850** distinct names (1,862 SKU IDs) · Units = **37,873**
- [ ] AOV = **$458.61** · Avg Discount = **15.62%**
- [ ] Loss exposure = **−$156,131** across **1,022** orders (20.4%)
- [ ] Furniture margin = **2.5%**; Tables profit = **−$17,725**; Copiers profit = **+$55,618**
- [ ] West margin **14.9%** vs Central **7.9%**; Texas profit **−$25,729**
- [ ] Discount band 41%+ margin = **−77.4%**; 0% band margin = **29.5%**
- [ ] Yearly sales: 2014 **$484,248** → 2015 **$470,533** → 2016 **$609,206** → 2017 **$733,215**
- [ ] Q4 revenue share = **38.2%**
- [ ] 8 KPI cards, 8 charts, ≥10 filters, 7 insight cards, 1 sortable/exportable table
- [ ] Every filter updates every visual; Reset All works; no `NaN`/`undefined`/blank chart in any filter combination
- [ ] Opens correctly from `file://` with no console errors; responsive at 375 / 768 / 1440 px
- [ ] Table CSV export matches the current filtered view row-for-row

---

## 11. DELIVERABLE

Return **exactly one** `aicw-sales-dashboard.html` file containing:

1. `<!DOCTYPE html>` with all CSS in `<style>` and all logic in `<script>` (no build tooling).
2. Header: **"AICW Project — Sales & Profitability Command Center"**, subtitle `FY2014–FY2017 · 5,009 orders · $2.30M revenue`, live date-range chip, Reset All.
3. The 8 KPI cards, the insight strip, the 8 visualizations, the global filter bar, and the detail table — wired together as specified.
4. A footer: data source `Sample_-_Superstore.csv (9,994 lines, Latin-1 encoded)`, refresh timestamp, and a one-line methodology note stating that margin = ΣProfit/ΣSales and order counts are distinct `Order ID`.

Do **not** ask clarifying questions. Do **not** substitute placeholder data. Do **not** omit a section to "keep it short". If a value is not computable from the data, render `—` rather than guessing.
