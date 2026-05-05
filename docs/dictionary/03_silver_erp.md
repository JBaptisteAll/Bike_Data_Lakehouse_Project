# Data Dictionary — Silver Layer (ERP)

Silver ERP tables normalize the three ERP bronze tables. The main challenge is aligning ERP customer keys to CRM format so that cross-source joins in the Gold layer work correctly.

> Schema: `data_lakehouse_project.silver`
> Notebooks: `02_Silver_Layer_Notebooks/erp/`

---

## `silver.erp_cust_az12`

**Source:** `bronze.erp_cust_az12`
**Notebook:** `erp/silver_layer_erp_cust_az12.ipynb`
**Description:** ERP customer demographics (birthdate, gender). Joined to CRM in the Gold layer via `customer_key`.

**Key transformations:**
- `CID` trimmed, uppercased, `-` removed, `NAS` prefix removed — aligns with `silver.crm_customers.customer_key`
- Gender normalized via regex: any value starting with `F` → `Female`, any value starting with `M` → `Male`

| Column | Type | Source field | Description | Notes |
|---|---|---|---|---|
| `customer_key` | string | `CID` | Business key — joins to `gold.dim_customers.customer_key` | Normalized: trimmed, uppercased, no `-`, no `NAS` prefix |
| `birthdate` | date | `BDATE` | Customer date of birth | Used to compute `age` in Platinum |
| `gender` | string | `GEN` | Gender | Values: `Male`, `Female` (normalized via regex) |

---

## `silver.erp_loc_a101`

**Source:** `bronze.erp_loc_a101`
**Notebook:** `erp/silver_layer_erp_loc_a101.ipynb`
**Description:** ERP customer country data. Joined to CRM in the Gold layer via `customer_key`.

**Key transformations:**
- `CID` normalized (same logic as `erp_cust_az12`)
- Country standardized: NULL / empty → `N/A`, all US variants (`US`, `United States`, `United-States`) → `USA`, all DE variants (`DE`, `Ger`) → `Germany`

| Column | Type | Source field | Description | Notes |
|---|---|---|---|---|
| `customer_key` | string | `CID` | Business key — joins to `gold.dim_customers.customer_key` | Same normalization as `erp_cust_az12.customer_key` |
| `country` | string | `CNTRY` | Customer country | Values: `USA`, `Germany`, `N/A`, and other standardized country names |

---

## `silver.erp_px_cat_g1v2`

**Source:** `bronze.erp_px_cat_g1v2`
**Notebook:** `erp/silver_layer_erp_px_cat_g1v2.ipynb`
**Description:** ERP product category reference. Joined to CRM products in the Gold layer via `id` ↔ `category_id`.

**Key transformations:**
- `ID` trimmed, uppercased, `_` replaced with `-` — aligns with `silver.crm_products.category_id` format (which comes from the first 5 chars of `prd_key`)

| Column | Type | Source field | Description | Notes |
|---|---|---|---|---|
| `id` | string | `ID` | Category identifier — joins to `silver.crm_products.category_id` | Normalized: trimmed, uppercased, `_` → `-` |
| `category` | string | `CAT` | Top-level category name (e.g. Bikes, Accessories) | |
| `sub_category` | string | `SUBCAT` | Sub-category name (e.g. Road Bikes, Helmets) | |
| `maintenance` | string | `MAINTENANCE` | Maintenance requirement flag | |
