# Column Definitions
## ABC Product Segmentation & Inventory Management
### Power BI Project - Phase 1: ETL Data Preparation Layer

---

> This document explains where each field in the Power BI report comes from, what it was renamed to, what new fields were created, what was removed, and why certain data formats were chosen. It is the main reference guide for understanding the project data.

---

## Source Dataset

**Dataset:** DataCo Smart Supply Chain for Big Data Analysis<br>
**Source:** Kaggle - https://www.kaggle.com/datasets/shashwatwork/dataco-smart-supply-chain-for-big-data-analysis<br>
**Format:** CSV - 53 columns, 180,519 rows<br>
**Original Language:** Spanish (translated to English - some column names are translation artifacts)<br>
**Staging Query:** MasterSet - load disabled, serves as single source of truth

---

## MasterSet - Staging Query

| # | Source Column | Data Type | Notes |
|---|---|---|---|
| 1 | Type | Text | Payment type - DEBIT, TRANSFER, CASH, PAYMENT |
| 2 | Days for shipping (real) | Whole Number | Actual shipping days |
| 3 | Days for shipment (scheduled) | Whole Number | Planned shipping days |
| 4 | Benefit per order | Decimal Number | Translation artifact - "beneficio" = gross profit |
| 5 | Sales per customer | Decimal Number | Total customer sales value |
| 6 | Delivery Status | Text | Categorical - Advance shipping, Late delivery, On time, Canceled |
| 7 | Late_delivery_risk | Whole Number | ML target variable - replaced in FactOrders |
| 8 | Category Id | Whole Number | Product category foreign key |
| 9 | Category Name | Text | Product category label |
| 10 | Customer City | Text | Customer city |
| 11 | Customer Country | Text | Customer country |
| 12 | Customer Email | Text | PII - masked as XXXXXXXX by dataset creators |
| 13 | Customer Fname | Text | Customer first name |
| 14 | Customer Id | Whole Number | Customer primary key |
| 15 | Customer Lname | Text | Customer last name |
| 16 | Customer Password | Text | PII - masked as XXXXXXXX by dataset creators |
| 17 | Customer Segment | Text | Consumer, Corporate, Home Office |
| 18 | Customer State | Text | Customer state |
| 19 | Customer Street | Text | Customer street address |
| 20 | Customer Zipcode | Whole Number | Customer zip code |
| 21 | Department Id | Whole Number | Department foreign key |
| 22 | Department Name | Text | Department label |
| 23 | Latitude | Decimal Number | Store location latitude |
| 24 | Longitude | Decimal Number | Store location longitude |
| 25 | Market | Text | Africa, Europe, LATAM, Pacific Asia, USCA |
| 26 | Order City | Text | Destination city |
| 27 | Order Country | Text | Destination country |
| 28 | Order Customer Id | Whole Number | Duplicate of Customer Id |
| 29 | order date (DateOrders) | DateTime | Order placement date and time |
| 30 | Order Id | Whole Number | Order primary key |
| 31 | Order Item Cardprod Id | Whole Number | RFID product scan code |
| 32 | Order Item Discount | Decimal Number | Discount amount per order item |
| 33 | Order Item Discount Rate | Decimal Number | Discount rate - ratio 0-1 |
| 34 | Order Item Id | Whole Number | Order line item key |
| 35 | Order Item Product Price | Decimal Number | Unit price before discount |
| 36 | Order Item Profit Ratio | Decimal Number | Profit ratio - ratio 0-1 |
| 37 | Order Item Quantity | Whole Number | Units ordered |
| 38 | Sales | Decimal Number | Revenue value |
| 39 | Order Item Total | Decimal Number | Total order line value |
| 40 | Order Profit Per Order | Decimal Number | Net profit per order |
| 41 | Order Region | Text | Geographic region |
| 42 | Order State | Text | Destination state |
| 43 | Order Status | Text | COMPLETE, PENDING, CLOSED, CANCELED, etc. |
| 44 | Order Zipcode | Whole Number | Destination zip code |
| 45 | Product Card Id | Whole Number | Product primary key |
| 46 | Product Category Id | Whole Number | Product category foreign key |
| 47 | Product Description | Text | Empty across all records - excluded from model |
| 48 | Product Image | Text | Product image URL |
| 49 | Product Name | Text | Product display name |
| 50 | Product Price | Decimal Number | Product list price |
| 51 | Product Status | Whole Number | 0 = Available, 1 = Not Available |
| 52 | shipping date (DateOrders) | DateTime | Shipment date and time |
| 53 | Shipping Mode | Text | Standard Class, First Class, Second Class, Same Day |

---

## DimDate - 22 Columns, 2,191 Rows

**Source:** Generated from MasterSet-derived min/max boundaries (not a reference query)<br>
**Date Range:** Jan 1, 2014 → Dec 31, 2019<br>
**Padding:** 1 year before min transaction date, 1 year after max transaction date<br>
**Grain:** One row per calendar day<br>
**Primary Key (PK):** Date<br>
**Foreign Key (FK):** None (dimension table)<br>
**Marked as Date Table:** Yes - Date column validated successfully

| Column | Data Type | Source | Notes |
|---|---|---|---|
| Date | Date | Generated | PK - unique contiguous daily date |
| Date Key | Whole Number | Derived | YYYYMMDD integer surrogate key |
| Year | Whole Number | Derived | Calendar year |
| Quarter Number | Whole Number | Derived | 1-4 |
| Quarter Label | Text | Derived | "Q1" through "Q4" |
| Month Number | Whole Number | Derived | 1-12 - sort key for Month Name |
| Month Name | Text | Derived | Full month name |
| Week Number | Whole Number | Derived | ISO week of year |
| Day of Month | Whole Number | Derived | 1-31 |
| Day Name | Text | Derived | Full day name |
| Day Number | Whole Number | Derived | 0=Sunday through 6=Saturday - sort key for Day Name |
| Year-Quarter Label | Text | Derived | "2015 Q1" - axis label for quarterly trends |
| Year-Month Key | Whole Number | Derived | YYYYMM integer - chronological sort for monthly trends |
| Year-Month Label | Text | Derived | "Jan 2015" - axis label for monthly trends |
| Is Weekday | Logical | Derived | True if Mon-Fri - filters supplier operating days |
| Is Weekend | Logical | Derived | True if Sat-Sun |
| Is Month Start | Logical | Derived | True if Day of Month = 1 |
| Is Month End | Logical | Derived | True if last day of month |
| Is Quarter Start | Logical | Derived | True if first day of quarter |
| Is Quarter End | Logical | Derived | True if last day of quarter |
| Relative Day Index | Whole Number | Derived | 0 = last date in range, negative = prior days |
| Relative Month Index | Whole Number | Derived | 0 = last month in range, negative = prior months |
| Relative Quarter Index | Whole Number | Derived | 0 = last quarter in range, negative = prior quarters |

---

## DimProduct - 11 Columns, 118 Rows

**Source:** Reference query from MasterSet<br>
**Grain:** One row per unique product<br>
**Primary Key (PK):** Product Card Id<br>
**Foreign Key (FK):** None (dimension table)<br>
**Duplicate Check Field:** Product Card Id (removed duplicate rows, kept one row per Product Card Id)

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Product Card Id | Whole Number | Product Card Id | No | PK |
| Product Name | Text | Product Name | No | |
| Product Category Id | Whole Number | Product Category Id | No | FK to Category |
| Category Id | Whole Number | Category Id | No | FK to Category |
| Category Name | Text | Category Name | No | |
| Department Id | Whole Number | Department Id | No | FK to Department |
| Department Name | Text | Department Name | No | |
| Product Price | Decimal Number | Product Price | No | List price |
| Product Status | Whole Number | Product Status | No | 0=Available, 1=Not Available. All 118 products = 0 |
| Product Image | Text | Product Image | No | URL string - acmesports.sports domain |
| Product Status Label | Text | Derived | N/A | "Available" or "Not Available" - readable label for report consumers |

**Excluded Columns:**
| Source Column | Reason |
|---|---|
| Product Description | Empty across all 118 products - zero analytical value |

---

## DimCustomer - 9 Columns

**Source:** Reference query from MasterSet<br>
**Grain:** One row per unique customer<br>
**Primary Key (PK):** Customer Id<br>
**Foreign Key (FK):** None (dimension table)<br>
**Duplicate Check Field:** Customer Id (removed duplicate rows, kept one row per Customer Id)

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Customer Id | Whole Number | Customer Id | No | PK |
| Customer Fname | Text | Customer Fname | No | |
| Customer Lname | Text | Customer Lname | No | |
| Customer Segment | Text | Customer Segment | No | Consumer, Corporate, Home Office |
| Customer City | Text | Customer City | No | |
| Customer State | Text | Customer State | No | |
| Customer Country | Text | Customer Country | No | |
| Customer Street | Text | Customer Street | No | |
| Customer Zipcode | Whole Number | Customer Zipcode | No | |

**Excluded Columns:**
| Source Column | Reason |
|---|---|
| Customer Email | PII - masked as XXXXXXXX. Zero analytical value. Data governance decision. |
| Customer Password | PII - masked as XXXXXXXX. Zero analytical value. Data governance decision. |

---

## FactOrders - 35 Columns

**Source:** Reference query from MasterSet<br>
**Grain:** One row per order line item<br>
**Primary Key (PK):** Order Id<br>
**Foreign Key (FK):** `Order Date` → `DimDate`, `Product Card Id` → `DimProduct`, `Customer Id` → `DimCustomer`

### Keys

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Order Id | Whole Number | Order Id | No | PK |
| Customer Id | Whole Number | Customer Id | No | FK → DimCustomer |
| Product Card Id | Whole Number | Product Card Id | No | FK → DimProduct |
| Product Card Id (RFID) | Whole Number | Order Item Cardprod Id | YES | RFID reader scan code - secondary product reference |
| Order Customer Id | Whole Number | Order Customer Id | No | Duplicate of Customer Id - retained from source |
| Order Date | Date | order date (DateOrders) | YES | FK → DimDate. Time stripped from DateTime. |
| Shipping Date | Date | shipping date (DateOrders) | YES | Shipment date. Time stripped from DateTime. |

### Order and status details

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Order Status | Text | Order Status | No | COMPLETE, PENDING, CLOSED, CANCELED, PROCESSING, etc. |
| Delivery Status | Text | Delivery Status | No | Advance shipping, Late delivery, On time, Canceled |
| Late Delivery Risk | Whole Number | Late_delivery_risk | YES | **Derived** - replaced ML target variable. 1 if Lead Time Variance > 0 |
| Is Late Delivery Risk | Logical | Late_delivery_risk | YES | **Derived** - True/False companion for report visuals |
| Payment Type | Text | Type | YES | DEBIT, TRANSFER, CASH, PAYMENT |
| Market | Text | Market | No | Africa, Europe, LATAM, Pacific Asia, USCA |
| Order Region | Text | Order Region | No | Geographic sub-region |
| Order State | Text | Order State | No | Destination state |
| Order City | Text | Order City | No | Destination city |
| Order Country | Text | Order Country | No | Destination country |
| Latitude | Decimal Number | Latitude | No | Store location coordinate |
| Longitude | Decimal Number | Longitude | No | Store location coordinate |

### Item-level order details

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Order Line Id | Whole Number | Order Item Id | YES | Order line item identifier |
| Discount Rate | Decimal Number | Order Item Discount Rate | YES | Ratio 0-1 - kept as Decimal not Currency |
| Shipping Mode | Text | Shipping Mode | No | Standard Class, First Class, Second Class, Same Day |

### Financial Measures

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Sales | Currency | Sales | No | Revenue value |
| Order Total | Currency | Order Item Total | YES | Total order line value |
| Order Discount | Currency | Order Item Discount | YES | Discount amount |
| Unit Price | Currency | Order Item Product Price | YES | Price before discount |
| Profit Ratio | Decimal Number | Order Item Profit Ratio | YES | Ratio 0-1 - kept as Decimal not Currency |
| Order Quantity | Whole Number | Order Item Quantity | YES | Units ordered |
| Order Profit Per Order | Currency | Order Profit Per Order | No | Net profit per order |
| Order Gross Profit | Currency | Benefit per order | YES | Translation artifact - "beneficio" = gross profit |
| Customer Sales Total | Currency | Sales per customer | YES | Total customer sales value |

### Shipping Facts

| Column | Data Type | Source Column | Renamed? | Notes |
|---|---|---|---|---|
| Shipping Days (Actual) | Whole Number | Days for shipping (real) | YES | Actual days from order to delivery |
| Shipping Days (Scheduled) | Whole Number | Days for shipment (scheduled) | YES | Planned days from order to delivery |

### Calculated fields

| Column | Data Type | Formula | Notes |
|---|---|---|---|
| Lead Time Variance | Whole Number | `Shipping Days (Actual) - Shipping Days (Scheduled)` | Positive = late, Negative = early, Zero = on time. Core safety stock input. |
| Late Delivery Risk | Whole Number | `if Lead Time Variance > 0 then 1 else 0` | Replaces source ML target variable. Integer for DAX arithmetic. |
| Is Late Delivery Risk | Logical | `Lead Time Variance > 0` | Logical companion. True/False for report visuals and slicers. |

---

## Data Type Reference

| Power BI Type | M Syntax | Used For |
|---|---|---|
| Whole Number | Int64.Type | IDs, counts, integer flags, date components |
| Decimal Number | type number | Ratios, percentages, coordinates |
| Fixed Decimal (Currency) | Currency.Type | All financial values - prevents floating point rounding |
| Text | type text | Labels, categories, descriptions |
| Date | type date | Date keys - time stripped from DateTime source |
| DateTime | type datetime | Not used in model - stripped to Date in ETL |
| Logical | type logical | Boolean flags - True/False for report consumers |

---

## Naming Conventions

| Convention | Applied To | Example |
|---|---|---|
| Title Case with spaces | All column names | `Order Gross Profit` |
| `Is` prefix | Logical boolean columns | `Is Late Delivery Risk` |
| `(Actual)` / `(Scheduled)` parenthetical | Paired comparative columns | `Shipping Days (Actual)` |
| No abbreviations | All columns | `Order Quantity` not `Ord Qty` |
| No snake_case | All columns | `Late Delivery Risk` not `Late_delivery_risk` |

---

*Document Version: 1.0 - Phase 1 ETL Complete*<br>
*Next Update: Phase 2 - Model Layer (Relationships + DAX)*
