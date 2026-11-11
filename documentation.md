# Executive Documentation: FY 26–27 Monthly Inventory KPI Model

## Executive Summary

This document details the architecture, computational logic, and business rules governing the dynamic **FY 26–27 Monthly Inventory KPI Dashboard**. 

Designed natively for PostgreSQL and Odoo ERP data structures, this model resolves a fundamental ERP limitation: `stock_quant` only captures real-time, live stock balances. By incorporating transaction-level rollback mechanisms from `stock_move`, historical sales trends from `sale_order`, and lot expiration data from `stock_lot`, this engine generates month-end inventory metrics for all 12 months of Fiscal Year 2026–2027 (April 2026 – March 2027).

---

## Technical Architecture & Processing Pipeline

The calculation engine operates via a 6-stage Common Table Expression (CTE) pipeline executed in a single read-only transaction:

1. **Date Backbone Generator (`fy_months`)**: Generates 12 calendar month-end dates for FY 26–27.
2. **Location & SKU Filtering (`location_filter`, `sku_unique`)**: Filters target warehouses and normalizes variant SKUs.
3. **Historical Stock Rollback (`live_stock`, `stock_movements`, `historical_inventory`)**: Reconstructs month-end inventory balances by reversing transactions post-date.
4. **Valuation Engine (`latest_aml_price`, `sku_monthly_metrics`)**: Fetches cost and list prices to compute SKU-level monthly metrics.
5. **Metric Aggregations (`monthly_kpis`)**: Aggregates SKU calculations into enterprise-level KPIs in INR Crores.
6. **Dashboard Pivot Output (`UNION ALL` Select)**: Cross-tabulates metrics into a standard horizontal month-by-month table.

---

## Detailed Logic Breakdown

### 1. Dynamic Fiscal Date Backbone (`fy_months`)
* **Logic:** Utilizes PostgreSQL's `generate_series` to construct 12 distinct month-end date boundaries starting from April 30, 2026 through March 31, 2027.
* **Business Purpose:** Eliminates static relative date references (such as `current_date`). Every inventory, sales velocity, and valuation metric is calculated retroactively or prospectively relative to each specific month-end snapshot date.

---

### 2. Location & SKU Normalization (`location_filter`, `sku_unique`)
* **Warehouse Scope:** Restricts stock data strictly to target Finished Goods warehouses (`FG Warehouse MP`, `FG Warehouse MP 1`, `FG Warehouse NS`) and operational sub-locations matching patterns such as `Domestic` and `Container`.
* **SKU Normalization:** Groups active, non-service product variants back to parent base SKUs by stripping ERP operational suffixes (e.g., `-IN`, `old`). This prevents artificial stock fragmentation across parent/child variant records.

---

### 3. Historical Stock Reconstruction (`live_stock`, `stock_movements`, `historical_inventory`)
* **Core Problem:** Odoo's `stock_quant` table only maintains current live balances, rendering historical point-in-time querying impossible out of the box.
* **Mathematical Rollback Logic:** To compute the stock balance at any historical month-end date ($N$), the model calculates:

$$\text{Ending Stock}_{\text{Month } N} = \text{Current Live Stock} + \text{Outward Moves}_{\text{after Month } N} - \text{Inward Moves}_{\text{after Month } N}$$

* **Outward Movements (Added Back):** Completed sales deliveries, scrap entries, and inventory damage write-offs that occurred *after* date $N$ are added back to current live stock.
* **Inward Movements (Deducted):** Purchase receipts, vendor returns, and inward transfers completed *after* date $N$ are subtracted from current live stock.

---

### 4. Pricing & Valuations (`latest_aml_price`, `sku_monthly_metrics`)
* **Selling Price / MRP:** Extracted from product list prices (`product_template.list_price`) combined with variant-specific attribute price extras (`price_extra`).
* **Cost Price (CP):** Retrieved from Odoo’s company-dependent property store (`ir_property` where `name = 'standard_price'`).
* **Valuations:** Computed at the SKU level for every month:
  $$\text{MRP Valuation} = \text{Reconstructed Quantity} \times \text{Unit Selling Price (MRP)}$$
  $$\text{CP Valuation} = \text{Reconstructed Quantity} \times \text{Unit Cost Price (CP)}$$

---

## Business Rules for Dashboard KPIs

| Row | Metric Name | Definition & Business Logic | Unit of Measure |
| :--- | :--- | :--- | :--- |
| **1** | **Total Inventory Value @ MRP** | Sum of all stock quantities at month-end multiplied by unit MRP. | INR Cr. |
| **2** | **Total Inventory Value @ CP** | Sum of all stock quantities at month-end multiplied by unit Cost Price. | INR Cr. |
| **3** | **Near Expiry Inventory Value** | Stock valuation of items whose lot expiration date (`stock_lot.expiration_date`) falls within **89 days** after the target month-end. | INR Cr. |
| **4** | **Excess Inventory Value (>2x TMC)** | Valuation of stock held above $2 \times \text{Target Monthly Consumption (TMC)}$.<br>$$\text{TMC} = \left(\frac{\text{Sales Qty (Trailing 3M)}}{\text{Days in Trailing 3M}}\right) \times \text{Days in Current Month}$$<br>$$\text{Excess Qty} = \max(0, \text{Total Stock} - 2 \times \text{TMC})$$ | INR Cr. |
| **5** | **Obsolete Inventory Value (P6M)** | Total Cost Price valuation of SKUs with **zero sales** in the 6-month period prior to the target month-end. | INR Cr. |
| **6** | **Dead Stock Value (P6M)** | Total Cost Price valuation of SKUs with **$>0$ but $<2$ units sold** in the 6-month period prior to the target month-end. | INR Cr. |
| **7** | **Dead + Obsolete Stock Value** | Combined valuation of non-moving and ultra-low moving inventory:<br>$$\text{Dead Valuation} + \text{Obsolete Valuation}$$ | INR Cr. |
| **8** | **Dead + Obsolete Stock %** | Ratio of unproductive inventory relative to total stock cost:<br>$$\left(\frac{\text{Dead Value} + \text{Obsolete Value}}{\text{Total Inventory CP Value}}\right) \times 100$$ | % |
| **9** | **Days of Inventory (DOI)** | Overall stock holding period based on daily Cost-of-Goods-Sold (COGS) velocity:<br>$$\text{DOI} = \frac{\text{Total Inventory Value @ CP}}{\sum (\text{Daily Avg Qty Sold (P3M)} \times \text{Unit Cost Price})}$$ | No. of Days |

---

## Data Safety & Governance Note

* **Non-Destructive Execution:** The query uses standard ANSI SQL `SELECT` operations and CTE structures. It executes strictly in **read-only** mode.
* **Zero Database Impact:** The query creates no temporary tables, performs no data modification (`INSERT`, `UPDATE`, `DELETE`), and makes no schema adjustments (`ALTER`, `DROP`). It can be run safely inside Metabase or directly against production replicas.