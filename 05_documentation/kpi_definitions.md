# KPI Definitions — KPI Chain Format

**Project:** Masseo E-commerce Furniture Sales Analysis
**Format:** Plain → Logic → DAX (KPI Chain standard)
**Scope:** 15 headline KPIs surfaced on the Operations Performance Pulse and Sales Report pages.

---

## Revenue KPIs

### Sales Amount
- **Plain:** Gross revenue across every order regardless of outcome — the top-of-funnel number before any leakage is removed.
- **Logic:** Iterate the fact table; multiply `sellingPrice × Quantity` per line; sum the result. `SUMX` is required because price and quantity are on separate columns.
- **DAX:** See `dax_measures.md` § Section 1

### Net Sales
- **Plain:** The headline revenue KPI — money that was actually delivered and kept by the business.
- **Logic:** `Sales amount (Delivered) − Sales amount (Return)`. Cancelled and Failed orders are not in the Delivered pool and are therefore already excluded.
- **Target:** Maximise; compare against prior month via MoM %.

### Leakage Amount
- **Plain:** The gap between what was ordered and what the company kept — the monetary cost of cancellations, delivery failures, and returns.
- **Logic:** `Sales amount − Net Sales`.
- **Target:** Minimise.

### Leakage Rate %
- **Plain:** Leakage as a share of gross revenue. The single most actionable operational risk indicator.
- **Logic:** `(Gross − Net) ÷ Gross`. Traffic-light bands: <5% green · 5–15% amber · >15% red.
- **Target:** <5%.

### AOV (Delivered Invoices)
- **Plain:** Average revenue per successfully delivered order.
- **Logic:** `Net Sales ÷ 2.Orders (Delivered)`.
- **Target:** Maximise; track MoM for pricing / product-mix signal.

---

## Order-Count KPIs

### Total Orders
- **Plain:** Count of unique orders in the current filter context (any status).
- **Logic:** `DISTINCTCOUNT(Fact_Sales[RowID])`. Using the business key protects against multi-line orders being double-counted.

### Orders (Delivered)
- **Plain:** Count of orders that successfully reached the customer.
- **Logic:** `1.Orders` filtered to `Status = "Delivered"`.
- **Target:** Maximise.

### Failure Rate %
- **Plain:** Share of all orders that ended in a negative outcome (cancelled + failed + returned).
- **Logic:** `7.Orders (Failure) ÷ 1.Orders (All)`.
- **Target:** <5%.

### Cancel Rate %
- **Plain:** Share of orders cancelled before dispatch.
- **Logic:** `4.Orders (Cancelled) ÷ 1.Orders (All)`.
- **Target:** <5%.

---

## Delivery Performance KPIs

### Lead Time (Days)
- **Plain:** Average elapsed days from order placement to actual delivery — the end-to-end operational cycle time.
- **Logic:** `AVERAGEX` over rows with a non-blank `ActualDeliveryDate` of `DATEDIFF(OrderDate, ActualDeliveryDate, DAY)`. Blanks excluded (cancellations, failures, in-flight).
- **Target:** Custom-furniture benchmark: ≤21 days.

### On-Time Delivery Rate
- **Plain:** Share of delivered orders that arrived on or before the date the operations team promised.
- **Logic:** Count of Delivered rows where `ActualDeliveryDate ≤ DeliveryDate`, divided by total delivered orders.
- **Target:** ≥85%.

### Net Sales Adherence % (Delivery vs Actual)
- **Plain:** How closely realised revenue tracked the promised delivery schedule across the full reporting period. 100% = schedule hit exactly.
- **Logic:** `Net Sales (actual Delivery Date) ÷ Net Sales (Delivery Date)` — activates both inactive date relationships via `USERELATIONSHIP`.
- **Target:** 95%–105% ("On track with plan" band).

---

## Customer KPIs

### Active Customers
- **Plain:** Distinct customers who had at least one successfully delivered order.
- **Logic:** `DISTINCTCOUNT(CustomerName)` filtered to `Status = "Delivered"`.

### Avg Days Between Orders
- **Plain:** Typical gap in days between a customer's repeat orders — the repeat-purchase interval.
- **Logic:** Two-level `SUMMARIZE + ADDCOLUMNS`: group delivered orders by customer, compute first-to-last order span and distinct-date count per customer, then average `span ÷ (orders − 1)` across all customers with more than one order.
- **Interpretation:** ≤7 days = Weekly · ≤30 = Monthly · ≤90 = Quarterly · >90 = Occasional · no repeat = No-Repeat.

### Frequnecy Segment
- **Plain:** Categorical label classifying the current customer aggregate into a repeat-purchase frequency bucket.
- **Logic:** `SWITCH` on `Avg Days Between Orders New`. Blank → No-Repeat; ≤7 → Weekly; ≤30 → Monthly; ≤90 → Quarterly; else → Occasional.
