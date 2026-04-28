# Bike_Data_Lakehouse_Project
Complete Data Lakehouse from scratch using Databricks and the Medallion Architecture.

SQL Queries for a potentiel future Business Sales Tables
```sql
WITH prd_price AS (
  SELECT 
    product_key,
    price / quantity AS price,
    ROW_NUMBER() OVER(PARTITION BY product_key ORDER BY quantity ASC, order_date DESC) AS latest_price
FROM data_lakehouse_project.silver.crm_sales
)

LEFT JOIN prd_price AS pp
  ON pp.product_key = prd.product_key
  AND pp.latest_price = 1
```