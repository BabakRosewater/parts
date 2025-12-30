````md
# Parts Inventory Dashboard (GL 2420)

A **client-side** (no backend) dashboard for Parts / GL 2420 analytics. It loads CSVs from the public `BabakRosewater/parts` repo via GitHub RAW, applies filters, and renders charts + tables across multiple tabs.

**Tabs**
- Overview (Net by Month chart + Journal Summary)
- Vendor Spend (group_control aggregation)
- Reconciler (detail vs totals validation)
- Document / RO Tracking (heuristic grouping)
- Counter Pad (on-hand snapshot)
- Aging (2025 inventory aging snapshot)

---

## Tech Stack (CDN)
- TailwindCSS (layout + utilities)
- PapaParse (CSV fetch + parse)
- Chart.js (line chart)

Included in HTML:
```html
<script src="https://cdn.tailwindcss.com"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.1/dist/chart.umd.min.js"></script>
````

---

## Data Sources (GitHub RAW)

The app loads four CSVs (with fallback URL candidates):

1. **Transactions (detail)**

   * `PARTS_INVENTORY_normalized_detail.csv`

2. **Audit totals**

   * `PARTS_INVENTORY_group_totals.csv`

3. **Counter Pad snapshot**

   * `Counter_Pad.csv`

4. **Inventory Aging snapshot (2025)**

   * `Inventory_Aging_Report_2025.csv`

Configured in JS:

```js
const URLS = { detail:[...], totals:[...], counter:[...], aging:[...] };
```

Loading uses:

* `parseCSV(url)` → PapaParse download+parse
* `loadFirstWorking(candidates)` → tries URLs until one succeeds

---

## How Filtering Works

### Main filters (Detail-driven tabs)

Controls:

* `Start date` (`#fStart`)
* `End date` (`#fEnd`)
* `Journal` (`#fJournal`)
* `Search` (`#fSearch`)

Applies to:

* Overview, Vendor Spend, Reconciler, Docs/RO Tracking

Search matches:

* `group_control`, `group_header_desc`, `line_description`, `document`, `reference`, `journal`, `date`

**Important:** detail `date` is assumed to be `YYYY-MM-DD` for string comparisons.

### Snapshot tabs (Counter Pad + Aging)

* Still use the **main Search** box (`#fSearch`)
* **Date + Journal are disabled** automatically (snapshots are not transaction-dated)

Counter Pad additional filters:

* Group (`#cpGroup`), Status (`#cpStatus`), Sort (`#cpSort`), Limit (`#cpLimit`), O/H > 0 (`#cpOnHandOnly`)

Aging additional filters:

* Stocking Group (`#agStockingGroup`), Status (`#agStatus`), Bucket (`#agBucket`)
* Sort (`#agSort`), Limit (`#agLimit`)
* Excess only (`#agExcessOnly`), O/H > 0 (`#agOnHandOnly`)

---

## Reconciler (What it validates)

Compares:

* Vendor/control totals computed from **filtered detail**
  vs
* Totals from `PARTS_INVENTORY_group_totals.csv`

Key:

* `group_control`

Tolerance:

* ± `$0.01`

Outputs mismatches:

* Δ Debit, Δ Credit, Δ Net (sorted by |Δ Net|)

---

## Document / RO Tracking (Heuristic)

Groups filtered detail rows by “RO-like” keys extracted from:

* `document`, `reference`, and `line_description`

Rules:

1. If matches `RO` + 4–8 digits → `RO ######`
2. Else if any 5–8 digit number → `DOC ######`
3. Else → `UNCLASSIFIED`

Then aggregates debit/credit/net per key.

---

## KPIs (Top Row)

KPIs change based on active tab:

### Detail tabs

* Transactions
* Total Debit
* Total Credit
* Net

### Counter Pad

* Lines
* On-Hand Units
* Total ExtCost
* Total List Value (`list * onHand`)

### Aging

* Lines
* On-Hand Units
* Total ExtCost
* Excess Value

---

## Expected CSV Columns

### `PARTS_INVENTORY_normalized_detail.csv`

Used fields:

* `date` (expected `YYYY-MM-DD`)
* `journal`
* `debit`, `credit`, `net`
* `group_control`, `group_header_desc`
* `line_description`, `document`, `reference`

### `PARTS_INVENTORY_group_totals.csv`

Used fields:

* `group_control`
* `total_debit`, `total_credit`, `total_net`

### `Counter_Pad.csv`

Used columns:

* `Part/Description` (parsed into `partNo` + `desc` using `" : "`)
* `MF`, `Grp`, `Sts`, `OEM`, `Bin/Shelf`
* `O/H`, `Cost`, `List`, `ExtCost`
* (optional) `O/O`, `B/O`, `Core`, `Trade`

### `Inventory_Aging_Report_2025.csv`

Used columns:

* `PartNumber`, `Description`, `Status`
* `BinLocation`, `Shelf`, `StockingGroup`
* `MonthsNoSale`, `MonthsNoReceipt`
* `OnHand`, `ExtCost`, `ExcessQty`, `ExcessValue`
* `LastSaleDate`, `LastReceiptDate`, `InInventoryDate` (MDY → ISO conversion)
* (optional) `GroupCode`, `StockCode`, `ReturnCode`

If headers change, update the mapping in the JS loader section.

---

## Chart.js Sizing Note (Prevents Resize Loops)

The Net-by-Month chart uses a fixed-height wrapper:

* `.chart-wrap { height: 260px; }`
* Chart options: `maintainAspectRatio: false`, `animation: false`

This prevents “infinite growth” resize loops in some layouts.

---

## Local Editing / Troubleshooting

**If you see “Load failed”:**

* Verify GitHub RAW URLs are correct
* Ensure repo + files are public
* Confirm file names match exactly
* Confirm CSV headers expected by the JS mapping

**If dates filter incorrectly:**

* Detail date must be `YYYY-MM-DD`
* Aging MDY dates are converted by `parseMDYDateString()`

---

## Roadmap Ideas

* Add export (filtered table CSV)
* Add “click-through” drilldowns (Vendor → transactions)
* Add saved views (persist filters in URL params)
* Add custom “RO classifier” rules per store conventions

```

::contentReference[oaicite:0]{index=0}
```
