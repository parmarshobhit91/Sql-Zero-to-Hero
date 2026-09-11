# SQL Aggregation Pitfalls — Fan-Out Joins, GROUPING SETS, ROLLUP & CUBE

**Topic:** Join-induced double counting, and multi-level aggregation with GROUPING SETS / ROLLUP / CUBE
**Category:** Data Engineering / SQL
**Level:** Intermediate → Advanced

## Table of Contents

1. [The Fan-Out Join Problem](#1-the-fan-out-join-problem)
2. [Sample Table: sales](#2-sample-table-sales)
3. [GROUPING SETS](#3-grouping-sets)
4. [ROLLUP](#4-rollup)
5. [CUBE](#5-cube)
6. [GROUPING SETS vs ROLLUP vs CUBE](#6-grouping-sets-vs-rollup-vs-cube)

---

## 1. The Fan-Out Join Problem

### Sample data

**Promotions**

| product_id | promotion |
|------------|-----------|
| p1 | summer sale |
| p1 | member discount |
| p2 | festival |

**Sales**

| order_id | product_id | amount |
|----------|------------|--------|
| 101 | p1 | 100 |
| 101 | p2 | 200 |
| 102 | p1 | 150 |

### The buggy query

```sql
SELECT *
FROM sales s
JOIN promotions p
    ON s.product_id = p.product_id;
```

### The problem

`sales` has 3 rows, but `promotions` has **two** rows for `p1` (`summer sale` and `member discount`). Because the join key `product_id` isn't unique on the `promotions` side, every `sales` row for `p1` gets duplicated once per matching promotion — this is called a **fan-out** or **row explosion**.

The join produces **5 rows** instead of 3:

| order_id | product_id | amount | promotion |
|----------|------------|--------|------------|
| 101 | p1 | 100 | summer sale |
| 101 | p1 | 100 | member discount |
| 101 | p2 | 200 | festival |
| 102 | p1 | 150 | summer sale |
| 102 | p1 | 150 | member discount |

If you now run `SUM(amount)` on this joined result, `amount = 100` and `amount = 150` are each counted **twice** (once per promotion row), so:

```
SUM(amount) on joined table = 700   ❌ wrong
Actual SUM(amount) in sales = 450   ✅ correct (100 + 200 + 150)
```

**Root cause:** the join duplicated `sales` rows *before* aggregation, so the duplicated `amount` values got summed as if they were separate orders.

### The fix: aggregate before joining to a one-to-many table

```sql
SELECT
    product_id,
    SUM(amount) AS revenue
FROM sales
GROUP BY product_id;
```

By aggregating `sales` on its own first — before it ever touches the one-to-many `promotions` table — each `amount` is counted exactly once. If promotion details are still needed afterward, join the *pre-aggregated* result to `promotions` rather than joining raw, unaggregated fact rows to a table that can multiply them.

> **General rule:** Whenever you join a fact table to a table that can have multiple matching rows per key (one-to-many or many-to-many), aggregate the fact table first, or be prepared for `SUM`/`COUNT`/`AVG` to be inflated by however many matches each row picks up.

---

## 2. Sample Table: sales

```sql
CREATE TABLE sales (
    id INT PRIMARY KEY,
    region VARCHAR(50),
    product VARCHAR(50),
    amount DECIMAL(10, 2)
);

INSERT INTO sales (id, region, product, amount)
VALUES
    (1, 'North', 'Laptop', 1000),
    (2, 'North', 'Phone',  500),
    (3, 'South', 'Laptop', 1200),
    (4, 'South', 'Phone',  700);
```

Instead of running `GROUP BY region` and `GROUP BY product` as two separate queries, `GROUPING SETS`, `ROLLUP`, and `CUBE` let you compute multiple grouping levels — including subtotals and a grand total — in a **single query and single pass over the data**.

---

## 3. GROUPING SETS

`GROUPING SETS` lets you explicitly list which combinations of columns to group by, including an empty set `()` for the grand total.

```sql
SELECT
    region,
    product,
    SUM(amount) AS revenue
FROM sales
GROUP BY GROUPING SETS (
    (region),
    (product),
    ()          -- grand total
);
```

### Output

| region | product | revenue |
|--------|---------|---------|
| North  | NULL    | 1500 |
| South  | NULL    | 1900 |
| NULL   | Laptop  | 2200 |
| NULL   | Phone   | 1200 |
| NULL   | NULL    | 3400 |

Each row corresponds to exactly one of the requested grouping sets: totals by `region` alone, totals by `product` alone, and one grand-total row where both columns are `NULL` (since neither was grouped on for that row). `GROUPING SETS` gives you full control — only the combinations you list appear, nothing more.

---

## 4. ROLLUP

`ROLLUP(region, product)` builds a **hierarchy** of subtotals based on the column order: it starts from the full `(region, product)` detail, then progressively "rolls up" by dropping the rightmost column, ending at the grand total.

```sql
SELECT
    region,
    product,
    SUM(amount) AS revenue
FROM sales
GROUP BY ROLLUP (region, product);
```

This is equivalent to:

```sql
GROUP BY GROUPING SETS (
    (region, product),
    (region),
    ()
)
```

### Output

| region | product | revenue |
|--------|---------|---------|
| North  | Laptop  | 1000 |
| North  | Phone   | 500 |
| North  | NULL    | 1500 |
| South  | Laptop  | 1200 |
| South  | Phone   | 700 |
| South  | NULL    | 1900 |
| NULL   | NULL    | 3400 |

Notice there is **no** `(product)`-only row (e.g. no row summing all `Laptop` revenue across regions) — `ROLLUP` only rolls up along the hierarchy from left to right (`region, product` → `region` → grand total), unlike `GROUPING SETS`, which can produce any arbitrary combination you ask for.

> **Typical use case:** subtotal reports with a natural hierarchy, e.g. `ROLLUP(year, month, day)` for year → year+month → year+month+day totals.

---

## 5. CUBE

`CUBE(region, product)` generates **every possible combination** of the given columns — it's the full cross-product of subtotals, not just one hierarchy.

```sql
SELECT
    region,
    product,
    SUM(amount) AS revenue
FROM sales
GROUP BY CUBE (region, product);
```

This is equivalent to:

```sql
GROUP BY GROUPING SETS (
    (region, product),
    (region),
    (product),
    ()
)
```

### Output

| region | product | revenue |
|--------|---------|---------|
| North  | Laptop  | 1000 |
| North  | Phone   | 500 |
| South  | Laptop  | 1200 |
| South  | Phone   | 700 |
| North  | NULL    | 1500 |
| South  | NULL    | 1900 |
| NULL   | Laptop  | 2200 |
| NULL   | Phone   | 1200 |
| NULL   | NULL    | 3400 |

`CUBE` includes everything `ROLLUP` has, **plus** the `(product)`-only subtotals (`Laptop = 2200`, `Phone = 1200`) that `ROLLUP` skips. With 2 columns, `CUBE` produces 2² = 4 grouping levels; with *n* columns it produces 2ⁿ, so it grows fast and should be used carefully on wide grouping lists.

---

## 6. GROUPING SETS vs ROLLUP vs CUBE

| Feature | Grouping levels produced | Use case |
|---------|---------------------------|----------|
| `GROUPING SETS` | Exactly the combinations you list | Full manual control over which subtotals appear |
| `ROLLUP(a, b, ...)` | Hierarchical: `(a,b,...)` → `(a,...)` → ... → `()` | Natural hierarchies (year→month→day, country→city) |
| `CUBE(a, b, ...)` | All 2ⁿ possible combinations | Every subtotal cross-section, e.g. for pivot-style reports |

> **Tip:** In all three cases, `NULL` in the result can mean either "the actual value was NULL" or "this column wasn't part of this grouping level." Most databases provide a `GROUPING()` function to distinguish the two, e.g. `GROUPING(region) = 1` means the `NULL` in that row is a subtotal marker, not a real `NULL` value.