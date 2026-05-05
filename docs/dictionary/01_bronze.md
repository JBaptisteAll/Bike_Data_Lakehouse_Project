# Data Dictionary — Bronze Layer

Bronze tables are loaded as-is from source CSVs. No transformation is applied. Column names, types, and values are exactly as received from the source systems.

> All Bronze tables are stored as Delta format in schema `data_lakehouse_project.bronze`.
> Notebook: `01_Bronze_Layer_Notebook/bronze_layer.ipynb`

---

## `bronze.crm_cust_info`

**Source:** `Volumes/.../source_crm/cust_info.csv`
**Description:** Raw CRM customer master records.

| Column | Type | Description | Notes |
|---|---|---|---|
| `cst_id` | int | Customer identifier in CRM | May contain duplicates — deduplication applied in Silver |
| `cst_key` | string | Alternate customer key used for cross-system joins | May contain `-` separators and `NAS` prefix — cleaned in Silver |
| `cst_firstname` | string | First name | May contain leading/trailing spaces |
| `cst_lastname` | string | Last name | May contain leading/trailing spaces |
| `cst_marital_status` | string | Marital status code | Raw values: `M`, `S`, and variants — normalized in Silver |
| `cst_gndr` | string | Gender code | Raw values: `M`, `F`, and variants — normalized in Silver |
| `cst_create_date` | string | Account creation date | String — cast to `date` in Silver |

---

## `bronze.crm_prd_info`

**Source:** `Volumes/.../source_crm/prd_info.csv`
**Description:** Raw CRM product catalogue.

| Column | Type | Description | Notes |
|---|---|---|---|
| `prd_id` | int | Product identifier | |
| `prd_key` | string | Compound product key encoding category and product | Format: `[CATID]-[PRDKEY]` — split in Silver into `category_id` + `product_key` |
| `prd_nm` | string | Product name | May contain leading/trailing spaces |
| `prd_cost` | int | Unit cost | 2 rows with NULL — imputed via window average in Silver |
| `prd_line` | string | Product line | NULL for some rows — replaced with `N/A` in Silver |
| `prd_start_dt` | date | Product catalogue start date | |
| `prd_end_dt` | date | Product catalogue end date | NULL = product still active |

---

## `bronze.crm_sales_details`

**Source:** `Volumes/.../source_crm/sales_details.csv`
**Description:** Raw CRM order line transactions.

| Column | Type | Description | Notes |
|---|---|---|---|
| `sls_ord_num` | string | Order number | |
| `sls_cust_id` | int | Customer identifier (FK to crm_cust_info) | |
| `sls_prd_key` | string | Product key (FK to crm_prd_info) | Matches `product_key` after Silver split |
| `sls_order_dt` | int | Order date as integer | Format `YYYYMMDD` — parsed to `date` in Silver |
| `sls_ship_dt` | int | Ship date as integer | Format `YYYYMMDD` — parsed to `date` in Silver |
| `sls_due_dt` | int | Due/delivery date as integer | Format `YYYYMMDD` — parsed to `date` in Silver |
| `sls_sales` | int | Order line revenue | Cross-imputed with `sls_price` in Silver when NULL |
| `sls_quantity` | int | Units ordered | |
| `sls_price` | int | Unit price | Cross-imputed with `sls_sales` in Silver when NULL |

---

## `bronze.erp_cust_az12`

**Source:** `Volumes/.../source_erp/CUST_AZ12.csv`
**Description:** Raw ERP customer demographic data (birthdate, gender).

| Column | Type | Description | Notes |
|---|---|---|---|
| `CID` | string | Customer identifier in ERP | Format differs from CRM — normalized in Silver (remove `-`, `NAS`) |
| `BDATE` | date | Date of birth | |
| `GEN` | string | Gender | Raw values include variants (`F`, `Female`, `M`, `Male`, etc.) — normalized in Silver via regex |

---

## `bronze.erp_loc_a101`

**Source:** `Volumes/.../source_erp/LOC_A101.csv`
**Description:** Raw ERP customer location data (country).

| Column | Type | Description | Notes |
|---|---|---|---|
| `CID` | string | Customer identifier in ERP | Same normalization issue as `erp_cust_az12.CID` |
| `CNTRY` | string | Country | Raw values include NULL, empty string, US variants (`US`, `USA`, `United States`), DE variants (`DE`, `Germany`) — normalized in Silver |

---

## `bronze.erp_px_cat_g1v2`

**Source:** `Volumes/.../source_erp/PX_CAT_G1V2.csv`
**Description:** Raw ERP product category reference table.

| Column | Type | Description | Notes |
|---|---|---|---|
| `ID` | string | Category identifier | Contains `_` separators — replaced with `-` in Silver to match CRM `category_id` format |
| `CAT` | string | Category name | |
| `SUBCAT` | string | Sub-category name | |
| `MAINTENANCE` | string | Maintenance flag | |
