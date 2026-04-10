# DAX Measures Reference

## ABC XYZ Inventory Segmentation & Management
### Power BI Project: Phase 3 (DAX Layer)

---

> This document is the authoritative developer reference for all DAX measures in this project. It covers measure name, table, display folder, DAX, format, dependencies, and implementation notes. **Section and measure order follow the `FactOrders` → `_Measures` display-folder tree** (see below), not DAX dependency order. See **Build Order Reference** when creating measures in Power BI. For architectural decisions behind each measure, see [decision-log.md](./decision-log.md). For column definitions and source-to-model mapping, see [column-definition.md](./column-definition.md).

---

## Document Conventions

| Convention | Meaning |
|---|---|
| `[Measure Name]` | DAX measure reference |
| `Table[Column]` | Column reference |
| `[Caution]` | Measure has a known limitation or requires careful interpretation |
| `[Fixed Context]` | Locked context measure - filter arguments intentionally remove slicers |
| `[Test]` | Testing/simulation infrastructure - not a business deliverable |
| `[Deprecated]` | Deprecated - retained for audit trail only |

---

## Table of Contents

1. [FactOrders - display folder tree](#factorders--display-folder-tree)
2. [\_Measures\\_Deprecated](#measures_deprecated)
3. [\_Measures\\_Validation](#measures_validation)
4. [\_Measures\Core Measures](#measurescore-measures)
5. [\_Measures\Inventory Planning](#measuresinventory-planning)
6. [\_Measures\Inventory Segmentation](#measuresinventory-segmentation)
7. [\_Measures\Supply Performance](#measuressupply-performance)
8. [\_Measures\Financial Impact](#measuresfinancial-impact)
9. [\_Measures\Trend Analysis](#measurestrend-analysis)
10. [DimProduct Calculated Columns](#dimproduct-calculated-columns)
11. [ABC XYZ Cycle Count Schedule](#abc-xyz-cycle-count-schedule)
12. [Build Order Reference](#build-order-reference)
13. [Measure Dependency Map](#measure-dependency-map)

---

## FactOrders - display folder tree

```text
FactOrders
│
├── _Measures
│   │
│   ├── _Deprecated
│   │   ├── Demand Std Dev (DEPRECATED - order level, not daily level)
│   │   ├── Demand Velocity Band (DEPRECATED - superseded by XYZ Classification)
│   │   └── SKU Count by Velocity Band (DEPRECATED - superseded by XYZ)
│   │
│   ├── _Validation
│   │   ├── Blank Customer Check
│   │   ├── Blank Product Check
│   │   ├── Canceled Line Items
│   │   ├── Canceled Order Units
│   │   ├── Customer Match Check
│   │   ├── CV 25th Percentile
│   │   ├── CV 50th Percentile
│   │   ├── CV 75th Percentile
│   │   ├── CV 90th Percentile
│   │   ├── Delivery Status Row Count
│   │   ├── Earliest Order Date
│   │   ├── Latest Order Date
│   │   ├── Orders With Canceled Lines
│   │   ├── Orders With Mixed Lines
│   │   ├── Product Match Check
│   │   ├── Row Count Integrity
│   │   ├── SKU Count - A Tier
│   │   ├── SKU Count - B Tier
│   │   ├── SKU Count - C Tier
│   │   ├── SKU Count - Inactive
│   │   ├── SKU Count - Unclassified
│   │   ├── SKU Count - X
│   │   ├── SKU Count - Y
│   │   ├── SKU Count - Z
│   │   └── SKU Count at Rank (Tie Check)
│   │
│   ├── Core Measures
│   │   ├── Avg Order Value
│   │   ├── Avg Profit Margin %
│   │   ├── Total Gross Profit
│   │   ├── Total Orders
│   │   ├── Total Revenue
│   │   └── Total Units Sold
│   │
│   ├── Inventory Planning
│   │   ├── Avg Daily Demand by SKU
│   │   ├── Demand Std Dev (Daily)
│   │   ├── Reorder Flag
│   │   ├── Reorder Point
│   │   ├── Reorder Quantity
│   │   ├── Safety Stock
│   │   └── Stock Coverage (Days)
│   │
│   ├── Inventory Segmentation
│   │   ├── ABC Tier
│   │   ├── Coefficient of Variation
│   │   ├── Cumulative Revenue %
│   │   ├── Revenue by SKU (12-Month Trailing)
│   │   ├── SKU Rank by Revenue
│   │   └── XYZ Classification
│   │
│   ├── Supply Performance
│   │   ├── Avg Lead Time (Actual)
│   │   ├── Avg Lead Time (Scheduled)
│   │   ├── Avg Lead Time Variance
│   │   ├── Late Delivery Rate %
│   │   ├── Lead Time Variance Std Dev
│   │   └── On-Time Delivery Rate %
│   │
│   ├── Financial Impact               ⬜ Pending
│   │   ├── Avg Margin % by ABC Tier
│   │   ├── Carrying Cost Estimate
│   │   ├── Gross Profit by ABC Tier
│   │   ├── Implied COGS
│   │   ├── Pareto Concentration Ratio
│   │   ├── Revenue at Risk (Supply)
│   │   └── Revenue at Risk (Stockout)
│   │
│   └── Trend Analysis                 ⬜ Pending
│       ├── Revenue vs Prior Period
│       ├── Revenue YoY
│       └── Rolling 90-Day Demand


DimProduct
│
└── Inventory Segmentation
    ├── ABC Tier (Classification)
    ├── Cycle Count Schedule
    ├── Demand Velocity Band (DEPRECATED - XYZ Classification)
    ├── Simulated Inventory Level
    ├── SKU Revenue Rank (DimColumn)
    └── XYZ Classification (DimColumn)
```

---

## \_Measures\\_Deprecated

Measures retained for audit trail and analytical evolution documentation. Do not use in report pages or downstream calculations.

---

### Demand Std Dev (DEPRECATED - order level, not daily level) `[Deprecated]`

**Replaced by:** `[Demand Std Dev (Daily)]`<br>
**Reason:** `STDEV.P(FactOrders[Order Quantity])` measures order size variability, not daily demand variability. Distorts safety stock. See [decision-log.md](./decision-log.md) Entry #20.

---

### Demand Velocity Band (DEPRECATED - superseded by XYZ Classification) `[Deprecated]`

**Replaced by:** `[XYZ Classification]`<br>
**Reason:** Velocity banding without CV (Coefficient of Variation) normalization does not account for demand predictability, it only accounts for speed. See [decision-log.md](./decision-log.md) Entry #17.

---

### SKU Count by Velocity Band (DEPRECATED - superseded by XYZ Classification) `[Deprecated]`

**Replaced by:** `SKU Count - X`, `SKU Count - Y`, `SKU Count - Z` in `_Validation`<br>
**Reason:** Superseded by XYZ classification. Velocity band counts no longer drive replenishment decisions. See [decision-log.md](./decision-log.md) Entry #17.

---

## \_Measures\\_Validation

Infrastructure measures for model testing and data quality verification. Not business deliverables. Folder sorts to top of field list indicating infrastructure context.

**Validation checks:**
- ABC completeness: `SKU Count - A + B + C + Inactive = 118` 
- XYZ completeness: `SKU Count - X + Y + Z + Unclassified = 118` 
- Row integrity: `Row Count Integrity = 180,519` 
- Dimension integrity: `Customer Match Check = Product Match Check = 180,519`

---

### Blank Customer Check

**Purpose:** Confirms no orphan customer keys in FactOrders.
**Report Page:** QA -

**Expected:** 0
**Actual**
**Status:**
 
**DAX:**
```dax
Blank Customer Check = 
COUNTROWS(
    FILTER(
        FactOrders,
        ISBLANK(
            RELATED(DimCustomer[Customer Segment])
        )
    )
)
```

---

### Blank Product Check

**Purpose:** Confirms no orphan product keys in FactOrders.
**Report Page:** QA -

**Expected:** 0<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Blank Product Check = 
COUNTROWS(
    FILTER(
        FactOrders,
        ISBLANK(
            RELATED(DimProduct[Product Name])
        )
    )
)
```

---

### Customer Match Check

**Purpose:** Confirms all FactOrders rows successfully join to DimCustomer.
**Report Page:** QA -

**Expected:** 180,519<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Customer Match Check = 
FORMAT(
    COUNTROWS(
        FILTER(
            FactOrders,
            NOT(
                ISBLANK(
                    RELATED(DimCustomer[Customer Segment])
                )
            )
        )
    ),
    "#,##0"
)
```

---

### CV 25th Percentile

**Purpose:** CV distribution analysis for data-driven XYZ threshold calibration.
**Report Page:** QA -

**DAX:**
```dax
CV 25th Percentile = 
PERCENTILEX.INC(
    ALL(DimProduct),
    [Coefficient of Variation],
    0.25
)
```

**Notes:**
- Validated value: **3.99** (post Entry #25 canceled order exclusion — prior value 3.96)
- Used as X/Y boundary in XYZ Classification thresholds. See Entry #17 and Entry #27.

---

### CV 50th Percentile

**Purpose:** Median CV across all active SKUs.
**Report Page:** QA -


**DAX:**
```dax
CV 50th Percentile = 
PERCENTILEX.INC(
    ALL(DimProduct),
    [Coefficient of Variation],
    0.50
)
```

**Notes:**
- Validated value: **7.81**

---

### CV 75th Percentile

**Purpose:** CV distribution analysis for data-driven XYZ threshold calibration.
**Report Page:** QA -

**DAX:**
```dax
CV 75th Percentile = 
PERCENTILEX.INC(
    ALL(DimProduct),
    [Coefficient of Variation],
    0.75
)
```

**Notes:**
- Validated value: **11.48** (post Entry #25 canceled order exclusion — prior value 10.91)
- Used as Y/Z boundary in XYZ Classification thresholds. See Entry #17 and Entry #27.

---

### CV 90th Percentile

**Purpose:** CV distribution tail analysis — confirms extent of erratic demand in top decile.
**Report Page:** QA -

**DAX:**
```dax
CV 90th Percentile = 
PERCENTILEX.INC(
    ALL(DimProduct),
    [Coefficient of Variation],
    0.90
)
```

**Notes:**
- Validated value: **17.95**

---

### Delivery Status Row Count

**Purpose:** Row count that respects Delivery Status filter context. Used on QA - Supply Performance page to verify delivery status distribution totals to 180,519.
**Report Page:** QA -

**DAX:**
```dax
Delivery Status Row Count = 
COUNTROWS(FactOrders)
```

**Notes:**
- Added because `Order Line Id` has Don't Summarize set per Entry #8 and cannot be implicitly aggregated. See [decision-log.md](./decision-log.md) Entry #8 downstream effect note.

---

### Earliest Order Date

**Purpose:** Confirms date range of loaded data matches source. Used on QA - Model Validation page to verify ....
**Report Page:** QA -

**Expected:** January 1, 2015<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Earliest Order Date = 
CALCULATE(
    MIN(FactOrders[Order Date]),
    ALL(FactOrders)
)
```

---

### Latest Order Date

**Purpose:** Confirms date range of loaded data matches source. 
**Report Page:** QA -

**Expected:** January 31, 2018<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Latest Order Date = 
CALCULATE(
    MAX(FactOrders[Order Date]),
    ALL(FactOrders)
)
```

---

### Product Match Check

**Purpose:** Confirms all FactOrders rows successfully join to DimProduct. 
**Report Page:** QA -

**Expected:** 180,519<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Product Match Check = 
FORMAT(
    COUNTROWS(
        FILTER(
            FactOrders,
            NOT(
                ISBLANK(
                    RELATED(DimProduct[Product Name])
                )
            )
        )
    ),
    "#,##0"
)
```

---

### Row Count Integrity

**Purpose:** Confirms source data loaded completely without row loss or duplication.
**Report Page:** QA -

**Expected:** 180,519<br>
**Actual**
**Status:**
 
**DAX:**
```dax
Row Count Integrity = 
FORMAT(
    COUNTROWS(FactOrders),
    "#,##0"
)
```

---

### SKU Count - A Tier
**Purpose:** Classification completeness check — A tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - A Tier = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "A"
)
```

**Notes:**
- Validated value: **11**
- A + B + C + Inactive must equal 118.

---

### SKU Count - B Tier
**Purpose:** Classification completeness check — B tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - B Tier = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "B"
)
```

**Notes:**
- Validated value: **22**

---

### SKU Count - C Tier
**Purpose:** Classification completeness check — C tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - C Tier = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "C"
)
```

**Notes:**
- Validated value: **85**

---

### SKU Count - Inactive
**Purpose:** Classification completeness check — inactive product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - Inactive = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[ABC Tier (Classification)] = "Inactive"
)
```

**Notes:**
- Validated value: **0** — all 118 products had revenue in the trailing 12-month window.

---

### SKU Count - Unclassified
**Purpose:** XYZ completeness check. Catches SKUs with revenue but a BLANK CV that fall outside X, Y, and Z.
**Report Page:** QA -

**Expected:** 0
**Actual:**
**Status:**

**DAX:**
```
SKU Count - Unclassified = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[XYZ Classification (DimColumn)] = "Unclassified"
)
```

**Notes:**
- Must equal 0. Any non-zero result means a SKU has trailing revenue but [Demand Std Dev (Daily)] or [Avg Daily Demand by SKU] returned BLANK, producing a BLANK CV and an unclassified label.
- These SKUs receive "Review" from [Cycle Count Schedule] and are excluded from X/Y/Z counts, so the standard completeness check (X + Y + Z = active SKU count) would not surface them.
- Investigate root cause if non-zero: most likely a SKU with a single transaction and no demand variability to calculate.

---

### SKU Count - X
**Purpose:** XYZ classification completeness check — X tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - X = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[XYZ Classification (DimColumn)] = "X"
)
```

**Notes:**
- Validated value:

---

### SKU Count - Y
**Purpose:** XYZ classification completeness check — Y tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - Y = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[XYZ Classification (DimColumn)] = "Y"
)
```

**Notes:**
- Validated value:

---

### SKU Count - Z
**Purpose:** XYZ classification completeness check — Z tier product count.
**Report Page:** QA -

**DAX:**
```dax
SKU Count - Z = 
CALCULATE(
    COUNTROWS(DimProduct),
    DimProduct[XYZ Classification (DimColumn)] = "Z"
)
```

**Notes:**
- Validated value:

---

### SKU Count at Rank (Tie Check)
**Purpose:** Regression check — identifies products sharing the same revenue rank due to Dense tie-handling. Returns 1 when no ties exist (all products have unique ranks). Any value > 1 flags a tied pair and signals that Dense ranking has produced a max displayed rank below 118.
**Report Page:** QA -

**DAX:**
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

**Notes:**
- `ThisRank` VAR captures this product's rank before the FILTER runs. Without it, the comparison always evaluates to TRUE and every product returns 118.
- Returns 1 when all products have unique revenue ranks — the normal healthy state. Returns 2 (or higher) when two or more products share the same revenue rank, which causes the ranking to skip numbers and the maximum rank to fall below 118.
- Validated value: **1** (post Entry #25 canceled order exclusion). Before exclusion, two products had identical 12-month revenue and this returned 2. Removing canceled orders changed one product's revenue enough to separate them.
- Monitor this on every model refresh. Any value above 1 means a tie exists and needs investigation.

---

### Canceled Line Items

**Purpose:** Confirms count of canceled order line items in the dataset. Validates the 7,754 figure used in Entry #23 canceled order exclusion decision.
**Report Page:** QA - Model Validation

**Expected:** 7,754<br>
**Validated:** 7,754 ✅

```dax
Canceled Line Items =
-- Count of FactOrders rows with Delivery Status = "Shipping canceled".
-- Validates the canceled row count used in Entry #23.
-- Expected: 7,754 -- matches validated delivery status distribution.

CALCULATE(
    COUNTROWS(FactOrders),
    FactOrders[Delivery Status] = "Shipping canceled"
)
```

**Notes:**
- Must equal 7,754. Any deviation signals a data load or ETL issue.
- Cross-reference with Delivery Status Row Count — Canceled Line Items + non-canceled rows must equal 180,519.
- See Entry #23.

---

### Canceled Order Units

**Purpose:** Confirms total units carried by canceled orders. Validates the 16,488 figure used in Entry #23. Confirms that excluding canceled orders meaningfully reduces unit totals used in demand calculations.
**Report Page:** QA - Model Validation

**Expected:** 16,488<br>
**Validated:** 16,488 ✅

```dax
Canceled Order Units =
-- Total Order Quantity units in canceled rows.
-- Validates the unit overstatement finding in Entry #23.
-- Expected: 16,488.

CALCULATE(
    SUM(FactOrders[Order Quantity]),
    FactOrders[Delivery Status] = "Shipping canceled"
)
```

**Notes:**
- Must equal 16,488. Confirms canceled orders carry meaningful unit values.
- If this returns 0 or BLANK, canceled orders carry no units and the exclusion has no impact on demand measures.
- See Entry #23.

---

### Orders With Canceled Lines

**Purpose:** Count of distinct orders containing at least one canceled line item. Used with Orders With Mixed Lines to confirm grain of canceled order exclusion.
**Report Page:** QA - Model Validation

**Expected:** 2,855<br>
**Validated:** 2,855 ✅

```dax
Orders With Canceled Lines =
-- Count of distinct Order IDs that have at least one Shipping canceled line.
-- Used alongside Orders With Mixed Lines to confirm exclusion grain.
-- Expected: 2,855.

CALCULATE(
    DISTINCTCOUNT(FactOrders[Order Id]),
    FactOrders[Delivery Status] = "Shipping canceled"
)
```

**Notes:**
- Must equal 2,855.
- 2,855 canceled orders × average 2.7 lines per order = 7,754 canceled line items. Consistent.
- See Entry #23.

---

### Orders With Mixed Lines

**Purpose:** Confirms no orders have both canceled and non-canceled line items. Validates that line-item grain exclusion and order-level exclusion produce identical results.
**Report Page:** QA - Model Validation

**Expected:** 0<br>
**Validated:** 0 ✅

```dax
Orders With Mixed Lines =
-- Count of orders containing BOTH canceled and non-canceled lines.
-- Must equal 0 to confirm line-item and order-level exclusion are equivalent.
-- If non-zero, split orders exist and exclusion logic must be reviewed.
-- See Entry #23.

CALCULATE(
    DISTINCTCOUNT(FactOrders[Order Id]),
    FactOrders[Delivery Status] = "Shipping canceled"
) -
CALCULATE(
    DISTINCTCOUNT(FactOrders[Order Id]),
    FILTER(
        VALUES(FactOrders[Order Id]),
        CALCULATE(
            COUNTROWS(FactOrders),
            FactOrders[Delivery Status] <> "Shipping canceled"
        ) = 0
    )
)
```

**Notes:**
- Must equal 0. Non-zero means split orders exist — some orders have both canceled and fulfilled lines. In that case line-item exclusion and order-level exclusion produce different results and the exclusion design must be revisited.
- Validated 0 across all customer states. See Entry #23.

---

## \_Measures\Core Measures

Six base measures that all downstream calculations depend on. Build these first.

All five aggregation measures exclude canceled orders using `KEEPFILTERS(Delivery Status <> "Shipping canceled")`. KEEPFILTERS intersects the exclusion with existing slicer context rather than replacing it — slicer responsiveness is preserved. Canceled orders were never fulfilled — including them overstates revenue, units, profit, and order counts. Validated: 7,754 canceled rows carrying 16,488 units across the dataset. See [decision-log.md](./decision-log.md) Entry #23.

`Avg Order Value` cascades automatically from `[Total Revenue]` and `[Total Orders]` — no DAX changes needed.

**Folder name:** Core Measures — signals base layer without implying a supply chain function. Previously named "Foundation." See [decision-log.md](./decision-log.md) Entry #16.

---

### Avg Order Value

**Table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `[Total Revenue]`, `[Total Orders]`<br>

```dax
Avg Order Value = 
DIVIDE(
    [Total Revenue],
    [Total Orders]
)
```

**Description:** Average revenue per fulfilled customer order in the selected period.

**Notes:**
- `DIVIDE` used instead of `/` — handles division by zero gracefully.
- Inherits canceled order exclusion from `[Total Revenue]` and `[Total Orders]`. No DAX changes needed.

---

### Avg Profit Margin %

**Table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `FactOrders[Profit Ratio]`, `FactOrders[Delivery Status]`<br>

```dax
Avg Profit Margin % =
-- Average gross profit margin across fulfilled order line items.
-- Canceled orders excluded -- never fulfilled.
-- KEEPFILTERS preserves slicer responsiveness. See Entry #23.

CALCULATE(
    AVERAGE(FactOrders[Profit Ratio]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

**Description:** Average gross profit margin across fulfilled order line items in the selected period.

**Notes:**
- `Profit Ratio` is a raw 0–1 decimal field in FactOrders. Format set to Percentage in model properties — no multiplication by 100 in DAX.
- Total row recalculates across all rows directly — not an average of averages.
- Canceled orders excluded. See Entry #23.

---

### Total Gross Profit

**Table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Gross Profit]`, `FactOrders[Delivery Status]`<br>

```dax
Total Gross Profit =
-- Total gross profit across fulfilled order line items.
-- Canceled orders excluded -- never fulfilled, overstate profit if included.
-- KEEPFILTERS preserves slicer responsiveness. See Entry #23.

CALCULATE(
    SUM(FactOrders[Order Gross Profit]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

**Description:** Total gross profit across fulfilled order line items in the selected period.

**Notes:**
- Enables implied COGS calculation per Limitation #2: `Implied COGS = [Total Revenue] - [Total Gross Profit]`.
- `Order Gross Profit` is a translation artifact from source Spanish column "beneficio" — confirmed as gross profit per ETL documentation.
- Canceled orders excluded. See Entry #23.

---

### Total Orders

**Table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `FactOrders[Order Id]`, `FactOrders[Delivery Status]`<br>

```dax
Total Orders =
-- Count of distinct fulfilled customer orders.
-- Canceled orders excluded -- never fulfilled.
-- Validated: 0 orders have mixed canceled and non-canceled lines.
-- KEEPFILTERS preserves slicer responsiveness. See Entry #23.

CALCULATE(
    DISTINCTCOUNT(FactOrders[Order Id]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

**Description:** Total number of distinct fulfilled customer orders in the selected period.

**Notes:**
- Uses `DISTINCTCOUNT` not `COUNTROWS`. FactOrders is at line-item grain. See [decision-log.md](./decision-log.md) Entry #13.
- Validated: 0 orders have mixed canceled and non-canceled lines — line-item exclusion correctly excludes all 2,855 canceled orders. See Entry #23.
- Canceled orders excluded. See Entry #23.

---

### Total Revenue

**Table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `FactOrders[Sales]`, `FactOrders[Delivery Status]`<br>

```dax
Total Revenue =
-- Total sales revenue across fulfilled order line items.
-- Canceled orders excluded -- never fulfilled, overstate revenue if included.
-- Base revenue measure. All downstream revenue calculations reference this.
-- KEEPFILTERS preserves slicer responsiveness. See Entry #23.

CALCULATE(
    SUM(FactOrders[Sales]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

**Description:** Total sales revenue across fulfilled order line items in the selected period.

**Notes:**
- Base revenue measure — do not modify. All downstream revenue calculations reference this.
- Respects all slicer and filter context. For locked classification revenue, use `[Revenue by SKU (12-Month Trailing)]`.
- Canceled orders excluded. See Entry #23.

---

### Total Units Sold

**Table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `FactOrders[Order Quantity]`, `FactOrders[Delivery Status]`<br>

```dax
Total Units Sold =
-- Total units across fulfilled order line items.
-- Canceled orders excluded -- 16,488 canceled units confirmed in dataset.
-- Including them overstates demand used in inventory planning calculations.
-- KEEPFILTERS preserves slicer responsiveness. See Entry #23.

CALCULATE(
    SUM(FactOrders[Order Quantity]),
    KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
)
```

**Description:** Total fulfilled units across all order line items in the selected period.

**Notes:**
- Canceled orders excluded. 16,488 canceled units confirmed in dataset. See Entry #23.

---

## \_Measures\Inventory Planning

**Architecture:** ABC tier drives service level (Z-score → Safety Stock). XYZ classification drives replenishment behavior (target days → Reorder Quantity). These are decoupled dimensions. See [decision-log.md](./decision-log.md) Entry #20.

**Canceled order exclusion:** `Avg Daily Demand by SKU` and `Demand Std Dev (Daily)` both exclude canceled orders. Canceled units overstate demand and inflate Safety Stock, Reorder Point, Reorder Quantity, and XYZ Classification downstream. Exclusion cascades automatically through all dependent measures. See [decision-log.md](./decision-log.md) Entry #25.

**Grain constraint:** Safety Stock, Reorder Point, Reorder Flag, Reorder Quantity, and Stock Coverage (Days) only calculate at Product Name grain. `[ABC Tier]` uses `ISINSCOPE(DimProduct[Product Name])` — returns BLANK at category, department, or total grain. This is intentional — these are per-SKU operational parameters. All report pages using these measures must include Product Name as a row dimension.

**Folder name:** Inventory Planning — aligned with APICS CPIM Planning domain terminology. Previously named "Safety Stock & Replenishment." See [decision-log.md](./decision-log.md) Entry #16.

---

### Avg Daily Demand by SKU `[Fixed Context]`

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Quantity]`, `FactOrders[Order Date]`, `FactOrders[Delivery Status]`, `DimDate[Date]`, `DimCustomer`<br>

```dax
Avg Daily Demand by SKU =
-- Average fulfilled units per day in the trailing 12-month window.
-- Canceled orders excluded -- 16,488 canceled units confirmed in dataset.
-- Including them overstates demand and inflates Safety Stock, Reorder Point,
-- Reorder Quantity, and XYZ Classification. See Entry #25.
-- `[Fixed Context]` ALL(DimCustomer) removes customer segment filters -- demand reflects total market.

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
        DimDate[Date] > WindowStart,
        DimDate[Date] <= WindowEnd,
        ALL(DimCustomer),
        -- Canceled orders excluded -- never fulfilled, overstate demand if included.
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
    )

VAR WindowDays =
    CALCULATE(
        COUNTROWS(DimDate),
        DimDate[Date] > WindowStart,
        DimDate[Date] <= WindowEnd
    )

RETURN
    DIVIDE(TotalUnits, WindowDays)
```

**Description:** Average fulfilled units per day in the trailing 12-month window. Represents expected daily demand for replenishment calculations.

**Notes:**
- `[Fixed Context]` `ALL(DimCustomer)` removes customer segment filters — demand reflects total market.
- `WindowDays` counts actual calendar days rather than hardcoding 365 — accounts for leap years.
- Window: Feb 1, 2017 → Jan 31, 2018 = 365 days for this dataset.
- Canceled orders excluded. See Entry #25.
- Primary input to Safety Stock and Reorder Point.

---

### Demand Std Dev (Daily) `[Fixed Context]`

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Order Quantity]`, `FactOrders[Order Date]`, `FactOrders[Delivery Status]`, `DimDate[Date]`<br>

```dax
Demand Std Dev (Daily) =
-- Standard deviation of daily fulfilled units in the trailing 12-month window.
-- Measures true daily demand variability -- not order size variability.
-- Canceled orders excluded -- never fulfilled, overstate variability if included.
-- See Entry #25.

VAR WindowEnd =
    CALCULATE(
        MAX(FactOrders[Order Date]),
        ALL(FactOrders)
    )

VAR WindowStart =
    EDATE(WindowEnd, -12)

VAR DailyDemandTable =
    CALCULATETABLE(
        SUMMARIZE(
            FILTER(
                FactOrders,
                FactOrders[Order Date] > WindowStart &&
                FactOrders[Order Date] <= WindowEnd &&
                -- Canceled orders excluded -- never fulfilled, overstate variability if included.
                FactOrders[Delivery Status] <> "Shipping canceled"
            ),
            FactOrders[Order Date],
            "DailyUnits", SUM(FactOrders[Order Quantity])
        ),
        ALL(DimCustomer)
    )

RETURN
    STDEVX.P(DailyDemandTable, [DailyUnits])
```

**Description:** Standard deviation of daily fulfilled units in the trailing 12-month window. Measures true daily demand variability — not order size variability.

**Notes:**
- `STDEV.P(FactOrders[Order Quantity])` (the original deprecated approach) measures order size variability, not daily demand variability. See Entry #20.
- `STDEVX.P` iterates over daily totals and correctly measures how much each day's demand deviates from the daily average.
- Canceled orders excluded inside the FILTER predicate — plain condition, not KEEPFILTERS. FILTER respects existing context natively; KEEPFILTERS is not needed here. See Entry #25.
- Critical input to Safety Stock formula: `σ²_demand`.

---

### Reorder Flag

**Table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Reorder Point]`, `DimProduct[Simulated Inventory Level]`<br>

**DAX:**
```dax
Reorder Flag = 

VAR SimInventory = SELECTEDVALUE(DimProduct[Simulated Inventory Level])

VAR ROP = [Reorder Point]

RETURN
    IF(
        ISBLANK(ROP) || ISBLANK(SimInventory),
        BLANK(),
        SWITCH(
            TRUE(),
            SimInventory = 0,       "🚨 Stockout",
            SimInventory <= ROP,    "⚠ Reorder Now",
            "✓ Stock OK"
        )
    )
```

**Description:** Current inventory status signal. Stockout = inventory depleted, order overdue. Reorder Now = place replenishment order today. Stock OK = no action needed.

**Notes:**
- Three-state logic. Stockout (inventory = 0) and Reorder Now (inventory ≤ Reorder Point) trigger different operational workflows.
- Returns BLANK at subtotal/total rows via `SELECTEDVALUE`.
- `[Test]` Depends on simulated inventory. In production replace source column only — no formula changes needed.

---

### Reorder Point

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `[Avg Daily Demand by SKU]`, `[Avg Lead Time (Actual)]`, `[Safety Stock]`<br>

**DAX:**
```dax

Reorder Point = 

VAR AvgDemand = [Avg Daily Demand by SKU]
VAR AvgLT = [Avg Lead Time (Actual)]
VAR SS = [Safety Stock]

VAR RawROP = (AvgDemand * AvgLT) + SS

RETURN
    IF(
        ISBLANK(SS),
        BLANK(),
        CEILING(RawROP, 1)
    )
```

**Description:** Inventory level at which a replenishment order should be triggered. Covers expected demand during lead time plus safety stock buffer.

**Notes:**
- `(Avg Daily Demand × Avg Lead Time)` covers expected demand during lead time. Safety Stock covers variability above that expectation.
- Unit: units of inventory (decimal — fractional trigger point).
- Returns BLANK for Inactive SKUs.

---

### Reorder Quantity

**Table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `DimProduct[XYZ Classification (DimColumn)]`, `[Avg Daily Demand by SKU]`, `[Safety Stock]`, `DimProduct[Simulated Inventory Level]`<br>

**DAX:**
```dax

Reorder Quantity = 

VAR XYZ =
    SELECTEDVALUE(DimProduct[XYZ Classification (DimColumn)])

VAR TargetDays =
    SWITCH(
        XYZ,
        "X", 60,    -- Stable — longer coverage, infrequent large orders
        "Y", 45,    -- Moderate — balanced replenishment cycle
        "Z", 30,    -- Erratic — short coverage, frequent small orders
        BLANK()
    )

VAR MaxDays = TargetDays * 1.5

VAR AvgDemand = [Avg Daily Demand by SKU]
VAR SS = [Safety Stock]

VAR SimInventory =
    SELECTEDVALUE(DimProduct[Simulated Inventory Level])

VAR TargetInventory = (TargetDays * AvgDemand) + SS
VAR MaxInventory = CEILING(MaxDays * AvgDemand, 1)
VAR RawQuantity = TargetInventory - SimInventory

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

**Description:** Suggested replenishment order quantity to bring inventory back to target coverage level based on XYZ demand predictability tier.

**Notes:**
- `[Caution]` Target days (60/45/30) are literature-based constants, not derived from this dataset's cost structure. In production, optimize using actual holding cost and ordering cost per SKU. See Limitation #7.
- `[Caution]` Continuous review model produces minimal differentiation for 81% of catalog (< 1 unit/day). Periodic review is the correct model for slow-moving SKUs. See Limitation #7.
- Lead time NOT added to formula — average demand during lead time is embedded in Reorder Point trigger. Adding here would double-count.
- Max cap = 1.5× target days per APICS min/max methodology.
- `[Test]` Depends on simulated inventory. In production replace source column only.

---

### Safety Stock

**Table:** FactOrders<br>
**Format:** Whole number, comma separator<br>
**Dependencies:** `[ABC Tier]`, `[Avg Lead Time (Actual)]`, `[Lead Time Variance Std Dev]`, `[Avg Daily Demand by SKU]`, `[Demand Std Dev (Daily)]`<br>

**DAX:**
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

VAR AvgLT = [Avg Lead Time (Actual)]
VAR StdDevLT = [Lead Time Variance Std Dev]
VAR AvgDemand = [Avg Daily Demand by SKU]
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

**Description:** Minimum buffer inventory required to maintain target service level given demand variability and lead time variability.

**Notes:**
- Combined variance formula — APICS standard. Source: APICS CPIM Body of Knowledge.
- Z-score by ABC tier: A = 1.65 (95%), B = 1.28 (90%), C = 1.04 (85%).
- `CEILING` rounds up — rounding down accepts more stockout risk than target service level.
- Returns BLANK for Inactive SKUs.
- Unit: units of inventory.

---

### Stock Coverage (Days)

**Table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `DimProduct[Simulated Inventory Level]`, `[Avg Daily Demand by SKU]`<br>

**DAX:**
```dax
Stock Coverage (Days) = 

VAR SimInventory =
    SELECTEDVALUE(DimProduct[Simulated Inventory Level])
VAR DailyDemand = [Avg Daily Demand by SKU]

VAR RawCoverage = DIVIDE(SimInventory, DailyDemand)

RETURN
    IF(
        ISBLANK(DailyDemand) || DailyDemand = 0,
        BLANK(),
        CEILING(RawCoverage, 1)
    )

```

**Description:** Estimated days of stock remaining at current average daily demand rate before stockout.

**Notes:**
- `CEILING` rounds up — partial days rounded up to avoid understating coverage.
- Returns BLANK at subtotal/total rows via `SELECTEDVALUE`.
- `[Test]` Depends on simulated inventory.
- Unit: days (whole number).

---

## \_Measures\Inventory Segmentation

The ABC and XYZ classification engine. **Measures below follow the display-folder tree**, not dependency order. When authoring in Power BI, build in **dependency order**: `Revenue by SKU` → `SKU Rank by Revenue` → `Cumulative Revenue %` → `ABC Tier` → `Coefficient of Variation` → `XYZ Classification`.

**Folder name:** Inventory Segmentation — aligned with APICS CPIM Planning domain terminology. Previously named "ABC Classification." See [decision-log.md](./decision-log.md) Entry #16.

---

### ABC Tier

**Table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Cumulative Revenue %]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**DAX:**
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

**Description:** ABC inventory tier for this product based on 12-month trailing revenue contribution. A = top 80% of revenue, B = next 15%, C = bottom 5%. Inactive = no revenue in trailing window.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows. Safety Stock, Reorder Point, Reorder Flag, and Reorder Quantity all depend on this measure — all require Product Name grain. See [decision-log.md](./decision-log.md) Entry #20.
- Inactive check fires first — products with zero trailing revenue excluded from classification before SWITCH evaluates.
- Thresholds are cumulative — `CumPct <= 0.95` for B tier implicitly means > 0.80 because A condition already failed.
- APICS 80/15/5 standard. Source: APICS CPIM Body of Knowledge.
- For slicing and filtering, use `DimProduct[ABC Tier (Classification)]` — measures cannot be used as slicer fields.

---

### Coefficient of Variation

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `[Demand Std Dev (Daily)]`, `[Avg Daily Demand by SKU]`<br>

**DAX:**
```dax
Coefficient of Variation = 
DIVIDE(
    [Demand Std Dev (Daily)],
    [Avg Daily Demand by SKU]
)
```

**Description:** Relative demand variability — standard deviation as a proportion of average daily demand. Normalizes variability across SKUs of different volumes so predictability can be compared regardless of scale.

**Notes:**
- CV strips out size difference between SKUs. A high-volume SKU with large absolute swings is not unfairly penalized vs a low-volume SKU.
- CV = 0.5 means demand typically varies ±50% around the average — regardless of whether a SKU sells 50 or 0.05 units/day.
- Primary input to `[XYZ Classification]`.
- `DIVIDE` handles zero average demand safely — returns BLANK for inactive SKUs.

---

### Cumulative Revenue %

**Table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `[SKU Rank by Revenue]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**DAX:**
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

**Description:** Percentage of total 12-month revenue accumulated by this product and all higher-ranked products. The Pareto curve expressed as a running percentage.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows.
- `TotalRev` uses `ALL(DimProduct)` to see all 118 products regardless of visual context.
- `CumulativeRev` uses FILTER to sum revenue of all products ranked at or below current rank.
- When this value crosses 0.80, next product enters B tier. When it crosses 0.95, next product enters C tier.
- Most computationally expensive measure in the model — FILTER iterates over all products for each row.

---

### Revenue by SKU (12-Month Trailing) `[Fixed Context]`

**Table:** FactOrders<br>
**Format:** Currency, 2 decimal places<br>
**Dependencies:** `[Total Revenue]`, `FactOrders[Order Date]`, `DimDate[Date]`, `DimCustomer`<br>

**DAX:**
```dax
Revenue by SKU (12-Month Trailing) = 
VAR WindowEnd = 
    CALCULATE(
        MAX(FactOrders[Order Date]),
        ALL(FactOrders) 
    )

VAR WindowStart = 
    EDATE(
        WindowEnd, 
        -12 
    )

RETURN
    CALCULATE (
        [Total Revenue],
        DimDate[Date] > WindowStart,
        DimDate[Date] <= WindowEnd,
        ALL (DimCustomer)
    )
```

**Description:** Total revenue for this product in the trailing 12-month window used for ABC classification. Not affected by date or customer slicers.

**Notes:**
- `[Fixed Context]` Locked context measure. `ALL(DimDate)` and `ALL(DimCustomer)` intentionally remove slicer filters — classification reflects total market demand.
- `WindowEnd` anchored to `MAX(FactOrders[Order Date])` not `MAX(DimDate[Date])` — DimDate includes 1-year padding. Using DimDate max produces an empty window. See [decision-log.md](./decision-log.md) Entry #17.
- `DimDate[Date] > WindowStart` (not `>=`) — ensures exactly 12 calendar months. `>=` adds an extra day.
- 12-month window: Feb 1, 2017 → Jan 31, 2018.
- For exploratory revenue analysis respecting slicer context, use `[Total Revenue]`.

---

### SKU Rank by Revenue

**Table:** FactOrders<br>
**Format:** Whole number<br>
**Dependencies:** `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**DAX:**
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

**Description:** Revenue rank of this product among all 118 products in the trailing 12-month window. Rank 1 = highest revenue.

**Notes:**
- `ISINSCOPE` guard returns BLANK at subtotal and total rows.
- `ALL(DimProduct)` iterator ensures ranking always against all 118 products regardless of visual filters.
- `Dense` tie-handling: if two products share identical trailing revenue, both receive the same rank and max displayed rank falls below 118. Post Entry #25 canceled order exclusion, no ties exist — max rank is 118. See `SKU Count at Rank (Tie Check)` in `_Validation`.
- Must evaluate at Product Name grain.

---

### XYZ Classification

**Table:** FactOrders<br>
**Format:** Text<br>
**Dependencies:** `[Coefficient of Variation]`, `[Revenue by SKU (12-Month Trailing)]`, `DimProduct`<br>

**DAX:**
```dax
XYZ Classification = 
IF(
    ISINSCOPE(DimProduct[Product Name]),
    VAR CV     = [Coefficient of Variation]
    VAR SKURev = [Revenue by SKU (12-Month Trailing)]
    RETURN
        IF(
            ISBLANK(SKURev) || SKURev = 0,
            "Inactive",
            SWITCH(
                TRUE(),
                ISBLANK(CV),    "Unclassified",
                CV <= 3.99,     "X",
                CV <= 11.48,    "Y",
                "Z"
            )
        )
)
```

**Description:** XYZ demand predictability classification based on Coefficient of Variation. X = stable demand, Y = moderate variability, Z = erratic demand. Drives replenishment cycle length in Reorder Quantity.

**Notes:**
- XYZ drives replenishment behavior (target days, order frequency). ABC drives service level (Z-score). These are separate dimensions. See Entry #17.
- **CV thresholds are data-driven, not assumed constants.** Standard literature thresholds (0.5/1.0) assume moderate-to-high velocity demand and are not applicable to this catalog where 81% of SKUs sell < 1 unit/day — near-zero demand mathematically produces very high CV values, making standard thresholds classify ~95% of SKUs as Z with no meaningful segmentation.
- Thresholds recalibrated post Entry #25 canceled order exclusion. Excluding canceled units shifted the entire CV distribution upward — demand variability increased relative to mean demand once inflated canceled units were removed. See Entry #27.
- Thresholds set at 25th percentile (3.99) and 75th percentile (11.48) of actual post-exclusion CV distribution — guarantees meaningful three-way split. X = bottom 25% most stable, Y = middle 50%, Z = top 25% most erratic.
- CV distribution (post-exclusion): 25th = 3.99, 50th = 8.30, 75th = 11.48, 90th = 18.67.
- For slicing and filtering, use `DimProduct[XYZ Classification (DimColumn)]` calculated column.

---

## \_Measures\Supply Performance

All six measures exclude canceled orders using `KEEPFILTERS(Delivery Status <> "Shipping canceled")` inside their RETURN CALCULATE blocks. KEEPFILTERS intersects the exclusion with existing slicer context rather than replacing it — slicer responsiveness is preserved. See [decision-log.md](./decision-log.md) Entry #24.

All six measures also include a row-grain guard using `NonCanceledCount` via `FILTER`. The FILTER-based guard uses a plain condition (not KEEPFILTERS) because FILTER already respects existing context without overriding it. See [decision-log.md](./decision-log.md) Entry #24.

**Folder name:** Supply Performance — aligned with SCOR Source domain terminology. Previously named "Lead Time & Reliability." See [decision-log.md](./decision-log.md) Entry #16.

**Validated delivery status distribution:**
| Status | Row Count |
|---|---|
| Late delivery | 98,977 |
| Advance shipping | 41,592 |
| Shipping on time | 32,196 |
| Shipping canceled | 7,754 |
| **Total** | **180,519** ✅ |

Non-canceled orders used in calculations: 172,765.

---

### Avg Lead Time (Actual)

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Shipping Days (Actual)]`, `FactOrders[Delivery Status]`<br>

```dax
Avg Lead Time (Actual) =
-- Average actual days from order placement to delivery.
-- Canceled orders excluded -- they never shipped, no delivery performance exists.
--
-- Row-grain guard: NonCanceledCount uses FILTER not CALCULATE.
-- CALCULATE overrides existing Delivery Status filter context.
-- At row grain in a canceled-only context, CALCULATE escapes and computes
-- across all non-canceled orders, returning a misleading average.
-- FILTER respects existing context. Returns BLANK when no non-canceled
-- rows exist in context. See Entry #24.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        CALCULATE(
            AVERAGE(FactOrders[Shipping Days (Actual)]),
            KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
        )
    )
```

**Description:** Average actual days from order placement to delivery, excluding canceled orders.

**Notes:**
- Unit: days. Validated total: 3.50 days.
- Primary lead time input to Safety Stock formula.
- `NonCanceledCount` guard prevents misleading average from appearing when canceled-only context is active. See Entry #24.

---

### Avg Lead Time (Scheduled)

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Shipping Days (Scheduled)]`, `FactOrders[Delivery Status]`<br>

```dax
Avg Lead Time (Scheduled) =
-- Average promised lead time at order placement.
-- Canceled orders excluded -- scheduled date is irrelevant if order never shipped.
--
-- Row-grain guard: same FILTER pattern as Avg Lead Time (Actual). See Entry #24.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        CALCULATE(
            AVERAGE(FactOrders[Shipping Days (Scheduled)]),
            KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
        )
    )
```

**Description:** Average promised lead time at order placement, excluding canceled orders.

**Notes:**
- Unit: days. Validated total: 2.93 days.
- `NonCanceledCount` guard applied for consistency. See Entry #24.

---

### Avg Lead Time Variance

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Lead Time Variance]`, `FactOrders[Delivery Status]`<br>

```dax
Avg Lead Time Variance =
-- Average days by which actual delivery exceeded scheduled delivery.
-- Positive = late on average. Canceled orders excluded.
--
-- Row-grain guard: same FILTER pattern as Avg Lead Time (Actual). See Entry #24.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        CALCULATE(
            AVERAGE(FactOrders[Lead Time Variance]),
            KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
        )
    )
```

**Description:** Average days by which actual delivery exceeded scheduled delivery. Positive = late on average.

**Notes:**
- Unit: days. Validated total: 0.57 days — orders are systematically late by over half a day on average.
- `NonCanceledCount` guard applied for consistency. See Entry #24.

---

### Late Delivery Rate %

**Table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `FactOrders[Late Delivery Risk]`, `FactOrders[Delivery Status]`<br>

```dax
Late Delivery Rate % =
-- Proportion of non-canceled orders that arrived later than scheduled.
-- Late Delivery Risk is a 0/1 column derived in Power Query from Lead Time Variance > 0.
-- Formula: Late orders / Total non-canceled orders.
-- Count-based pattern -- consistent with OTIF % formula structure.
-- AVERAGE of a 0/1 field produces the same result but obscures the formula intent.
--
-- Row-grain guard: NonCanceledCount uses FILTER not CALCULATE.
-- FILTER respects existing context. Returns BLANK when no non-canceled
-- rows exist in context. See Entry #24.
--
-- Filter string: "Shipping canceled" -- the exact value in the dataset.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

-- Count orders where Late Delivery Risk = 1 (arrived after scheduled date).
VAR LateOrders =
    CALCULATE(
        COUNTROWS(FactOrders),
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled"),
        FactOrders[Late Delivery Risk] = 1
    )

-- Total non-canceled orders -- the denominator.
VAR TotalOrders =
    CALCULATE(
        COUNTROWS(FactOrders),
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        DIVIDE(LateOrders, TotalOrders)
    )
```

**Description:** Proportion of non-canceled orders that arrived later than scheduled.

**Notes:**
- Count-based formula — consistent with OTIF % pattern. Validated total: 57.3%. High rate reflects DataCo dataset characteristics — not a measure error. See Limitation #6.
- 98,977 late ÷ 172,765 non-canceled = 57.3%

---

### Lead Time Variance Std Dev

**Table:** FactOrders<br>
**Format:** Decimal number, 2 decimal places<br>
**Dependencies:** `FactOrders[Lead Time Variance]`, `FactOrders[Delivery Status]`<br>

```dax
Lead Time Variance Std Dev =
-- Standard deviation of lead time variance across non-canceled orders.
-- Measures how consistently late or early orders arrive.
-- Higher value = more unpredictable fulfillment.
-- Canceled orders excluded.
--
-- Row-grain guard: same FILTER pattern as Avg Lead Time (Actual). See Entry #24.
-- Note: at true single-row grain, STDEV.P of one value returns 0, not BLANK.
-- The guard prevents the more misleading case where a canceled-only context
-- causes CALCULATE to escape and compute std dev across the full dataset.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        CALCULATE(
            STDEV.P(FactOrders[Lead Time Variance]),
            KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
        )
    )
```

**Description:** Standard deviation of lead time variance — measures how consistently late or early orders arrive. Higher value = more unpredictable fulfillment.

**Notes:**
- Unit: days. Validated total: 1.49 days.
- `STDEV.P` used — 180,519 rows is the full population, not a sample. `STDEV.S` overstates variance.
- Critical input to Safety Stock formula: `σ²_LT`. Squaring inside the formula converts standard deviation to variance.
- At true single-row grain, `STDEV.P` of one value returns 0, not BLANK. Expected behavior. See Entry #24.

---

### On-Time Delivery Rate %

**Table:** FactOrders<br>
**Format:** Percentage, 2 decimal places<br>
**Dependencies:** `FactOrders[Late Delivery Risk]`, `FactOrders[Delivery Status]`<br>

```dax
On-Time Delivery Rate % =
-- Proportion of non-canceled orders that arrived on time or early.
-- Formula: On-time orders / Total non-canceled orders.
-- Count-based pattern -- consistent with OTIF % and Late Delivery Rate % formula structure.
--
-- Cannot write 1 - [Late Delivery Rate %] and inherit the canceled guard.
-- When Late Delivery Rate % returns BLANK for canceled rows,
-- 1 - BLANK() = 1 - 0 = 100% in DAX. Arithmetic on BLANK treats it as 0.
-- Explicit guard and independent calculation required. See Entry #24.

VAR NonCanceledCount =
    COUNTROWS(
        FILTER(
            FactOrders,
            FactOrders[Delivery Status] <> "Shipping canceled"
        )
    )

-- Count orders where Late Delivery Risk = 0 (arrived on or before scheduled date).
VAR OnTimeOrders =
    CALCULATE(
        COUNTROWS(FactOrders),
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled"),
        FactOrders[Late Delivery Risk] = 0
    )

-- Total non-canceled orders -- the denominator.
VAR TotalOrders =
    CALCULATE(
        COUNTROWS(FactOrders),
        KEEPFILTERS(FactOrders[Delivery Status] <> "Shipping canceled")
    )

RETURN
    IF(
        NonCanceledCount = 0,
        BLANK(),
        DIVIDE(OnTimeOrders, TotalOrders)
    )
```

**Description:** Proportion of non-canceled orders that arrived on time or early.

**Notes:**
- Count-based formula — consistent with Late Delivery Rate % and OTIF % pattern. Validated total: 42.7%.
- Explicit guard required — `1 - BLANK()` evaluates to 100% in DAX, not BLANK. Arithmetic on BLANK treats it as 0. See Entry #24.

---

## \_Measures\Financial Impact

⬜ **Pending — Phase 3 build in progress.**

Measures to be built:

| Measure | Description | Data type |
|---|---|---|
| `Implied COGS` | `[Total Revenue] - [Total Gross Profit]` | Real data |
| `Gross Profit by ABC Tier` | Gross profit filtered to A/B/C tier | Real data |
| `Avg Margin % by ABC Tier` | Average profit margin by tier | Real data |
| `Pareto Concentration Ratio` | % of SKUs driving 80% of revenue | Real data |
| `Carrying Cost Estimate` | Inventory value × 25% annual rate (APICS standard) | Modeled estimate `[Caution]` |
| `Revenue at Risk (Supply)` | A-tier revenue × Late Delivery Rate % — exposure to late supplier delivery | Real data |
| `Revenue at Risk (Stockout)` | Flagged SKUs × Avg Daily Demand × Avg Lead Time × Unit Price — potential lost revenue during active stockout window | Modeled estimate `[Test]` |

**Carrying cost rate:** 25% annual. APICS standard range 20–30%. Components: capital cost ~12%, storage ~4%, obsolescence/shrinkage ~5%, insurance/taxes ~4%. Source: APICS CPIM Body of Knowledge.

**Revenue at Risk note:** Two measures answer different questions:
- `Revenue at Risk (Supply)` — how much A-tier revenue is exposed to late supplier delivery? Uses only real data.
- `Revenue at Risk (Stockout)` — how much revenue could be lost during active stockout windows? Labeled as modeled estimate — depends on simulated inventory.

---

## \_Measures\Trend Analysis

⬜ **Pending — Phase 3 build in progress.**

Measures to be built:

| Measure | Description |
|---|---|
| `Revenue YoY` | Year-over-year revenue comparison using `SAMEPERIODLASTYEAR` |
| `Revenue vs Prior Period` | Period-over-period comparison using `DATEADD` |
| `Rolling 90-Day Demand` | Trailing 90-day demand using `DATESINPERIOD` |

**Folder name:** Trend Analysis — business-facing label replacing "Time Intelligence" which is a Power BI developer term. See [decision-log.md](./decision-log.md) Entry #16.

---

## DimProduct Calculated Columns

Calculated columns stored in DimProduct. Evaluated at model refresh time — stable between refreshes. Required for slicers, row-level filtering, and visual grouping. Measure equivalents exist in FactOrders for dynamic filter context evaluation.

**Folder:** Inventory Segmentation — previously named "ABC Classification." Aligned with APICS CPIM Planning domain. See [decision-log.md](./decision-log.md) Entry #16.

---

### ABC Tier (Classification)

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Format:** Text<br>
**Measure equivalent:** `[ABC Tier]` in FactOrders<br>

**DAX:**
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
- For dynamic filter context, use `[ABC Tier]` measure.
- Naming conflict: Power BI does not allow same name for measure and column across model. Column named `ABC Tier (Classification)` to distinguish.
- Column and measure refresh in same operation — consistency guaranteed post-refresh. Gap only exists between refreshes. See [decision-log.md](./decision-log.md) Entry #17.

---

### Cycle Count Schedule

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Format:** Text<br>
**Dependencies:** `DimProduct[ABC Tier (Classification)]`, `DimProduct[XYZ Classification (DimColumn)]`<br>

**DAX:**
```dax
Cycle Count Schedule =
VAR ABCTier = [ABC Tier (Classification)]
VAR XYZTier = [XYZ Classification (DimColumn)]
VAR Matrix  = ABCTier & XYZTier
RETURN
    IF(
        ABCTier = "Inactive",
        "Annual",
        SWITCH(
            Matrix,
            "AX", "Monthly",
            "AY", "Monthly",
            "AZ", "Weekly",
            "BX", "Quarterly",
            "BY", "Quarterly",
            "BZ", "Monthly",
            "CX", "Semi-Annual",
            "CY", "Semi-Annual",
            "CZ", "Quarterly",
            "Review"
        )
    )
```

**Description:** Recommended cycle count frequency for this product based on combined ABC revenue tier and XYZ demand predictability classification.

**Notes:**
- APICS-aligned: count frequency proportional to both revenue contribution and demand variability.
- AZ (high revenue, erratic demand) = weekly - most dangerous combination. Stockout is expensive AND demand is unpredictable.
- CX (low revenue, stable demand) = semi-annual - safest combination. Low stockout cost AND predictable demand.
- "Review" returned for unexpected combinations - signals unclassified SKUs needing manual assessment.
- Stamped at refresh time. Updates automatically when ABC or XYZ classification changes on refresh.

**ABC XYZ Cycle Count Matrix:**

| | X (Stable) | Y (Moderate) | Z (Erratic) |
|---|---|---|---|
| **A** | Monthly | Monthly | **Weekly** |
| **B** | Quarterly | Quarterly | Monthly |
| **C** | Semi-Annual | Semi-Annual | Quarterly |
| **Inactive** | Annual | Annual | Annual |

---

### Demand Velocity Band (DEPRECATED - XYZ Classification) `[Deprecated]`

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Replaced by:** `DimProduct[XYZ Classification (DimColumn)]`<br>
**Reason:** Velocity banding without CV normalization. Superseded by XYZ Classification (DimColumn). See Entry #17.

---

### Simulated Inventory Level `[Test]`

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Format:** Whole number<br>
**Dependencies:** `[Reorder Point]`, `DimProduct[SKU Revenue Rank (DimColumn)]`<br>

**DAX:**
```dax
Simulated Inventory Level = 
-- Simulates a realistic inventory position for testing reorder flag logic.
-- Three states distributed deterministically by revenue rank:
--   10% Stockout    (MOD = 0)       → inventory = 0
--   30% Reorder Now (MOD = 1, 2, 3) → inventory at 50% of Reorder Point
--   60% Stock OK    (MOD = 4–9)     → inventory at 150% of Reorder Point
--
-- SKU Revenue Rank (DimColumn) used as MOD input — sequential 1–118,
-- guarantees even bucket distribution across all 118 SKUs.
-- Product Card Id avoided — non-sequential arbitrary identifier,
-- MOD of arbitrary integers does not guarantee even bucket distribution.
-- 118 SKUs ÷ 10 buckets = ~12 SKUs per bucket, achieving ~10/30/60 split.
-- Bucket assignment is stable across refreshes — deterministic, not random.
-- Reorder Point derived from actual historical demand — simulation is operationally grounded.
-- See Limitation #1 and Entry #26.

-- Pull the current Reorder Point for this SKU.
-- BLANK for Inactive SKUs — no simulation needed outside active inventory scope.
VAR ROP = [Reorder Point]

-- Reference the pre-computed revenue rank column on DimProduct.
-- Stored at refresh time — no measure evaluation needed at runtime.
-- Avoids context issues with [Revenue by SKU (12-Month Trailing)] in column context.
VAR RevenueRank = DimProduct[SKU Revenue Rank (DimColumn)]

-- MOD(RevenueRank, 10) maps the sequential rank to a bucket 0–9.
-- Rank 10 → bucket 0, Rank 20 → bucket 0, Rank 11 → bucket 1, etc.
-- Even distribution: each bucket receives ~11–12 of the 118 SKUs.
VAR Bucket = MOD(RevenueRank, 10)

-- 50% of ROP places inventory below the reorder threshold.
-- CEILING rounds up to nearest whole unit — no fractional inventory.
-- This level triggers Reorder Now in Reorder Flag.
VAR ReorderNowLevel = CEILING(ROP * 0.5, 1)

-- 150% of ROP places inventory comfortably above the reorder threshold.
-- CEILING rounds up to nearest whole unit.
-- This level triggers Stock OK in Reorder Flag.
VAR StockOKLevel = CEILING(ROP * 1.5, 1)

RETURN
    IF(
        -- Guard: Inactive SKUs have no Reorder Point — return BLANK.
        -- No simulation needed for products outside active inventory management scope.
        ISBLANK(ROP),
        BLANK(),
        SWITCH(
            TRUE(),
            -- Bucket 0: ~10% of SKUs (ranks 10, 20, 30...) → Stockout
            Bucket = 0,     0,
            -- Buckets 1–3: ~30% of SKUs (ranks ending in 1, 2, 3...) → Reorder Now
            Bucket <= 3,    ReorderNowLevel,
            -- Buckets 4–9: ~60% of SKUs (ranks ending in 4–9...) → Stock OK
            StockOKLevel
        )
    )
```

**Notes:**
- `[Test]` Testing infrastructure only. No analytical conclusions from this column.
- Three-state ROP-anchored design: Stockout (0), Reorder Now (50% of ROP), Stock OK (150% of ROP).
- Revenue rank used as MOD input for guaranteed even bucket distribution — Product Card Id is non-sequential and produces uneven distribution.
- Validated distribution: 11 Stockout (~10%), 36 Reorder Now (~30%), 71 Stock OK (~60%). See Entry #26.
- In production: replace with live WMS/ERP on-hand snapshot. `Reorder Flag`, `Stock Coverage (Days)`, `Reorder Quantity` require no formula changes.

---

### SKU Revenue Rank (DimColumn) `[Test]`

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Format:** Whole number<br>
**Dependencies:** `[Revenue by SKU (12-Month Trailing)]`<br>

**DAX:**
```dax
SKU Revenue Rank (DimColumn) = 
RANKX(
    ALL(DimProduct),
    [Revenue by SKU (12-Month Trailing)],
    ,
    DESC,
    Dense
)
```

**Notes:**
- `[Test]` Infrastructure column supporting `Simulated Inventory Level`. Not a business-facing column.
- Stores revenue rank per product at refresh time — avoids context evaluation issues with `[Revenue by SKU (12-Month Trailing)]` inside calculated column iteration.
- Dense tie-handling: if two products share identical trailing revenue, both receive the same rank. Post Entry #25 canceled order exclusion, no ties exist — all 118 products have unique ranks producing ranks 1–118.
- Mirrors `[SKU Rank by Revenue]` measure in FactOrders but without the `ISINSCOPE` guard — that guard is needed in measure context but not in calculated column context where row context is always active.
- See Entry #26.

---

### XYZ Classification (DimColumn)

**Table:** DimProduct<br>
**Folder:** Inventory Segmentation<br>
**Format:** Text<br>
**Measure equivalent:** `[XYZ Classification]` in FactOrders<br>

**DAX:**
```dax
XYZ Classification (DimColumn) = 
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
            CV <= 3.99,     "X",
            CV <= 11.48,    "Y",
            "Z"
        )
    )
```

**Notes:**
- Use for slicers and visual row grouping.
- For dynamic filter context, use `[XYZ Classification]` measure.
- Thresholds: X ≤ 3.99 (bottom 25%), Y 3.99–11.48 (middle 50%), Z > 11.48 (top 25%). Recalibrated post Entry #25 canceled order exclusion. See Entry #27.

---

## ABC XYZ Cycle Count Schedule

The combined ABC XYZ matrix is the industry-standard idea for tying cycle count frequency to both revenue concentration and demand variability. In this model, neither axis assigns cadence alone: `DimProduct[Cycle Count Schedule]` evaluates the pair (ABC tier letter + XYZ class letter), e.g. `AX` or `BZ`, and returns one label via `SWITCH`. **ABC tier** encodes how costly an inventory error is in revenue terms. **XYZ class** encodes demand variability (CV), a proxy for how easily system-to-physical error can stay hidden between counts. Together they yield nine active combinations (plus Inactive → Annual) with differentiated schedules.

Operationally, count frequency is always a joint outcome of that pair—not “XYZ only.” **Z** (high CV) tightens counts relative to **X** and **Y** at the same ABC row because erratic demand provides less natural signal between formal counts. In this DataCo catalog, **X and Y share the same frequency at every ABC tier** (see [decision-log.md](./decision-log.md) Entry #22): breakpoints are this dataset’s 25th / 75th CV percentiles (3.99 / 11.48, recalibrated post Entry #25 exclusion — see Entry #27), and low average daily demand inflates CV so X vs Y does not separate discrepancy risk enough to merit different cadences—Z is the operationally meaningful split. A miscounted A item still drives the highest revenue exposure; a Z item still compounds error longest. Under a different population—e.g. textbook XYZ bands on a normalized or low CV scale (such as 0–1) suited to higher-velocity catalogs—X and Y would often map to different frequencies and the matrix would need to be rebuilt.

**Source:** APICS CPIM Body of Knowledge — cycle count frequency proportional to revenue contribution and demand variability.

### Matrix

| | X (Stable, CV ≤ 3.99) | Y (Moderate, CV 3.99–11.48) | Z (Erratic, CV > 11.48) |
|---|---|---|---|
| **A** (top 80% revenue) | Monthly | Monthly | **Weekly** |
| **B** (80–95% revenue) | Quarterly | Quarterly | Monthly |
| **C** (bottom 5% revenue) | Semi-Annual | Semi-Annual | Quarterly |
| **Inactive** | Annual | Annual | Annual |

### Operational Rationale by Combination

| Combination | Frequency | Rationale |
|---|---|---|
| AZ | Weekly | Highest revenue + most erratic demand = maximum risk of costly undetected discrepancy |
| AX / AY | Monthly | High revenue justifies frequent counts regardless of demand stability |
| BZ | Monthly | Moderate revenue + erratic demand — discrepancy risk warrants elevated frequency |
| BX / BY | Quarterly | Moderate revenue, manageable risk with quarterly review |
| CZ | Quarterly | Low revenue but erratic — periodic check prevents accumulation of small errors |
| CX / CY | Semi-Annual | Low revenue, stable demand — minimal discrepancy risk, low count frequency acceptable |
| Inactive | Annual | Outside active classification scope — annual physical audit sufficient |

### Implementation

`Cycle Count Schedule` is a calculated column in DimProduct. It concatenates ABC and XYZ tier labels and uses SWITCH to assign frequency. Refreshes automatically with model refresh — no manual updates needed when tier assignments change.

---

## Build Order Reference

| Layer | Measures | Must build after |
|---|---|---|
| 1 - Core Measures | Total Revenue, Total Gross Profit, Total Units Sold, Total Orders, Avg Order Value, Avg Profit Margin % | Nothing |
| 2 - Inventory Segmentation | Revenue by SKU (12-Month Trailing), SKU Rank by Revenue, Cumulative Revenue %, ABC Tier | Layer 1 complete |
| 2a - DimProduct columns | ABC Tier (Classification) | Layer 2 complete |
| 3 - Supply Performance | All six lead time measures | Layer 1 complete |
| 4 - Inventory Planning | Avg Daily Demand by SKU, Demand Std Dev (Daily), Coefficient of Variation (**Note: Layer 4 build sequence only**), XYZ Classification, Safety Stock, Reorder Point, Reorder Quantity, Reorder Flag, Stock Coverage (Days) | Layers 2 and 3 complete |
| 4a - DimProduct columns | SKU Revenue Rank (DimColumn), XYZ Classification (DimColumn), Simulated Inventory Level, Cycle Count Schedule | Layer 4 complete |
| 5 - Financial Impact | Implied COGS, Gross Profit by ABC Tier, Avg Margin % by ABC Tier, Pareto Concentration Ratio, Carrying Cost Estimate, Revenue at Risk (Supply), Revenue at Risk (Stockout) | Layers 1–4 complete |
| 6 - Trend Analysis | Revenue YoY, Revenue vs Prior Period, Rolling 90-Day Demand | Layer 1 complete |

---

## Measure Dependency Map

```
Total Revenue ──────────────────────────────────────────► Revenue by SKU (12-Month Trailing)
                                                                    │
                                                          SKU Rank by Revenue
                                                                    │
                                                          Cumulative Revenue %
                                                                    │
                                                              ABC Tier ──────────────► Z-Score input
                                                                    │                        │
                                                     ABC Tier (Classification)               │
                                                     Cycle Count Schedule ◄─────────────────┤
                                                                                             │
Avg Daily Demand by SKU ──────────────────────────────────► Safety Stock ◄── Lead Time Variance Std Dev
        │                                                         │                    │
        │                                                   Reorder Point       Avg Lead Time (Actual)
        │                                                         │
        │                                                   Reorder Flag
        │                                                   Reorder Quantity ◄── XYZ Classification
        │                                                   Stock Coverage (Days) ◄── Simulated Inventory ◄── SKU Revenue Rank (DimColumn)
        │                                                                                              ◄── Reorder Point
        │
Demand Std Dev (Daily) ──► Coefficient of Variation ──► XYZ Classification
                                                               │
                                                    XYZ Classification (column)
                                                    Cycle Count Schedule ◄──────────────────┘
```

---

*Document Version: 1.6 — Phase 3 DAX Layer (XYZ threshold recalibration)*<br>
*Changes from v1.5:*
- *XYZ Classification: thresholds updated from 3.96/10.91 to 3.99/11.48 — recalibrated post Entry #25 canceled order exclusion (Entry #27)*
- *XYZ Classification (DimColumn): same threshold update (Entry #27)*
- *CV 25th Percentile: validated value updated from 3.96 to 3.99 (Entry #27)*
- *CV 75th Percentile: validated value updated from 10.91 to 11.48 (Entry #27)*
- *XYZ Classification notes: CV distribution updated to post-exclusion values (Entry #27)*
- *Cycle Count Schedule narrative: threshold references updated to 3.99/11.48 (Entry #27)*
- *Cycle Count Schedule matrix header: threshold ranges updated (Entry #27)*<br>
*Next Update: OTIF % added to Supply Performance*
