# Data Dictionary — Silver Layer (CRM)

Silver CRM tables clean, type and normalize the three CRM bronze tables. Each notebook produces one Delta table via `CREATE OR REPLACE TABLE`.

> Schema: `data_lakehouse_project.silver`
> Notebooks: `02_Silver_Layer_Notebooks/crm/`

---

## `silver.crm_customers`

**Source:** `bronze.crm_cust_info`
**Notebook:** `crm/silver_layer_crm_cust_info.ipynb`
**Description:** Deduplicated, normalized CRM customer master. One row per unique customer.

**Key transformations:**
- Deduplication on `cst_id` — rows with the same ID but NULL values removed, latest record kept
- All string columns trimmed
- `cst_marital_status` → `M` / `S` → `Married` / `Single`
- `cst_gndr` → `M` / `F` → `Male` / `Female`
- `cst_key` → remove `-` and `NAS` prefix (alignment with ERP key format)

| Column | Type | Source field | Description | Notes |
|---|---|---|---|---|
| `customer_id` | int | `cst_id` | Surrogate customer identifier | Primary key |
| `customer_key` | string | `cst_key` | Business key used for CRM ↔ ERP join | Normalized: no `-`, no `NAS` prefix |
| `firstname` | string | `cst_firstname` | First name | Trimmed |
| `lastname` | string | `cst_lastname` | Last name | Trimmed |
| `marital_status` | string | `cst_marital_status` | Marital status | Values: `Married`, `Single` |
| `gender` | string | `cst_gndr` | Gender | Values: `Male`, `Female` |
| `create_date` | date | `cst_create_date` | CRM account creation date | |

---

## `silver.crm_products`

**Source:** `bronze.crm_prd_info`
**Notebook:** `crm/silver_layer_crm_prd_info.ipynb`
**Description:** Cleaned CRM product catalogue with split keys and imputed costs.

**Key transformations:**
- All strings trimmed
- `prd_key` split into `category_id` (first 5 chars) and `product_key` (chars 7+)
- `prd_cost` NULL imputation via window average over `prd_nm` prefix (first 12 chars), cast to int
- `prd_line` NULL → `"N/A"`

| Column | Type | Source field | Description | Notes |
|---|---|---|---|---|
| `product_id` | int | `prd_id` | Product identifier | Primary key |
| `product_key` | string | `prd_key` (chars 7+) | Business key used in sales transactions and Gold join | Extracted from positions 7 onwards of `prd_key` |
| `category_id` | string | `prd_key` (chars 0–4) | FK to ERP category reference (`erp_px_cat_g1v2.id`) | First 5 characters of `prd_key` |
| `product_name` | string | `prd_nm` | Product name | Trimmed |
| `product_cost` | int | `prd_cost` | Unit production cost | 2 NULLs imputed via window average over product name prefix |
| `product_line` | string | `prd_line` | Product line (e.g. Road, Mountain) | NULL replaced with `N/A` |
| `product_start_date` | date | `prd_start_dt` | Date product entered catalogue | |
| `product_end_date` | date | `prd_end_dt` | Date product left catalogue | NULL = product still active |

---

## `silver.crm_sales`

**Source:** `bronze.crm_sales_details`
**Notebook:** `crm/silver_layer_crm_sales_details.ipynb`
**Description:** Typed and cleaned order line transactions. One row per order line.

**Key transformations:**
- All strings trimmed and uppercased
- `sls_sales` and `sls_price` cross-imputed: both columns carry equivalent values; NULLs in one filled from the other
- Date integers (`YYYYMMDD`) parsed by inserting separators and casting to `date`

| Column | Type | Source field | Description | Transformation | Notes |
|---|---|---|---|---|---|
| `order_number` | string | `sls_ord_num` | Order identifier | Trimmed, uppercased | |
| `customer_id` | int | `sls_cust_id` | FK to `silver.crm_customers.customer_id` | | |
| `product_key` | string | `sls_prd_key` | FK to `silver.crm_products.product_key` | Trimmed, uppercased | |
| `order_date` | date | `sls_order_dt` | Date the order was placed | Parsed from `YYYYMMDD` int | |
| `ship_date` | date | `sls_ship_dt` | Date the order was shipped | Parsed from `YYYYMMDD` int | |
| `due_date` | date | `sls_due_dt` | Expected delivery date | Parsed from `YYYYMMDD` int | |
| `sales` | int | `sls_sales` | Order line revenue | Cross-imputed from `price` when NULL | Equivalent to `price` for single-unit lines |
| `quantity` | int | `sls_quantity` | Number of units | | |
| `price` | int | `sls_price` | Unit price | Cross-imputed from `sales` when NULL | |
