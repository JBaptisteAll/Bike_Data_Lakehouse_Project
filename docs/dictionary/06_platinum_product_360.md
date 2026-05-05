# Data Dictionary — `platinium.product_360`

**Source tables:** `gold.dim_products`, `gold.fact_sales`
**Notebook:** `04_Platinium_Layer_Notebook/platinium_layer_product_360.ipynb`
**Grain:** One row per product
**Description:** 360° product analytical view. Combines product catalogue data with sales performance, logistics metrics, RFM scoring, and ABC Pareto classification. Rebuilt daily via `CREATE OR REPLACE TABLE`.

---

## Temporal anchor

All time-based calculations use:
```sql
WITH max_date AS (SELECT MAX(order_date) AS max_date FROM data_lakehouse_project.gold.fact_sales)
```
This anchors recency to the most recent transaction in the dataset — not to `CURRENT_DATE`.

---

## Column reference

### Identity & catalogue

| Column | Type | Source | Description | Notes |
|---|---|---|---|---|
| `product_id` | int | `dim_products.product_id` | Surrogate key | Primary key |
| `product_name` | string | `dim_products.product_name` | | |
| `sub_category` | string | `dim_products.sub_category` | | From ERP via Gold |
| `category` | string | `dim_products.category` | | From ERP via Gold |
| `product_line` | string | `dim_products.product_line` | | e.g. `Road`, `Mountain`, `N/A` |
| `product_start_date` | date | `dim_products.product_start_date` | Catalogue entry date | |
| `status` | string | Computed | Product lifecycle status | `'Active'` if `product_end_date IS NULL`, else `'Inactive'` |

### Sales activity dates

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `first_order` | date | `fact_sales.order_date` | Date of first sale | `MIN(order_date)` per product — NULL if never sold |
| `last_order` | date | `fact_sales.order_date` | Date of most recent sale | `MAX(order_date)` per product |
| `days_since_last_order` | int | Computed | Days between last sale and dataset reference date | `DATE_DIFF(max_date, MAX(order_date))` — 0 = last sold on reference date |
| `last_ship` | date | `fact_sales.ship_date` | Most recent ship date | `MAX(ship_date)` per product |
| `last_due` | date | `fact_sales.due_date` | Most recent due date | `MAX(due_date)` per product |

### Logistics metrics

All logistics averages are computed across all historical order lines for the product.

| Column | Type | Description | Calculation |
|---|---|---|---|
| `avg_ship_days` | double | Average days from order to shipment | `AVG(DATE_DIFF(ship_date, order_date))` |
| `avg_delivery_days` | double | Average days from shipment to delivery | `AVG(DATE_DIFF(due_date, ship_date))` |
| `avg_total_preparation_days` | double | Average total lead time (order to delivery) | `AVG(DATE_DIFF(due_date, order_date))` = `avg_ship_days + avg_delivery_days` |

### Financial performance

| Column | Type | Source | Description | Calculation |
|---|---|---|---|---|
| `price` | int | `fact_sales.price` | Observed selling price | `MIN(price)` across all order lines |
| `product_cost` | int | `dim_products.product_cost` | Unit production cost | `MAX(product_cost)` — cost is stable per product |
| `total_sales` | bigint | `fact_sales.sales` | Total revenue generated | `SUM(sales)` |
| `total_quantity` | bigint | `fact_sales.quantity` | Total units sold | `SUM(quantity)` |
| `total_profit` | bigint | Computed | Total gross profit | `SUM((sales - product_cost) * quantity)` |
| `total_orders` | bigint | `fact_sales.order_number` | Number of distinct orders | `COUNT(DISTINCT order_number)` |
| `avg_order_value` | double | Computed | Average revenue per order | `ROUND(total_sales / NULLIF(total_orders, 0), 2)` |

### RFM scores

Product RFM scores are quintiles (NTILE 5) partitioned to separate products with no sales history (score = 0) from products with sales (scored 1–5).

| Column | Type | Description | Calculation | Score 5 means |
|---|---|---|---|---|
| `recency_score` | int | Recency score | `NTILE(5) OVER(PARTITION BY (days_since_last_order >= 0) ORDER BY days_since_last_order DESC)` | Most days since last sale (least recently active) |
| `frequency_score` | int | Frequency score | `NTILE(5) OVER(PARTITION BY (total_orders > 0) ORDER BY total_orders ASC)` | Most orders |
| `monetary_score` | int | Monetary score | `NTILE(5) OVER(PARTITION BY (total_profit > 0) ORDER BY total_profit ASC)` | Highest total profit |
| `rfm_score` | double | Composite RFM score | `ROUND((recency_score + frequency_score + monetary_score) / 3, 2)` | — |

> **Note on recency direction:** Unlike `customer_360`, `recency_score` for products orders by `days_since_last_order DESC` — a score of 5 means the product has been inactive the longest (most stale). Products with no sales (`first_order IS NULL`) receive score 0, not in the quintile distribution.

### Revenue share & ABC classification

| Column | Type | Description | Calculation |
|---|---|---|---|
| `revenue_share` | double | Product's share of total portfolio revenue (%) | `ROUND(total_sales / SUM(total_sales) OVER () * 100, 2)` — expressed as percentage (e.g. `2.34` = 2.34%) |
| `abc_class` | string | Pareto ABC classification | See table below |

**ABC classification thresholds** — based on cumulative revenue share ordered by `total_sales DESC`:

| Class | Cumulative revenue | Interpretation |
|---|---|---|
| `A` | ≤ 80% | Top revenue drivers — prioritize stock, marketing, availability |
| `B` | 81–95% | Moderate contributors — monitor, optimize |
| `C` | > 95% | Tail products — low revenue, review for discontinuation |

Computed via:
```sql
SUM(total_sales) OVER (ORDER BY total_sales DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
/ SUM(total_sales) OVER ()
AS cumulative_revenue_share
```

---

## Query examples

```sql
-- ABC class summary
SELECT abc_class,
       COUNT(*)                          AS nb_products,
       ROUND(SUM(revenue_share), 1)      AS pct_revenue
FROM data_lakehouse_project.platinium.product_360
GROUP BY abc_class
ORDER BY abc_class;

-- Top 10 most profitable products
SELECT product_name, category, total_profit, total_quantity, abc_class
FROM data_lakehouse_project.platinium.product_360
ORDER BY total_profit DESC
LIMIT 10;

-- Products inactive > 180 days (potential discontinuation candidates)
SELECT product_name, sub_category, last_order, days_since_last_order, status
FROM data_lakehouse_project.platinium.product_360
WHERE days_since_last_order > 180
ORDER BY days_since_last_order DESC;
```
