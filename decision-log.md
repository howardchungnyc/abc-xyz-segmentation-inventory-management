# Decision Log

## ABC XYZ Inventory Segmentation & Management

### Power BI Project — Phases 1–3 (ETL + Model Layer + DAX Layer)

---

> This log explains the main build decisions for this project, why each decision was made, and how each decision affects reporting and operations.

---

## Phase 1 — ETL Data Preparation Layer

## Entry #1 — Single Source Data Staging (MasterSet)

**Date:** March 2026  

**Layer:** Power Query `MasterSet`

**Decision:**
Use `MasterSet` as one central source table. All report tables pull from it, and `MasterSet` itself is not loaded into the final report model.

**Context:**
Initial build had all five queries (`DimDate`, `DimProduct`, `DimCustomer`, `FactOrders`, `MasterSet`) each loading the source CSV independently. This meant the CSV was being read five times on every refresh, creating five separate file-read operations on a 180K+ row dataset.

**What Changed:**

- Designated MasterSet as the sole CSV loading query
- Set MasterSet `Enable Load = False` removes it from the model entirely
- `DimProduct`, `DimCustomer`, `FactOrders` set as reference queries: `Source = MasterSet`
- `DimDate` reads `List.Min` and `List.Max` directly from `MasterSet` column inline

**Architecture:**

```
CSV Source
    └── MasterSet (load disabled, staging only)
            ├── DimProduct (reference query)
            ├── DimCustomer (reference query)
            ├── FactOrders (reference query)
            └── DimDate (reads Min/Max from MasterSet inline)
```

**Benefits:**

- CSV loads exactly once per refresh — no redundant file reads
- Single source of truth — source schema changes propagate automatically
- MasterSet does not appear in the published model — cleaner field list
- Zero-touch updates when source data is refreshed or extended

---

## Entry #2 — Auto-Updating DimDate Table with 1-Year Buffer

**Date:** March 2026  

**Layer:** Power Query `DimDate`

**Decision:**
Generate DimDate dynamically from MasterSet min/max dates with 1-year padding on each boundary rather than using hardcoded start and end dates.

**Implementation:**

```
MinDate    = Date.From(List.Min(MasterSet[#"order date (DateOrders)"]))
MaxDate    = Date.From(List.Max(MasterSet[#"order date (DateOrders)"]))
StartDate  = Date.StartOfYear(Date.AddYears(MinDate, -1))
EndDate    = Date.EndOfYear(Date.AddYears(MaxDate, 1))
```

**Source Data Range:** Jan 1, 2015 → Jan 31, 2018  

**Generated Range:** Jan 1, 2014 → Dec 31, 2019  

**Total Rows:** 2,191 contiguous daily dates

**Benefits:**

- Zero hardcoded dates — model self-maintains as source data grows
- Padding ensures time intelligence coverage beyond transaction boundaries
- If source data extends into 2019, DimDate automatically extends to Dec 31, 2020

**Important downstream note:**
`Revenue by SKU (12-Month Trailing)`, `Avg Daily Demand by SKU`, and `Demand Std Dev (Daily)` all anchor their trailing windows to `MAX(FactOrders[Order Date])`, not `MAX(DimDate[Date])`. DimDate's 1-year padding beyond the last transaction would otherwise produce an empty classification window. See Entry #17.

---

## Entry #3 — Dynamic Late Delivery Risk Rule

**Date:** March 2026  

**Layer:** Power Query `FactOrders`

**Decision:**
Replace the source `Late_delivery_risk` column with a transparent derivation based on actual shipping delay. If `Lead Time Variance > 0`, mark as late risk.

**Context:**
The DataCo source data includes a `Late_delivery_risk` flag. Review showed it was a pre-labeled machine learning training field created by the dataset authors, not a field calculated from actual shipping performance. Empirical testing found 5 mismatches per 1,000 rows between the source flag and actual Lead Time Variance outcomes, confirming the flag was generated independently from the data it appears to describe.

**What Changed:**

- Removed the source `Late_delivery_risk` column
- Replaced with two derived columns:
  - `Late Delivery Risk` (Whole Number) — `1` if `Lead Time Variance > 0`, else `0` for DAX calculations
  - `Is Late Delivery Risk` (True/False) — `True` if `Lead Time Variance > 0` for report filters and visuals

**Formula:**

```m
Late Delivery Risk    = if [Lead Time Variance] > 0 then 1 else 0
Is Late Delivery Risk = [Lead Time Variance] > 0
```

**Why This Is Better:**

- Fully transparent logic auditable by any developer
- Dynamically recalculates when source data is refreshed
- Derived from actual shipping performance
- Two columns serve both analytical and presentation layers

---

## Entry #4 — Two Late-Risk Fields for Different Uses

**Date:** March 2026  

**Layer:** Power Query `FactOrders`

**Decision:**
Maintain two versions of the `Late Delivery Risk` binary flag:

1. Numeric integer (0/1) for DAX calculations
2. Logical (True/False) for report filters and visuals

**Context:**
`Late Delivery Risk` needed to serve two purposes simultaneously:

1. DAX calculations — rates, averages, and safety stock formulas
2. Report visuals filtering — slicers, conditional formatting, and KPI cards

One column does not serve both needs well. Numeric values are better for calculations; True/False is easier for report users to filter and read.

**Pattern Applied:**

- `Late Delivery Risk` (Int64) — `0` or `1` (DAX arithmetic layer)
- `Is Late Delivery Risk` (Logical) — `True` or `False` (Presentation layer)

**Naming Convention:** `Is` prefix denotes logical boolean columns throughout the model.

**Downstream DAX Usage:**

```dax
-- Integer version — arithmetic
Late Delivery Rate % = AVERAGE(FactOrders[Late Delivery Risk])

-- Logical version — filter
Late Orders = CALCULATE(COUNTROWS(FactOrders), FactOrders[Is Late Delivery Risk] = TRUE())
```

---

## Entry #5 — Consistent Column Ordering Standard

**Date:** March 2026  

**Layer:** Power Query (`DimProduct`, `DimCustomer`, `FactOrders`)

**Decision:**
Apply a consistent column order so tables are easier to read, validate, and maintain:
`Primary Key → Foreign Keys → Date Keys → Attributes → Measures → Calculated Fields`

**Context:**
Without a fixed column order, query outputs become inconsistent, harder to scan, and more error-prone during maintenance when new columns are added.

**What Changed:**

- Reordered columns in each model query to follow the same enterprise sequence
- Grouped business attributes after key fields for faster validation and troubleshooting
- Kept derived and analytical fields at the end to preserve readability during ETL review

**Note:** `FactOrders` primary key at line-item grain is `Order Line Id`, not `Order Id` (documented in Entry #13).

---

## Phase 2 — Model Layer (Semantic Model)

## Entry #6 — Sort By Column Assignments

**Date:** March 2026  

**Layer:** Model properties — `DimDate`

**Decision:**
Set **Sort by column** on `DimDate` text labels so charts and slicers follow calendar order, not A–Z.

**Assignments:**

- `Month Name` → `Month Number`
- `Day Name` → `Day Number`
- `Quarter Label` → `Quarter Number`
- `Year-Month Label` → `Year-Month Key`

**Rationale:**
Power BI's default sort for text is alphabetical. Month names like "Apr" and "Jan" sort wrong alphabetically. Calendar labels require an explicit numeric or key sort column for correct axis and slicer behavior.

---

## Entry #7 — Hidden Columns

**Date:** March 2026  

**Layer:** Model — all loaded tables

**Decision:**
Hide surrogate keys, foreign keys, and DAX-only columns from report view across `FactOrders`, `DimDate`, `DimProduct`, and `DimCustomer`.

**Rationale:**
Hiding these fields keeps day-to-day reporting focused on names, dates, and measures. Hidden columns remain available for relationships and DAX.

---

## Entry #8 — Default Summarization

**Date:** March 2026  

**Layer:** Model — `FactOrders` (and related columns as applicable)

**Decision:**
Set **Don't summarize** on ID columns, date keys, geographic coordinates, and ratio/rate columns where Sum is semantically wrong.

**Rationale:**
Prevents implicit aggregation mistakes. Phase 3 DAX measures own all intentional aggregations.

**Note:** Columns such as `Order Discount`, `Order Gross Profit`, `Order Profit Per Order`, `Order Quantity`, `Order Total`, and `Sales` remain summable where additive semantics apply.

**Downstream effect:** Columns set to Don't Summarize cannot be used as implicit aggregates in visuals. Any count, sum, or aggregation of these columns requires an explicit DAX measure. Example: `Order Line Id` cannot be dropped into a visual Values well as a count — `Delivery Status Row Count = COUNTROWS(FactOrders)` in the `_Validation` folder is the correct pattern. This is intentional — explicit DAX measures own all aggregations per this entry.

---

## Entry #9 — Display Folders on FactOrders

**Date:** March 2026  

**Layer:** Model — `FactOrders`

**Decision:**
Organize `FactOrders` columns into seven display folders: **Keys**, **Dates**, **Financial**, **Rates & Ratios**, **Order Details**, **Shipping & Logistics**, **Delivery Risk**.

**Rationale:**
Improves field-list navigation during DAX measure authoring.

---

## Entry #10 — Power Query Groups

**Date:** March 2026  

**Layer:** Power Query Editor

**Decision:**
Group queries into **Staging** (`MasterSet`), **Facts** (`FactOrders`), and **Dimensions** (`DimDate`, `DimProduct`, `DimCustomer`).

**Rationale:**
Makes the ETL architecture readable without opening individual queries.

**Note:** When you create named groups, Power BI sometimes leaves a default **Other Queries** bucket. Refresh behavior and the loaded model are unchanged.

---

## Entry #11 — Date Hierarchies on DimDate

**Date:** March 2026  

**Layer:** Model — `DimDate`

**Decision:**
Create two explicit hierarchies on `DimDate`:

- **Year–Month–Day:** `Year` → `Month Name` → `Day Name`
- **Year–Qtr–Month:** `Year` → `Quarter Label` → `Month Name`

**Rationale:**
Custom hierarchies match calendar attributes and sort keys, so drill-down order in time-series visuals stays consistent with how the dimension was built.

---

## Entry #12 — Shared DimDate: Order Date vs. Shipping Date (dual relationships)

**Date:** March 2026  

**Layer:** Model — `FactOrders` ↔ `DimDate` (role-playing dates)

**Decision:**
**Order Date** and **Shipping Date** both use the same calendar table (`DimDate`). **Order Date** is the active relationship for most time intelligence analysis. **Shipping Date** is inactive so the model keeps a single default timeline; analysis on the shipment calendar uses `**USERELATIONSHIP`** in DAX when needed.

**Rationale:**
**Order Date** answers demand questions — when customers buy, revenue trends, seasonal patterns. **Shipping Date** answers fulfillment questions — how long orders take to ship, whether late deliveries cluster by period, where the operation is falling behind. Keeping both on the same calendar table enables either view to use identical date filters, hierarchies, and time intelligence without building a second calendar.

---

## Entry #13 — FactOrders: Line-Item Grain, Primary Key, and Order-Header Denormalization

**Date:** March 2026  

**Layer:** Model — `FactOrders`; documentation — `column-definition.md`, MasterSet source notes

**Observation #1 (FactOrders Primary Key):** `FactOrders` operates at line-item grain: 180,519 rows and 65,752 distinct `**Order Id`** values (~2.75 line items per order on average).

**Decision #1:** At this grain the primary key is `**Order Line Id`** (DataCo source `Order Item Id`). `**Order Id`** is the order-level natural key — it repeats across line items on the same order, is not unique at line-item grain, and must not be documented as the fact PK.

**Degenerate Dimension:** `**Order Id`** stays on `**FactOrders`** as a **degenerate dimension** — the business order identifier carried on the fact without a separate `**DimOrder`** table. Matches common Kimball usage when the source does not justify a full order dimension and the id is mainly for grouping or drill-through.

**Documentation Updates:** `column-definition.md` FactOrders Keys and MasterSet Order Id notes updated.

**Observation #2 (Order-Header Denormalization):** Because the fact is at line grain, order-header attributes (`Market`, `Order Status`, `Order Region`, `Order City`, `Order Country`, `Payment Type`) repeat on every row that shares the same `Order Id`.

**Architectural Options Considered:**

- `FactOrderHeader` — one row per `Order Id`, holding order-level attributes
- `FactOrderLines` — one row per line item, with `Order Id` as a foreign key to the header

That two-grain fact pattern is standard in enterprise-scale order domains when both header-level and line-level metrics are substantial.

**Decision #2:** Retain the collapsed single fact. The DataCo dataset does not expose order-header-level measures (no order-level shipping cost, tax, or fee) that would create aggregation conflicts. Without header-level facts, splitting adds join complexity with no analytical return for this reporting scope.

**Trade-offs Considered:** Order-header attributes repeat N times per order. Measures using these columns as filter context (e.g. revenue by Market) still aggregate correctly because additive facts are defined at line-item grain. No double-counting arises under the current measure design.

**What Triggers Schema Change:** Order-level costs (freight, handling, taxes) that must be allocated across lines would justify `FactOrderHeader`.

---

## Entry #14 — Model Validation Suite

**Date:** March 2026  

**Layer:** DAX measures — `FactOrders` → `**_Validation`** display folder

**Decision:**
Run a short set of automated tests in the model: row totals, order-date range, and every order row links to a real customer and product (no orphan keys).

**Organization:**
Validation measures live in `FactOrders` under the `_Validation` display folder. The folder name starts with `_` so it sorts to the top of the field list and reads as infrastructure, not business reporting.

**Scope note:** Entry #14 reflects the **Phase 2** validation suite only — the seven tests in the table below and their measures. Phase 3 added many more `_Validation` measures. The **authoritative** list of measures and folder structure is `dax-measures.md`; use that file for current inventory, not the "seven measures" wording from the original write-up.

**Tests and Results:**

| Test                 | Expected     | Result |
| -------------------- | ------------ | ------ |
| Row Count Integrity  | 180,519      | Pass   |
| Earliest Order Date  | Jan 1, 2015  | Pass   |
| Latest Order Date    | Jan 31, 2018 | Pass   |
| Blank Customer Check | 0            | Pass   |
| Blank Product Check  | 0            | Pass   |
| Customer Match Check | 180,519      | Pass   |
| Product Match Check  | 180,519      | Pass   |

---

## Entry #15 — Model Validation Page Preserved

**Date:** March 2026  

**Layer:** Report page — **QA - Model Validation**

**Decision:**
Keep the Model Validation report page in the shipped `.pbix` after all validation tests pass. Page renamed to `QA - Model Validation` per QA page naming convention established in Entry #16.

**Rationale:**
The page is a reference for anyone opening the file later: it shows what was tested and the outcomes without hunting through the measures list.

**Note:** Measure definitions and the `_Validation` folder are documented under Entry #14.

---

## Phase 3 — DAX Measures Layer

## Entry #16 — QA Validation Pages and Measure Display Folder Naming

**Date:** March 2026

**Layer:** Report pages; `FactOrders` → `_Measures` display folders; `DimProduct` column folders

**Decision: QA Validation Pages:**
Retain QA measure validation pages in the shipped `.pbix` for each phase of measure development. Pages are prefixed `QA -` to distinguish infrastructure from business report pages. Infrastructure folders `_Deprecated` and `_Validation` retain underscore prefix.

**Rationale:** Validation pages to audit measures.

| Page                          | Purpose                                                                                                                                                      |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `QA - Model Validation`       | Row count, date range, dimension integrity                                                                                                                   |
| `QA - Core Measures`          | Core measures validated against raw source column aggregates                                                                                                 |
| `QA - Inventory Segmentation` | ABC analysis per APICS CPIM, SKU ranking, cumulative revenue, Dense tie verification, XYZ distribution                                                       |
| `QA - Supply Performance`     | Supplier lead time, delivery reliability measures validated by category, delivery status distribution                                                        |
| `QA - Inventory Planning`     | APICS CPIM - safety stock, reorder point, and replenishment parameters, Safety stock, reorder point, reorder flag, stock coverage validated at product grain |
| `QA - Financial Impact`       | `P&L Impact`                                                                                                                                                 |
| `QA - Trend Analysis`         | `Time Intelligence`                                                                                                                                          |

**Decision 2: Measure Display Folder Names - APICS/SCOR Alignment:**
Display folder labels align with APICS CPIM Body of Knowledge and SCOR Model functional domain terminology used by enterprise supply chain operations.

| Display Folder           | Replaces                       | Canonical Source                                                                                      |
| ------------------------ | ------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `Core Measures`          | `Foundation`                   | Enterprise BI convention - signals base layer without implying a supply chain function                |
| `Inventory Segmentation` | `ABC Classification`           | APICS CPIM - ABC analysis falls under Inventory Segmentation in the Planning domain                   |
| `Supply Performance`     | `Lead Time & Reliability`      | SCOR Source domain - supplier lead time and delivery reliability under Supply Chain Reliability       |
| `Inventory Planning`     | `Safety Stock & Replenishment` | APICS CPIM - safety stock, reorder point, and replenishment parameters are Inventory Planning outputs |
| `Financial Impact`       | `P&L Impact`                   | Enterprise BI convention - more precise than P&L which implies a full income statement                |
| `Trend Analysis`         | `Time Intelligence`            | Enterprise BI convention - Time Intelligence is a Power BI developer term, not a business term        |

---

## Entry #17 - Inventory Segmentation: ABC and XYZ Classification

**Date:** March 2026  

**Layer:** `_Measures\Inventory Segmentation` (DAX measures); `DimProduct` (calculated columns)

**ABC Classification Methodology:**
ABC inventory classification ranks every SKU by revenue contribution using the Pareto principle. Tier thresholds follow the APICS 80/15/5 standard:

- A tier: SKUs whose combined revenue fills the first 80% of `Total Revenue`
- B tier: next 15% (80–95% cumulative)
- C tier: remaining 5%

Source: APICS CPIM Body of Knowledge.

**Time Window:**
12-month trailing window anchored to `MAX(FactOrders[Order Date])`. Using `MAX(DimDate[Date])` would produce an empty classification window because `DimDate[Date]` includes 1-year padding beyond the most recent transaction per Entry #2. APICS guidance is a minimum one full demand cycle; 12 months is the most widely cited standard in mid-market enterprise operations.

**12-month window for this dataset:** Feb 1, 2017 → Jan 31, 2018.

**Inactive SKU Handling:**
Products with zero or blank revenue in the trailing window are flagged as **Inactive** and excluded from classification. Inactive SKUs carry no rank and no cumulative revenue percentage. 

**Dense Tie Handling:**
`SKU Rank by Revenue` uses Dense ranking. Two products in this dataset share identical 12-month trailing revenue, producing a maximum displayed rank of 116 against 118 total active products. With Dense ranking, both receive the same rank and the next rank increments by one. The two tied products are identifiable with a count of 2 using `SKU Count at Rank (Tie Check)` in `FactOrders[_Validation]`. 

**Two-Measure Pattern:**
`Revenue by SKU (12-Month Trailing)` is a helper measure with locked context `ALL(DimDate)` and `ALL(DimCustomer)` removed so inventory classification reflects total demand. `[Total Revenue]` remains the general-purpose analytical measure respecting all filter context.

**ABC XYZ Decoupling:**
ABC and XYZ answer different questions and must drive different outputs. Conflating them produces incorrect reorder quantities, e.g. a high-price slow-mover classified A-tier by revenue needs a larger safety stock buffer, not more frequent orders.

| Dimension | Question                            | Outputs                                       |
| --------- | ----------------------------------- | --------------------------------------------- |
| ABC       | How much does a stockout cost?      | Service level Z-score → `Safety Stock` buffer |
| XYZ       | How predictably does this SKU sell? | How much to reorder → Reorder Quantity        |

**Safety Stock Formula — Combined Variance (APICS Standard):**

```
Safety Stock = Z × SQRT((Avg LT × σ²_demand) + (Avg demand² × σ²_LT))
```

Safety stock covers demand variance above average inventory consumption during lead time.

**XYZ Classification - Coefficient of Variation:**
XYZ uses Coefficient of Variation to measure relative demand predictability: 

```
CV = daily demand std dev ÷ avg daily demand 
```

CV normalizes variability across SKUs of different sales volumes, so a high-volume SKU with large absolute demand swings is not classified as more erratic than a low-volume SKU with smaller but proportionally equivalent swings. Without this normalization, high-volume SKUs would be pushed into Z tier simply because they sell more. Their reorder quantities would be cut and their inventory coverage reduced, creating unnecessary stockout risk on high-demand products.

**Why Standard CV Thresholds Don't Apply:**
Standard thresholds (CV ≤ 0.5 = X, 0.5–1.0 = Y, > 1.0 = Z) assume moderate-to-high velocity demand. 81% of this dataset sell less than 1 unit per day. Near-zero average demand mathematically produces very high CV values, classifying a significant majority of this catalog's SKUs as Z with no useful segmentation. Thresholds set at 25th and 75th percentiles of actual CV distribution guarantee a meaningful three-way split relative to this dataset's actual behavior.

| Class | CV Threshold | SKU Share                         |
| ----- | ------------ | --------------------------------- |
| X     | ≤ 3.96       | Bottom 25% (most stable)          |
| Y     | 3.96–10.91   | Middle 50% (moderate variability) |
| Z     | > 10.91      | Top 25% (most erratic)            |

**Validated CV Distribution:**

| Percentile | Coefficient of Variation |
| ---------- | ------------------------ |
| 25th       | 3.96                     |
| 50th       | 7.81                     |
| 75th       | 10.91                    |
| 90th       | 17.95                    |

**Dual Implementation: Measure and Calculated Column:**
Both `ABC Tier` and `XYZ Classification` follow the same dual implementation pattern: a `DAX` measure for dynamic filter context in visuals and downstream DAX, and a calculated column in `DimProduct` for slicers and row-level grouping: 

- Measure evaluates dynamically in filter context used in visuals and downstream DAX
- Calculated column stored as `DimProduct` dimension attribute at refresh required for slicers and row-level filtering
- Both refresh in the same operation immediately post-refresh
- Between refreshes a gap can exist, but is acceptable because ABC tiers are reviewed periodically, not in real time

**ISINSCOPE Guard:**
`ABC Tier`, `Cumulative Revenue %`, and `SKU Rank by Revenue` wrapped with `ISINSCOPE(DimProduct[Product Name])` to return BLANK at subtotal and total rows. All downstream replenishment measures (`Safety Stock, Reorder Point, Reorder Flag, Reorder Quantity`) require `Product Name` grain as a direct consequence. See Entry #20.

---

## Entry #18 — Core Measures

**Date:** March 2026  

**Layer:** `_Measures\Core Measures` (DAX measures)

| Measure               | DAX Pattern                               | Note                                                              |
| --------------------- | ----------------------------------------- | ----------------------------------------------------------------- |
| `Total Revenue`       | `SUM(FactOrders[Sales])`                  | Base revenue measure (all downstream calculations reference this) |
| `Total Gross Profit`  | `SUM(FactOrders[Order Gross Profit])`     | Enables implied COGS per Limitation #2                            |
| `Total Units Sold`    | `SUM(FactOrders[Order Quantity])`         | Demand volume                                                     |
| `Total Orders`        | `DISTINCTCOUNT(FactOrders[Order Id])`     | Distinct orders (not line item grain). See Entry #13.             |
| `Avg Order Value`     | `DIVIDE([Total Revenue], [Total Orders])` | `DIVIDE` handles division by zero safely                          |
| `Avg Profit Margin %` | `AVERAGE(FactOrders[Profit Ratio])`       | Raw 0–1 field, formatted as %                                     |

**Validated Totals:**

| Measure             | Value          |
| ------------------- | -------------- |
| Total Revenue       | $36,784,734.31 |
| Total Gross Profit  | $3,966,902.97  |
| Total Units Sold    | 384,079        |
| Total Orders        | 65,752         |
| Avg Order Value     | $559.45        |
| Avg Profit Margin % | 12.06%         |

---

## Entry #19 — Supply Performance Measures

**Date:** March 2026  

**Layer:** DAX measures — `_Measures\Supply Performance`

| Measure                      | Format          | Validated Total |
| ---------------------------- | --------------- | --------------- |
| `Avg Lead Time (Actual)`     | Decimal, 2dp    | 3.50 days       |
| `Avg Lead Time (Scheduled)`  | Decimal, 2dp    | 2.93 days       |
| `Avg Lead Time Variance`     | Decimal, 2dp    | 0.57 days       |
| `Lead Time Variance Std Dev` | Decimal, 2dp    | 1.49 days       |
| `Late Delivery Rate %`       | Percentage, 2dp | 57.3%           |
| `On-Time Delivery Rate %`    | Percentage, 2dp | 42.7%           |

**Canceled Order Exclusion:**
All six measures filter `Delivery Status <> "Shipping canceled"`. Canceled orders never shipped so including them would distort lead time averages toward zero and inflate the on-time rate. `Delivery Status` chosen over `Order Status` because it captures shipping outcome specifically, not order processing status.

| Status            | Row Count   |
| ----------------- | ----------- |
| Late delivery     | 98,977      |
| Advance shipping  | 41,592      |
| Shipping on time  | 32,196      |
| Shipping canceled | 7,754       |
| **Total**         | **180,519** |

Non-canceled orders used in calculations: 172,765.

**STDEV.P over STDEV.S:**
180,519 rows is the full population of orders, not a sample. `STDEV.P` is statistically correct. `STDEV.S` overstates variance producing an inflated `Lead Time Variance Std Dev` that feeds directly into the Safety Stock formula and produces excess safety stock.

**On-Time Delivery Rate % derivation:**
`1 - [Late Delivery Rate %]` is not independently calculated. It inherits the canceled order filter from its source measure, ensuring the two rates are always mathematically complementary with filter logic maintained in one place. See Entry #4.

**Data quality note:**
Late delivery rate of 57.3% reflects actual `DataCo` source dataset characteristics. It is not a measure error. See Limitation #6.

---

## Entry #20 — Inventory Planning: Safety Stock and Replenishment Design

**Date:** March 2026  

**Layer:** `_Measures\Inventory Planning` (DAX measures); `DimProduct` (calculated columns)

**Safety Stock Formula - Combined Variance (APICS Standard):**

```
Safety Stock = Z × SQRT((Avg LT × σ²_demand) + (Avg demand² × σ²_LT))
```

Chosen over simpler single-variance formulas because this dataset has both demand variability and lead time variability. Using only one dimension understates the true buffer required. Source: APICS CPIM Body of Knowledge.

**ABC Tier Z-Score:**

```
Higher Revenue Tier = Higher Opportunity Cost per Stockout = Higher Service Level Target
```

| Tier | Service Level | Z Score |
| ---- | ------------- | ------- |
| A    | 95%           | 1.65    |
| B    | 90%           | 1.28    |
| C    | 85%           | 1.04    |

**Safety Stock Grain Constraint:**
`Safety Stock` references `[ABC Tier]` which uses `ISINSCOPE(DimProduct[Product Name])`. This propagates BLANK at category, department, and total grain through `Safety Stock` → `Reorder Point` → `Reorder Flag` → `Reorder Quantity`. This is intentional. Safety stock is a per-SKU parameter. A category-level aggregate would mislead warehouse operators into thinking a single buffer covers all SKUs in a category. All report pages using these measures must include `Product Name` as a row dimension.

**Demand Std Dev Correction:**
Original `Demand Std Dev` used `STDEV.P(FactOrders[Order Quantity])` which measures how customers place orders, not how daily demand fluctuates. A corporate buyer placing one order for 50 units produces the same total daily demand as five buyers each ordering 10 units, but very different order-level standard deviations. The original measure would flag the first scenario as low variability when the daily demand pattern — one spike, 29 days of zero — is actually highly volatile. 

Safety stock answers: *how much buffer do I need when daily demand exceeds the average during supply lead time?*. That requires daily demand variability, not transaction size variability. `Demand Std Dev (Daily)` uses `STDEVX.P` over a daily aggregated demand table. The deprecated measure is retained in `FactOrders[_Deprecated]` to document the correction. 

**Reorder Quantity:**

```
Reorder Quantity = (Target Days × Avg Daily Demand) + Safety Stock − Current Inventory
```

How much to reorder is driven by XYZ demand predictability. X-tier SKUs hold 60 days of stock on hand; Y-tier 45 days; Z-tier 30 days. Erratic demand SKUs hold less stock to avoid overcommitting inventory to SKUs that may stop selling unexpectedly. 

**Max order cap = 1.5× target days**. Source: APICS min/max methodology.

**XYZ CV Thresholds: Why Standard Values Don't Apply:**
Standard XYZ thresholds (CV ≤ 0.5 = X, 0.5–1.0 = Y, > 1.0 = Z) are designed for catalogs with moderate-to-high velocity demand. 81% of this catalog sells less than 1 unit per day. Near-zero average demand mathematically inflates CV. A product averaging 0.03 units/day with normal day-to-day variation produces CV values of 5–15. Applying standard thresholds classifies ~95% of SKUs as Z, producing no useful segmentation.

Thresholds set at 25th and 75th percentiles of actual CV distribution guarantees a meaningful three-way split relative to this catalog's behavior:

| Class | CV Threshold                      | Interpretation                    | Replenishment                           |
| ----- | --------------------------------- | --------------------------------- | --------------------------------------- |
| X     | ≤ 3.96 (25th percentile)          | Stable, predictable - bottom 25%  | 60-day target - infrequent large orders |
| Y     | 3.96–10.91 (25th–75th percentile) | Moderate variability - middle 50% | 45-day target - balanced replenishment  |
| Z     | > 10.91 (75th percentile)         | Erratic, unpredictable - top 25%  | 30-day target - frequent small orders   |

CV percentile distribution (validated): 25th = 3.96, 50th = 7.81, 75th = 10.91, 90th = 17.95.

**Known Limitations:**

- Target days of supply (X=60, Y=45, Z=30) are industry standard starting points, not optimized for this catalog's specific costs. A production system would tune these using actual carrying and ordering cost data.
- For the 81% of SKUs selling less than 1 unit per day in this catalog, suggested reorder quantities are too small to differentiate meaningfully between tiers. Use these as a starting reference, not as a precise order instruction. 
- Periodic review is the correct model for slow-moving SKUs. See Limitation #7.

---

## Entry #21 — Simulated Inventory: Testing Infrastructure

**Date:** March 2026  

**Layer:** `DimProduct[Simulated Inventory Level]` (Calculated column)

No on-hand inventory snapshots exist in the `DataCo` source dataset (See Limitation #1). Without real inventory levels, `Reorder Flag`, `Stock Coverage`, and `Reorder Quantity` cannot be validated operationally. A `Simulated Inventory Level` column was added to `DimProduct` for testing purposes only.

**Why Deterministic Over Random Values**
`Number.RandomBetween` regenerates on every refresh, so simulated values would change each session, making it impossible to verify reorder flag logic against stable inputs. `MOD(Product Card Id, SafeRange)` assigns a consistent, demand-proportionate inventory level per product, where `UpperBound` = 60 days of supply at that SKU's average daily demand. Upper bound scales to each SKU's demand velocity: a high-velocity SKU (50 units/day) produces a range of 0–3,000; a slow-moving SKU (0.5 units/day) produces a range of 0–30. Stable, auditable, and reproducible across refreshes.

`SafeRange = IF(UpperBound > 0, UpperBound, 1)` guards against division by zero for near-zero demand SKUs. `MAX()` cannot accept scalar VAR arguments in calculated column context, so an IF guard is required instead.

**Production Replacement**
In production, `Simulated Inventory Level` is replaced by a live WMS/ERP on-hand snapshot. `Reorder Flag`, `Stock Coverage (Days)`, and `Reorder Quantity` require no DAX changes. Only the source column changes.

---

## Entry #22 — Cycle Count Schedule: ABC XYZ Combined Matrix

**Date:** March 2026  

**Layer:** Calculated column — `DimProduct[Cycle Count Schedule]`

**Decision:**
Implement cycle count frequency as a calculated column in `DimProduct` using the combined ABC XYZ matrix. Cycle count frequency is proportional to both revenue contribution (ABC) and demand variability (XYZ).

**Rationale:**  
Cycle counting is one of the most operationally tangible outputs of inventory segmentation. The combined matrix operationalizes both classification dimensions simultaneously. ABC tells you how costly an inventory error would be; XYZ tells you how likely a discrepancy is to exist.

**Why Combined ABC XYZ Matrix Over ABC-Only:**
ABC-only cycle counting (count A-items most frequently) is the baseline standard. Adding XYZ increases precision. Erratic demand SKUs are more likely to have system-to-physical discrepancies because unpredictable consumption creates more opportunities for unrecorded variance. An AZ product (high revenue, erratic demand) warrants weekly counting due to maximum revenue exposure combined with maximum discrepancy probability. A CX product (low revenue, stable demand) warrants semi-annual counting considering low cost of error and low probability of discrepancy.

Source: APICS CPIM Body of Knowledge recommends cycle count frequency proportional to revenue contribution and demand variability.

**Why X and Y Share the Same Count Frequency at Every ABC Tier:**
Standard XYZ thresholds assume moderate-to-high velocity demand where a CV of 0.8 represents a meaningful absolute swing in daily units. This catalog does not fit that profile -- 81% of SKUs sell less than 1 unit per day. At near-zero average demand, CV is mathematically inflated regardless of actual behavioral variability, and standard thresholds would classify approximately 95% of the catalog as Z (see Entry #20).

Thresholds were set at the 25th and 75th percentiles of this catalog's actual CV distribution instead: X at or below 3.96, Y between 3.96 and 10.91, Z above 10.91. Within that structure, the absolute unit volume difference between an X and Y SKU in this catalog is too small to produce meaningfully different discrepancy accumulation or detection risk. Z is the operationally significant threshold -- where CV exceeds 10.91 and movement is genuinely erratic at whatever velocity the SKU carries. X/Y parity is a data-derived conclusion, not a simplification.

**ABC XYZ Cycle Count Matrix:**

|                           | X (Stable, CV ≤ 3.96) | Y (Moderate, CV 3.96–10.91) | Z (Erratic, CV > 10.91) |
| ------------------------- | --------------------- | --------------------------- | ----------------------- |
| **A** (top 80% revenue)   | Monthly               | Monthly                     | **Weekly**              |
| **B** (80–95% revenue)    | Quarterly             | Quarterly                   | Monthly                 |
| **C** (bottom 5% revenue) | Semi-Annual           | Semi-Annual                 | Quarterly               |
| **Inactive**              | Annual                | Annual                      | Annual                  |

| Combination | Frequency   | Rationale                                                                                                    |
| ----------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| AZ          | Weekly      | Maximum revenue exposure AND maximum demand unpredictability - highest risk of costly undetected discrepancy |
| AX / AY     | Monthly     | High revenue justifies frequent counts regardless of demand stability                                        |
| BZ          | Monthly     | Moderate revenue + erratic demand - elevated frequency warranted                                             |
| BX / BY     | Quarterly   | Moderate revenue, manageable risk with quarterly review                                                      |
| CZ          | Quarterly   | Low revenue but erratic - periodic check prevents error accumulation                                         |
| CX / CY     | Semi-Annual | Low revenue, stable demand - minimal discrepancy risk                                                        |
| Inactive    | Annual      | Outside active classification scope - annual physical audit sufficient                                       |

**Implementation:**  
The DAX calculated column `Cycle Count Schedule` concatenates `ABC Tier (Classification)` and `XYZ Classification (DimColumn)` into a matrix key ("AX", "BZ", etc.) and uses `SWITCH` to return the frequency label. Updates automatically on model refresh when tier assignments change.

---

## Limitation Log

| #   | Limitation                                                                          | Impact                                                                                                                      | Mitigation                                                                                                                                                                                                                                                                                                      |
| --- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | No on-hand inventory snapshots in source data                                       | True inventory turnover cannot be calculated. Reorder Flag and Stock Coverage cannot use real inventory levels              | Demand velocity used as proxy. Simulated Inventory Level added for model testing (Entry #21). In production, replace with live WMS/ERP snapshot. Documented in Executive Summary                                                                                                                                |
| 2   | No raw COGS column in source data                                                   | Cannot directly sum cost of goods as a native sourced field                                                                 | Implied COGS derivable: `Sales − Order Gross Profit`. Revenue-based ABC classification method used as primary industry standard approach                                                                                                                                                                        |
| 3   | Late Delivery Risk was Machine Learning target variable                             | Source flag not calculated from actual shipping performance                                                                 | Replaced with transparent Lead Time Variance derivation (Entry #3)                                                                                                                                                                                                                                              |
| 4   | Product Description empty across all products                                       | Cannot build product detail page                                                                                            | Column excluded                                                                                                                                                                                                                                                                                                 |
| 5   | Single flat file source. No supplier dimension                                      | Cannot segment lead time by supplier                                                                                        | Shipping Mode used as a substitute                                                                                                                                                                                                                                                                              |
| 6   | Late delivery rate of 57.3% in source data                                          | Elevated rate may concern report consumers                                                                                  | Reflects actual DataCo dataset characteristics (not a measure error). Documented on QA page                                                                                                                                                                                                                     |
| 7   | Continuous review replenishment model applied to predominantly slow-moving products | Suggested reorder quantities are too small to differentiate meaningfully between tiers for 81% of SKUs selling < 1 unit/day | Use as directional starting reference. Periodic review is the production-grade solution for slow movers. Target days (60/45/30) are industry standard starting points, not data-derived from cost structure. Full optimization requires actual holding cost and ordering cost per SKU. Documented in Entry #20. |
| 8   | No Shipped Quantity column in source data | In Full component of OTIF cannot be independently measured | Order Quantity used as proxy in OTIF % — assumes all orders shipped in full. In production: replace the right side of the In Full condition with `FactOrders[Shipped Quantity]`. No other formula changes needed. See Entry #28. |

---

## Entry #23 — Core Measures: Canceled Order Exclusion

**Date:** April 2026
**Layer:** DAX Layer — Core Measures folder (FactOrders)

### Finding: Canceled Orders Carrying Revenue and Unit Values

Validation confirmed 7,754 rows with `Delivery Status = "Shipping canceled"` carry non-zero values: 16,488 canceled units and $1,570,305.33 in canceled revenue across the dataset. The base model included these in all Core Measures aggregations — overstating Total Revenue, Total Gross Profit, Total Units Sold, Total Orders, and Avg Profit Margin %.

Four permanent validation measures added to `_Validation` to document and reproduce these findings: `Canceled Line Items` (expected 7,754), `Canceled Order Units` (expected 16,488), `Orders With Canceled Lines` (expected 2,855), `Orders With Mixed Lines` (expected 0). Any reviewer cloning the repo can refresh and verify these figures independently.

### Finding: Delivery Status is the Correct Exclusion Filter

`Delivery Status = "Shipping canceled"` is the correct and complete exclusion filter. It catches CANCELED, SUSPECTED_FRAUD, and any other Order Status that was never fulfilled. Validation confirmed: COUNTROWS(FILTER(FactOrders, Order Status = "CANCELED" && Delivery Status <> "Shipping canceled")) returned BLANK — all CANCELED order status rows have Delivery Status = "Shipping canceled". Using Order Status alone would miss rows.

### Finding: No Mixed-Line Orders

Orders With Mixed Lines returned 0. Every order is either fully canceled or fully fulfilled. Line-item-level exclusion at FactOrders grain correctly excludes all 2,855 canceled orders and their 7,754 line items.

### Decision: Exclude Canceled Orders from All Core Measures

Canceled orders were never fulfilled. Including them represents ordered value not fulfilled value — operationally inaccurate and not aligned with inventory planning intent.

Five measures updated: `Total Revenue`, `Total Gross Profit`, `Total Units Sold`, `Total Orders`, `Avg Profit Margin %`. `Avg Order Value` cascades automatically from Total Revenue and Total Orders — no DAX changes required.

### KEEPFILTERS: Preserving Slicer Responsiveness

Initial implementation used `CALCULATE(..., FactOrders[Delivery Status] <> "Shipping canceled")` directly. This broke slicer responsiveness. When a Delivery Status slicer is active, CALCULATE replaces the existing filter context on that column entirely — the slicer is ignored.

Fix: `KEEPFILTERS` wraps the exclusion condition in all CALCULATE blocks:

```dax
CALCULATE(
    SUM(FactOrders[Sales]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

KEEPFILTERS intersects the new condition with existing context rather than replacing it. With a "Late delivery" slicer active, the result is `Late delivery AND not Shipping canceled = Late delivery only` — slicer respected, canceled orders excluded. With a "Shipping canceled" slicer, the intersection is empty — returns BLANK, which is correct.

### Downstream Cascade

`Revenue by SKU (12-Month Trailing)` calls `[Total Revenue]` inside a CALCULATE that only modifies DimDate and DimCustomer context. KEEPFILTERS on Delivery Status persists through nested CALCULATE calls — canceled orders are automatically excluded from the trailing window calculation and all dependent measures: ABC tier assignments, SKU revenue ranking, cumulative revenue %, XYZ classification, safety stock, and all replenishment measures.

### Validated Baselines (Post-Exclusion)

| Measure | Value |
|---|---|
| Total Revenue | $35,214,428.98 |
| Total Gross Profit | $3,806,420.63 |
| Total Units Sold | 367,591 |
| Total Orders | 62,897 |
| Avg Order Value | $559.87 |
| Avg Profit Margin % | 12.08% |

### QA: Cross-Check Validation (Scaffolding Removed)

Three ephemeral validation measures were built to verify the canceled order exclusion and window calculation logic. Removed after confirmation — ephemeral QA scaffolding, not permanent regression checks. Findings recorded here as the audit trail.

- `Total Revenue Trailing 12M (Check)` — direct 12-month window calculation over `[Total Revenue]` with `ALL(DimProduct)`. Confirmed match with `Revenue by SKU Check`: **$10,609,791.02**
- `Revenue by SKU Check` — `[Revenue by SKU (12-Month Trailing)]` summed across `ALL(DimProduct)`. Confirmed match with `Total Revenue Trailing 12M (Check)`: **$10,609,791.02**
- `Total Revenue (Canceled)` — confirmed canceled revenue amount excluded from `[Total Revenue]`: **$1,570,305.33**

All three confirmed. The window logic and KEEPFILTERS cascade are working correctly post-exclusion.

---

## Entry #24 — Supply Performance: Canceled Order Exclusion, FILTER Guard, Count-Based Rate Measures

**Date:** April 2026
**Layer:** DAX Layer — Supply Performance folder (FactOrders)

### Finding: Plain CALCULATE Overrides Delivery Status Filter Context

The base Supply Performance measures used `CALCULATE(..., FactOrders[Delivery Status] <> "Shipping canceled")`. CALCULATE does not add to existing filter context on the same column — it replaces it. At row grain in a canceled-only visual context (e.g. a Delivery Status slicer set to "Shipping canceled"), CALCULATE discards the canceled-only context and computes across all non-canceled orders, returning a misleading value instead of BLANK.

This is distinct from the Core Measures issue in Entry #23. Core Measures are aggregations — the slicer override causes incorrect values at the total level. Supply Performance measures also compute at row grain in product-level visuals — the override produces a wrong result in a cell that should be BLANK.

### Decision: NonCanceledCount FILTER Guard on All Six Measures

A `NonCanceledCount` VAR was added to all six Supply Performance measures:

```dax
VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )
```

FILTER respects existing context — it does not override it. When context is canceled-only, FILTER finds zero matching rows, COUNTROWS returns 0, and the measure returns BLANK. This is the correct behavior — no delivery performance data exists for orders that never shipped.

FILTER-based conditions do not require KEEPFILTERS. FILTER evaluates within existing context natively. Only CALCULATE-based Delivery Status arguments need KEEPFILTERS.

### Decision: KEEPFILTERS on All CALCULATE-Based Delivery Status Arguments

All CALCULATE-based Delivery Status exclusion arguments in the RETURN block were wrapped with KEEPFILTERS:

```dax
CALCULATE(
    ...,
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

KEEPFILTERS intersects the new filter with existing context rather than replacing it. With a "Late delivery" slicer active, the result is `Late delivery AND not Shipping canceled = Late delivery only` — slicer respected, canceled orders excluded. With a "Shipping canceled" slicer, the intersection is empty — returns BLANK, and the NonCanceledCount guard fires first.

### Decision: Count-Based Rewrite for Late Delivery Rate % and On-Time Delivery Rate %

`Late Delivery Rate %` was rewritten from `AVERAGE(FactOrders[Late Delivery Risk])` to `LateOrders / TotalOrders`. The two formulas are mathematically identical — AVERAGE of a 0/1 field equals the proportion of 1s. The count-based pattern is used for consistency with the OTIF % formula structure that will be added in a future entry.

`On-Time Delivery Rate %` was rewritten from `1 - [Late Delivery Rate %]` to an independent count-based formula with its own NonCanceledCount guard. `1 - BLANK()` evaluates to 100% in DAX because arithmetic on BLANK treats it as 0, not BLANK. BLANK does not propagate through arithmetic — each derived measure requires its own explicit guard. The inherited formula from v1.2 would return 100% for every canceled row in a row-level visual.

### Validated Baselines (Post-Entry #24, Unchanged)

| Measure | Value |
|---|---|
| Late Delivery Rate % | 57.3% |
| On-Time Delivery Rate % | 42.7% |
| Avg Lead Time (Actual) | 3.50 days |
| Avg Lead Time (Scheduled) | 2.93 days |
| Avg Lead Time Variance | 0.57 days |
| Lead Time Variance Std Dev | 1.49 days |

---

## Entry #25 — Inventory Planning: Canceled Order Exclusion

**Date:** April 2026
**Layer:** DAX Layer — Inventory Planning folder (FactOrders)

### Finding: Canceled Units Inflate Demand Inputs

`Avg Daily Demand by SKU` and `Demand Std Dev (Daily)` both use trailing 12-month window aggregations over `FactOrders[Order Quantity]`. Prior to this entry, both measures included canceled rows. Validated dataset: 16,488 canceled units across 7,754 rows. Including them overstates average daily demand and distorts the standard deviation of daily demand, which inflates Safety Stock, Reorder Point, Reorder Quantity, and shifts XYZ Classification thresholds.

### Decision: KEEPFILTERS on Avg Daily Demand by SKU TotalUnits CALCULATE Block

The `TotalUnits` VAR inside `Avg Daily Demand by SKU` uses `CALCULATE` with multiple filter arguments (DimDate window, ALL(DimCustomer)). A plain `Delivery Status <> "Shipping canceled"` argument inside CALCULATE would override any existing Delivery Status slicer context. KEEPFILTERS wraps the exclusion to intersect with rather than replace that context:

```dax
VAR TotalUnits =
    CALCULATE(
        SUM(FactOrders[Order Quantity]),
        DimDate[Date] > WindowStart,
        DimDate[Date] <= WindowEnd,
        ALL(DimCustomer),
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
    )
```

`WindowDays` counts calendar days from DimDate, not order rows — no Delivery Status filter needed there.

### Decision: Plain Condition Inside FILTER Predicate for Demand Std Dev (Daily)

`Demand Std Dev (Daily)` builds `DailyDemandTable` via `CALCULATETABLE(SUMMARIZE(FILTER(...)))`. The canceled order exclusion belongs inside the FILTER predicate alongside the existing date window conditions:

```dax
FILTER(
    FactOrders,
    FactOrders[Order Date] > WindowStart &&
    FactOrders[Order Date] <= WindowEnd &&
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

FILTER evaluates within existing context without overriding it — KEEPFILTERS is not needed here. Using KEEPFILTERS inside a FILTER predicate would be a category error: KEEPFILTERS is a CALCULATE modifier, not a boolean predicate.

### Cascade: No Changes Required to Dependent Measures

Excluding canceled orders from `Avg Daily Demand by SKU` and `Demand Std Dev (Daily)` cascades automatically through every downstream measure — `Coefficient of Variation`, `XYZ Classification`, `Safety Stock`, `Reorder Point`, `Reorder Quantity`, `Reorder Flag`, `Stock Coverage (Days)`. No formula changes needed to any of these measures.

`Revenue by SKU (12-Month Trailing)` calls `[Total Revenue]` directly, which already carries KEEPFILTERS canceled order exclusion from Entry #23. No changes needed.

---

## Entry #26 — Simulated Inventory Level Redesign

**Date:** April 2026
**Layer:** DAX Layer — DimProduct Inventory Segmentation (calculated columns)

### Finding: Original Design Produced Uneven Bucket Distribution

The original `Simulated Inventory Level` used `MOD([Product Card Id], SafeRange)` where `SafeRange` was derived from each SKU's average daily demand. `Product Card Id` is a non-sequential arbitrary identifier — MOD of arbitrary integers against a variable range does not guarantee even bucket distribution. The three-state distribution (Stockout, Reorder Now, Stock OK) could not be controlled or validated, and the simulation range scaled to demand velocity rather than to the reorder logic being tested.

### Decision: Three-State ROP-Anchored Design

The redesign replaces the velocity-proportionate approach with three explicit states anchored to `[Reorder Point]`:

- **Stockout** (Bucket 0): inventory = 0. Tests the stockout escalation path in `Reorder Flag`.
- **Reorder Now** (Buckets 1–3): inventory = `CEILING(ROP * 0.5, 1)`. Places inventory at 50% of ROP — below threshold, triggers Reorder Now.
- **Stock OK** (Buckets 4–9): inventory = `CEILING(ROP * 1.5, 1)`. Places inventory at 150% of ROP — above threshold, triggers Stock OK.

ROP-anchored levels mean the simulation is operationally grounded — each SKU's simulated inventory is proportionate to its actual replenishment trigger, not an arbitrary velocity range.

### Decision: SKU Revenue Rank (DimColumn) as MOD Input

`Product Card Id` was replaced by `DimProduct[SKU Revenue Rank (DimColumn)]` as the MOD input. Revenue rank is sequential 1–118 — MOD of a sequential integer against 10 guarantees even bucket distribution. 118 SKUs ÷ 10 buckets = ~11–12 SKUs per bucket, producing the target ~10/30/60 distribution.

`RANKX` with `[Revenue by SKU (12-Month Trailing)]` cannot be used directly inside a calculated column — the measure's fixed context window returns the aggregate rather than the per-row value. A dedicated `SKU Revenue Rank (DimColumn)` calculated column stores the rank at refresh time, sidestepping the context evaluation issue.

### Validated Distribution (Post-Redesign)

| Flag | Expected | Actual |
|---|---|---|
| 🚨 Stockout | ~12 (10%) | 11 (9.3%) |
| ⚠ Reorder Now | ~35 (30%) | 36 (30.5%) |
| ✓ Stock OK | ~71 (60%) | 71 (60.2%) |

Small deviations from exact 10/30/60 are expected — 118 SKUs cannot distribute evenly across 10 buckets (118 ÷ 10 = 11.8). Each bucket receives either 11 or 12 SKUs. Distribution is correct.

### Finding: Revenue Rank Tie Resolved Post Canceled Order Exclusion

Prior to Entry #25, `SKU Count at Rank (Tie Check)` returned 2 — two products shared identical 12-month trailing revenue, producing a max displayed rank of 116 against 118 active products. This was documented as expected behavior in the original model.

After applying canceled order exclusion in Entry #25, the tie resolved. Excluding canceled revenue changed one product's 12-month trailing revenue enough to break the equality. `SKU Count at Rank (Tie Check)` now returns 1 — all 118 products have unique revenue ranks, max rank is 118.

`SKU Count at Rank (Tie Check)` remains active in `_Validation` as a regression check. Any future value > 1 signals a new tie has emerged and max displayed rank has fallen below 118.

---

## Entry #27 — XYZ Threshold Recalibration Post Canceled Order Exclusion

**Date:** April 2026
**Layer:** DAX Layer — Inventory Segmentation (FactOrders measure + DimProduct column)

### Finding: CV Distribution Shifted Upward After Canceled Order Exclusion

After applying canceled order exclusion in Entry #25, the CV distribution across all 118 SKUs shifted upward. Excluding canceled units reduced the demand totals used in `Avg Daily Demand by SKU` and `Demand Std Dev (Daily)`. With canceled units removed, the ratio of variability to mean demand increased for affected SKUs — demand became less predictable relative to its new lower baseline.

| Percentile | Pre-Exclusion | Post-Exclusion | Delta |
|---|---|---|---|
| CV 25th (X/Y boundary) | 3.96 | 3.99 | +0.03 |
| CV 50th | 7.81 | 8.30 | +0.49 |
| CV 75th (Y/Z boundary) | 10.91 | 11.48 | +0.57 |
| CV 90th | 17.95 | 18.67 | +0.72 |

### Decision: Recalibrate Thresholds to Post-Exclusion CV Percentiles

The design principle from Entry #17 is data-driven percentile thresholds, not fixed constants. The thresholds must reflect the actual CV distribution of the post-exclusion model. Updated thresholds:

- X: CV ≤ **3.99** (25th percentile)
- Y: CV 3.99 – **11.48** (25th–75th percentile)
- Z: CV > **11.48** (above 75th percentile)

Applied to both `[XYZ Classification]` (FactOrders measure) and `DimProduct[XYZ Classification (DimColumn)]`.

### Finding: SKU Distribution Shifted — Six SKUs Moved from Y to Z

| Class | Pre-Exclusion | Post-Exclusion |
|---|---|---|
| X | ~30 | 29 |
| Y | ~59 | 54 |
| Z | ~29 | 35 |

Six SKUs shifted from Y to Z after recalibration. This is expected — excluding canceled orders exposed higher demand variability in those SKUs, and the recalibrated Y/Z boundary (11.48 vs 10.91) moved some borderline SKUs into Z. These SKUs will now receive shorter replenishment coverage (30 days vs 45 days in `Reorder Quantity`) and will be monitored more frequently per the cycle count matrix.

---

*Document Version: 3.0 — Phase 1 ETL + Phase 2 Model Layer + Phase 3 DAX Layer*
*Phase 3 Entries #16–#22: QA pages, display folder naming, ABC XYZ segmentation, core measures, supply performance, inventory planning, simulation, cycle count schedule*
*Phase 3 Entry #23: Core Measures canceled order exclusion — KEEPFILTERS pattern, validated 7,754 canceled rows / 16,488 units / 0 mixed-line orders*
*Phase 3 Entry #24: Supply Performance canceled order exclusion — NonCanceledCount FILTER guard, KEEPFILTERS on all CALCULATE blocks, count-based rewrites for Late Delivery Rate % and On-Time Delivery Rate %, 1 - BLANK() fix*
*Phase 3 Entry #25: Inventory Planning canceled order exclusion — KEEPFILTERS on Avg Daily Demand TotalUnits CALCULATE block, plain FILTER predicate condition on Demand Std Dev (Daily), deprecated references fixed in Coefficient of Variation, XYZ Classification (DimColumn), Cycle Count Schedule, SKU Count - Unclassified, cascade through all dependent measures confirmed*
*Phase 3 Entry #26: Simulated Inventory Level redesign — three-state ROP-anchored design, SKU Revenue Rank (DimColumn) as MOD input, validated 11/36/71 distribution, rank tie resolved post Entry #25 exclusion*
*Phase 3 Entry #27: XYZ threshold recalibration — thresholds updated from 3.96/10.91 to 3.99/11.48, CV distribution shift documented, SKU distribution change confirmed*
*Next Update: OTIF % added to Supply Performance*