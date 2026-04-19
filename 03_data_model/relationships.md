# Data Model — Relationships

**Model Type:** Star Schema (Kimball) with role-playing date dimension
**Compatibility Level:** 1600
**Total Tables:** 12 (1 fact · 4 dimensions · 7 measures-host tables)

---

## 1. Schema Overview

![ERD](erd_diagram.png)

The model centres on a single fact table — `Fact_Sales` — surrounded by four conformed dimensions. `Dim_Date` is a **role-playing dimension** — the fact carries three date columns (Order, promised Delivery, Actual Delivery) and only the Order relationship is active; the other two are activated on-demand via `USERELATIONSHIP` inside the `Test` measure group. All business logic (118 DAX measures) is centralised across seven headless measures-host tables.

| Table | Type | Rows | Key | Purpose |
|---|---|---|---|---|
| `Fact_Sales` | Fact | 112 | Composite (RowID + line) | Order-line grain |
| `Dim_Date` | Dimension | 479 | `Date` | Calendar 2024-01-09 → 2025-05-01 (marked Date Table) |
| `Dim_Product` | Dimension | 70 | `ProductID` | Product master with type/colour/size |
| `Dim_Channel` | Dimension | 8 | `ChannelID` | Sales-channel master |
| `Dim_Customer_Type` | Dimension | 2 | `CustomerID` | Customer segment (Individual / Company) |
| `Measured` | Measures host | — | — | 21 core aggregate measures |
| `Delivery performance` | Measures host | — | — | 6 delivery-KPI measures |
| `Assumptions - Dependencies` | Measures host | — | — | 15 segmentation measures |
| `Dynamic Title` | Measures host | — | — | 9 report-header measures |
| `Test` | Measures host | — | — | 7 role-playing-date measures |
| `Orders analysis` | Measures host | — | — | 13 order-count and rate measures |
| `MoM %` | Measures host | — | — | 47 month-over-month presentation measures |

---

## 2. Active Relationships (4)

| From (Many) | To (One) | Cardinality | Cross-Filter | Active |
|---|---|---|---|---|
| `Fact_Sales[ChannelID]` | `Dim_Channel[ChannelID]` | Many-to-One | Single → | ✅ |
| `Fact_Sales[ProductID]` | `Dim_Product[ProductID]` | Many-to-One | Single → | ✅ |
| `Fact_Sales[CustomerTypeID]` | `Dim_Customer_Type[CustomerID]` | Many-to-One | Single → | ✅ |
| `Fact_Sales[OrderDate]` | `Dim_Date[Date]` | Many-to-One | Single → | ✅ |

All relationships use natural keys stored as strings. Filter flow is single-direction, single-active, dimensions → fact. No bi-directional filtering.

---

## 3. Inactive Relationships (2 — role-playing on Dim_Date)

| From (Many) | To (One) | Cardinality | Active | Activated by |
|---|---|---|---|---|
| `Fact_Sales[DeliveryDate]` | `Dim_Date[Date]` | Many-to-One | ❌ | `USERELATIONSHIP(...)` inside `Net Sales (Delivery Date)` |
| `Fact_Sales[ActualDeliveryDate]` | `Dim_Date[Date]` | Many-to-One | ❌ | `USERELATIONSHIP(...)` inside `Net Sales (actual Delivery Date)` |

This pattern enables **three temporal views of the same revenue** (order-based · promise-based · actual-delivery-based) without duplicating `Dim_Date` into three separate role tables. The comparison of the promise vs the actual drives the Schedule Adherence KPI in the `Test` measure group.

---

## 4. Grain & Referential Integrity

**Fact grain:** One row per `RowID × ProductID` line — an order may split into multiple fact rows if it contains multiple products. `1.Orders = DISTINCTCOUNT(Fact_Sales[RowID])` collapses those line rows into a single order count.

**Checks:**
- Every `ChannelID` resolves to `Dim_Channel` (no orphan channels)
- Every `ProductID` resolves to `Dim_Product` (no orphan SKUs)
- Every `CustomerTypeID` resolves to `Dim_Customer_Type`
- `Dim_Date` is marked as the Date Table (enables time intelligence across all DAX)

---

## 5. Design Decisions

**Why seven measures-host tables instead of one.** The 118 measures are split by functional group so each pane in the Fields list surfaces a coherent set. Revenue basics live in `Measured`, delivery KPIs in `Delivery performance`, segmentation in `Assumptions - Dependencies`, role-playing-date analytics in `Test`, MoM presentation in `MoM %`, order counts in `Orders analysis`, and report chrome in `Dynamic Title`.

**Why natural keys instead of surrogate keys.** The source `ProductID` and `ChannelID` are already stable string codes assigned upstream. Adding a surrogate layer would duplicate the column without compression benefit. `Dim_Customer_Type` has only 2 rows so the key choice is irrelevant at scale.

**Why role-playing dates instead of three date dimensions.** Three physical copies of `Dim_Date` would fragment slicers, bloat the model, and create ambiguity in any time-intelligence measure that could be joined to multiple role tables. `USERELATIONSHIP` keeps the slicer surface clean — one `Dim_Date`, three activation points — and makes the author's intent explicit at the measure level.

**Why the calendar extends past the fact range.** `Dim_Date` runs to 2025-05-01 while the latest `OrderDate` in the fact is within 2024-12-31. The forward extension accommodates promised/actual delivery dates that land after the order date — those dates are used by the inactive relationships and must exist in the calendar for `USERELATIONSHIP` patterns to resolve.

---

## 6. Time Intelligence Readiness

- `Dim_Date` marked as Date Table on the `Date` column
- Integer sort keys exist for every descriptive column: `Month Number`, `Month Sort`, `Quarter Num`, `Quarter Year Sort`, `DayOfWeek_SatStart`
- Sort-by relationships configured: `Month` sorts by `Month Number` · `Month Short` by `Month Sort` · `Day Name` / `Day Short` by `DayOfWeek_SatStart` · `Quarter Year` by `Quarter Year Sort`
- `Year Hierarchy`: Year → Quarter → Month Short → Day Short

---

## 7. Saturday-Start Week Convention

The Egyptian retail week starts on Saturday. Two supporting columns implement this:

- `DayOfWeek_SatStart` — 1 = Saturday, 7 = Friday
- `Start of Week (Sat)` — anchors each date to its Saturday-week for weekly aggregations
