# ABC Product Segmentation & Inventory Management
### Power BI Portfolio Project

---

## Project Overview

This project builds an enterprise-grade ABC Product Segmentation and Inventory Management System in Power BI using the DataCo Smart Supply Chain dataset. It converts operational inventory management instincts — developed through years of warehouse and DC management — into a repeatable, data-driven analytical framework.

---

## Background

ABC inventory classification is one of the most fundamental tools in supply chain operations. In practice it's often done manually, inconsistently, and stored entirely in someone's head. This project systematizes that process — connecting product revenue segmentation directly to cycle counting programs, PAR levels, safety stock calculations, and replenishment triggers.

---

## What This System Does

- Classifies every SKU into A, B, or C tiers based on revenue contribution using the Pareto principle
- Generates PAR levels and replenishment thresholds with automated reorder flags
- Calculates safety stock using demand volatility and supplier lead time variance
- Produces a cycle counting program by ABC tier
- Surfaces stockout risk before it becomes a stockout
- Provides an executive summary with key findings and actionable recommendations

---

## Dataset

**Source:** DataCo Smart Supply Chain for Big Data Analysis
**URL:** https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis
**Size:** 180,519 rows, 53 columns
**Note:** Raw CSV not included in this repository. Download from Kaggle and update the MasterSet source path in Power Query before refreshing.

---

## Data Model Architecture

Star schema with one fact table and three dimension tables:

```
DimDate (2,191 rows)
    │
    ├── [Order Date] ──────────────────────────────────┐
                                                        │
DimCustomer ──── [Customer Id] ──── FactOrders ──── [Product Card Id] ──── DimProduct
                                    (180,519 rows)
```

### Query Architecture

```
CSV Source
    └── MasterSet (load disabled — staging only)
            ├── DimProduct (reference query — 118 unique products)
            ├── DimCustomer (reference query — unique customers)
            ├── FactOrders (reference query — all transactions)
            └── DimDate (generated — reads min/max from MasterSet)
```

---

## Report Pages

| Page | Title | Description |
|---|---|---|
| 1 | ABC Classification Dashboard | Product revenue contribution, Pareto curve, segment breakdown, cycle count program |
| 2 | Operational Implications | Reorder frequency, PAR levels, safety stock, lead time variance, stockout risk, carrying cost |
| 3 | Executive Summary | 3 findings, 2 recommendations, 1 KPI |

---

## Key Architectural Decisions

See `DECISION_LOG.md` for full documentation. Summary:

1. **Dynamic DimDate** — self-maintaining date range derived from source data with 1-year padding
2. **MasterSet staging pattern** — single source load, all queries reference one table
3. **Dynamic Late Delivery Risk** — replaced pre-labeled ML target variable with transparent Lead Time Variance derivation
4. **Dual boolean columns** — integer (0/1) for DAX arithmetic, logical (True/False) for report consumers
5. **Enterprise column ordering** — PK → FKs → Date Keys → Dimensions → Measures → Derived

---

## Project Phases

| Phase | Status | Commit |
|---|---|---|
| Phase 1 — ETL Layer | ✅ Complete | `Initial data model — ETL layer complete` |
| Phase 2 — Model Layer | 🔄 In Progress | Relationships + DimDate marking |
| Phase 3 — DAX Layer | ⬜ Pending | All measures + calculated columns |
| Phase 4 — Page 1 Visuals | ⬜ Pending | ABC Classification Dashboard |
| Phase 5 — Page 2 Visuals | ⬜ Pending | Operational Implications |
| Phase 6 — Page 3 Visuals | ⬜ Pending | Executive Summary |
| Phase 7 — v1.0 Complete | ⬜ Pending | Final polished version |

---

## Documentation

| File | Description |
|---|---|
| `README.md` | Project overview and architecture |
| `DECISION_LOG.md` | Architectural and analytical decisions with reasoning |
| `COLUMN_DEFINITIONS.md` | Complete data dictionary — source to model column mapping |

---

## Tools & Technologies

- **Power BI Desktop** — report development
- **Power Query (M)** — ETL and data transformation
- **DAX** — measures and calculated columns
- **Star Schema** — Kimball dimensional modeling

---

## Author

**Howard Chung**
Operations Manager | Analytics and Business Operations Transition
Long Island City, NY

LinkedIn: [linkedin.com/in/howardchungnyc](https://www.linkedin.com/in/howardchungnyc/)
