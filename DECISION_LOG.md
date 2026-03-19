# Decision Log
## ABC Product Segmentation & Inventory Management
### Power BI Project — Phase 1: ETL Layer

---

> This log documents key architectural, modeling, and analytical decisions made during the ETL build phase. Each entry captures the decision, the reasoning, and the operational or analytical implication. This log serves as the authoritative reference for understanding why the model is built the way it is.

---

## Entry #1 — Dynamic Late Delivery Risk Derivation

**Date:** March 2026
**Layer:** Power Query — FactOrders

**Decision:**
Replace the source `Late_delivery_risk` column with a dynamically derived column based on actual Lead Time Variance.

**Context:**
The DataCo source dataset includes a `Late_delivery_risk` binary flag. Investigation revealed this column was a pre-labeled machine learning target variable created by the DataCo researchers for classification model training — not an operationally derived metric. Empirical testing found 5 mismatches per 1,000 rows between the source flag and actual Lead Time Variance outcomes, confirming the flag was generated independently from the data it appears to describe.

**What We Did:**
- Removed the source `Late_delivery_risk` column after renaming
- Replaced with two derived columns:
  - `Late Delivery Risk` (Int64) — `1` if `Lead Time Variance > 0`, else `0` — for DAX arithmetic
  - `Is Late Delivery Risk` (Logical) — `True` if `Lead Time Variance > 0` — for report visuals and slicers

**Formula:**
```
Late Delivery Risk = if [Lead Time Variance] > 0 then 1 else 0
Is Late Delivery Risk = [Lead Time Variance] > 0
```

**Why This Is Better:**
- Fully transparent — logic is auditable by any developer
- Dynamically recalculates when source data is refreshed
- Operationally grounded — derived from actual shipping performance
- Dual column pattern serves both analytical and presentation layers

**Storytelling Angle:**
*"I found a black box in my dataset, tested it empirically, and replaced it with a transparent derivation. Here's how."*

---

## Entry #2 — MasterSet Staging Pattern with Load Disabled

**Date:** March 2026
**Layer:** Power Query — MasterSet

**Decision:**
Implement MasterSet as a load-disabled staging query. All dimension and fact queries reference MasterSet as their single source of truth.

**Context:**
Initial build had all five queries — DimDate, DimProduct, DimCustomer, FactOrders, and MasterSet — each loading the source CSV independently. This meant the CSV was being read five times on every refresh — five separate I/O operations on a 180K+ row file.

**What We Did:**
- Designated MasterSet as the sole CSV loading query
- Set MasterSet `Enable Load = False` — removes it from the model entirely
- DimProduct, DimCustomer, FactOrders set as reference queries: `Source = MasterSet`
- DimDate reads `List.Min` and `List.Max` directly from MasterSet column inline

**Architecture:**
```
CSV Source
    └── MasterSet (load disabled — staging only)
            ├── DimProduct (reference query)
            ├── DimCustomer (reference query)
            ├── FactOrders (reference query)
            └── DimDate (reads Min/Max from MasterSet inline)
```

**Benefits:**
- CSV loads exactly once per refresh — zero redundant I/O
- Single source of truth — schema changes propagate automatically
- MasterSet never surfaces in the published model — clean field list
- Zero-touch updates when source data is refreshed or extended

**Storytelling Angle:**
*"One source, one load, zero redundancy. This is what enterprise ETL architecture actually looks like."*

---

## Entry #3 — PII Column Exclusion from DimCustomer

**Date:** March 2026
**Layer:** Power Query — DimCustomer

**Decision:**
Exclude `Customer Email` and `Customer Password` from DimCustomer even though both fields exist in MasterSet.

**Context:**
The source dataset contained Customer Email and Customer Password columns. Both were already masked by the DataCo dataset creators — values displayed as `XXXXXXXX` across all records — indicating the original researchers recognized PII sensitivity and anonymized before publishing.

**What We Did:**
- Excluded both columns from DimCustomer's `Table.SelectColumns` step
- Combined with MasterSet load-disabled setting — masked fields never surface at any layer of the published model

**Why This Matters:**
Retaining encrypted or placeholder PII columns — even masked ones — adds zero analytical value while creating unnecessary questions about data governance intent. Removing them is a deliberate modeling decision, not a cleanup step.

Note: Disabling MasterSet load is a modeling configuration, not a security measure. Anyone with access to the .pbix file can re-enable load and view raw query data. True security requires publishing to Power BI Service with workspace access controls.

**Storytelling Angle:**
*"The data was already anonymized by the researchers. I took it further — removing PII fields entirely from the analytical model. Good data governance means not carrying what you don't need."*

---

## Entry #4 — Column Definitions Documentation

**Date:** March 2026
**Layer:** Documentation

**Decision:**
Maintain a `COLUMN_DEFINITIONS.md` file in the GitHub repository mapping all renamed columns to their source names, documenting derived columns with formulas, and recording dropped columns with reasons for exclusion.

**Context:**
Significant column renaming occurred across FactOrders — 16 columns renamed from source names to business-friendly labels. Without documentation, any developer or future self attempting to trace a Power BI field back to the source CSV would have no reference.

**What We Built:**
- `COLUMN_DEFINITIONS.md` — covers all five queries
- Maps every renamed column to its source name
- Documents derived columns with calculation logic
- Records dropped columns with reason for exclusion
- Documents data type changes with justification

**Storytelling Angle:**
*"Renaming columns for business users is standard practice. Documenting those renames for developers is what separates a maintainable model from a black box."*

---

## Entry #5 — Boolean Flag Dual Column Pattern

**Date:** March 2026
**Layer:** Power Query — FactOrders

**Decision:**
Maintain two versions of each binary flag — integer (0/1) for DAX arithmetic and logical (True/False) for report consumers.

**Context:**
`Late Delivery Risk` needed to serve two purposes simultaneously:
1. DAX aggregations — `SUM`, `AVERAGE`, safety stock formula inputs
2. Report visuals — slicers, conditional formatting, KPI cards

A single column cannot cleanly serve both purposes. DAX arithmetic requires integer type. Report consumers find True/False more intuitive than 0/1 in slicers and visuals without requiring a legend.

**Pattern Applied:**
- `Late Delivery Risk` (Int64) — `0` or `1` — DAX arithmetic layer
- `Is Late Delivery Risk` (Logical) — `True` or `False` — presentation layer

**Naming Convention:**
`Is` prefix denotes logical boolean columns throughout the model.

**DAX Usage Pattern:**
```dax
// Integer version — arithmetic
Late Delivery Rate = AVERAGE(FactOrders[Late Delivery Risk])

// Logical version — filter
Late Orders = CALCULATE(COUNTROWS(FactOrders), FactOrders[Is Late Delivery Risk] = TRUE())
```

**Storytelling Angle:**
*"A column's data type is a UX decision as much as a technical one. I maintained two versions of each flag — one for the machine, one for the human."*

---

## Entry #6 — Product Description Column Exclusion

**Date:** March 2026
**Layer:** Power Query — DimProduct

**Decision:**
Exclude `Product Description` from DimProduct after confirming the column is empty across all 118 products.

**Validation:**
Column profiling on `Product Description` in DimProduct showed:
- Count: 118
- Empty: 118
- Distinct: 0

**Decision Criteria:**
- Does it have data? No
- Could it have data in a refreshed dataset? No — consistently empty
- Does any downstream calculation depend on it? No

All three criteria confirm exclusion. Empty columns consume memory, appear in the field list creating confusion, and signal poor data hygiene.

---

## Entry #7 — Enterprise Fact Table Column Ordering

**Date:** March 2026
**Layer:** Power Query — FactOrders

**Decision:**
Reorder FactOrders columns following enterprise fact table column ordering convention.

**Standard Applied:**
```
Primary Key → Foreign Keys → Date Keys → Degenerate Dimensions → Measures → Derived Columns
```

**Why Column Order Matters:**
Column order in a fact table is a communication tool. It tells any developer who opens the file exactly what the table's grain is, what dimensions it connects to, and where the measurable facts live. Predictable structure reduces onboarding time for collaborators.

**Reference:**
Kimball dimensional modeling methodology — fact table design principles.

**Storytelling Angle:**
*"Column order in a fact table isn't cosmetic — it's documentation. The first thing an experienced data modeler reads when they open your file is the fact table column order."*

---

## Entry #8 — "Benefit per order" Translation Artifact

**Date:** March 2026
**Layer:** Power Query — FactOrders

**Decision:**
Rename `Benefit per order` to `Order Gross Profit` in FactOrders.

**Context:**
The source column name `Benefit per order` is a translation artifact. The DataCo dataset was originally created in Spanish where "beneficio" means profit/benefit. The field represents gross profit per order — revenue minus cost.

**Renamed To:** `Order Gross Profit`

**Why This Matters for Documentation:**
Without this note, any developer seeing the rename from "Benefit per order" to "Order Gross Profit" might question whether the business logic changed. It did not — only the label was corrected to accurately reflect the field's meaning in English.

---

## Entry #9 — Dynamic DimDate with Source-Derived Padding

**Date:** March 2026
**Layer:** Power Query — DimDate

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
- Zero hardcoded dates — model self-maintains as source data grows
- Padding ensures time intelligence coverage beyond transaction boundaries
- If source data extends into 2019, DimDate automatically extends to Dec 31, 2020

**Storytelling Angle:**
*"A date table that hardcodes its own boundaries is a maintenance liability. Mine reads the data and sets its own range."*

---

## Limitation Log

| # | Limitation | Impact | Mitigation |
|---|---|---|---|
| 1 | No on-hand inventory snapshots in source data | True inventory turnover cannot be calculated | Demand velocity used as proxy — documented in Executive Summary |
| 2 | No COGS field — only profit and revenue | Cannot calculate margin-based ABC weighting | Revenue-based ABC used — industry standard approach |
| 3 | Late Delivery Risk was ML target variable | Source flag not operationally derived | Replaced with transparent Lead Time Variance derivation |
| 4 | Product Description empty across all products | Cannot build product detail page | Column excluded — noted in column definitions |
| 5 | Single flat file source — no supplier dimension | Cannot segment lead time by supplier | Segmented by Shipping Mode as proxy |

---

*Document Version: 1.0 — Phase 1 ETL Complete*
*Next Update: Phase 2 — Model Layer (Relationships + DAX)*
