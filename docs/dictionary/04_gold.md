# Data Dictionary — Gold Layer

Gold layer tables are produced by cross-source joins between CRM and ERP Silver tables. No business logic or aggregation — the Gold layer is the stable analytical interface consumed by the Platinum layer and ad-hoc SQL queries.

> Schema: `data_lakehouse_project.gold`
> Notebooks: `03_Gold_Layer_Notebooks/`

---

## `gold.dim_customers`

**Sources:** `silver.crm_customers` (base) + `silver.erp_loc_a101` + `silver.erp_cust_az12`
**Notebook:** `gold_layer_dim_customer.ipynb`
**Description:** Customer dimension. One row per customer, enriched with country and birthdate from ERP.
**Join key:** `customer_key`

```sql
SELECT crm_c.*, erp_loc.country, erp_cust.birthdate
FROM silver.crm_customers AS crm_c
LEFT JOIN silver.erp_loc_a101  AS erp_loc  ON crm_c.customer_key = erp_loc.customer_key
LEFT JOIN silver.erp_cust_az12 AS erp_cust ON crm_c.customer_key = erp_cust.customer_key
```

| Column | Type | Source | Description | Notes |
|---|---|---|---|---|
| `customer_id` | int | `crm_customers.customer_id` | Surrogate key | Primary key |
| `customer_key` | string | `crm_customers.customer_key` | Business key | Normalized — no `-`, no `NAS` prefix |
| `firstname` | string | `crm_customers.firstname` | | |
| `lastname` | string | `crm_customers.lastname` | | |
| `marital_status` | string | `crm_customers.marital_status` | | Values: `Married`, `Single` |
| `gender` | string | `crm_customers.gender` | | Values: `Male`, `Female` |
| `birthdate` | date | `erp_cust_az12.birthdate` | Date of birth | NULL if no ERP match |
| `country` | string | `erp_loc_a101.country` | Customer country | NULL if no ERP match; otherwise `USA`, `Germany`, etc. |
| `create_date` | date | `crm_customers.create_date` | CRM account creation date | |

---

## `gold.dim_products`

**Sources:** `silver.crm_products` (base) + `silver.erp_px_cat_g1v2`
**Notebook:** `gold_layer_dim_products.ipynb`
**Description:** Product dimension. One row per product, enriched with category information from ERP.
**Join key:** `category_id` ↔ `erp_px_cat_g1v2.id`

```sql
SELECT prd.*, px.category, px.sub_category, px.maintenance
FROM silver.crm_products AS prd
LEFT JOIN silver.erp_px_cat_g1v2 AS px ON px.id = prd.category_id
```

| Column | Type | Source | Description | Notes |
|---|---|---|---|---|
| `product_id` | int | `crm_products.product_id` | Surrogate key | Primary key |
| `product_key` | string | `crm_products.product_key` | Business key | FK in `fact_sales` |
| `category_id` | string | `crm_products.category_id` | Join key to ERP category | First 5 chars of original `prd_key` |
| `product_name` | string | `crm_products.product_name` | | |
| `product_cost` | int | `crm_products.product_cost` | Unit production cost | Imputed for 2 products (window avg) |
| `product_line` | string | `crm_products.product_line` | Product line | Values: `Road`, `Mountain`, `Touring`, `N/A`, etc. |
| `product_start_date` | date | `crm_products.product_start_date` | Catalogue entry date | |
| `product_end_date` | date | `crm_products.product_end_date` | Catalogue exit date | NULL = still active |
| `category` | string | `erp_px_cat_g1v2.category` | Top-level category | e.g. `Bikes`, `Accessories` |
| `sub_category` | string | `erp_px_cat_g1v2.sub_category` | Sub-category | e.g. `Road Bikes`, `Helmets` |
| `maintenance` | string | `erp_px_cat_g1v2.maintenance` | Maintenance requirement flag | |

---

## `gold.fact_sales`

**Source:** `silver.crm_sales` (direct promotion — no join required)
**Notebook:** `gold_layer_fact_sales.ipynb`
**Description:** Sales fact table. One row per order line. The sales table is already complete after Silver transformations.

| Column | Type | Source | Description | Notes |
|---|---|---|---|---|
| `order_number` | string | `crm_sales.order_number` | Order identifier | |
| `customer_id` | int | `crm_sales.customer_id` | FK to `dim_customers.customer_id` | |
| `product_key` | string | `crm_sales.product_key` | FK to `dim_products.product_key` | |
| `order_date` | date | `crm_sales.order_date` | Date order was placed | Parsed from `YYYYMMDD` integer in Silver |
| `ship_date` | date | `crm_sales.ship_date` | Date order was shipped | Parsed from `YYYYMMDD` integer in Silver |
| `due_date` | date | `crm_sales.due_date` | Expected delivery date | Parsed from `YYYYMMDD` integer in Silver |
| `sales` | int | `crm_sales.sales` | Order line revenue | Cross-imputed in Silver |
| `quantity` | int | `crm_sales.quantity` | Units ordered | |
| `price` | int | `crm_sales.price` | Unit price | Cross-imputed in Silver |
