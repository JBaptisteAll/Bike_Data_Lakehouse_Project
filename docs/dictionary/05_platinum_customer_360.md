# Data Dictionary — `platinium.customer_360`

**Source tables:** `gold.dim_customers`, `gold.fact_sales`, `gold.dim_products`
**Notebook:** `04_Platinium_Layer_Notebook/platinium_layer_customer_360.ipynb`
**Grain:** One row per customer
**Description:** 360° customer analytical view. Combines CRM/ERP demographics with full purchase history, behavioural aggregates, and RFMV segmentation. Rebuilt daily via `CREATE OR REPLACE TABLE`.

---

## Temporal anchor

All time-based calculations use:
```sql
WITH max_date AS (SELECT MAX(order_date) AS max_date FROM data_lakehouse_project.gold.fact_sales)
```
This anchors recency and age to the most recent transaction in the dataset — not to `CURRENT_DATE` — ensuring reproducibility.

---

## Column reference

### Identity & demographics

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `customer_id` | int | `dim_customers.customer_id` | Surrogate key | Primary key |
| `fullname` | string | `dim_customers` | Full name | `CONCAT(firstname, ' ', lastname)` |
| `gender` | string | `dim_customers.gender` | | Values: `Male`, `Female` |
| `birthdate` | date | `dim_customers.birthdate` | Date of birth | |
| `age` | int | Computed | Age in years | `DATE_DIFF(year, birthdate, max_date)` |
| `country` | string | `dim_customers.country` | Country of residence | |

### Purchase history

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `first_order` | date | `fact_sales.order_date` | Date of first ever purchase | `MIN(order_date)` per customer |
| `last_order` | date | `fact_sales.order_date` | Date of most recent purchase | `MAX(order_date)` per customer |
| `total_orders` | bigint | `fact_sales.order_number` | Number of distinct orders placed | `COUNT(DISTINCT order_number)` |
| `total_sales` | bigint | `fact_sales.sales` | Total revenue generated | `SUM(sales)` |
| `total_quantity` | bigint | `fact_sales.quantity` | Total units purchased | `SUM(quantity)` |
| `avg_order_value` | double | Computed | Average revenue per order | `ROUND(total_sales / total_orders, 2)` |
| `avg_days_between_delivery_and_order` | double | Computed | Average lead time (order → due) | `AVG(DATE_DIFF(due_date, order_date))` |

### Product preferences

Preferences are ranked by quantity purchased. In case of tie, no secondary sort is applied — ranking is deterministic but arbitrary among equals.

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `first_product_ordered` | string | `dim_products.product_name` | First product ever purchased | Product name at `MIN(order_date)`, tie-break on `MIN(order_number)` |
| `top_category` | string | `dim_products.category` | Category with most units bought | `ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY SUM(quantity) DESC)` = 1 |
| `top_subcategory` | string | `dim_products.sub_category` | Sub-category with most units bought | Same ranking method |
| `top_product` | string | `dim_products.product_name` | Product with most units bought | Same ranking method |

### Recency

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `days_since_last_order` | int | Computed | Days between last order and dataset reference date | `DATEDIFF(MAX(last_order) OVER(), last_order)` — distance from the most recent customer's last order across all customers. Equivalent to distance from `max_date`. |

### RFMV scores

Scores are computed as quintiles (NTILE 5) — each score ranges from 1 (lowest) to 5 (highest). Scoring is relative: a score of 5 means the customer is in the top 20% of the portfolio for that dimension.

| Column | Type | Description | Calculation | Score 5 means |
|---|---|---|---|---|
| `r_score` | int | Recency score | `NTILE(5) OVER(ORDER BY last_order ASC)` | Most recently active customer |
| `f_score` | int | Frequency score | `NTILE(5) OVER(ORDER BY total_orders ASC)` | Highest order count |
| `m_score` | int | Monetary score | `NTILE(5) OVER(ORDER BY total_sales ASC)` | Highest total revenue |
| `v_score` | int | Value score | `NTILE(5) OVER(ORDER BY avg_order_value ASC)` | Highest average order value |

> **Note on NTILE direction:** `ORDER BY ... ASC` means the last quintile (5) gets the best values. Example for `r_score`: ascending order puts the oldest last orders in NTILE 1, the most recent in NTILE 5.

### RFMV composite

| Column | Type | Description | Calculation |
|---|---|---|---|
| `rfmv_score` | double | Composite RFMV score | `(r_score + f_score + m_score + v_score) / 4.0` — range: 1.0 to 5.0 |
| `rfmv_segment` | string | Business segment label | See segmentation table below |

### RFMV segmentation thresholds

| Segment | Condition | Business meaning |
|---|---|---|
| `VIP` | `rfmv_score >= 4.5` | Top customers across all dimensions |
| `Premium` | `rfmv_score >= 3.5` | High-value, active customers |
| `Standard` | `rfmv_score >= 2.5` | Average engagement |
| `At Risk` | `rfmv_score >= 1.5` | Declining engagement, worth re-activating |
| `Dormant` | `rfmv_score < 1.5` | Inactive, low-value customers |

---

## Query example

```sql
-- Segment distribution with average revenue
SELECT rfmv_segment,
       COUNT(*)                              AS nb_customers,
       ROUND(AVG(total_sales), 2)            AS avg_total_revenue,
       ROUND(AVG(avg_order_value), 2)        AS avg_order_value
FROM data_lakehouse_project.platinium.customer_360
GROUP BY rfmv_segment
ORDER BY avg_total_revenue DESC;
```
