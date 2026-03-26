# DAX Measures Reference

## ABC Product Segmentation & Inventory Management
### Power BI Project: Phase 3 (DAX Layer)

---

> This document is the authoritative developer reference for all DAX measures in this project. It covers measure name, home table, display folder, formula, format, dependencies, and implementation notes. **Section and measure order follow the `FactOrders` → `_Measures` display-folder tree** (see below), not DAX dependency order—use **Build Order Reference** when creating measures in Power BI. For architectural decisions behind each measure, see `decision-log.md`. For column definitions and source-to-model mapping, see `column-definition.md`.

---

## Document Conventions

| Convention | Meaning |
|---|---|
| `[Measure Name]` | DAX measure reference |
| `Table[Column]` | Column reference |
| ⚠ | Measure has a known limitation or requires careful interpretation |
| 🔒 | Locked context measure — filter arguments intentionally remove slicers |
| 🧪 | Testing/simulation infrastructure — not a business deliverable |
| ❌ | Deprecated — retained for audit trail only |

---

## Table of Contents

1. [FactOrders — display folder tree](#factorders--display-folder-tree)
2. [\_Measures\_Deprecated](#measures_deprecated)
3. [\_Measures\_Validation](#measures_validation)
4. [\_Measures\ABC Classification](#measuresabc-classification)
5. [\_Measures\Foundation](#measuresfoundation)
6. [\_Measures\Lead Time & Reliability](#measureslead-time--reliability)
7. [\_Measures\Safety Stock & Replenishment](#measuressafety-stock--replenishment)
8. [DimProduct Calculated Columns](#dimproduct-calculated-columns)
9. [Measure Dependency Map](#measure-dependency-map)
10. [Build Order Reference](#build-order-reference)

---

## FactOrders — display folder tree

```text
FactOrders
│
├── _Measures
│   │
│   ├── _Deprecated
│   │   ├── Demand Std Dev (DEPRECATED - order level, not daily level)
│   │   └── Demand Velocity Band (DEPRECATED - superseded by XYZ Classification)
│   │
│   ├── _Validation
│   │   ├── Blank Customer Check
│   │   ├── Blank Product Check
│   │   ├── Customer Match Check
│   │   ├── Delivery Status Row Count
│   │   ├── Earliest Order Date
│   │   ├── Latest Order Date
│   │   ├── Product Match Check
│   │   ├── Row Count Integrity
│   │   ├── SKU Count at Rank (Tie Check)
│   │   ├── Demand 25th Percentile
│   │   ├── Demand 50th Percentile
│   │   ├── Demand 75th Percentile
│   │   └── Demand 90th Percentile
│   │
│   ├── ABC Classification
│   │   ├── ABC Tier
│   │   ├── Coefficient of Variation
│   │   ├── Cumulative Revenue %
│   │   ├── Revenue by SKU (12-Month Trailing)
│   │   ├── SKU Count - A Tier
│   │   ├── SKU Count - B Tier
│   │   ├── SKU Count - C Tier
│   │   ├── SKU Count - Inactive
│   │   ├── SKU Count by Velocity Band
│   │   ├── SKU Rank by Revenue
│   │   └── XYZ Classification
│   │
│   ├── Foundation
│   │   ├── Avg Order Value
│   │   ├── Avg Profit Margin %
│   │   ├── Total Gross Profit
│   │   ├── Total Orders
│   │   ├── Total Revenue
│   │   └── Total Units Sold
│   │
│   ├── Lead Time & Reliability
│   │   ├── Avg Lead Time (Actual)
│   │   ├── Avg Lead Time (Scheduled)
│   │   ├── Avg Lead Time Variance
│   │   ├── Late Delivery Rate %
│   │   ├── Lead Time Variance Std Dev
│   │   └── On-Time Delivery Rate %
│   │
│   └── Safety Stock & Replenishment
│       ├── Avg Daily Demand by SKU
│       ├── Demand Std Dev (Daily)
│       ├── Reorder Flag
│       ├── Reorder Point
│       ├── Reorder Quantity
│       ├── Safety Stock
│       └── Stock Coverage (Days)
```

---

## \_Measures\\_Deprecated

Measures retained for audit trail and analytical evolution documentation. Do not use in report pages or downstream calculations.

---

### Demand Std Dev (DEPRECATED - order level, not daily level)

**Replaced by:** `[Demand Std Dev (Daily)]`<br>
**Reason:** `STDEV.P(FactOrders[Order Quantity])` measures order size variability, not daily demand variability. Distorts safety stock. See Entry #22.

---

### Demand Velocity Band (DEPRECATED - superseded by XYZ Classification)

**Replaced by:** `[XYZ Classification]`<br>
**Reason:** Velocity banding without CV normalization does not account for demand predictability — only speed. Unfairly penalizes high-volume SKUs with large absolute swings. See Entry #23.

---

## \_Measures\\_Validation

Infrastructure measures for model testing and data quality verification. Not business deliverables. Folder sorts to top of field list — signals infrastructure context.

---

### Blank Customer Check

**Formula:**
```dax
Blank Customer Check =
COUNTROWS(
    FILTER(
        FactOrders,
        ISBLANK(RELATED(DimCustomer[Customer Id]))
    )
)
```

**Expected value:** 0<br>
**Purpose:** Confirms no orphan customer keys in FactOrders.

---

### Blank Product Check

**Formula:**
```dax
Blank Product Check =
COUNTROWS(
    FILTER(
        FactOrders,
        ISBLANK(RELATED(DimProduct[Product Card Id]))
    )
)
```

**Expected value:** 0<br>
**Purpose:** Confirms no orphan product keys in FactOrders.

---

### Customer Match Check

**Formula:**
```dax
Customer Match Check =
COUNTROWS(
    FILTER(
        FactOrders,
        NOT ISBLANK(RELATED(DimCustomer[Customer Id]))
    )
)
```

**Expected value:** 180,519<br>
**Purpose:** Confirms all FactOrders rows successfully join to DimCustomer.

---

### Delivery Status Row Count

**Formula:**
```dax
Delivery Status Row Count =
COUNTROWS(FactOrders)
```

**Purpose:** Row count that respects Delivery Status filter context. Used on QA - Lead Time & Reliability page to verify delivery status distribution totals to 180,519.

**Notes:**
- Added because `Order Line Id` has Don't Summarize set per Entry #8 and cannot be implicitly aggregated. See decision-log.md Entry #8 downstream effect note.

---

### Earliest Order Date

**Formula:**
```dax
Earliest Order Date =
CALCULATE(
    MIN(FactOrders[Order Date]),
    ALL(FactOrders)
)
```

**Expected value:** January 6, 2015<br>
**Purpose:** Confirms date range of loaded data matches source.

---

### Latest Order Date

**Formula:**
```dax
Latest Order Date =
CALCULATE(
    MAX(FactOrders[Order Date]),
    ALL(FactOrders)
)
```

**Expected value:** January 31, 2018<br>
**Purpose:** Confirms date range of loaded data matches source.

---

### Product Match Check

**Formula:**
```dax
Product Match Check =
COUNTROWS(
    FILTER(
        FactOrders,
        NOT ISBLANK(RELATED(DimProduct[Product Card Id]))
    )
)
```

**Expected value:** 180,519<br>
**Purpose:** Confirms all FactOrders rows successfully join to DimProduct.

---

### Row Count Integrity

**Formula:**
```dax
Row Count Integrity =
COUNTROWS(FactOrders)
```

**Expected value:** 180,519<br>
**Purpose:** Confirms source data loaded completely without row loss or duplication.

---

### SKU Count at Rank (Tie Check)

**Formula:**
```dax
SKU Count at Rank (Tie Check) =
VAR ThisRank = [SKU Rank by Revenue]
RETURN
    CALCULATE(
        COUNTROWS(DimProduct),
        FILTER(
            ALL(DimProduct),
            [SKU Rank by Revenue] = ThisRank
        )
    )
```

**Purpose:** Identifies products sharing the same revenue rank due to Dense tie-handling. Value of 2 flags a tied pair. Used to explain why maximum displayed rank is 116 against 118 active products.

**Notes:**
- `ThisRank` VAR freezes current row's rank before FILTER iterates. Without VAR, both sides of `=` evaluate in iterator context — always TRUE — returning 118 for every row.

---

### Demand 25th Percentile

**Formula:**
```dax
Demand 25th Percentile =
PERCENTILX.INC(
    ALL(DimProduct),
    [Avg Daily Demand by SKU],
    0.25
)
```

**Purpose:** Demand distribution analysis for XYZ threshold calibration.

---

### Demand 50th Percentile

**Formula:**
```dax
Demand 50th Percentile =
PERCENTILX.INC(
    ALL(DimProduct),
    [Avg Daily Demand by SKU],
    0.50
)
```

**Purpose:** Median daily demand across all SKUs.

---

### Demand 75th Percentile

**Formula:**
```dax
Demand 75th Percentile =
PERCENTILX.INC(
    ALL(DimProduct),
    [Avg Daily Demand by SKU],
    0.75
)
```

**Purpose:** Demand distribution analysis for XYZ threshold calibration.

---

### Demand 90th Percentile

**Formula:**
```dax
Demand 90th Percentile =
PERCENTILX.INC(
    ALL(DimProduct),
    [Avg Daily Demand by SKU],
    0.90
)
```

**Purpose:** Demand distribution analysis for XYZ threshold calibration.

---

## \_Measures\ABC Classification

The ABC classification engine. **Measures below follow the display-folder tree**, not dependency order. When authoring in Power BI, build in **dependency order**: `Revenue by SKU` → `SKU Rank by Revenue` → `Cumulative Revenue %` → `ABC Tier` (then downstream SKU counts, XYZ, etc.)—the chain breaks if built out of sequence.

---

### ABC Tier

**Home table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Cumulative Revenue %]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**Formula:**
```dax
ABC Tier =
IF(
    ISINSCOPE(DimProduct[Product Name]),
    VAR CumPct = [Cumulative Revenue %]
    VAR SKURev = [Revenue by SKU (12-Month Trailing)]
    RETURN
        IF(
            ISBLANK(SKURev) || SKURev = 0,
            "Inactive",
            SWITCH(
                TRUE(),
                CumPct <= 0.80, "A",
                CumPct <= 0.95, "B",
                "C"
            )
        )
)
```

**Consumer description:** ABC inventory tier for this product based on 12-month trailing revenue contribution. A = top 80% of revenue, B = next 15%, C = bottom 5%. Inactive = no revenue in trailing window.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows.
- Inactive check fires first — products with zero trailing revenue are excluded from classification entirely before SWITCH evaluates.
- Thresholds are cumulative — `CumPct <= 0.95` for B tier implicitly means > 0.80 because A condition already failed.
- APICS 80/15/5 standard. Source: APICS CPIM Body of Knowledge.
- For slicing and filtering by tier, use the calculated column `DimProduct[ABC Tier (Classification)]` — measures cannot be used as slicer fields.

---

### Coefficient of Variation

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `[Demand Std Dev (Daily)]`, `[Avg Daily Demand by SKU]`<br>

**Formula:**
```dax
Coefficient of Variation =
DIVIDE(
    [Demand Std Dev (Daily)],
    [Avg Daily Demand by SKU]
)
```

**Consumer description:** Relative demand variability — standard deviation as a proportion of average daily demand. Normalizes variability across SKUs of different volumes.

**Notes:**
- CV strips out the size difference between SKUs and measures pure predictability. A high-volume SKU with large absolute swings is not unfairly penalized vs a low-volume SKU.
- CV = 0.5 means demand typically varies ±50% around average — regardless of whether the SKU sells 50 or 0.05 units/day.
- Primary input to `[XYZ Classification]`.
- `DIVIDE` handles zero average demand safely.

---

### Cumulative Revenue %

**Home table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `[SKU Rank by Revenue]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**Formula:**
```dax
Cumulative Revenue % =
IF(
    ISINSCOPE(DimProduct[Product Name]),
    VAR CurrentRank = [SKU Rank by Revenue]
    VAR TotalRev =
        CALCULATE(
            [Revenue by SKU (12-Month Trailing)],
            ALL(DimProduct)
        )
    VAR CumulativeRev =
        CALCULATE(
            [Revenue by SKU (12-Month Trailing)],
            FILTER(
                ALL(DimProduct),
                [SKU Rank by Revenue] <= CurrentRank
            )
        )
    RETURN
        DIVIDE(CumulativeRev, TotalRev)
)
```

**Consumer description:** Percentage of total 12-month revenue accumulated by this product and all higher-ranked products. The Pareto curve expressed as a running percentage.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows.
- `TotalRev` uses `ALL(DimProduct)` to see all 118 products regardless of visual context.
- `CumulativeRev` uses FILTER to sum revenue of all products ranked at or below current rank.
- When this value crosses 0.80, the next product enters B tier. When it crosses 0.95, the next product enters C tier.
- This is the most computationally expensive measure in the model — FILTER iterates over all products for each row.

---

### Revenue by SKU (12-Month Trailing) 🔒

**Home table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `[Total Revenue]`, `FactOrders[Order Date]`, `DimDate[Date]`, `DimCustomer`<br>

**Formula:**
```dax
Revenue by SKU (12-Month Trailing) =
VAR WindowEnd =
    CALCULATE(
        MAX(FactOrders[Order Date]),
        ALL(FactOrders)
    )
VAR WindowStart =
    EDATE(WindowEnd, -12)
RETURN
    CALCULATE(
        [Total Revenue],
        ALL(DimCustomer),
        ALL(DimDate),
        DimDate[Date] >  WindowStart,
        DimDate[Date] <= WindowEnd
    )
```

**Consumer description:** Total revenue for this product in the trailing 12-month window used for ABC classification. Not affected by date or customer slicers.

**Notes:**
- 🔒 Locked context measure. `ALL(DimDate)` and `ALL(DimCustomer)` intentionally remove slicer filters so classification reflects total market demand.
- `WindowEnd` anchored to `MAX(FactOrders[Order Date])` not `MAX(DimDate[Date])` — DimDate includes 1-year padding beyond transaction boundaries. Using DimDate max would produce an empty window. See decision-log.md Entry #17.
- `DimDate[Date] > WindowStart` (not `>=`) — ensures exactly 12 calendar months. `>=` would add an extra day.
- For exploratory revenue analysis that respects slicer context, use `[Total Revenue]` instead.
- 12-month window: Feb 1, 2017 → Jan 31, 2018.

---

### SKU Count - A Tier

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[ABC Tier (Classification)]`<br>

**Formula:**
```dax
SKU Count - A Tier =
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "A"
)
```

**Consumer description:** Number of products classified as A tier.

**Notes:**
- Uses calculated column `ABC Tier (Classification)` not the measure `[ABC Tier]` — calculated column is available without product-level filter context.
- Validation check: A + B + C + Inactive must equal 118.

---

### SKU Count - B Tier

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[ABC Tier (Classification)]`<br>

**Formula:**
```dax
SKU Count - B Tier =
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "B"
)
```

**Consumer description:** Number of products classified as B tier.

---

### SKU Count - C Tier

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[ABC Tier (Classification)]`<br>

**Formula:**
```dax
SKU Count - C Tier =
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "C"
)
```

**Consumer description:** Number of products classified as C tier.

---

### SKU Count - Inactive

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[ABC Tier (Classification)]`<br>

**Formula:**
```dax
SKU Count - Inactive =
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "Inactive"
)
```

**Consumer description:** Number of products with no revenue in the trailing 12-month window — excluded from ABC classification.

---

### SKU Count by Velocity Band

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[Demand Velocity Band]`<br>

**Formula:**
```dax
SKU Count by Velocity Band =
COUNTROWS(DimProduct)
```

**Consumer description:** Number of products in the current velocity band filter context.

**Notes:**
- Use with `DimProduct[Demand Velocity Band]` as row grouping in a table visual to see distribution across bands.

---

### Demand Velocity Band ⚠

**Home table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Avg Daily Demand by SKU]`, `DimProduct`<br>

**Formula:**
```dax
Demand Velocity Band =
IF(
    ISINSCOPE(DimProduct[Product Name]),
    SWITCH(
        TRUE(),
        [Avg Daily Demand by SKU] >= 10,  "High (≥10/day)",
        [Avg Daily Demand by SKU] >= 1,   "Moderate (1-10/day)",
        [Avg Daily Demand by SKU] > 0,    "Low (<1/day)",
        "Inactive"
    )
)
```

**Consumer description:** Demand velocity band for this product based on average daily units sold in the trailing 12-month window.

**Notes:**
- ⚠ Superseded by `[XYZ Classification]` for replenishment decisions. Velocity banding without CV normalization does not account for demand predictability — only speed. See Entry #23.
- Retained as diagnostic measure. For replenishment logic, use `DimProduct[XYZ Classification]`.
- Distribution in this dataset: High = 8 products, Moderate = 14 products, Low = 96 products.

---

### SKU Rank by Revenue

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**Formula:**
```dax
SKU Rank by Revenue =
IF(
    ISINSCOPE(DimProduct[Product Name]),
    RANKX(
        ALL(DimProduct),
        [Revenue by SKU (12-Month Trailing)],
        ,
        DESC,
        Dense
    )
)
```

**Consumer description:** Revenue rank of this product among all 118 products in the trailing 12-month window. Rank 1 = highest revenue.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows — prevents misleading rank 1 at non-product grain.
- `ALL(DimProduct)` iterator ensures ranking is always against all 118 products regardless of visual filters.
- `Dense` tie-handling: two products share identical 12-month trailing revenue, producing max displayed rank of 116 against 118 active products. Expected behavior — see `SKU Count at Rank (Tie Check)` in `_Validation`.
- Must evaluate at `Product Name` grain. Returns incorrect results at category or department grain.

---

### XYZ Classification

**Home table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Coefficient of Variation]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**Formula:**
```dax
XYZ Tier =
IF(
    ISINSCOPE(DimProduct[Product Name]),
    VAR CV = [Coefficient of Variation]
    VAR SKURev = [Revenue by SKU (12-Month Trailing)]
    RETURN
        IF(
            ISBLANK(SKURev) || SKURev = 0,
            "Inactive",
            SWITCH(
                TRUE(),
                ISBLANK(CV),    "Unclassified",
                CV <= 0.5,      "X",
                CV <= 1.0,      "Y",
                "Z"
            )
        )
)
```

**Consumer description:** XYZ demand predictability classification. X = stable demand, Y = moderate variability, Z = erratic demand. Drives replenishment cycle length. Decoupled from ABC tier.

**Notes:**
- XYZ drives replenishment behavior (target days, order frequency). ABC drives service level (Z-score). These are separate dimensions. See Entry #23.
- CV thresholds: X ≤ 0.5, Y 0.5–1.0, Z > 1.0. Based on APICS supply chain segmentation methodology.
- For slicing and filtering, use the calculated column `DimProduct[XYZ Classification]`.

---

## \_Measures\Foundation

Six base measures that all downstream calculations depend on. Build these first. Every other measure in the model either references these directly or inherits their logic.

---

### Avg Order Value

**Home table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `[Total Revenue]`, `[Total Orders]`<br>

**Formula:**
```dax
Avg Order Value =
DIVIDE([Total Revenue], [Total Orders])
```

**Consumer description:** Average revenue per customer order in the selected period.

**Notes:**
- `DIVIDE` used instead of `/` — handles division by zero gracefully without returning errors.
- Validated: $36,784,734.31 ÷ 65,752 = $559.45 unfiltered. ✅

---

### Avg Profit Margin %

**Home table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `FactOrders[Profit Ratio]`<br>

**Formula:**
```dax
Avg Profit Margin % =
AVERAGE(FactOrders[Profit Ratio])
```

**Consumer description:** Average gross profit margin across all order line items in the selected period.

**Notes:**
- `Profit Ratio` is a raw 0–1 decimal field in FactOrders. Format set to Percentage in model properties — no multiplication by 100 in DAX.
- Total row recalculates across all rows directly — it does not average the department averages. DAX evaluates in total filter context, not as an average of averages.
- Validated: 12.06% unfiltered. Varies meaningfully by department — confirms correct filter context behavior.

---

### Total Gross Profit

**Home table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Gross Profit]`<br>

**Formula:**
```dax
Total Gross Profit =
SUM(FactOrders[Order Gross Profit])
```

**Consumer description:** Total gross profit across all order line items in the selected period.

**Notes:**
- Enables implied COGS calculation per Limitation #2 in decision log: `Implied COGS = [Total Revenue] - [Total Gross Profit]`.
- `Order Gross Profit` is a translation artifact from source Spanish column "beneficio" — confirmed as gross profit per ETL documentation.

---

### Total Orders

**Home table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `FactOrders[Order Id]`<br>

**Formula:**
```dax
Total Orders =
DISTINCTCOUNT(FactOrders[Order Id])
```

**Consumer description:** Total number of distinct customer orders in the selected period.

**Notes:**
- Uses `DISTINCTCOUNT` not `COUNTROWS`. FactOrders is at line-item grain — 180,519 rows, 65,752 distinct orders. `COUNTROWS` would return line item count, not order count. See decision-log.md Entry #13.
- Validated: unfiltered value = 65,752, matching model validation suite.

---

### Total Revenue

**Home table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `FactOrders[Sales]`<br>

**Formula:**
```dax
Total Revenue =
SUM(FactOrders[Sales])
```

**Consumer description:** Total sales revenue across all order line items in the selected period.

**Notes:**
- Base revenue measure — do not modify. All downstream revenue calculations reference this.
- Respects all slicer and filter context. For locked classification revenue, use `[Revenue by SKU (12-Month Trailing)]`.

---

### Total Units Sold

**Home table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `FactOrders[Order Quantity]`<br>

**Formula:**
```dax
Total Units Sold =
SUM(FactOrders[Order Quantity])
```

**Consumer description:** Total units ordered across all order line items in the selected period.

**Notes:**
- Demand volume measure. Used as input to `[Avg Daily Demand by SKU]` in Safety Stock layer.

---

## \_Measures\Lead Time & Reliability

All six measures exclude canceled orders (`Delivery Status <> "Shipping canceled"`). Canceled orders never shipped — including them distorts lead time averages toward zero and inflates on-time rate. See decision-log.md Entry #19.

**Validated delivery status distribution:**
- Late delivery: 98,977 rows
- Advance shipping: 41,592 rows
- Shipping on time: 32,196 rows
- Shipping canceled: 7,754 rows
- Total: 180,519 rows ✅
- Non-canceled orders used in calculations: 172,765

---

### Avg Lead Time (Actual)

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Shipping Days (Actual)]`, `FactOrders[Delivery Status]`<br>

**Formula:**
```dax
Avg Lead Time (Actual) =
CALCULATE(
    AVERAGE(FactOrders[Shipping Days (Actual)]),
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

**Consumer description:** Average actual days from order placement to delivery, excluding canceled orders.

**Notes:**
- Unit: days.
- Validated total: 3.50 days.
- Primary lead time input to Safety Stock formula.

---

### Avg Lead Time (Scheduled)

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Shipping Days (Scheduled)]`, `FactOrders[Delivery Status]`<br>

**Formula:**
```dax
Avg Lead Time (Scheduled) =
CALCULATE(
    AVERAGE(FactOrders[Shipping Days (Scheduled)]),
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

**Consumer description:** Average promised lead time at order placement, excluding canceled orders.

**Notes:**
- Unit: days.
- Validated total: 2.93 days.
- Compare to `[Avg Lead Time (Actual)]` to quantify systemic delay.

---

### Avg Lead Time Variance

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Lead Time Variance]`, `FactOrders[Delivery Status]`<br>

**Formula:**
```dax
Avg Lead Time Variance =
CALCULATE(
    AVERAGE(FactOrders[Lead Time Variance]),
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

**Consumer description:** Average days by which actual delivery exceeded scheduled delivery. Positive = late on average.

**Notes:**
- Unit: days.
- Validated total: 0.57 days — orders are systematically late by over half a day on average.
- `Lead Time Variance` is a calculated column in FactOrders: `Shipping Days (Actual) - Shipping Days (Scheduled)`.

---

### Late Delivery Rate %

**Home table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `FactOrders[Late Delivery Risk]`, `FactOrders[Delivery Status]`<br>

**Formula:**
```dax
Late Delivery Rate % =
CALCULATE(
    AVERAGE(FactOrders[Late Delivery Risk]),
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

**Consumer description:** Proportion of non-canceled orders that arrived later than scheduled.

**Notes:**
- `Late Delivery Risk` is a 0/1 integer column in FactOrders. `AVERAGE` of a binary field equals the proportion of 1s — mathematically equivalent to a rate. Standard analytics pattern.
- Validated total: 57.3%. High rate reflects DataCo dataset characteristics — not a measure error. See Limitation #6 in decision-log.md.
- 98,977 late deliveries ÷ 172,765 non-canceled orders = 57.3% ✅

---

### Lead Time Variance Std Dev

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Lead Time Variance]`, `FactOrders[Delivery Status]`<br>

**Formula:**
```dax
Lead Time Variance Std Dev =
CALCULATE(
    STDEV.P(FactOrders[Lead Time Variance]),
    FactOrders[Delivery Status] <> "Shipping canceled"
)
```

**Consumer description:** Standard deviation of lead time variance — measures how consistently late or early orders arrive. Higher value = more unpredictable fulfillment.

**Notes:**
- Unit: days.
- Validated total: 1.49 days.
- `STDEV.P` used — 180,519 rows is the full population, not a sample. `STDEV.S` would overstate variance.
- Critical input to Safety Stock formula: `σ²_LT` (lead time variance). Squaring this value inside the formula converts standard deviation to variance.
- A std dev of 1.49 days against a scheduled lead time of 2.93 days = coefficient of variation of ~51% — highly unpredictable fulfillment.

---

### On-Time Delivery Rate %

**Home table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `[Late Delivery Rate %]`<br>

**Formula:**
```dax
On-Time Delivery Rate % =
1 - [Late Delivery Rate %]
```

**Consumer description:** Proportion of non-canceled orders that arrived on time or early.

**Notes:**
- Inherits canceled order filter from `[Late Delivery Rate %]`. No redundant filter needed.
- Includes both "Shipping on time" and "Advance shipping" statuses.
- Validated total: 42.7%.

---

## \_Measures\Safety Stock & Replenishment

**Architecture:** ABC tier drives service level (Z-score → Safety Stock). XYZ classification drives replenishment behavior (target days → Reorder Quantity). These are decoupled dimensions. See decision-log.md Entry #23.

**Measure list order** below matches the **Safety Stock & Replenishment** display folder (not the sequence you use to type DAX). **Build order** when authoring: `Avg Daily Demand by SKU` → `Demand Std Dev (Daily)` → `Coefficient of Variation` (ABC section) → `Safety Stock` → `Reorder Point` → `Reorder Quantity` → `Reorder Flag` → `Stock Coverage (Days)`.

---

### Avg Daily Demand by SKU 🔒

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Quantity]`, `FactOrders[Order Date]`, `DimDate[Date]`, `DimCustomer`<br>

**Formula:**
```dax
Avg Daily Demand by SKU =
VAR WindowEnd =
    CALCULATE(
        MAX(FactOrders[Order Date]),
        ALL(FactOrders)
    )
VAR WindowStart =
    EDATE(WindowEnd, -12)
VAR TotalUnits =
    CALCULATE(
        SUM(FactOrders[Order Quantity]),
        DimDate[Date] >  WindowStart,
        DimDate[Date] <= WindowEnd,
        ALL(DimCustomer)
    )
VAR WindowDays =
    CALCULATE(
        COUNTROWS(DimDate),
        DimDate[Date] >  WindowStart,
        DimDate[Date] <= WindowEnd
    )
RETURN
    DIVIDE(TotalUnits, WindowDays)
```

**Consumer description:** Average units sold per day in the trailing 12-month window.

**Notes:**
- 🔒 `ALL(DimCustomer)` removes customer segment filters — demand reflects total market.
- `WindowDays` counts actual calendar days rather than hardcoding 365 — accounts for leap years where the window spans February 29 and contains 366 days.
- `DimDate[Date] > WindowStart` (not `>=`) — ensures exactly 12 calendar months. Window: Feb 1, 2017 → Jan 31, 2018 = 365 days for this dataset.
- Primary input to Safety Stock and Reorder Point formulas.

---

### Demand Std Dev (Daily) 🔒

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Quantity]`, `FactOrders[Order Date]`, `DimDate[Date]`, `DimCustomer`<br>

**Formula:**
```dax
Demand Std Dev (Daily) =
VAR WindowEnd =
    CALCULATE(
        MAX(FactOrders[Order Date]),
        ALL(FactOrders)
    )
VAR WindowStart =
    EDATE(WindowEnd, -12)
VAR DailyDemandTable =
    SUMMARIZE(
        FILTER(
            FactOrders,
            FactOrders[Order Date] >  WindowStart &&
            FactOrders[Order Date] <= WindowEnd
        ),
        FactOrders[Order Date],
        "DailyUnits", SUM(FactOrders[Order Quantity])
    )
RETURN
    STDEVX.P(DailyDemandTable, [DailyUnits])
```

**Consumer description:** Standard deviation of daily units sold in the trailing 12-month window. Measures true demand variability — how much daily demand fluctuates around the average.

**Notes:**
- `SUMMARIZE` aggregates to daily totals first, then `STDEVX.P` measures spread across those daily totals. This is the correct grain for safety stock input.
- `STDEV.P(FactOrders[Order Quantity])` — the original approach — measures order size variability, not daily demand variability. A corporate buyer ordering 50 units in one transaction vs five buyers each ordering 10 units produces identical daily demand but different order-level standard deviations. See Entry #22.
- `STDEVX.P` used — full population, not sample.
- Critical input to Safety Stock formula: `σ²_demand` (demand variance). Squaring inside formula converts to variance.

---

### Reorder Flag

**Home table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Reorder Point]`, `DimProduct[Simulated Inventory Level]`<br>

**Formula:**
```dax
Reorder Flag =
VAR SimInventory = SELECTEDVALUE(DimProduct[Simulated Inventory Level])
VAR ROP          = [Reorder Point]
RETURN
    IF(
        ISBLANK(ROP) || ISBLANK(SimInventory),
        BLANK(),
        SWITCH(
            TRUE(),
            SimInventory = 0,    "🚨 Stockout",
            SimInventory <= ROP, "⚠ Reorder Now",
            "✓ Stock OK"
        )
    )
```

**Consumer description:** Current inventory status signal. Stockout = inventory depleted, order overdue. Reorder Now = place replenishment order today. Stock OK = no action needed.

**Notes:**
- Three-state logic. Stockout (inventory = 0) and Reorder Now (inventory ≤ Reorder Point) trigger different operational workflows — stockout requires immediate escalation.
- Returns BLANK at subtotal/total rows via `SELECTEDVALUE` — no single product in context.
- Returns BLANK for Inactive SKUs — Reorder Point is BLANK for inactive products.
- 🧪 Depends on simulated inventory. In production, replace `DimProduct[Simulated Inventory Level]` with live WMS/ERP snapshot. No formula changes needed.

---

### Reorder Point

**Home table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `[Avg Daily Demand by SKU]`, `[Avg Lead Time (Actual)]`, `[Safety Stock]`<br>

**Formula:**
```dax
Reorder Point =
VAR AvgDemand = [Avg Daily Demand by SKU]
VAR AvgLT     = [Avg Lead Time (Actual)]
VAR SS        = [Safety Stock]
VAR RawROP    = (AvgDemand * AvgLT) + SS
RETURN
    IF(
        ISBLANK(SS),
        BLANK(),
        RawROP
    )
```

**Consumer description:** Inventory level at which a replenishment order should be triggered. Covers expected demand during lead time plus safety stock buffer.

**Notes:**
- Formula: `(Avg Daily Demand × Avg Lead Time) + Safety Stock`.
- When on-hand inventory reaches this level, place replenishment order immediately.
- `(Avg Daily Demand × Avg Lead Time)` covers expected demand during the lead time window. Safety Stock covers variability above that expectation.
- Returns BLANK for Inactive SKUs — inherited from `[Safety Stock]`.
- Unit: units of inventory (decimal — represents fractional trigger point).

---

### Reorder Quantity

**Home table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `DimProduct[XYZ Classification]`, `[Avg Daily Demand by SKU]`, `[Safety Stock]`, `DimProduct[Simulated Inventory Level]`<br>

**Formula:**
```dax
Reorder Quantity =
VAR XYZ = SELECTEDVALUE(DimProduct[XYZ Classification])
VAR TargetDays =
    SWITCH(
        XYZ,
        "X", 60,
        "Y", 45,
        "Z", 30,
        BLANK()
    )
VAR MaxDays         = TargetDays * 1.5
VAR AvgDemand       = [Avg Daily Demand by SKU]
VAR SS              = [Safety Stock]
VAR SimInventory    = SELECTEDVALUE(DimProduct[Simulated Inventory Level])
VAR TargetInventory = (TargetDays * AvgDemand) + SS
VAR MaxInventory    = CEILING(MaxDays * AvgDemand, 1)
VAR RawQuantity     = TargetInventory - SimInventory
VAR ControlledQuantity =
    IF(
        RawQuantity <= 0,
        0,
        MIN(CEILING(RawQuantity, 1), MaxInventory)
    )
RETURN
    IF(
        ISBLANK(TargetDays) || ISBLANK(SS),
        BLANK(),
        ControlledQuantity
    )
```

**Consumer description:** Suggested replenishment order quantity to bring inventory back to target coverage level.

**Notes:**
- Target days driven by XYZ demand predictability — not ABC tier. XYZ drives replenishment behavior. ABC drives service level. See Entry #23.
- X = 60 days (stable demand → longer coverage, infrequent large orders), Y = 45 days, Z = 30 days (erratic demand → shorter coverage, frequent small orders).
- Max cap = 1.5× target days per APICS min/max methodology. Tier-relative — prevents over-ordering proportionally.
- Lead time is NOT added to the formula. Safety stock already covers variability above average demand during lead time. Average demand during lead time is embedded in Reorder Point trigger. Adding it here would double-count.
- Floor at 0: when simulated inventory exceeds target, no order needed.
- `CEILING` rounds up — cannot order fractional units.
- 🧪 Depends on `DimProduct[Simulated Inventory Level]` — simulated data for testing only. In production, replace simulated column with live WMS/ERP snapshot. No formula changes needed.

---

### Safety Stock

**Home table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `[ABC Tier]`, `[Avg Lead Time (Actual)]`, `[Lead Time Variance Std Dev]`, `[Avg Daily Demand by SKU]`, `[Demand Std Dev (Daily)]`<br>

**Formula:**
```dax
Safety Stock =
VAR ZScore =
    SWITCH(
        [ABC Tier],
        "A", 1.65,
        "B", 1.28,
        "C", 1.04,
        BLANK()
    )
VAR AvgLT       = [Avg Lead Time (Actual)]
VAR StdDevLT    = [Lead Time Variance Std Dev]
VAR AvgDemand   = [Avg Daily Demand by SKU]
VAR StdDevDemand = [Demand Std Dev (Daily)]
VAR RawSafetyStock =
    ZScore * SQRT(
        (AvgLT * StdDevDemand ^ 2) +
        (AvgDemand ^ 2 * StdDevLT ^ 2)
    )
RETURN
    IF(
        ISBLANK(ZScore),
        BLANK(),
        CEILING(RawSafetyStock, 1)
    )
```

**Consumer description:** Minimum buffer inventory required to maintain target service level given demand variability and lead time variability.

**Notes:**
- Combined variance formula — APICS standard for environments with both demand and lead time variability. Source: APICS CPIM Body of Knowledge.
- Z-score by ABC tier: A = 1.65 (95% service level), B = 1.28 (90%), C = 1.04 (85%). ABC drives service level — how much does a stockout cost? Higher revenue tier = higher cost per stockout = higher protection.
- `CEILING` rounds up — rounding down holds less buffer than formula requires, accepting more stockout risk than target service level.
- Returns BLANK for Inactive SKUs — no replenishment parameters for products outside active classification scope.
- Unit: units of inventory.

---

### Stock Coverage (Days)

**Home table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[Simulated Inventory Level]`, `[Avg Daily Demand by SKU]`<br>

**Formula:**
```dax
Stock Coverage (Days) =
VAR SimInventory  = SELECTEDVALUE(DimProduct[Simulated Inventory Level])
VAR DailyDemand   = [Avg Daily Demand by SKU]
VAR RawCoverage   = DIVIDE(SimInventory, DailyDemand)
RETURN
    IF(
        ISBLANK(DailyDemand) || DailyDemand = 0,
        BLANK(),
        CEILING(RawCoverage, 1)
    )
```

**Consumer description:** Estimated days of stock remaining at current average daily demand rate before stockout.

**Notes:**
- `CEILING` rounds up — rounding down understates remaining coverage and could trigger premature reorder decisions.
- Returns BLANK when demand is zero — undefined stock coverage for products with no demand.
- Returns BLANK at subtotal/total rows via `SELECTEDVALUE`.
- 🧪 Depends on simulated inventory. In production, replace with live WMS/ERP snapshot.
- Unit: days (whole number — partial days rounded up to nearest full day).

---

## DimProduct Calculated Columns

Calculated columns stored in DimProduct. Evaluated at model refresh time — stable between refreshes. Required for slicers, row-level filtering, and visual grouping. Measure equivalents exist in FactOrders for dynamic filter context evaluation.

---

### ABC Tier (Classification)

**Table:** DimProduct
**Folder:** ABC Classification
**Format:** Text<br>
**Measure equivalent:** `[ABC Tier]` in FactOrders

**Formula:**
```dax
ABC Tier (Classification) =
VAR SKURev = [Revenue by SKU (12-Month Trailing)]
VAR CumPct = [Cumulative Revenue %]
RETURN
    IF(
        ISBLANK(SKURev) || SKURev = 0,
        "Inactive",
        SWITCH(
            TRUE(),
            CumPct <= 0.80, "A",
            CumPct <= 0.95, "B",
            "C"
        )
    )
```

**Notes:**
- Stamped at refresh time. Use for slicers and visual row grouping.
- For dynamic filter context evaluation, use `[ABC Tier]` measure.
- Naming conflict: Power BI does not allow a measure and column with the same name across the model. Column named `ABC Tier (Classification)` to distinguish from `[ABC Tier]` measure.

---

### Demand Velocity Band (DEPRECATED) ❌

**Table:** DimProduct
**Folder:** ABC Classification
**Replaced by:** `DimProduct[XYZ Classification]`
**Reason:** Velocity banding without CV normalization. Superseded by XYZ Classification.

---

### Simulated Inventory Level 🧪

**Table:** DimProduct
**Folder:** ABC Classification
**Format:** Whole number<br>

**Formula:**
```dax
Simulated Inventory Level =
VAR AvgDailyDemand = [Avg Daily Demand by SKU]
VAR UpperBound     = ROUND(AvgDailyDemand * 60, 0)
VAR SafeRange      = IF(UpperBound > 0, UpperBound, 1)
RETURN
    MOD([Product Card Id], SafeRange)
```

**Notes:**
- 🧪 Testing infrastructure only. No analytical conclusions should be drawn from this column.
- Demand-proportionate deterministic simulation. Upper bound = 60 days of supply at average daily demand. `MOD(Product Card Id, range)` assigns stable values across refreshes.
- No on-hand inventory data exists in DataCo source dataset. See Limitation #1 in decision-log.md.
- In production: replace with live WMS/ERP on-hand snapshot. `Reorder Flag`, `Stock Coverage (Days)`, and `Reorder Quantity` require no formula changes — only this source column changes.

---

### XYZ Classification

**Table:** DimProduct
**Folder:** ABC Classification
**Format:** Text<br>
**Measure equivalent:** `[XYZ Classification]` in FactOrders

**Formula:**
```dax
XYZ Classification =
VAR CV =
    DIVIDE(
        [Demand Std Dev (Daily)],
        [Avg Daily Demand by SKU]
    )
VAR SKURev = [Revenue by SKU (12-Month Trailing)]
RETURN
    IF(
        ISBLANK(SKURev) || SKURev = 0,
        "Inactive",
        SWITCH(
            TRUE(),
            ISBLANK(CV),    "Unclassified",
            CV <= 0.5,      "X",
            CV <= 1.0,      "Y",
            "Z"
        )
    )
```

**Notes:**
- Use for slicers and visual row grouping.
- For dynamic filter context evaluation, use `[XYZ Classification]` measure.

---

## Measure Dependency Map

```
Total Revenue ──────────────────────────────────────────────► Revenue by SKU (12-Month Trailing)
                                                                        │
                                                              SKU Rank by Revenue
                                                                        │
                                                            Cumulative Revenue %
                                                                        │
                                                                  ABC Tier ──────► Z-Score input
                                                                        │                  │
                                                           ABC Tier (Classification)        │
                                                                                           ▼
Avg Daily Demand by SKU ──────────────────────────────────────► Safety Stock ◄── Lead Time Variance Std Dev
        │                                                               │                  │
        │                                                        Reorder Point      Avg Lead Time (Actual)
        │                                                               │
Demand Std Dev (Daily) ──► Coefficient of Variation ──► XYZ Classification
                                                                │
                                                         TargetDays input
                                                                │
                                                         Reorder Quantity ◄── Safety Stock
                                                                              Simulated Inventory Level
                                                                │
                                                          Reorder Flag
                                                          Stock Coverage (Days)
```

---

## Build Order Reference

| Layer | Measures | Must build after |
|---|---|---|
| 1 — Foundation | Total Revenue, Total Gross Profit, Total Units Sold, Total Orders, Avg Order Value, Avg Profit Margin % | Nothing |
| 2 — ABC Classification | Revenue by SKU (12-Month Trailing), SKU Rank by Revenue, Cumulative Revenue %, ABC Tier | Layer 1 complete |
| 3 — Lead Time & Reliability | All six lead time measures | Layer 1 complete |
| 4 — Safety Stock & Replenishment | Avg Daily Demand, Demand Std Dev (Daily), CV, Safety Stock, Reorder Point, Reorder Quantity, Reorder Flag, Stock Coverage | Layers 2 and 3 complete |
| 5 — P&L Impact | Pending | Layers 1–4 complete |
| 6 — Time Intelligence | Pending | Layer 1 complete |

---

*Document Version: 1.0 — Phase 3 DAX Layer*
*Covers: FactOrders `_Measures` folders in display-folder order (_Deprecated → _Validation → ABC → Foundation → Lead Time → Safety Stock), then DimProduct columns*
*Next Update: Phase 4 — P&L Impact and Time Intelligence measures*
