# SQL Performance — Cardinality & Aggregation Pushdown

**Topic:** Cardinality and its cost impact, and aggregation pushdown as a query optimization
**Category:** Data Engineering / SQL
**Level:** Intermediate → Advanced

## Table of Contents

1. [What Is Cardinality?](#1-what-is-cardinality)
2. [Why High Cardinality Is Expensive](#2-why-high-cardinality-is-expensive)
3. [Aggregation Pushdown in SQL](#3-aggregation-pushdown-in-sql)
4. [Example: Orders and Customers](#4-example-orders-and-customers)
5. [Why Is This Faster?](#5-why-is-this-faster)
6. [Important Caveat](#6-important-caveat)
7. [Partial Aggregation](#7-partial-aggregation)
8. [Simple Mental Model](#8-simple-mental-model)

---

## 1. What Is Cardinality?

**Cardinality** is the number of distinct (unique) values in a column, or the number of rows in a table/result set.

- **Low cardinality:** a column with few distinct values relative to the number of rows. Example: a `status` column with only `'Pending'` / `'Delivered'` / `'Cancelled'` across millions of rows — 3 distinct values no matter how large the table gets.
- **High cardinality:** a column with many distinct values relative to the number of rows — often close to one distinct value per row. Example: `user_id`, `order_id`, `email`, or a UUID column.

Cardinality also applies to entire result sets and to the intermediate output of a query step. A `GROUP BY customer_id` on a table with 100 million orders and 1 million distinct customers produces a **high-cardinality** grouping (1 million output groups) compared to `GROUP BY status`, which might produce only 3 groups (**low cardinality**) regardless of how many rows go in.

---

## 2. Why High Cardinality Is Expensive

The more distinct values/groups/rows an operation has to track, the more work the database has to do to keep them organized. This shows up in several concrete costs:

- **More memory** — the database has to keep track of every distinct group (or value) it has seen so far, e.g. a hash table entry per distinct `customer_id`. A `GROUP BY` over a low-cardinality column needs only a handful of buckets in memory; a high-cardinality `GROUP BY` may need millions of buckets, which can force data out of memory and onto disk.
- **More CPU** — computing and comparing more distinct keys means more hashing operations, more comparisons, and more work building/maintaining data structures (hash tables, trees) as cardinality grows.
- **More sorting/hashing** — many execution strategies for `GROUP BY`, `DISTINCT`, and joins rely on sorting or hashing the data by key first, so that matching/duplicate values end up next to each other or in the same bucket. The cost of sorting or hashing scales with the number of distinct keys — a low-cardinality sort/hash is cheap and fast; a high-cardinality one is much heavier.
- **More network shuffle** — in a distributed query engine (Spark, BigQuery, Snowflake, etc.), rows with the same group-by key or join key often need to be moved to the same worker node so they can be combined. High cardinality means many more distinct keys spread across the cluster, which means far more data movement ("shuffle") between nodes compared to a low-cardinality key that condenses naturally into a handful of groups.
- **More intermediate state** — every stage of a query plan that hasn't finished aggregating yet has to hold its partial results somewhere. High-cardinality aggregations (or high-cardinality joins, like the fan-out case) keep much larger intermediate result sets alive throughout the query, increasing the amount of temporary state the engine has to manage before it can produce the final, smaller output.

> **Rule of thumb:** cardinality is a good predictor of how "heavy" an operation will be. Aggregating, joining, or sorting on a low-cardinality column is cheap because the data naturally collapses into a small number of groups. Doing the same on a high-cardinality column (like a unique ID) means the database is doing nearly as much work as if it weren't grouping at all — which is exactly why techniques like aggregation pushdown (below) try to reduce cardinality as early as possible, before expensive operations like joins.

---

## 3. Aggregation Pushdown in SQL

Aggregation pushdown is a query-optimization technique where the database performs an aggregation (`GROUP BY`, `SUM`, `COUNT`, `AVG`, etc.) as early as possible, before expensive operations such as joins.

The main goal is to reduce the amount of data that needs to be processed.

---

## 4. Example: Orders and Customers

Suppose we have:

```sql
orders (
    order_id,
    customer_id,
    amount
)

customers (
    customer_id,
    customer_name
)
```

A query might ask: *Find the total amount spent by each customer.*

### A straightforward query

```sql
SELECT
    c.customer_name,
    SUM(o.amount) AS total_spent
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_name;
```

Conceptually, the database could do:

```text
Orders
   │
   ▼
JOIN Customers
   │
   ▼
GROUP BY customer
   │
   ▼
SUM(amount)
```

If `orders` contains 100 million rows, the join may have to process a huge amount of data.

### With aggregation pushdown

We can aggregate `orders` before joining:

```sql
SELECT
    c.customer_name,
    o.total_spent
FROM customers c
JOIN (
    SELECT
        customer_id,
        SUM(amount) AS total_spent
    FROM orders
    GROUP BY customer_id
) o
    ON o.customer_id = c.customer_id;
```

Now the conceptual execution becomes:

```text
100 million Orders
        │
        ▼
GROUP BY customer_id
        │
        ▼
maybe 1 million rows
        │
        ▼
JOIN Customers
        │
        ▼
Final result
```

Instead of joining 100 million order rows, the database may only need to join the aggregated customer totals.

---

## 5. Why Is This Faster?

Imagine:

```text
orders:       100,000,000 rows
customers:      1,000,000 rows
```

After:

```sql
GROUP BY customer_id
```

the orders might become:

```text
1,000,000 rows
```

So the later join processes roughly:

```text
1 million rows
```

instead of:

```text
100 million rows
```

That's the key idea:

> **Aggregate early → produce fewer rows → perform expensive operations on less data.**

---

## 6. Important Caveat

You cannot always push an aggregation below a join. For example, this can change the result:

```sql
SELECT
    c.customer_name,
    COUNT(*)
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id
GROUP BY c.customer_name;
```

Here, `COUNT(*)` is counting *joined rows*, not pre-aggregated order totals — pushing an aggregation below this join without adjusting the logic would silently change what's being counted (similar in spirit to the fan-out join problem, where aggregating at the wrong stage relative to a join gives the wrong number).

Whether aggregation can safely be pushed down depends on things such as:

- join type (`INNER`, `LEFT`, etc.)
- join cardinality
- grouping columns
- whether the aggregation is distributive/algebraic
- filters and conditions applied to the joined tables

Also, modern databases such as PostgreSQL, SQL Server, Oracle, and distributed query engines may automatically perform aggregation pushdown in their query optimizer. You don't necessarily have to rewrite the SQL manually.

---

## 7. Partial Aggregation

**Partial aggregation** (also called **pre-aggregation** or **map-side aggregation**) is closely related to aggregation pushdown, but applies specifically to **distributed** query engines (Spark, Trino/Presto, BigQuery, Snowflake, etc.), where data lives on many different worker nodes.

### The problem it solves

In a distributed engine, computing `GROUP BY customer_id` normally requires rows with the same `customer_id` to end up on the **same worker node**, so they can be combined into one group. Moving rows between nodes like this is called a **shuffle**, and — as covered in the cardinality section above — shuffling is expensive, especially at high cardinality.

If every raw row is shuffled across the network before any aggregation happens, the engine ends up moving the *entire* dataset over the network just to group it.

### How partial aggregation helps

Instead of shuffling raw rows, each worker node first aggregates the rows **it already has locally** — computing a partial `SUM`/`COUNT`/etc. per key, using only the data on that node. Only these much smaller partial results are then shuffled across the network and combined into the final, fully aggregated result.

```text
Without partial aggregation:

Node A rows ──┐
Node B rows ──┼─► Shuffle ALL raw rows ─► Aggregate ─► Final result
Node C rows ──┘


With partial aggregation:

Node A rows ─► local aggregate ─┐
Node B rows ─► local aggregate ─┼─► Shuffle partial sums ─► Merge ─► Final result
Node C rows ─► local aggregate ─┘
```

### Example

Say `orders` is spread across 3 worker nodes, and we run:

```sql
SELECT
    customer_id,
    SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

If customer `C1` has rows on all three nodes:

- **Without partial aggregation:** every individual `C1` row from all 3 nodes is shuffled to one node, which then sums them all — the network moves as many rows as `C1` has, no matter how many.
- **With partial aggregation:** each node first computes its own local `SUM(amount)` for `C1` (say `Node A → 300`, `Node B → 150`, `Node C → 200`). Only these **3 partial sums** are shuffled across the network, and the receiving node just adds `300 + 150 + 200 = 650`. However many raw rows `C1` had, only one partial number per node crosses the network.

### Why this works: distributive/algebraic aggregates

Partial aggregation only works cleanly for aggregates that can be computed in stages and then combined — these are called **distributive** or **algebraic** aggregates:

- `SUM`, `COUNT`, `MIN`, `MAX` — straightforward: partial sums can be summed, partial counts can be summed, partial mins/maxes can be re-compared.
- `AVG` — not directly distributive on its own (you can't average partial averages), but it *is* algebraic: each node computes a partial `SUM` and partial `COUNT`, both of which combine cleanly, and the final `AVG` is computed as `total_sum / total_count` at the end.
- `COUNT(DISTINCT ...)` — the hard case. A raw distinct count generally *can't* be split into simple partial results and merged, because the same distinct value could appear on multiple nodes, and naively summing partial distinct counts would double-count it. This is part of why exact `COUNT(DISTINCT)` is so expensive at scale (as noted earlier), and why approximate techniques like `APPROX_COUNT_DISTINCT` / HyperLogLog exist — they use mergeable summary structures specifically so this kind of value *can* be partially aggregated across nodes.

### How this relates to aggregation pushdown

Aggregation pushdown (moving a `GROUP BY` before a `JOIN`) and partial aggregation (aggregating locally before a network shuffle) are the same underlying idea applied at different layers:

| Technique | Expensive operation it protects | What it reduces |
|-----------|----------------------------------|-------------------|
| Aggregation pushdown | `JOIN` | Rows flowing into the join |
| Partial aggregation | Network shuffle | Rows flowing across the network |

Both follow the same principle: **aggregate as early as possible, so every downstream step processes fewer rows.** Most modern query optimizers apply partial aggregation automatically whenever the aggregate function allows it — it's rarely something you need to write into the SQL yourself, but understanding it explains why `GROUP BY`-heavy queries on distributed engines don't cost nearly as much network traffic as the raw row count might suggest.

---

## 8. Simple Mental Model

**Without pushdown:**

```text
Scan → Join → Aggregate
```

**With pushdown:**

```text
Scan → Aggregate → Join
```

The second approach is often better because the join receives fewer rows.