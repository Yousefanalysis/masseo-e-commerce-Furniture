# Raw Data — Source Description

**Folder:** `01_raw_data/`
**Purpose:** Original Excel files as received from the client, unmodified.

---

## Files in This Folder

| File | Format | Grain | Rows | Description |
|---|---|---|---|---|
| `Fact_Sales.xlsx` | `.xlsx` | Order-line (RowID × ProductID) | 112 | Master order ledger: dates, status, price, quantity, channel, customer type, governorate, delivery persons |
| `Dim_Product.xlsx` | `.xlsx` | SKU | 70 | Product master: type, name, description, colour/metal, fabric, size |
| `Dim_Channel.xlsx` | `.xlsx` | Channel | 8 | Sales channel master: ChannelID + Channel display name |
| `Dim_Customer_Type.xlsx` | `.xlsx` | Segment | 2 | Customer type master: فرد (Individual) · شركة (Company) |

---

## Date Range

- Earliest `OrderDate` in `Fact_Sales`: **2024-01-09**
- Latest `OrderDate`: **2024-12-31**
- `DeliveryDate` and `ActualDeliveryDate` may extend into 2025 for orders placed in late December 2024.

---

## Status Values in Fact_Sales

| Status | Count | Meaning |
|---|---|---|
| `Delivered` | 91 | Order reached customer |
| `Cancelled request` | 17 | Order cancelled before delivery |
| `Delivery failed` | 3 | Carrier returned undeliverable |
| `Return` | 1 | Delivered but returned by customer |

---

## Notes

- `IncorrectSellingPrice` column flags rows where `sellingPrice` diverges from the expected price list — retained for data-quality audit, not used in any KPI calculation.
- `Dim_Date` calendar (479 rows, 2024-01-09 → 2025-05-01) was **generated in Power Query** and is not stored here — see `02_cleaned_data/`.
- `Dim_Project` and `Dim_Customer_Type` are small enough (2–8 rows) to have been received as separate files; no transformation was required beyond loading.
