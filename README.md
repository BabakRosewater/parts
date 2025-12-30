Parts Inventory Dashboard (GL 2420)

A client-side (no backend) dashboard that loads multiple CSV sources from the public BabakRosewater/parts GitHub repo, applies filters, and renders summaries + tables across multiple tabs:

Overview (Net by Month chart + Journal Summary)

Vendor Spend (group_control aggregation)

Reconciler (detail vs totals validation)

Document / RO Tracking (heuristic grouping)

Counter Pad (on-hand snapshot)

Aging (2025 inventory aging snapshot)

Everything runs in the browser using CDN libraries.

1) Runtime Dependencies (CDN)

Included directly in the HTML (no CodePen settings required):

TailwindCSS (layout + utility classes)

PapaParse (CSV fetch + parse)

Chart.js (Net by Month line chart)

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>

2) Data Sources (GitHub RAW)

All datasets are loaded from GitHub RAW URLs using a “first working” fallback list.

Source files

PARTS_INVENTORY_normalized_detail.csv

“Transaction detail” (GL activity rows)

PARTS_INVENTORY_group_totals.csv

Audit totals per group_control (used for reconciliation)

Counter_Pad.csv

Snapshot of on-hand by part/bin (counter screen)

Inventory_Aging_Report_2025.csv

Snapshot of aging (months since sale/receipt, excess, etc.)

URL configuration (JS)
const URLS = {
  detail: [...],
  totals: [...],
  counter: [...],
  aging: [...]
};


Loading uses:

parseCSV(url) → PapaParse download+parse

loadFirstWorking(candidates) → tries candidates in order

3) UI Structure (HTML)

The UI is divided into:

A) Header + Filters

Status line: #statusLine

Filters:

#fStart (date)

#fEnd (date)

#fJournal (select)

#fSearch (text search)

#btnReset (resets all controls)

Important behavior

Date + journal filters apply only to the “detail-driven” tabs:

Overview, Vendor Spend, Reconciler, Docs

Date + journal filters are disabled on snapshot tabs:

Counter Pad, Aging
(because those sources are snapshots, not dated transaction rows)

B) Tabs

Buttons with data-tab:

overview

vendors

reconcile

docs

counter

aging

Panels with IDs:

#tab-overview

#tab-vendors

#tab-reconcile

#tab-docs

#tab-counter

#tab-aging

C) KPI Row

Four KPI boxes updated depending on the active tab:

Detail tabs: Transactions / Debit / Credit / Net

Counter tab: Lines / On-Hand Units / ExtCost / List Value

Aging tab: Lines / On-Hand Units / ExtCost / Excess Value

IDs:

Labels: #kpiLabel1..4

Values: #kpiTx #kpiDebit #kpiCredit #kpiNet

D) Tab-Specific Controls

Vendor Spend

#vendorSort, #vendorLimit

Docs

#docLimit

Counter

#cpGroup, #cpStatus, #cpSort, #cpLimit, #cpOnHandOnly

Aging

#agStockingGroup, #agStatus, #agBucket, #agSort, #agLimit

#agExcessOnly, #agOnHandOnly

4) Styling (CSS)

The CSS provides a consistent “dark UI” theme with soft borders and readable inputs.

Key goals:

Stable dark theme and contrast

More readable dropdown lists on Edge/Windows 11

Prevent Chart.js “infinite resize” issues

Theme variables
:root{
  --bg: #0b1220;
  --card:#0f1a2e;
  --text:#e9eef9;
  --border: rgba(255,255,255,0.10);
  --pill: rgba(255,255,255,0.05);
  --dot: #5eead4;
  color-scheme: dark;
}

Dropdown readability fix (Edge/Win11)

Edge often renders the dropdown popup menu as light. We force option text to dark for readability:

select.ui-input option,
select.ui-input optgroup{
  color: #0b1220;
  background-color: #ffffff;
}

Chart sizing

A fixed-height wrapper prevents reflow loops:

.chart-wrap{
  position: relative;
  width: 100%;
  height: 260px;
  overflow: hidden;
}
#monthChart{
  display: block;
  width: 100%;
  height: 100%;
}

5) JavaScript Architecture

The JS is organized into the following functional areas:

A) State container

All app state is stored in one object:

const state = {
  activeTab: "overview",
  detail: [], totals: [], filtered: [], journals: [],
  counter: [], counterFiltered: [], counterGroups: [], counterStatuses: [],
  aging: [], agingFiltered: [],
  monthChart: null,
  debounceTimer: null,
};

B) Parsing + loading

parseCSV(url) uses PapaParse with:

header: true

dynamicTyping: true

skipEmptyLines: true

loadFirstWorking([...urls]) tries sequential URLs

C) Filters

Main filters (detail tabs):

readMainFilters() returns {start,end,journal,search}

rowMatchesDetail(row, f) applies:

date range comparisons (row.date must already be YYYY-MM-DD)

journal matching

search across multiple fields

applyDetailFilters() populates state.filtered

Counter filters (snapshot):

applyCounterFilters() uses:

group/status selects

on-hand-only checkbox

main search box (#fSearch) across part/bin/etc.

Aging filters (snapshot):

applyAgingFilters() uses:

stockingGroup/status/bucket

excessOnly + onHandOnly toggles

main search box across part/bin/status codes

Bucketing logic:

inAgingBucket(monthsNoSale, bucket)

D) Normalization / Mapping

Because CSV headers differ per file, the loader maps raw columns into consistent JS object shapes.

Counter Pad normalization

parsePartDesc("PARTNO : DESC") → { partNo, desc }

Produces fields like:

partNo, desc, grp, sts, bin, onHand, cost, list, extCost

Aging normalization
Maps CSV headers into canonical keys:

partNumber, description, status, bin, shelf, stockingGroup

monthsNoSale, monthsNoReceipt

onHand, extCost, excessQty, excessValue

Date fields are converted from M/D/YYYY to YYYY-MM-DD:

parseMDYDateString()

E) Aggregations

sumDetail(rows) → debit/credit/net/count

sumCounter(rows) → onHand/extCost/listValue/count

sumAging(rows) → onHand/extCost/excessValue/count

aggByJournal(detailFiltered)

aggByVendor(detailFiltered)

aggNetByMonth(detailFiltered)

F) Reconciler

Compares detail-derived vendor totals against the audit totals file:

buildTotalsMap(state.totals)

reconcile(vendorAgg, totalsMap) with tolerance ±$0.01
Outputs mismatch rows {control, dDebit, dCredit, dNet}.

G) Document / RO tracking (heuristic)

Extracts likely RO/document keys from text:

Looks for “RO ######”

Else falls back to 5–8 digit sequences

Else “UNCLASSIFIED”
Then aggregates debit/credit/net by extracted key.

H) Rendering

Each panel has a renderer:

renderKPIs()

renderJournalTable()

renderMonthChart() (Chart.js line)

renderVendorTable()

renderReconciler()

renderDocTable()

renderCounterTable()

renderAgingTable()

Chart safety
ensureChartContainerHeight() guards against a missing height that can cause Chart.js resize loops in some layouts.

I) Tabs + events

setTab(name) switches active tab, toggles .is-active, hides/shows panels, disables date/journal on snapshot tabs.

wireUI() binds all filters + tab clicks to scheduleRecalc()

scheduleRecalc() debounces to avoid re-rendering on every keystroke.

J) Recalc pipeline

recalcAndRender() is the single “update loop”:

apply filters (detail/counter/aging)

render KPIs

render all panels (safe even if hidden)

update status line based on active tab

6) Expected CSV Field Assumptions

These are important for stability:

Detail (normalized_detail)

Expected fields used by code:

date (assumed YYYY-MM-DD)

journal

debit, credit, net

group_control, group_header_desc

line_description, document, reference

Totals (group_totals)

group_control

total_debit, total_credit, total_net

Counter Pad (Counter_Pad.csv)

Columns referenced:

Part/Description, O/H, Cost, List, ExtCost

MF, Grp, Sts, OEM, Bin/Shelf

(optional) O/O, B/O, Core, Trade

Aging (Inventory_Aging_Report_2025.csv)

Columns mapped (case-sensitive as in code):

PartNumber, Description, Status

BinLocation, Shelf, StockingGroup

MonthsNoSale, MonthsNoReceipt

OnHand, ExtCost, ExcessQty, ExcessValue

LastSaleDate, LastReceiptDate, InInventoryDate

plus optional codes: GroupCode, StockCode, ReturnCode

If a CSV header changes, the dashboard will load but filters/columns may go blank until the mapping is updated.

7) How to Extend Safely

Common extension pattern:

Add UI controls in HTML (new select/checkbox)

Add state fields if needed

Bind events in wireUI()

Update filter function (applyXFilters)

Update renderer (renderXTable + meta)

Ensure the new renderer is called in recalcAndRender()

8) Known Browser/UI Constraints

Native <select> dropdown popup styling is limited (especially on Windows).
The CSS improves readability but cannot fully theme the popup in all browsers.

Chart.js requires a stable container height; we already enforce a fixed-height wrapper.

9) Quick Developer Checklist

✅ All required HTML IDs exist (tables, metas, selects)

✅ CSV URLs are public and correct

✅ Detail date format is YYYY-MM-DD

✅ Aging date strings are M/D/YYYY (or compatible)

✅ Recalc runs after tab switch + filter changes

✅ Status line updates by active tab
