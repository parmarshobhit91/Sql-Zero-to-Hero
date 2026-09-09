# SQL Aggregation — Conditional Aggregation, COUNT, and GROUP BY

**Topic:** Conditional aggregation, COUNT variants, GROUP BY (single & multiple dimensions)
**Category:** Data Engineering / SQL
**Level:** Intermediate

## Table of Contents

1. [Sample Tables](#1-sample-tables)
2. [Conditional Aggregation](#2-conditional-aggregation)
3. [COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)](#3-count-vs-countcolumn-vs-countdistinct-column)
4. [Aggregations with GROUP BY](#4-aggregations-with-group-by)
5. [GROUP BY: Multiple Dimensions](#5-group-by-multiple-dimensions)

---

## 1. Sample Tables

### Customers

| customer_id | first_name | last_name | age | country | salary |
|-------------|------------|-----------|-----|---------|--------|
| 1 | John | Doe | 31 | USA | 12000 |
| 2 | Robert | Luna | 22 | USA | 12000 |
| 3 | David | Robinson | 22 | UK | 12000 |
| 4 | John | Reinhardt | 25 | UK | 12000 |
| 5 | Betty | Doe | 28 | UAE | 12000 |
| 6 | Shobhit | Parmar | 23 | *(empty string)* | 15000 |
| 7 | Kinshu | Parmar | 18 | *(NULL)* | 1000 |

### Orders

| order_id | item | amount | customer_id |
|----------|------|--------|--------------|
| 1 | Keyboard | 400 | 4 |
| 2 | Mouse | 300 | 4 |
| 3 | Monitor | 12000 | 3 |
| 4 | Keyboard | 400 | 1 |
| 5 | Mousepad | 250 | 2 |

### Shippings

| shipping_id | status | customer |
|-------------|--------|----------|
| 1 | Pending | 2 |
| 2 | Pending | 4 |
| 3 | Delivered | 3 |
| 4 | Pending | 5 |
| 5 | Delivered | 1 |

---

## 2. Conditional Aggregation

Conditional aggregation combines `CASE` with an aggregate function (`SUM`, `COUNT`, etc.) to compute multiple metrics — split by condition — in a single pass over the data, instead of running separate queries per condition.

### Total revenue from delivered orders only

```sql
SELECT
    SUM(
        CASE
            WHEN s.status = 'Delivered' THEN o.amount
            ELSE 0
        END
    ) AS total_revenue
FROM orders o
INNER JOIN shippings s
    ON o.customer_id = s.customer;
```

Only rows where `status = 'Delivered'` contribute their `amount` to the sum; every other row contributes `0`.

### Counting orders by shipping status

```sql
SELECT
    SUM(
        CASE
            WHEN s.status = 'Delivered' THEN 1
            ELSE 0
        END
    ) AS completed_orders,
    SUM(
        CASE
            WHEN s.status = 'Pending' THEN 1
            ELSE 0
        END
    ) AS pending_orders
FROM shippings s;
```

This pattern — `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` — is the standard way to get a per-category count as a column instead of as separate grouped rows. It's often called a "pivot" or "spreadsheet-style" aggregation.

---

## 3. COUNT(*) vs COUNT(column) vs COUNT(DISTINCT column)

| Form | Behavior |
|------|----------|
| `COUNT(*)` | Counts **all rows**, including rows where any column is `NULL`. |
| `COUNT(column_name)` | Counts only rows where `column_name` is **NOT NULL**. |
| `COUNT(DISTINCT column_name)` | Counts **distinct, non-NULL** values of `column_name`. |

Using the `customers` table above (7 rows total; `country` is `NULL` for customer 7, and an empty string `''` for customer 6; distinct non-NULL country values are `USA`, `UK`, `UAE`, `''` — 4 distinct values):

### COUNT(*) — counts every row

```sql
SELECT COUNT(*) AS total_customers
FROM customers;
```

**Result:** `7` — every row is counted, regardless of any `NULL` values in any column.

### COUNT(column_name) — counts non-NULL values only

```sql
SELECT COUNT(country) AS customers_with_country
FROM customers;
```

**Result:** `6` — customer 7 (`country IS NULL`) is excluded. Customer 6 (`country = ''`) *is* counted, because an empty string is a non-NULL value.

### COUNT(DISTINCT column_name) — counts distinct non-NULL values

```sql
SELECT COUNT(DISTINCT country) AS distinct_countries
FROM customers;
```

**Result:** `4` — the distinct non-NULL values are `'USA'`, `'UK'`, `'UAE'`, and `''` (empty string counts as its own distinct value). The `NULL` for customer 7 is dropped entirely and does not count as a 5th "unknown" group.

> **Performance note:** `COUNT(DISTINCT column)` requires the database to identify every unique value before counting, which typically means sorting or hashing the full column. On small tables this is instant, but at large scale (e.g. a **1-billion-row** events or clickstream table), an exact `COUNT(DISTINCT user_id)` can be one of the most expensive operations in a query — it can't be answered from a simple running total the way `COUNT(*)` or `SUM()` can.
>
> For very large datasets, many warehouses offer an approximate alternative that trades a small amount of accuracy for a large speed/cost improvement, e.g.:
>
> ```sql
> -- BigQuery
> SELECT APPROX_COUNT_DISTINCT(user_id) AS approx_unique_users
> FROM events;
>
> -- Snowflake / Postgres (via HyperLogLog extension)
> SELECT APPROX_COUNT_DISTINCT(user_id) AS approx_unique_users
> FROM events;
> ```
>
> These use algorithms like **HyperLogLog** to estimate cardinality with a small, bounded error (often <2%), without scanning and deduplicating every value.

---

## 4. Aggregations with GROUP BY

```sql
SELECT
    customer_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spend,
    AVG(amount) AS avg_order_value,
    MIN(amount) AS min_order_value,
    MAX(amount) AS max_order_value
FROM orders
GROUP BY customer_id;
```

This produces one row per `customer_id`, with each aggregate function computed only over that customer's orders — e.g. `SUM(amount)` gives each customer's total spend, `COUNT(*)` gives their number of orders, and so on.

---

## 5. GROUP BY: Multiple Dimensions

```sql
SELECT
    country,
    city,
    product,
    SUM(amount) AS revenue
FROM sales
GROUP BY
    country,
    city,
    product;
```

When `GROUP BY` lists multiple columns, the result has **one row per unique combination** of those columns — not one row per distinct value of each column individually. So if the same `product` is sold in multiple `(country, city)` pairs, each pair gets its own row and its own `revenue` total, rather than being merged together.