# Data Dictionary — Masseo E-commerce Furniture

**Scope:** All physical columns and DAX measures in the Power BI semantic model.
**Measures:** 118 total, organized across 7 measures-host tables.
**Tables:** 12 (1 fact · 4 dimensions · 7 measures-host tables).
**Source of truth:** Extracted verbatim from the live Power BI model via DMV (`$SYSTEM.MDSCHEMA_MEASURES`, `$SYSTEM.DBSCHEMA_COLUMNS`).

---

## 1. Column Dictionary

### 1.1 Fact_Sales (18 columns, 112 rows — order-line grain)

| Column | Data Type | Role | Description |
|---|---|---|---|
| `RowID` | String | Business key | Source order/line identifier — used by `DISTINCTCOUNT` for the `1.Orders` measure |
| `CustomerName` | String | Attribute | Customer display name — used for customer-level distinct counts and retention analysis |
| `CustomerTypeID` | String | FK | → `Dim_Customer_Type[CustomerID]` — segments فرد (Individual) vs شركة (Company) |
| `ChannelID` | String | FK | → `Dim_Channel[ChannelID]` — sales channel / marketplace |
| `ProductID` | String | FK | → `Dim_Product[ProductID]` — SKU identifier |
| `Quantity` | Int64 | Measure source | Units sold on the line |
| `sellingPrice` | Double | Measure source | Unit price actually charged (may differ from price list) |
| `paymentMethodType` | String | Attribute | Prepaid · check · Company account · unknown |
| `IncorrectSellingPrice` | Double | Audit | Flagged price mismatches for data-quality review |
| `OrderDate` | DateTime | Active date FK | → `Dim_Date[Date]` (active relationship) — drives all order-period analysis |
| `DeliveryDate` | DateTime | Inactive date FK | → `Dim_Date[Date]` (inactive, role-playing) — promised delivery date |
| `ActualDeliveryDate` | DateTime | Inactive date FK | → `Dim_Date[Date]` (inactive, role-playing) — realised delivery date |
| `DeliveryPerson` | String | Attribute | Assigned delivery agent |
| `Governorate` | String | Attribute | Egyptian governorate — used for geographic analysis |
| `Address` | String | Attribute | Free-text shipping address |
| `Status` | String | Driver | Delivered · Return · Cancelled request · Delivery failed — drives all status-split measures |
| `Count` | Int64 | Helper | Pre-materialized row count |
| `RepeatingRows` | Int64 | Helper | Deduplication helper |

### 1.2 Dim_Date (18 columns, 479 rows — 2024-01-09 → 2025-05-01)

| Column | Data Type | Sort By | Description |
|---|---|---|---|
| `Date` | DateTime | — | Calendar date (marked Date Table) |
| `Year` | Int64 | — | 4-digit year |
| `Month Number` | Int64 | — | 1–12 |
| `Month` | String | `Month Number` | January, February … |
| `Month Short` | String | `Month Sort` | Jan, Feb … |
| `Quarter` | String | `Quarter Num` | Q1–Q4 |
| `Quarter Num` | Int64 | — | 1–4 |
| `Month Start` | DateTime | — | First-of-month anchor (used by MoM engine) |
| `Month Year` | String | `Month Sort` | "Jan 2024", "Feb 2024" … |
| `Month Sort` | Int64 | — | YYYYMM integer for chronological sort |
| `Day` | Int64 | — | Day of month 1–31 |
| `Day Name` | String | `DayOfWeek_SatStart` | Saturday–Friday (EN) |
| `Day Short` | String | `DayOfWeek_SatStart` | Sat, Sun, Mon … |
| `DayOfWeek_SatStart` | Int64 | — | 1=Saturday … 7=Friday (Egyptian week convention) |
| `Start of Week (Sat)` | DateTime | — | Saturday-anchored week start |
| `Year Start` | DateTime | — | First-of-year anchor |
| `Quarter Year` | String | `Quarter Year Sort` | "Q1 2024" … |
| `Quarter Year Sort` | Int64 | — | YYYYQ integer for chronological sort |

**Hierarchy:** `Year Hierarchy` — Year → Quarter → Month Short → Day Short

### 1.3 Dim_Product (9 columns, 70 rows — SKU master)

| Column | Data Type | Description |
|---|---|---|
| `index` | Int64 | Original source index |
| `ProductID` | String | Primary key (→ `Fact_Sales[ProductID]`) |
| `ProductType` | String | Category: سرير · كنبه · طاولة · ركنة · كرسى · شيزلونج |
| `ProductName` | String | Display name (e.g., "سرير كينج", "كنبه سرير") |
| `ProductDescription` | String | Extended description |
| `Color/Metal` | String | Metal colour finish |
| `FabricColor` | String | Fabric colour |
| `Size` | String | Dimensions / size code |
| `TextproductID` | String | Display-friendly product ID for visuals |

### 1.4 Dim_Channel (2 columns, 8 rows — sales-channel master)

| Column | Data Type | Description |
|---|---|---|
| `ChannelID` | String | Primary key (→ `Fact_Sales[ChannelID]`) |
| `Channel` | String | Channel display name: هومزمارت · شيك هومز · تاكى · اوسكار ريتن · فيس بوك · منزلى · كيمت · لا يوجد |

### 1.5 Dim_Customer_Type (2 columns, 2 rows — customer-segment master)

| Column | Data Type | Description |
|---|---|---|
| `CustomerID` | String | Primary key (→ `Fact_Sales[CustomerTypeID]`) |
| `CustomerType` | String | فرد (Individual) · شركة (Company) |

---

## 2. Measure Catalog (118 measures — organized by host table)

Format string is shown where relevant. All 118 measures are extracted directly from the live model.

### 2.1 `Measured` (21 measures) — core aggregates

| Measure | Format | Description |
|---|---|---|
| `Sales amount` | `#,0` | `SUMX(Fact_Sales, sellingPrice × Quantity)` — gross revenue across all statuses |
| `Sales amount (Delivered)` | `#,0` | Gross revenue from Delivered orders only |
| `Sales amount (Return)` | `#,0` | Gross revenue from Return orders |
| `Sales amount (Cancelled)` | `#,0` | Gross revenue from Cancelled request orders |
| `Sales amount (Failed)` | `#,0` | Gross revenue from Delivery failed orders |
| `Net Sales` | `#,0` | `Sales amount (Delivered) − Sales amount (Return)` — the revenue KPI |
| `Leakage Amount` | `#,0` | `Sales amount − Net Sales` — revenue lost to non-delivered orders |
| `Leakage Rate %` | `0.00%` | Leakage Amount ÷ Sales amount. Bands: <5% green · 5–15% amber · >15% red |
| `Quantity (Delivered)` | `#,0` | Units in Delivered orders |
| `Quantity (Return)` | `#,0` | Units in Return orders |
| `Quantity (Net)` | `#,0` | Delivered − Return units |
| `Quantity (All)` | `#,0` | All units regardless of status |
| `Return Rate` | `0.00%` | `Sales amount (Return) ÷ Sales amount (Delivered)` — value-weighted return rate |
| `Return Rate QTY` | `0.00%` | Unit-weighted return rate |
| `sales amount upper` | `#,0` | Alias for waterfall visual headroom |
| `Net Sales lower` | `#,0` | Constant 0 for waterfall visual baseline |
| `net sales upper` | `#,0` | Alias for waterfall visual headroom |
| `Best Product (by Sales amount) - Name` | — | `TOPN(1,…)` over product SKU — returns winning product name |
| `Best Product (by Sales amount) - Value` | `#,0` | Amount delivered by the winning product |
| `Avg Days Between Orders (Global)` | `0.00` | Average days between any two consecutive orders, ignoring customer |
| `Distinct Customers` | `#,0` | `DISTINCTCOUNT(Fact_Sales[CustomerName])` |

### 2.2 `Delivery performance` (6 measures)

| Measure | Format | Description |
|---|---|---|
| `Lead Time (Days)` | `0.00` | `AVERAGEX(DATEDIFF(OrderDate, ActualDeliveryDate, DAY))` over rows with a non-blank actual delivery |
| `On-Time Deliveries` | `#,0` | Count of Delivered rows where `ActualDeliveryDate ≤ DeliveryDate` |
| `On-Time Delivery Rate` | `0.00%` | On-Time Deliveries ÷ 2.Orders (Delivered) |
| `Late Deliveries` | `#,0` | `MAX(Delivered − On-Time, 0)` — defensive floor |
| `Late Delivery Rate` | `0.00%` | Late Deliveries ÷ 2.Orders (Delivered) |
| `Avg Days Between Orders New` | `0.00` | Per-customer average gap between consecutive orders — drives Frequency Segment |

### 2.3 `Assumptions - Dependencies` (15 measures) — segmentation

| Measure | Format | Description |
|---|---|---|
| `Net Sales (by Customer Type)` | `#,0` | `ALLEXCEPT` pattern preserving only the `CustomerType` filter |
| `Quantity Net (by Customer Type)` | `#,0` | Same pattern, for units |
| `Net Sales (by Channel)` | `#,0` | `ALLEXCEPT` isolating `Channel` |
| `Quantity Net (by Channel)` | `#,0` | Same pattern, for units |
| `Net Sales (by Fabric Color)` | `#,0` | `ALLEXCEPT` isolating `FabricColor` |
| `Net Sales (by Metal Color)` | `#,0` | `ALLEXCEPT` isolating `Color/Metal` |
| `Quantity Net (by Fabric Color)` | `#,0` | Same pattern |
| `Quantity Net (by Metal Color)` | `#,0` | Same pattern |
| `AOV (Delivered invoices)` | `#,0.00` | `Net Sales ÷ 2.Orders (Delivered)` — average order value on delivered invoices |
| `Avg Net Sales per Month` | `#,0` | `AVERAGEX(DISTINCT(Month Start), [Net Sales])` |
| `Avg Quantity Net per Month` | `#,0` | Same pattern, for units |
| `Active Customers` | `#,0` | Distinct customers with at least one Delivered order |
| `Cancelled Customers` | `#,0` | Distinct customers with at least one Cancelled order |
| `Avg Days Between Orders` | `0.00` | Per-customer span ÷ (order count − 1) — smoother than Global version |
| `Frequnecy Segment` | — | SWITCH on Avg DBO New → Weekly / Monthly / Quarterly / Occasional / No-Repeat |

### 2.4 `Dynamic Title` (9 measures) — report chrome

| Measure | Format | Description |
|---|---|---|
| `Reporting Period Start` | Date | `MINX(ALLSELECTED(Dim_Date), Date)` |
| `Reporting Period End` | Date | `MAXX(ALLSELECTED(Dim_Date), Date)` |
| `Dynamic Title` | — | "For the period from : DD MMMM YYYY To DD MMMM YYYY" |
| `Dynamic Title KPIS` | — | " Sales Amount For the period from : …" — KPI page variant |
| `Title (Visible Period)` | — | "Sales vs Net - Leakage Amount MMM YYYY → MMM YYYY" |
| `Count Active Filters` | — | Sums 7 `ISFILTERED()` flags (month · channel · product type · governorate · status · customer name · product name) |
| `Count Active Filters Tooltip` | — | Concatenates active filter names into "Active Filters: A, B, C" |
| `count Active Filter Counter Color` | — | `#a7754770` when 0 filters · `#16a34a` otherwise (with alpha) |
| `count Active Filter Counter Color Really count` | — | Solid-colour variant for the counter badge |

### 2.5 `Test` (7 measures) — role-playing date analytics

| Measure | Format | Description |
|---|---|---|
| `Net Sales (Delivery Date)` | `#,0` | `CALCULATE([Net Sales], USERELATIONSHIP(DeliveryDate, Dim_Date[Date]))` — recognised on promised delivery date |
| `Net Sales (actual Delivery Date)` | `#,0` | Same pattern, using `ActualDeliveryDate` |
| `Net sales YTD (Delivery Date)` | `#,0` | `DATESYTD` over the delivery-date relationship |
| `Net Sales Adherence % (Delivery vs Actual)` | `0.0%` | Actual ÷ Planned (promised) |
| `Net Sales Timing Gap` | `#,0` | Actual − Planned (signed) |
| `Net Sales Adherence Status` | — | SWITCH: Ahead ≥105% · On track 95–105% · Slightly behind 80–95% · Significantly behind <80% |
| `Net Sales Adherence Status Color` | — | Hex palette for the status (green / red / pale red / deep red / grey) |

### 2.6 `Orders analysis` (13 measures) — count & rate KPIs

| Measure | Format | Description |
|---|---|---|
| `1.Orders` | `#,0` | `DISTINCTCOUNT(Fact_Sales[RowID])` — any status |
| `2.Orders (Delivered)` | `#,0` | `1.Orders` filtered to Delivered |
| `4.Orders (Cancelled)` | `#,0` | `1.Orders` filtered to Cancelled request |
| `6.Orders (Delivery failed)` | `#,0` | `1.Orders` filtered to Delivery failed |
| `9.Orders (Return)` | `#,0` | `1.Orders` filtered to Return |
| `1.Orders (All)` | `#,0` | Sum of Delivered + Cancelled + Failed + Return — intentional explicit sum |
| `7.Orders (Failure)` | `#,0` | Sum of Cancelled + Failed + Return (failure categories only) |
| `8.Failure Rate %` | `0.00%` | 7.Orders (Failure) ÷ 1.Orders (All) |
| `Return Rate %` | `0.00%` | 9.Orders (Return) ÷ 1.Orders (All) — order-count-weighted |
| `Cancel Rate %` | `0.00%` | 4.Orders (Cancelled) ÷ 1.Orders (All) |
| `Delivery Failed Rate %` | `0.00%` | 6.Orders (Delivery failed) ÷ 1.Orders (All) |
| `5.Orders (Cancel Rate)` | `0.00%` | 4.Orders (Cancelled) ÷ 1.Orders (pre-All total) — legacy ratio |
| `3.On-Time Delivery Rate (c)` | `0.00%` | Duplicate of Delivery-performance version for this folder |

### 2.7 `MoM %` (47 measures) — month-over-month presentation engine

Month-over-month output is structured as triplets of value / label / colour. For 15 KPIs there are 15 × 3 = 45 measures, plus 2 standalone members.

**Triplet pattern (identical structure for each KPI):**
1. `n-<KPI> MoM %` — the value. Pattern: capture latest visible `Month Start`, build current- and prior-month date windows via `EOMONTH` ± offsets, compute `(Cur − Prev) ÷ Prev`, return blank if prior is zero/blank.
2. `n-<KPI> MoM % Label` — the formatted string. SWITCH on sign: `↑ +x.x%` (green) · `↓ x.x%` (red) · `→ x.x%` (neutral).
3. `n-<KPI> MoM % Color` — the hex colour. Palette: `#16A34A` green · `#DC2626` red · `#6B7280` grey (blank or zero).

**Covered KPIs (15):**

| # | KPI Prefix | KPI Covered |
|---|---|---|
| 0 | `0-Net Sales` | Net Sales (primary, prefixed `0-` to anchor top of Fields pane) |
| 1 | `1-Sales amount` | Sales amount |
| 2 | `2-Quantity (Net)` | Quantity (Net) |
| 3 | `3-Orders` | Orders (`1.Orders`) |
| 4 | `4-Active Customers` | Active Customers |
| 5 | `5-Sales amount (Delivered)` | Sales amount (Delivered) |
| 6 | `6-Sales amount (Cancelled)` | Sales amount (Cancelled) |
| 7 | `7-Sales amount (Failed)` | Sales amount (Failed) |
| 8 | `8-Avg Net Sales per Month` | Avg Net Sales per Month |
| 9 | `9-Quantity (Net)` | Quantity (Net) — duplicate KPI covered twice |
| 910 | `910-Avg Quantity Net per Month` | Avg Quantity Net per Month |
| 911 | `911-Orders (Delivered)` | 2.Orders (Delivered) |
| 912 | `912-Orders (Cancelled)` | 4.Orders (Cancelled) |
| 913 | `913-Active Customers` | Active Customers — duplicate KPI covered twice |
| 914 | `914-Cancelled Customers` | Cancelled Customers — **inverted colour logic** (decrease in cancellations = green) |

**Standalone members (2):**

| Measure | Format | Description |
|---|---|---|
| `Net Sales Rolling 3M` | `#,0` | `CALCULATE([Net Sales], DATESINPERIOD(Dim_Date[Date], MAX(Date), -3, MONTH))` — trailing 3-month window smoother |
| `Leakage Rate Color` | — | Traffic-light palette for Leakage Rate %: <5% green · 5–15% amber · >15% red |

---

## 3. Documentation Coverage

- **Columns:** 49 physical columns across 5 data tables — all documented above
- **Measures:** 118 total — 30 core measures carry full KPI-Chain documentation in `dax_measures.md`; the remaining 88 (primarily MoM triplets and colour/status helpers) follow the patterns documented in §2.7 above
- **Calculated columns with DAX:** none (all `Dim_Date` columns are Power Query-generated)
- **Hierarchies:** 1 (`Dim_Date[Year Hierarchy]`)
- **Relationships:** 6 — 4 active, 2 inactive (role-playing on `Dim_Date`) — documented in `relationships.md`
