# DAX Measures — KPI Chain Documentation

**Format:** Each measure is documented in the **KPI Chain** pattern: **Plain → Logic → DAX**.
**Scope:** 30 core measures that drive the entire dashboard surface. The remaining 88 presentation-layer measures (MoM % triplets, colour/status helpers, dynamic titles) follow derivative patterns from these core measures and are catalogued in `data_dictionary.md`.
**Source of truth:** All DAX below is extracted verbatim from the live Power BI model via `$SYSTEM.MDSCHEMA_MEASURES`.

---

## Section 1 — Revenue Base Measures

### Sales amount
- **Plain:** Gross revenue across every order regardless of its final status — the top-of-funnel number before any leakage is removed.
- **Logic:** Iterate the fact table and sum the line-level price × quantity. `SUMX` is required here rather than `SUM` because price and quantity live on separate columns and the product must be computed per row.
- **DAX:**
```DAX
Sales amount = 
SUMX ( Fact_Sales, Fact_Sales[sellingPrice] * Fact_Sales[Quantity] )
```

### Sales amount (Delivered)
- **Plain:** Gross revenue from orders that successfully reached the customer.
- **Logic:** Filter the base `Sales amount` to rows where Status = Delivered.
- **DAX:**
```DAX
Sales amount (Delivered) = 
CALCULATE ( [Sales amount], Fact_Sales[Status] = "Delivered" )
```

### Sales amount (Return)
- **Plain:** Gross revenue tied to returned orders — these are subtracted from Delivered to get Net Sales.
- **Logic:** Same pattern, filtered to Status = Return.
- **DAX:**
```DAX
Sales amount (Return) = 
CALCULATE ( [Sales amount], Fact_Sales[Status] = "Return" )
```

### Sales amount (Cancelled)
- **Plain:** Gross revenue that was written on cancelled orders — counted as leakage.
- **Logic:** Filtered to Status = Cancelled request.
- **DAX:**
```DAX
Sales amount (Cancelled) = 
CALCULATE ( [Sales amount], Fact_Sales[Status] = "Cancelled request" )
```

### Sales amount (Failed)
- **Plain:** Gross revenue tied to orders that the carrier failed to deliver — also leakage.
- **Logic:** Filtered to Status = Delivery failed.
- **DAX:**
```DAX
Sales amount (Failed) = 
CALCULATE ( [Sales amount], Fact_Sales[Status] = "Delivery failed" )
```

### Net Sales
- **Plain:** The headline revenue KPI — money that was actually delivered and kept.
- **Logic:** Delivered revenue minus returned revenue. Cancelled and Failed orders are already excluded because they are not in the Delivered pool.
- **DAX:**
```DAX
Net Sales = 
[Sales amount (Delivered)] - [Sales amount (Return)]
```

### Leakage Amount
- **Plain:** The revenue gap between what was ordered and what the company actually kept — the sum of cancelled, failed, and returned money.
- **Logic:** Gross Sales minus Net Sales. The arithmetic guarantees this equals `Sales amount (Cancelled) + Sales amount (Failed) + Sales amount (Return)`.
- **DAX:**
```DAX
Leakage Amount = 
[Sales amount] - [Net Sales]
```

### Leakage Rate %
- **Plain:** Leakage as a share of gross revenue. The traffic-light target bands are <5% (green), 5–15% (amber), >15% (red).
- **Logic:** Guard the divisor with `IF` to avoid division by blank/zero, then divide Leakage by Gross.
- **DAX:**
```DAX
Leakage Rate % = 
VAR g = [Sales amount]
VAR n = [Net Sales]
RETURN
    IF ( NOT ISBLANK ( g ) && g <> 0, DIVIDE ( g - n, g ), BLANK () )
```

---

## Section 2 — Unit Base Measures

### Quantity (Delivered)
- **Plain:** Units actually delivered to customers.
- **Logic:** Sum the Quantity column filtered to Delivered status.
- **DAX:**
```DAX
Quantity (Delivered) = 
CALCULATE ( SUM ( Fact_Sales[Quantity] ), Fact_Sales[Status] = "Delivered" )
```

### Quantity (Return)
- **Plain:** Units that were returned.
- **Logic:** Sum Quantity filtered to Return status.
- **DAX:**
```DAX
Quantity (Return) = 
CALCULATE ( SUM ( Fact_Sales[Quantity] ), Fact_Sales[Status] = "Return" )
```

### Quantity (Net)
- **Plain:** Net units sold after returns — the unit equivalent of Net Sales.
- **Logic:** Delivered minus Return.
- **DAX:**
```DAX
Quantity (Net) = 
[Quantity (Delivered)] - [Quantity (Return)]
```

---

## Section 3 — Order-Count KPIs

### 1.Orders
- **Plain:** Count of unique orders in the current filter context, any status.
- **Logic:** Distinct count of the business key `RowID`. Using `RowID` rather than counting fact rows protects against multi-line orders being double-counted.
- **DAX:**
```DAX
1.Orders = 
DISTINCTCOUNT ( Fact_Sales[RowID] )
```

### 2.Orders (Delivered)
- **Plain:** Count of orders that reached the customer.
- **Logic:** Reuse the `1.Orders` base inside `CALCULATE` with a Status filter.
- **DAX:**
```DAX
2.Orders (Delivered) = 
CALCULATE ( [1.Orders], Fact_Sales[Status] = "Delivered" )
```

### 4.Orders (Cancelled)
- **Plain:** Count of cancelled orders.
- **DAX:**
```DAX
4.Orders (Cancelled) = 
CALCULATE ( [1.Orders], Fact_Sales[Status] = "Cancelled request" )
```

### 6.Orders (Delivery failed)
- **Plain:** Count of orders that the carrier returned as undeliverable.
- **DAX:**
```DAX
6.Orders (Delivery failed) = 
CALCULATE (
    DISTINCTCOUNT ( Fact_Sales[RowID] ),
    Fact_Sales[Status] = "Delivery failed"
)
```

### 9.Orders (Return)
- **Plain:** Count of orders that were returned after delivery.
- **DAX:**
```DAX
9.Orders (Return) = 
CALCULATE (
    DISTINCTCOUNT ( Fact_Sales[RowID] ),
    Fact_Sales[Status] = "Return"
)
```

### 1.Orders (All)
- **Plain:** Explicit roll-up of every status so ratios that use this as denominator cannot be skewed by unexpected statuses appearing in the source data.
- **Logic:** Sum of the four status-specific counts. This is deliberately *not* `[1.Orders]` — if the source ever introduces a new status code, this measure will surface the discrepancy instead of silently absorbing it.
- **DAX:**
```DAX
1.Orders (All) = 
[2.Orders (Delivered)] +
[4.Orders (Cancelled)] +
[6.Orders (Delivery failed)] +
[9.Orders (Return)]
```

### 7.Orders (Failure)
- **Plain:** Count of orders in any negative outcome — cancelled, delivery failed, or returned.
- **Logic:** Sum of the three failure categories.
- **DAX:**
```DAX
7.Orders (Failure) = 
[4.Orders (Cancelled)] +
[6.Orders (Delivery failed)] +
[9.Orders (Return)]
```

### 8.Failure Rate %
- **Plain:** Share of all orders that ended in a negative outcome.
- **Logic:** Failure count ÷ All count.
- **DAX:**
```DAX
8.Failure Rate % = 
DIVIDE ( [7.Orders (Failure)], [1.Orders (All)] )
```

---

## Section 4 — Delivery Performance

### Lead Time (Days)
- **Plain:** Average elapsed days from order placement to actual delivery — the operational cycle time.
- **Logic:** Restrict the fact table to rows with a non-blank actual delivery date, then average the per-row day-difference. Blanks are excluded because they represent orders not yet delivered (cancellations, failures, in-flight).
- **DAX:**
```DAX
Lead Time (Days) = 
AVERAGEX (
    FILTER ( Fact_Sales, NOT ISBLANK ( Fact_Sales[ActualDeliveryDate] ) ),
    DATEDIFF ( Fact_Sales[OrderDate], Fact_Sales[ActualDeliveryDate], DAY )
)
```

### On-Time Deliveries
- **Plain:** Count of delivered orders whose actual delivery date landed on or before the promised date.
- **Logic:** Row-level filter requiring both dates to be non-blank, the actual date to be `≤` the promised date, and Status = Delivered. Then count the surviving rows.
- **DAX:**
```DAX
On-Time Deliveries = 
COUNTROWS (
    FILTER (
        Fact_Sales,
        NOT ISBLANK ( Fact_Sales[ActualDeliveryDate] )
            && NOT ISBLANK ( Fact_Sales[DeliveryDate] )
            && Fact_Sales[ActualDeliveryDate] <= Fact_Sales[DeliveryDate]
            && Fact_Sales[Status] = "Delivered"
    )
)
```

### On-Time Delivery Rate
- **Plain:** Percentage of delivered orders that arrived on or before the promise date.
- **Logic:** On-Time ÷ Delivered.
- **DAX:**
```DAX
On-Time Delivery Rate = 
DIVIDE ( [On-Time Deliveries], [2.Orders (Delivered)] )
```

### Late Deliveries
- **Plain:** Count of delivered orders that arrived after the promise date.
- **Logic:** Delivered minus On-Time. The `MAX(..., 0)` floor is defensive — if On-Time ever exceeds Delivered because of a data anomaly the measure returns 0 instead of a negative count.
- **DAX:**
```DAX
Late Deliveries = 
VAR _del = [2.Orders (Delivered)]
RETURN
    MAX ( _del - [On-Time Deliveries], 0 )
```

### Late Delivery Rate
- **Plain:** Percentage of delivered orders that arrived late.
- **DAX:**
```DAX
Late Delivery Rate = 
DIVIDE ( [Late Deliveries], [2.Orders (Delivered)] )
```

---

## Section 5 — Customer & Retention

### Distinct Customers
- **Plain:** Total number of unique customer names present in the filter context — includes every status.
- **Logic:** `DISTINCTCOUNT` on CustomerName.
- **DAX:**
```DAX
Distinct Customers = 
DISTINCTCOUNT ( Fact_Sales[CustomerName] )
```

### Active Customers
- **Plain:** Unique customers who had at least one order that was actually delivered — the retained base.
- **Logic:** `DISTINCTCOUNT` under a `CALCULATE` status filter.
- **DAX:**
```DAX
Active Customers = 
CALCULATE (
    DISTINCTCOUNT ( Fact_Sales[CustomerName] ),
    Fact_Sales[Status] = "Delivered"
)
```

### Cancelled Customers
- **Plain:** Unique customers who cancelled at least one order — early churn signal.
- **Logic:** Iterate the distinct customer list, keep only those whose cancelled-row count is greater than zero, then count the surviving list.
- **DAX:**
```DAX
Cancelled Customers = 
VAR __cust_with_cancelled =
    FILTER (
        VALUES ( Fact_Sales[CustomerName] ),
        CALCULATE (
            COUNTROWS ( Fact_Sales ),
            Fact_Sales[Status] = "Cancelled request"
        ) > 0
    )
RETURN
    COUNTROWS ( __cust_with_cancelled )
```

### Avg Days Between Orders
- **Plain:** Average spacing in days between a customer's own repeated orders — the repeat-purchase interval.
- **Logic:** Four steps: (1) restrict to Delivered rows; (2) summarise per customer with min order date, max order date, and distinct-order count; (3) compute the span between first and last order; (4) average `span ÷ (n − 1)` across customers, which returns an average gap per customer that has at least two orders. `DIVIDE` returns blank when `n = 1` so single-order customers do not collapse the mean.
- **DAX:**
```DAX
Avg Days Between Orders = 
VAR __delivered =
    FILTER (
        ALLSELECTED ( Fact_Sales ),
        Fact_Sales[Status] = "Delivered"
    )
VAR __per_customer =
    ADDCOLUMNS (
        SUMMARIZE ( __delivered, Fact_Sales[CustomerName] ),
        "@min", CALCULATE ( MIN ( Fact_Sales[OrderDate] ), __delivered ),
        "@max", CALCULATE ( MAX ( Fact_Sales[OrderDate] ), __delivered ),
        "@n",   CALCULATE ( DISTINCTCOUNT ( Fact_Sales[OrderDate] ), __delivered )
    )
VAR __with_span =
    ADDCOLUMNS (
        __per_customer,
        "@span", DATEDIFF ( [@min], [@max], DAY )
    )
RETURN
    AVERAGEX ( __with_span, DIVIDE ( [@span], [@n] - 1 ) )
```

### Frequnecy Segment
- **Plain:** A categorical label that classifies each customer (or the current aggregate) into a frequency bucket based on their repeat-order gap.
- **Logic:** SWITCH on the `Avg Days Between Orders New` value. Blank → No-Repeat (one-time buyers). ≤7d Weekly, ≤30d Monthly, ≤90d Quarterly, anything longer Occasional.
- **DAX:**
```DAX
Frequnecy Segment = 
VAR _AvgDays = [Avg Days Between Orders New]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( _AvgDays ), "No - Repeat",
        _AvgDays <= 7,  "Weekly",
        _AvgDays <= 30, "Monthly",
        _AvgDays <= 90, "Quarterly",
        "Occasional"
    )
```

### AOV (Delivered invoices)
- **Plain:** Average order value on delivered orders — the primary revenue-per-order KPI.
- **Logic:** Net Sales ÷ Delivered Orders.
- **DAX:**
```DAX
AOV (Delivered invoices) = 
DIVIDE ( [Net Sales], [2.Orders (Delivered)] )
```

---

## Section 6 — Composite / Role-Playing Date

### Net Sales (Delivery Date)
- **Plain:** Revenue recognised on the **promised** delivery date rather than the order date — lets leadership see the revenue curve as the operations team promised it.
- **Logic:** Activate the inactive `DeliveryDate ↔ Dim_Date[Date]` relationship via `USERELATIONSHIP`, then evaluate `[Net Sales]` in that context.
- **DAX:**
```DAX
Net Sales (Delivery Date) = 
CALCULATE (
    [Net Sales],
    USERELATIONSHIP ( Fact_Sales[DeliveryDate], Dim_Date[Date] )
)
```

### Net Sales (actual Delivery Date)
- **Plain:** Revenue recognised on the date the order actually arrived — shows the real revenue curve.
- **Logic:** Same pattern, activating the `ActualDeliveryDate` relationship instead.
- **DAX:**
```DAX
Net Sales (actual Delivery Date) = 
CALCULATE (
    [Net Sales],
    USERELATIONSHIP ( Fact_Sales[ActualDeliveryDate], Dim_Date[Date] )
)
```

### Net Sales Adherence % (Delivery vs Actual)
- **Plain:** How closely the realised revenue tracked the planned delivery schedule. 100% means operations hit the plan exactly.
- **Logic:** Actual ÷ Planned.
- **DAX:**
```DAX
Net Sales Adherence % (Delivery vs Actual) = 
VAR Planned = [Net Sales (Delivery Date)]
VAR Actual  = [Net Sales (actual Delivery Date)]
RETURN
    DIVIDE ( Actual, Planned )
```

### Net Sales Adherence Status
- **Plain:** A categorical status label for the adherence percentage — lets stakeholders read the schedule performance at a glance without interpreting a number.
- **Logic:** Sequential SWITCH on the adherence ratio. Bands: ≥105% ahead, 95–105% on track, 80–95% slightly behind, <80% significantly behind.
- **DAX:**
```DAX
Net Sales Adherence Status = 
VAR Adherence = [Net Sales Adherence % (Delivery vs Actual)]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( Adherence ), "No data",
        Adherence >= 1.05, "Ahead of planned schedule",
        Adherence >= 0.95, "On track with plan",
        Adherence >= 0.80, "Slightly behind schedule",
        "Significantly behind schedule"
    )
```

---

## Section 7 — MoM Engine (Representative Pattern)

The `MoM %` host table contains 45 measures produced as triplets (value · label · colour) across 15 KPIs, plus 2 standalone members (`Net Sales Rolling 3M`, `Leakage Rate Color`). The three examples below are the canonical patterns — every other triplet in the folder is a structural copy with the KPI substituted.

### 0-Net Sales MoM %
- **Plain:** The change in Net Sales from the previous month to the currently-selected month, expressed as a percentage.
- **Logic:** Anchor on the latest visible `Month Start`; build an inclusive current-month date window and a prior-month window by `EOMONTH` offsets; `REMOVEFILTERS(Dim_Date)` so the `DATESBETWEEN` can redefine the range cleanly; return blank when the prior period has no revenue (prevents misleading "+∞%" outputs when a month starts cold).
- **DAX:**
```DAX
0-Net Sales MoM % = 
VAR _LatestVisiblePeriod = MAX ( Dim_Date[Month Start] )
VAR _CurrentStart =
    CALCULATE ( MIN ( Dim_Date[Date] ), Dim_Date[Month Start] = _LatestVisiblePeriod )
VAR _CurrentEnd = EOMONTH ( _CurrentStart, 0 )
VAR _PrevEnd    = EOMONTH ( _CurrentStart, -1 )
VAR _PrevStart  = EOMONTH ( _CurrentStart, -2 ) + 1
VAR _CurSales =
    CALCULATE ( [Net Sales],
        REMOVEFILTERS ( Dim_Date ),
        DATESBETWEEN ( Dim_Date[Date], _CurrentStart, _CurrentEnd )
    )
VAR _PrevSales =
    CALCULATE ( [Net Sales],
        REMOVEFILTERS ( Dim_Date ),
        DATESBETWEEN ( Dim_Date[Date], _PrevStart, _PrevEnd )
    )
RETURN
    IF ( _PrevSales > 0, DIVIDE ( _CurSales - _PrevSales, _PrevSales ), BLANK () )
```

### 0-Net Sales MoM % Label
- **Plain:** The human-readable badge rendered on the KPI card — sign-aware with arrow glyphs.
- **Logic:** SWITCH on the sign of the underlying percentage — up-arrow for positive, down-arrow for negative, right-arrow for zero.
- **DAX:**
```DAX
0-Net Sales MoM % Label = 
VAR v = [0-Net Sales MoM %]
RETURN
    IF (
        ISBLANK ( v ), BLANK (),
        SWITCH (
            TRUE (),
            v > 0, "↑ " & "+" & FORMAT ( v, "0.0%" ),
            v < 0, "↓ "       & FORMAT ( v, "0.0%" ),
                   "→ "       & FORMAT ( v, "0.0%" )
        )
    )
```

### 0-MoM Color
- **Plain:** The hex colour applied to the MoM badge — green for improvement, red for decline, grey for no data.
- **Logic:** SWITCH on the sign of the MoM value. Inline palette: `#16A34A` green · `#DC2626` red · `#6B7280` grey.
- **DAX:**
```DAX
0-MoM Color = 
VAR v = [0-Net Sales MoM %]
RETURN
    SWITCH ( TRUE (),
        ISBLANK ( v ), "#6B7280",
        v > 0,         "#16A34A",
        v < 0,         "#DC2626",
                       "#6B7280"
    )
```

### Net Sales Rolling 3M
- **Plain:** Net Sales over the trailing 3-month window anchored at the current date — smooths month-to-month volatility for the trend line.
- **Logic:** `DATESINPERIOD` anchored at the max date in context with a `-3 MONTH` window.
- **DAX:**
```DAX
Net Sales Rolling 3M = 
CALCULATE (
    [Net Sales],
    DATESINPERIOD ( Dim_Date[Date], MAX ( Dim_Date[Date] ), -3, MONTH )
)
```

### Leakage Rate Color
- **Plain:** The traffic-light hex applied to the Leakage Rate KPI card.
- **Logic:** Three thresholds, evaluated top-down. <5% green · >15% red · anything between amber · blank grey.
- **DAX:**
```DAX
Leakage Rate Color = 
VAR v = [Leakage Rate %]
RETURN
    SWITCH ( TRUE (),
        ISBLANK ( v ), "#6B7280",
        v < 0.05,      "#16A34A",
        v > 0.15,      "#DC2626",
                       "#F59E0B"
    )
```

---

## DAX Authoring Principles Applied

1. **Always use `DIVIDE` over `/`** — every ratio measure uses `DIVIDE(…)` to return blank rather than infinity on divide-by-zero.
2. **Stack measures** — ratios reference base measures (`[Net Sales]`, `[1.Orders]`), never re-summing columns, so a change in the base propagates everywhere.
3. **Use `VAR` for readability and performance** — complex measures (MoM engine, Avg Days Between Orders, Best Product) declare intermediate values as variables so they evaluate once.
4. **Keep measures on dedicated host tables** — seven headless measures-host tables group measures by purpose; no measure lives on a data-bearing table.
5. **Use `USERELATIONSHIP` for role-playing dates** — the `Test` group demonstrates three temporal views (Order / Delivery / Actual Delivery) of the same revenue using inactive relationships, without duplicating the date dimension.
6. **Pre-aggregate for iterator-heavy calcs** — `Avg Days Between Orders` uses `SUMMARIZE + ADDCOLUMNS` to materialise per-customer stats once rather than re-scanning the fact table inside an `AVERAGEX`.
7. **Guard all percentage calcs** — MoM and Leakage both wrap the numerator/denominator in `IF` / `ISBLANK` checks to suppress misleading output.
8. **Defensive arithmetic floors** — `MAX(Late − OnTime, 0)` and `DIVIDE([@span], [@n] − 1)` prevent data anomalies from producing negative counts or divide-by-zero blow-ups.
