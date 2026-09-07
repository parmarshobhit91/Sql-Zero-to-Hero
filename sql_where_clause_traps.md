# SQL WHERE Clause: Operator Precedence & NULL Traps

**Topic:** AND/OR precedence, IN/NOT IN, and NULL pitfalls in WHERE
**Category:** Data Engineering / SQL
**Level:** Intermediate

## Table of Contents

1. [Operator Precedence: AND vs OR](#1-operator-precedence-and-vs-or)
2. [IN, NOT IN, and NOT vs `<>`](#2-in-not-in-and-not-vs-)
3. [The WHERE Clause Requires a Clear TRUE](#3-the-where-clause-requires-a-clear-true)
4. [The Dangerous Bug: A NULL Inside NOT IN](#4-the-dangerous-bug-a-null-inside-not-in)

---

## 1. Operator Precedence: AND vs OR

`AND` has a higher precedence than `OR`, so it is evaluated first — even if `OR` appears earlier in the query.

```sql
SELECT *
FROM orders
WHERE item = 'Mousepad'
   OR item = 'Keyboard'
  AND amount > 300;
```

Because `AND` binds tighter, SQL reads this as:

```sql
WHERE item = 'Mousepad'
   OR (item = 'Keyboard' AND amount > 300);
```

**Output:**

| order_id | item | amount | customer_id |
|----------|------|--------|--------------|
| 1 | Keyboard | 400 | 4 |
| 4 | Keyboard | 400 | 1 |
| 5 | Mousepad | 250 | 2 |

This explains the result: every `Mousepad` row qualifies on its own (regardless of amount), while `Keyboard` rows only qualify when `amount > 300`. That's why the Mousepad row with `amount = 250` is included — the `amount > 300` condition never applies to it.

> **Best practice:** Use parentheses to make intent explicit, e.g. `WHERE item = 'Mousepad' OR (item = 'Keyboard' AND amount > 300)`, so the logic doesn't depend on the reader remembering precedence rules.

---

## 2. IN, NOT IN, and NOT vs `<>`

`IN` checks whether a value matches any item in a list. It's shorthand for chaining multiple `OR` conditions.

```sql
SELECT * FROM customers
WHERE country = 'USA';

SELECT * FROM customers
WHERE country IN ('USA', 'UK');
```

### Negating a condition

There are two equivalent ways to negate equality:

```sql
SELECT * FROM customers
WHERE NOT country = 'USA';

SELECT * FROM customers
WHERE country <> 'USA';
```

And two equivalent ways to negate `IN`:

```sql
SELECT * FROM customers
WHERE NOT country IN ('USA', 'UK');

SELECT * FROM customers
WHERE country NOT IN ('USA', 'UK');
```

---

## 3. The WHERE Clause Requires a Clear TRUE

A `WHERE` condition only keeps a row if it evaluates to `TRUE`. If it evaluates to `FALSE` **or** `UNKNOWN`, the row is discarded.

This matters because comparisons involving `NULL` never evaluate to `TRUE` or `FALSE` — they evaluate to `UNKNOWN`.

### Classic production bug: empty string vs. NULL

```sql
SELECT *
FROM customers
WHERE country NOT IN ('USA');
```

**Customers table:**

| customer_id | first_name | last_name | age | country | salary |
|-------------|------------|-----------|-----|---------|--------|
| 1 | John | Doe | 31 | USA | 12000 |
| 2 | Robert | Luna | 22 | USA | 12000 |
| 3 | David | Robinson | 22 | UK | 12000 |
| 4 | John | Reinhardt | 25 | UK | 12000 |
| 5 | Betty | Doe | 28 | UAE | 12000 |
| 6 | Shobhit | Parmar | 23 | *(empty string)* | 15000 |
| 7 | Kinshu | Parmar | 18 | *(NULL)* | 1000 |

**Output:**

| customer_id | first_name | last_name | age | country | salary |
|-------------|------------|-----------|-----|---------|--------|
| 3 | David | Robinson | 22 | UK | 12000 |
| 4 | John | Reinhardt | 25 | UK | 12000 |
| 5 | Betty | Doe | 28 | UAE | 12000 |
| 6 | Shobhit | Parmar | 23 | *(empty string)* | 15000 |

Kinshu (row 7) is missing, but Shobhit (row 6) is included — even though both look "blank" in the table. The difference:

- **Shobhit's `country`** is an **empty string** `''`. An empty string is still a defined value, so `'' <> 'USA'` evaluates to `TRUE` → row is kept.
- **Kinshu's `country`** is **`NULL`**, meaning unknown/missing. `NULL <> 'USA'` evaluates to `UNKNOWN`, not `TRUE` → row is discarded.

**Concept:** `WHERE` requires a clear `TRUE` to include a row. If the result is `FALSE` or `UNKNOWN`, the row is dropped — and any comparison against `NULL` (other than `IS NULL` / `IS NOT NULL`) always evaluates to `UNKNOWN`.

---

## 4. The Dangerous Bug: A NULL Inside NOT IN

```sql
SELECT *
FROM customers
WHERE country NOT IN ('USA', NULL);
```

This silently returns **zero rows** from the entire table — a well-known SQL trap.

### Why this happens

Internally, `NOT IN (a, b)` is evaluated as a chain of `AND`-ed inequalities:

```sql
WHERE country <> 'USA' AND country <> NULL
```

Test this against a row where `country = 'UK'`:

```
'UK' <> 'USA'  → TRUE
'UK' <> NULL   → UNKNOWN
TRUE AND UNKNOWN → UNKNOWN
```

Since the combined result is `UNKNOWN` (not `TRUE`), the row is excluded. This happens for **every row**, regardless of its `country` value, because `country <> NULL` is always `UNKNOWN` — so the whole table returns 0 rows.

### The fix: use NOT EXISTS

```sql
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM excluded_countries e
    WHERE e.country = c.country
);
```

`NOT EXISTS` compares rows directly rather than expanding into a chain of `<> NULL` comparisons, so a stray `NULL` in the exclusion list can't silently poison the entire result set.

> **Rule of thumb:** Before using `NOT IN` with a subquery or a list that might contain `NULL`, either filter out `NULL`s explicitly (`WHERE column IS NOT NULL`) or use `NOT EXISTS` instead.