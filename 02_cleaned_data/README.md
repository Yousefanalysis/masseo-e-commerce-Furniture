# Cleaned Data — Transformation Log

**Folder:** `02_cleaned_data/`
**Purpose:** Documents all Power Query (M) transformations applied to the raw source files before loading into the semantic model.

---

## Transformation Summary

### Fact_Sales

| Step | Transformation | Reason |
|---|---|---|
| Promote headers | First row promoted to column headers | Source file uses row 1 as header |
| Type assignment | `OrderDate`, `DeliveryDate`, `ActualDeliveryDate` → DateTime; `Quantity` → Int64; `sellingPrice`, `IncorrectSellingPrice` → Double; all others → Text | Ensures correct data type compression in VertiPaq |
| Null handling | Rows with blank `RowID` removed | RowID is the business key; blank rows are load artifacts |
| Status normalization | Confirmed four discrete values (Delivered / Return / Cancelled request / Delivery failed) — no trimming or case correction required | Consistent source encoding |
| No surrogate keys added | `ChannelID`, `ProductID`, `CustomerTypeID` kept as natural string keys | Already stable upstream identifiers |

### Dim_Date (generated in Power Query)

| Step | Detail |
|---|---|
| Calendar generation | `List.Dates(#date(2024,1,9), 479, #duration(1,0,0,0))` — contiguous daily calendar |
| Range | 2024-01-09 → 2025-05-01 (extended past fact range to accommodate promised/actual delivery dates landing after December 2024) |
| `DayOfWeek_SatStart` | Saturday = 1, Friday = 7 — Egyptian retail-week convention |
| `Start of Week (Sat)` | `Date.StartOfWeek(Date, Day.Saturday)` |
| `Month Start` | `Date.StartOfMonth(Date)` — used by MoM engine as anchor column |
| `Month Sort` | `Date.Year * 100 + Date.Month` — integer for chronological axis sorting |
| `Quarter Num` | `Date.QuarterOfYear` |
| `Quarter Year Sort` | `Date.Year * 10 + Date.QuarterOfYear` — integer for sorting |
| Sort-by configuration | `Month` → `Month Number` · `Month Short` → `Month Sort` · `Day Name` → `DayOfWeek_SatStart` · `Quarter Year` → `Quarter Year Sort` |
| Marked as Date Table | On the `Date` column — enables time-intelligence functions in DAX |

### Dim_Product

| Step | Transformation |
|---|---|
| Type assignment | `ProductID` → Text; `index` → Int64; all descriptive columns → Text |
| No deduplication required | Source had 70 unique SKUs, confirmed |

### Dim_Channel / Dim_Customer_Type

| Step | Transformation |
|---|---|
| Load only | No transformation required — files already clean at 8 rows and 2 rows respectively |

---

## Output Tables Loaded into Model

| Table | Source | Rows after transform |
|---|---|---|
| `Fact_Sales` | `Fact_Sales.xlsx` | 112 |
| `Dim_Date` | Generated | 479 |
| `Dim_Product` | `Dim_Product.xlsx` | 70 |
| `Dim_Channel` | `Dim_Channel.xlsx` | 8 |
| `Dim_Customer_Type` | `Dim_Customer_Type.xlsx` | 2 |
