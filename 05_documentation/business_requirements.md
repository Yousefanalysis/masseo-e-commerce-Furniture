# Business Requirements

**Project:** Masseo E-commerce Furniture Sales Analysis
**Client:** Masseo — multi-channel Egyptian furniture e-commerce operation
**Prepared by:** Yousef Tarek (Sole Data Analyst / BI Developer)

---

## 1. Background

Masseo sells furniture through eight external channels (هومزمارت · شيك هومز · تاكى · اوسكار ريتن · فيس بوك · منزلى · كيمت · لا يوجد) alongside a direct customer base. Orders are managed end-to-end by an internal operations team responsible for dispatch, delivery coordination, and returns. Before this project, performance reporting was done ad-hoc in Excel with no standardized KPI definitions, no visibility into delivery-schedule adherence, and no customer-retention view.

---

## 2. Business Questions (CONVO Framework)

| # | Question | Priority |
|---|---|---|
| BQ-01 | Where is revenue leaking between order placement and successful delivery, and what is the monetary magnitude? | Critical |
| BQ-02 | Which sales channels and product types concentrate the most net revenue — and what is the channel-concentration risk? | High |
| BQ-03 | How well is the operations team keeping to promised delivery dates, and where does the plan vs actual diverge? | High |
| BQ-04 | Are customers returning after their first purchase, and at what frequency interval? | Medium |
| BQ-05 | Which products carry the highest cancellation and return exposure (profitability risk) alongside their gross revenue? | Medium |
| BQ-06 | Is there a geographic concentration in orders that represents either a risk or an expansion opportunity? | Medium |

---

## 3. KPI Requirements

| KPI | Required Granularity | Required Comparison |
|---|---|---|
| Net Sales | Total · by channel · by product type · by governorate · by customer type | MoM % · Rolling 3M |
| Leakage Rate % | Total · by status category | Traffic-light status (green/amber/red) |
| Total Orders | Total · by status | MoM % |
| On-Time Delivery Rate | Total | vs 85% target |
| Lead Time (Days) | Total | vs 21-day benchmark |
| Net Sales Adherence % | Total | vs 100% (Ahead / On track / Behind labels) |
| Active Customers | Total | MoM % |
| AOV (Delivered Invoices) | Total | MoM % |
| Avg Days Between Orders | Per customer / aggregate | Frequency Segment label |

---

## 4. Report Structure Requirements

| Page | Audience | Primary KPIs |
|---|---|---|
| Menu | All | Navigation landing page |
| Operations Performance Pulse | Management / Operations lead | Net Sales · Leakage · Orders · Delivery adherence |
| Sales Report | Commercial / Channel manager | Sales amount · Channel breakdown · Product trends · Payment method |
| Insight & Recommended | Management | Derived insights, recommended actions |
| Inventory Analysis | Operations / Procurement | [FILL: confirm scope with client] |

---

## 5. Data Requirements

| Requirement | Detail |
|---|---|
| Fact grain | Order-line (one row per RowID × ProductID) |
| Date axes | Three: OrderDate (active) · DeliveryDate (inactive) · ActualDeliveryDate (inactive) |
| Status values | Delivered · Cancelled request · Delivery failed · Return |
| Bilingual support | Arabic product names, channel names, governorates preserved as-is in the model |
| Calendar convention | Saturday-start week (Egyptian retail standard) |
| Historical range | 2024-01-09 onwards; calendar extended to 2025-05-01 for delivery-date resolution |

---

## 6. Non-Functional Requirements

- Report must open without requiring external data refresh (data embedded in `.pbix`)
- All KPI definitions standardized and documented in `03_data_model/dax_measures.md` and `03_data_model/data_dictionary.md`
- Conditional formatting applied via DAX measures (not per-visual rules) for portability
- Model must support XMLA read for Tabular Editor / DAX Studio inspection
