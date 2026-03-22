# Decision Log
## ABC Product Segmentation & Inventory Management
### Power BI Project — Phases 1–2 (ETL + Model Layer)

---

> This log explains the main build decisions for this project, why each decision was made, and how each decision affects reporting and operations.

---

## Phase 1 — ETL Data Preparation Layer

## Entry #1 — Single Source Data Staging (MasterSet)

**Date:** March 2026<br>
**Layer:** Power Query `MasterSet`

**Decision:**
Use `MasterSet` as one central source table. All report tables pull from it, and `MasterSet` itself is not loaded into the final report model.

**Context:**
Initial build had all five queries ( `DimDate, DimProduct, DimCustomer, FactOrders, MasterSet` ) each loading the source CSV independently. This meant the CSV was being read five times on every refresh, creating five separate file-read operations on a 180K+ row dataset.

**What Changed:**
- Designated MasterSet as the sole CSV loading query
- Set MasterSet `Enable Load = False` removes it from the model entirely
- `DimProduct, DimCustomer, FactOrders` set as reference queries: `Source = MasterSet`
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
- CSV loads exactly once per refresh - no redundant file reads
- Single source of truth - source schema changes propagate automatically
- MasterSet does not appear in the published model - cleaner field list
- Zero-touch updates when source data is refreshed or extended

---

## Entry #2 — Auto-Updating DimDate Table with 1-Year Buffer

**Date:** March 2026<br>
**Layer:** Power Query `DimDate`

**Decision:**
Generate DimDate dynamically from MasterSet min/max dates with 1-year padding on each boundary rather than using hardcoded start and end dates.

**Implementation:**
```m
MinDate = Date.From(List.Min(MasterSet[#"order date (DateOrders)"]))
MaxDate = Date.From(List.Max(MasterSet[#"order date (DateOrders)"]))
StartDate = Date.StartOfYear(Date.AddYears(MinDate, -1))
EndDate = Date.EndOfYear(Date.AddYears(MaxDate, 1))
```

**Source Data Range:** Jan 6, 2015 → Jan 31, 2018
**Generated Range:** Jan 1, 2014 → Dec 31, 2019
**Total Rows:** 2,191 contiguous daily dates

**Benefits:**
- Zero hardcoded dates. Model self-maintains as source data grows
- Padding ensures time intelligence coverage beyond transaction boundaries
- If source data extends into 2019, DimDate automatically extends to Dec 31, 2020

---

## Entry #3 — Dynamic Late Delivery Risk Rule

**Date:** March 2026<br>
**Layer:** Power Query `FactOrders`

**Decision:**
Replace the source `Late_delivery_risk` column with a simple check based on actual shipping delay. If `Lead Time Variance > 0`, mark as `late risk`.

**Context:**
The DataCo source data includes a `Late_delivery_risk` flag. Review showed it was a pre-labeled machine learning training field created by the dataset authors, not a field calculated from actual shipping performance. Empirical testing found 5 mismatches per 1,000 rows between the source flag and actual Lead Time Variance outcomes, confirming the flag was generated independently from the data it appears to describe.

**What Changed:**
- Removed the source `Late_delivery_risk` column
- Replaced with two derived columns:
  - `Late Delivery Risk` (Whole Number) - `1` if `Lead Time Variance > 0`, else `0` for DAX calculations
  - `Is Late Delivery Risk` (True/False) - `True` if `Lead Time Variance > 0` for report filters and visuals

**Formula:**
```
Late Delivery Risk = if [Lead Time Variance] > 0 then 1 else 0
Is Late Delivery Risk = [Lead Time Variance] > 0
```

**Why This Is Better:**
- Fully transparent logic is auditable by any developer
- Dynamically recalculates when source data is refreshed
- Derived from actual shipping performance
- Two columns serve both analytical and presentation layers

---

## Entry #4 — Two Late-Risk Fields for Different Uses

**Date:** March 2026<br>
**Layer:** Power Query `FactOrders`

**Decision:**
Maintain two versions of `Late Delivery Risk` column binary flag<br>
1. Numeric integer (0/1) field for DAX calculations 
2. Logical (True/False) field for report filters and visuals

**Context:**
`Late Delivery Risk` needed to serve two purposes simultaneously:
1. DAX calculations - rates, averages, and safety stock formulas
2. Report visuals filtering - slicers, conditional formatting, and KPI cards

One column does not serve both needs well. Numeric values are better for calculations, while True/False is easier for report users to filter and read.

**Pattern Applied:**
- `Late Delivery Risk` (Int64) - `0` or `1` (DAX arithmetic layer)
- `Is Late Delivery Risk` (Logical) - `True` or `False` (Presentation layer)

**Naming Convention:**
`Is` prefix denotes logical boolean columns throughout the model.

**Downstream DAX Usage:**
```dax
// Integer version — arithmetic
Late Delivery Rate = AVERAGE(FactOrders[Late Delivery Risk])

// Logical version — filter
Late Orders = CALCULATE(COUNTROWS(FactOrders), FactOrders[Is Late Delivery Risk] = TRUE())
```

---

## Entry #5 — Consistent Column Ordering Standard

**Date:** March 2026<br>
**Layer:** Power Query (`DimProduct`, `DimCustomer`, `FactOrders`)

**Decision:**
Apply a consistent column order so tables are easier to read, validate, and maintain across model queries:
`Primary Key -> Foreign Keys -> Date Keys -> Attributes -> Measures -> Calculated Fields`.

**Context:**
Without a fixed column order, query outputs become inconsistent, harder to scan, and more error-prone during maintenance when new columns are added.

**What Changed:**
- Reordered columns in each model query to follow the same enterprise sequence
- Grouped business attributes after key fields for faster validation and troubleshooting
- Kept derived and analytical fields at the end to preserve readability during ETL review

---

## Phase 2 — Model Layer (Semantic Model)

## Entry #6 — Sort By Column Assignments

**Date:** March 2026<br>
**Layer:** Model properties — `DimDate`

**Decision:**
In the **model**, set **Sort by column** on `DimDate` text labels so charts and slicers follow calendar order, not A–Z.

**Assignments:**
- `Month Name` → `Month Number`
- `Day Name` → `Day Number`
- `Quarter Label` → `Quarter Number`
- `Year-Month Label` → `Year-Month Key`

**Note:**
Power Query already produced the correct **data** (label column plus matching sort key in the same table). Phase 2 was the step where the **report model** was wired so each label **depends on** its numeric/key column for sorting.

**Rationale:**
Power BI’s default sort for text is alphabetical. Month names like “Apr” and “Jan” sort wrong alphabetically. Calendar labels require an explicit numeric or key sort column for correct axis and slicer behavior. Sort by column tells the product which column defines the intended order.

---

## Entry #7 — Hidden Columns

**Date:** March 2026<br>
**Layer:** Model — all loaded tables

**Decision:**
Hide surrogate keys, foreign keys, and DAX-only columns from **report view** across `FactOrders`, `DimDate`, `DimProduct`, and `DimCustomer`.


**Rationale:**
Hiding these fields keeps day-to-day reporting focused on names, dates, and measures. Hidden columns remain available for relationships and DAX.

---

## Entry #8 — Default Summarization

**Date:** March 2026<br>
**Layer:** Model — `FactOrders` (and related columns as applicable)

**Decision:**
Set **Don’t summarize** on ID columns, date keys, geographic coordinates, and ratio/rate columns where **Sum** is semantically wrong.

**Rationale:**
Prevents implicit aggregation mistakes. Phase 3 DAX measures will own intentional aggregations.

**Note:** Columns such as `Order Discount`, `Order Gross Profit`, `Order Profit Per Order`, `Order Quantity`, `Order Total`, and `Sales` remain summable where additive semantics apply.

---

## Entry #9 — Display Folders on FactOrders

**Date:** March 2026<br>
**Layer:** Model — `FactOrders`

**Decision:**
Organize `FactOrders` columns into seven display folders: **Keys**, **Dates**, **Financial**, **Rates & Ratios**, **Order Details**, **Shipping & Logistics**, **Delivery Risk**.

**Rationale:**
Improves field-list navigation during Phase 3 DAX measure authoring.

---

## Entry #10 — Power Query Groups

**Date:** March 2026<br>
**Layer:** Power Query Editor — query organization

**Decision:**
Group queries into **\_Staging** (`MasterSet`), **Facts** (`FactOrders`), and **Dimensions** (`DimDate`, `DimProduct`, `DimCustomer`).

**Rationale:**
Makes the ETL architecture readable without opening individual queries.

**Note:**
When you create named groups, Power BI sometimes leaves a default **Other Queries** bucket. Refresh behavior and the loaded model are unchanged.

---

## Entry #11 — Date Hierarchies on DimDate

**Date:** March 2026<br>
**Layer:** Model — `DimDate`

**Decision:**
Create two **explicit** hierarchies on `DimDate`:
- **Year–Month–Day:** `Year` → `Month Name` → `Day Name`
- **Year–Qtr–Month:** `Year` → `Quarter Label` → `Month Name`

**Rationale:**
Custom hierarchies match calendar attributes and sort keys, so drill-down order in time-series visuals stays consistent with how the dimension was built.

---

## Entry #12 — Model Validation Suite

**Date:** March 2026<br>
**Layer:** DAX measures — `FactOrders` → **`_Validation`** display folder

**Decision:**
Run a short set of **automated tests** in the model: row totals, order-date range, and **every order row links to a real customer and product** (no orphan keys).

**Organization:**
Seven validation measures live in **`FactOrders`** under the **`_Validation`** display folder. The folder name starts with **`_`** so it sorts to the **top** of the field list and reads as **infrastructure**, not business reporting.

**Tests and results** (values below match the **DataCo** source loaded in this model)

| Test | Expected | Result |
|------|----------|--------|
| Row Count Integrity | 180,519 | Pass |
| Earliest Order Date | Jan 1, 2015 | Pass |
| Latest Order Date | Jan 31, 2018 | Pass |
| Blank Customer Check | 0 | Pass |
| Blank Product Check | 0 | Pass |
| Customer Match Check | 180,519 | Pass |
| Product Match Check | 180,519 | Pass |

**DAX patterns:**  
`RELATED` needs **row context** on `FactOrders`, so row-by-row checks use an iterator such as `FILTER`. The **authoritative** formulas are the measures inside the published `.pbix`; the patterns below are representative.

```dax
-- Row count

Row Count Integrity = COUNTROWS ( FactOrders )

-- Date range over entire fact (ignore visual filters)

Earliest Order Date =
CALCULATE (
    MIN ( FactOrders[Order Date] ),
    ALL ( FactOrders )
)

Latest Order Date =
CALCULATE (
    MAX ( FactOrders[Order Date] ),
    ALL ( FactOrders )
)

-- Example: count rows where dimension lookup is missing (adjust to your exact keys/blanks logic)

Blank Customer Check =
COUNTROWS (
    FILTER (
        FactOrders,
        ISBLANK ( RELATED ( DimCustomer[Customer Id] ) )
    )
)

Blank Product Check =
COUNTROWS (
    FILTER (
        FactOrders,
        ISBLANK ( RELATED ( DimProduct[Product Card Id] ) )
    )
)

-- Example: count rows where lookup succeeds (should equal COUNTROWS(FactOrders) when integrity holds)

Customer Match Check =
COUNTROWS (
    FILTER (
        FactOrders,
        NOT ISBLANK ( RELATED ( DimCustomer[Customer Id] ) )
    )
)

Product Match Check =
COUNTROWS (
    FILTER (
        FactOrders,
        NOT ISBLANK ( RELATED ( DimProduct[Product Card Id] ) )
    )
)
```

---

## Entry #13 — Model Validation Page Preserved

**Date:** March 2026<br>
**Layer:** Report — page **“Model Validation”**

**Decision:**
Keep the **Model Validation** report page in the shipped `.pbix` after all validation **tests pass**.

**Rationale:**
The page is a **reference** for anyone opening the file later: it shows **what was tested** and **the outcomes** without hunting through the measures list.

**Note:** Measure definitions and the `_Validation` folder are documented under **Entry #12**.

---

## Limitation Log

| # | Limitation | Impact | Mitigation |
|---|---|---|---|
| 1 | No on-hand inventory snapshots in source data | True inventory turnover cannot be calculated | Demand velocity used as proxy. Documented in Executive Summary |
| 2 | No Cost of Goods Sold (COGS) field. Only profit and revenue | Cannot calculate margin-based ABC weighting | Revenue-based ABC used. Industry standard approach |
| 3 | Late Delivery Risk was ML target variable | Source flag not calculated from actual shipping performance | Replaced with transparent Lead Time Variance derivation |
| 4 | Product Description empty across all products | Cannot build product detail page | Column excluded |
| 5 | Single flat file source. No supplier dimension | Cannot segment lead time by supplier | Used Shipping Mode as a substitute |

---

*Document Version: 2.0 - Phase 1 ETL & Phase 2 Model Layer Complete*<br>
*Next Update: Phase 3 - DAX Measures Layer*
