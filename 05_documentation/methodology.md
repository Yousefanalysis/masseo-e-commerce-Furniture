# Methodology — 4D Cycle Application

**Project:** Masseo E-commerce Furniture Sales Analysis
**Cycle:** Discover → Design → Develop → Deliver

---

## 1. Discover

**Objective:** Understand the raw data before touching it. Identify grain, quality issues, and the questions the business actually needs answered.

**Activities performed:**
- Profiled `Fact_Sales.xlsx` for null patterns, duplicate `RowID` values, and status-value distribution.
- Confirmed fact grain: one row per `RowID × ProductID` line. Multi-product orders split into multiple fact rows; `DISTINCTCOUNT(RowID)` collapses them back to order-level for count KPIs.
- Verified four discrete `Status` values drive every downstream filter: `Delivered`, `Return`, `Cancelled request`, `Delivery failed`. No additional cleansing required.
- Identified three date columns on every fact row (`OrderDate`, `DeliveryDate`, `ActualDeliveryDate`). Recognised that analysing revenue on each axis separately was required to answer schedule-adherence questions — this became the role-playing date dimension decision in Design.
- Spotted that `sellingPrice` occasionally diverges from the expected price list. Retained the `IncorrectSellingPrice` audit column; excluded it from all KPI calculations.
- Identified the four core business questions the dashboard must answer:
  1. Where is revenue leaking between order placement and delivery?
  2. Which channels and products concentrate the most value?
  3. Are we delivering on the dates we promised?
  4. Are customers returning, and at what frequency?

---

## 2. Design

**Objective:** Define the data model, KPI list, and report structure before writing a single DAX expression.

**Data model decisions:**
- Adopted Kimball-style star schema: one fact (`Fact_Sales`) + four conformed dimensions (`Dim_Date`, `Dim_Product`, `Dim_Channel`, `Dim_Customer_Type`).
- Natural keys preserved for `ProductID`, `ChannelID`, `CustomerTypeID` — already stable upstream string codes; no surrogate layer needed.
- `Dim_Date` designed as a **role-playing dimension**: one physical table, three relationships to the fact. `OrderDate` active; `DeliveryDate` and `ActualDeliveryDate` inactive, activated via `USERELATIONSHIP` at measure level.
- Calendar extended to 2025-05-01 — beyond the last `OrderDate` (2024-12-31) — to accommodate promised and actual delivery dates landing after year-end.
- Seven headless measures-host tables scoped by functional group: `Measured` · `Orders analysis` · `Delivery performance` · `Assumptions - Dependencies` · `Test` · `MoM %` · `Dynamic Title`.

**KPI list scoped against the four business questions:**

| Question | KPIs scoped |
|---|---|
| Revenue leakage | Net Sales · Leakage Amount · Leakage Rate % · Sales amount by status |
| Channel/product concentration | Net Sales by Channel · Net Sales by Product Type · Best Product · AOV |
| Schedule adherence | Lead Time (Days) · On-Time Delivery Rate · Net Sales Adherence % (Delivery vs Actual) |
| Customer retention | Active Customers · Avg Days Between Orders · Frequnecy Segment · Distinct Customers |

---

## 3. Develop

**Objective:** Build the Power Query transformations and DAX measure library in a controlled, layered sequence.

**Power Query layer:**
- Saturday-start calendar generated with `DayOfWeek_SatStart` (1 = Saturday, 7 = Friday) and `Start of Week (Sat)` anchors — reflects Egyptian retail-week convention.
- `Month Start`, `Month Sort` (YYYYMM integer), `Quarter Year Sort` (YYYYQ integer) added as sort-key columns so month and quarter axes sort chronologically rather than alphabetically.
- `Dim_Date` marked as Date Table on the `Date` column.

**DAX measure sequence:**
1. **Base aggregations** — `Sales amount`, `Sales amount (Delivered)`, `Sales amount (Return)`, `Sales amount (Cancelled)`, `Sales amount (Failed)`, `Quantity (Delivered)`, `Quantity (Return)`
2. **Revenue integrity** — `Net Sales`, `Leakage Amount`, `Leakage Rate %`
3. **Order counts** — `1.Orders`, `2.Orders (Delivered)`, `4.Orders (Cancelled)`, `6.Orders (Delivery failed)`, `9.Orders (Return)`, `1.Orders (All)`, `7.Orders (Failure)`, `8.Failure Rate %`
4. **Delivery performance** — `Lead Time (Days)`, `On-Time Deliveries`, `On-Time Delivery Rate`, `Late Deliveries`, `Late Delivery Rate`
5. **Customer retention** — `Active Customers`, `Distinct Customers`, `Avg Days Between Orders`, `Frequnecy Segment`, `AOV (Delivered invoices)`
6. **Role-playing date analytics** — `Net Sales (Delivery Date)`, `Net Sales (actual Delivery Date)`, `Net Sales Adherence % (Delivery vs Actual)`, `Net Sales Adherence Status`
7. **Presentation layer** — 47 MoM % triplets (value · label · colour) covering 15 KPIs; 9 Dynamic Title measures; `Leakage Rate Color`

**Quality checks applied:**
- All ratio measures use `DIVIDE(…, …, BLANK())` — no division by zero.
- All MoM measures guard against blank prior periods with `IF(_PrevSales > 0, …, BLANK())`.
- `1.Orders (All)` is an explicit sum of four status counts — not `1.Orders` — so new source statuses surface as discrepancies rather than being silently absorbed.

---

## 4. Deliver

**Objective:** Build the report surfaces and ensure every KPI card is self-explaining.

**Report pages:** Menu · Operations Performance Pulse · Sales Report · Insight & Recommended · Inventory Analysis

**Design decisions:**
- Warm gold-and-cream theme with glass-card styling — consistent with the Masseo brand palette visible in the source materials.
- Unified conditional colour palette applied via DAX measures, not per-visual conditional formatting rules: `#16A34A` green · `#DC2626` red · `#F59E0B` amber · `#6B7280` grey.
- Every KPI card carries its MoM label+colour measures inline — the stakeholder reads direction and delta without opening a second visual.
- `Count Active Filters` and `Count Active Filters Tooltip` badge surfaces the active slicer set at a glance, replacing the need for the user to check each slicer manually.
- Dynamic title measures (`Dynamic Title`, `Dynamic Title KPIS`, `Title (Visible Period)`) adapt the report header to whatever date period is currently selected.
- Arabic channel names, product names, and governorate labels are preserved in the data model — no Arabic text is hard-coded in visuals, so filter context always renders in the correct script.
