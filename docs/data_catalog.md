# Data Catalog — Bike Data Lakehouse

**Catalog:** `data_lakehouse_project`
**Owner:** Jean-Baptiste Allombert
**Refresh:** Daily at 04:00 AM via Databricks Workflows (full reload — all layers)
**Dictionary:** see [`dictionary/`](dictionary/)

---

## Table Inventory

| # | Table | Schema | Layer | Grain | Description |
|---|---|---|---|---|---|
| 1 | `crm_cust_info` | `bronze` | Bronze | 1 row / CRM customer record | Raw CRM customer data — as-is from CSV |
| 2 | `crm_prd_info` | `bronze` | Bronze | 1 row / CRM product record | Raw CRM product catalogue — as-is from CSV |
| 3 | `crm_sales_details` | `bronze` | Bronze | 1 row / order line | Raw CRM sales transactions — as-is from CSV |
| 4 | `erp_cust_az12` | `bronze` | Bronze | 1 row / ERP customer record | Raw ERP customer demographics — as-is from CSV |
| 5 | `erp_loc_a101` | `bronze` | Bronze | 1 row / ERP customer location | Raw ERP customer country data — as-is from CSV |
| 6 | `erp_px_cat_g1v2` | `bronze` | Bronze | 1 row / product category | Raw ERP product category reference — as-is from CSV |
| 7 | `crm_customers` | `silver` | Silver | 1 row / customer (deduplicated) | CRM customers — typed, cleaned, normalized |
| 8 | `crm_products` | `silver` | Silver | 1 row / product | CRM products — typed, cleaned, cost imputed |
| 9 | `crm_sales` | `silver` | Silver | 1 row / order line | CRM sales — typed, dates parsed, NULLs cross-imputed |
| 10 | `erp_cust_az12` | `silver` | Silver | 1 row / ERP customer | ERP customer demographics — keys normalized, gender standardized |
| 11 | `erp_loc_a101` | `silver` | Silver | 1 row / ERP customer location | ERP locations — keys normalized, country standardized |
| 12 | `erp_px_cat_g1v2` | `silver` | Silver | 1 row / product category | ERP product categories — keys normalized |
| 13 | `dim_customers` | `gold` | Gold | 1 row / customer | Customer dimension — CRM + ERP enriched |
| 14 | `dim_products` | `gold` | Gold | 1 row / product | Product dimension — CRM + ERP enriched |
| 15 | `fact_sales` | `gold` | Gold | 1 row / order line | Sales fact table |
| 16 | `customer_360` | `platinium` | Platinum | 1 row / customer | Customer analytics — behavioural aggregates, RFMV scoring & segmentation |
| 17 | `product_360` | `platinium` | Platinum | 1 row / product | Product analytics — sales performance, logistics metrics, RFM scoring & ABC classification |

---

## Lineage

```
Sources (CSV)
│
├── source_crm/cust_info.csv      → bronze.crm_cust_info      → silver.crm_customers   ─┐
├── source_crm/prd_info.csv       → bronze.crm_prd_info        → silver.crm_products    ─┤→ gold.dim_customers
├── source_crm/sales_details.csv  → bronze.crm_sales_details   → silver.crm_sales       ─┤→ gold.dim_products
├── source_erp/CUST_AZ12.csv      → bronze.erp_cust_az12       → silver.erp_cust_az12   ─┤→ gold.fact_sales
├── source_erp/LOC_A101.csv       → bronze.erp_loc_a101        → silver.erp_loc_a101    ─┘
└── source_erp/PX_CAT_G1V2.csv   → bronze.erp_px_cat_g1v2     → silver.erp_px_cat_g1v2

gold.dim_customers + gold.dim_products + gold.fact_sales → platinium.customer_360
gold.dim_products  + gold.fact_sales                     → platinium.product_360
```

---

## Layer Contracts

| Layer | Contract |
|---|---|
| **Bronze** | Never modified after load. Exact copy of source CSV. No type casting, no filtering. |
| **Silver** | One notebook per source table. Full reload (`CREATE OR REPLACE`). Type-safe, no NULLs in key columns, normalized string values. |
| **Gold** | Cross-source joins only. No business logic. Full reload. Serves as the stable analytical interface for downstream consumers. |
| **Platinum** | Business analytics. Full rebuild. All time calculations anchored to `MAX(order_date)` from `gold.fact_sales` — not `CURRENT_DATE`. |

---

## Key column: `order_date` as temporal anchor

All time-based metrics in the Platinum layer (customer age, recency, `days_since_last_order`) use:

```sql
WITH max_date AS (
  SELECT MAX(order_date) AS max_date FROM data_lakehouse_project.gold.fact_sales
)
```

This makes the pipeline fully reproducible: re-running on any date produces identical output for the same dataset.

---

## Naming conventions

| Convention | Example |
|---|---|
| Bronze raw field names kept as-is from source | `cst_id`, `CID`, `CNTRY` |
| Silver renames to snake_case readable names | `customer_id`, `country` |
| Gold preserves Silver naming, no prefix | `customer_id`, `product_key` |
| Platinum adds computed columns as new names | `rfmv_score`, `abc_class`, `days_since_last_order` |
| Key join column across CRM/ERP | `customer_key` (normalized: no `-`, no `NAS` prefix) |
| Key join column CRM products/ERP categories | `category_id` (first 5 chars of `prd_key`) |
