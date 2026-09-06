# SQL Basics — Data Engineering Notes

**Topic:** SQL Fundamentals
**Category:** Data Engineering / SQL
**Level:** Beginner → Intermediate

## Table of Contents

1. [Basic SELECT](#1-basic-select)
2. [ALTER TABLE and UPDATE](#2-alter-table-and-update)
3. [Comparison Operators](#3-comparison-operators)
4. [NULL and IS NULL](#4-null-and-is-null)
5. [GROUP BY](#5-group-by)
6. [HAVING](#6-having)
7. [WHERE vs HAVING](#7-where-vs-having)
8. [SQL Query Structure](#8-sql-query-structure)
9. [SQL Logical Execution Order](#9-sql-logical-execution-order)
10. [Subqueries](#10-subqueries)
11. [Aggregate Aliases and WHERE](#11-aggregate-aliases-and-where)
12. [ORDER BY](#12-order-by)
13. [DISTINCT](#13-distinct)
14. [SQL Clause Summary](#14-sql-clause-summary)
15. [Key Takeaways](#15-key-takeaways)

---

## 1. Basic SELECT

The basic structure of a SQL query is:

```sql
SELECT -- what do I want? (columns)
FROM   -- where does it come from? (table)
```

Example:

```sql
SELECT name, salary
FROM employees;
```

### Calculated Columns

```sql
SELECT name, salary * 12 AS annual_salary
FROM employees;
```

`salary * 12` represents annual salary, so `annual_salary` is a more appropriate alias than `avg_salary`.

---

## 2. ALTER TABLE and UPDATE

### Add a New Column

```sql
ALTER TABLE customers
ADD COLUMN avg_salary DECIMAL(10,2);
```

This adds a new `avg_salary` column to the `customers` table.

### Update the Column

```sql
UPDATE customers
SET avg_salary = (SELECT AVG(salary) FROM customers);
```

This calculates the overall average salary and assigns the same value to `avg_salary` for every row.

---

## 3. Comparison Operators

SQL provides comparison operators for filtering data.

### Equal To `=`

```sql
SELECT *
FROM customers
WHERE salary = 12000;
```

### Not Equal To `<>`

```sql
SELECT *
FROM customers
WHERE salary <> 12000;
```

### Greater Than `>`

```sql
SELECT *
FROM customers
WHERE salary > 12000;
```

### Less Than `<`

```sql
SELECT *
FROM customers
WHERE salary < 12000;
```

### Greater Than or Equal To `>=`

```sql
SELECT *
FROM customers
WHERE salary >= 12000;
```

### Less Than or Equal To `<=`

```sql
SELECT *
FROM customers
WHERE salary <= 12000;
```

---

## 4. NULL and IS NULL

`NULL` represents a missing or unknown value.

### IS NULL

```sql
SELECT *
FROM customers
WHERE salary IS NULL;
```

### IS NOT NULL

```sql
SELECT *
FROM customers
WHERE salary IS NOT NULL;
```

### IS vs =

`IS` is primarily used for checking `NULL`.

For normal value comparisons, use `=`:

```sql
SELECT *
FROM customers
WHERE salary = 12000;
```

Do not use:

```sql
SELECT *
FROM customers
WHERE salary IS 12000;
```

The important distinction is:

```
=           → compare values
IS NULL     → check for NULL
IS NOT NULL → check that value is not NULL
```

---

## 5. GROUP BY

`GROUP BY` groups rows based on one or more columns.

It is commonly used with aggregate functions such as:

- `COUNT()`
- `SUM()`
- `AVG()`
- `MIN()`
- `MAX()`

Example:

```sql
SELECT city, AVG(salary) AS average_salary
FROM customers
GROUP BY city;
```

This groups customers by city and calculates the average salary for each city.

---

## 6. HAVING

`HAVING` filters groups after `GROUP BY`.

Example:

```sql
SELECT city, AVG(salary) AS average_salary
FROM customers
GROUP BY city
HAVING AVG(salary) > 30000;
```

This returns only cities where the average salary is greater than 30000.

### WHERE + GROUP BY + HAVING

```sql
SELECT city, AVG(salary) AS average_salary
FROM customers
WHERE salary IS NOT NULL
GROUP BY city
HAVING AVG(salary) > 30000;
```

The query works in the following way:

1. `WHERE` removes rows where salary is `NULL`.
2. `GROUP BY` groups the remaining rows by city.
3. `AVG()` calculates the average salary for each group.
4. `HAVING` filters the groups.

---

## 7. WHERE vs HAVING

The main difference is:

```
WHERE  → filters rows before GROUP BY
HAVING → filters groups after GROUP BY
```

### WHERE

Use `WHERE` when filtering individual rows.

```sql
SELECT
    item,
    SUM(amount) AS total_amount
FROM orders
WHERE item = 'Keyboard'
GROUP BY item;
```

Here, `WHERE` filters the rows first, and then `GROUP BY` groups the remaining rows.

### HAVING

Use `HAVING` when filtering aggregated groups.

```sql
SELECT
    item,
    SUM(amount) AS total_amount
FROM orders
GROUP BY item
HAVING SUM(amount) > 400;
```

Here, the rows are grouped first, and then the groups are filtered based on the aggregate result.

### Key Difference

```
WHERE  → filter rows
HAVING → filter groups
```

---

## 8. SQL Query Structure

We write SQL clauses in this order:

```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

A general SQL query looks like:

```sql
SELECT columns
FROM table
WHERE row_condition
GROUP BY columns
HAVING group_condition
ORDER BY columns
LIMIT number;
```

Not every query needs every clause.

Example:

```sql
SELECT *
FROM customers
WHERE salary > 30000;
```

---

## 9. SQL Logical Execution Order

Although we write SQL like this:

```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

SQL logically processes the query approximately in this order:

```
FROM
WHERE
GROUP BY
HAVING
SELECT
ORDER BY
LIMIT
```

### Written Order

```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

### Logical Execution Order

```
FROM
WHERE
GROUP BY
HAVING
SELECT
ORDER BY
LIMIT
```

This difference is important when understanding SQL aliases, filtering, grouping, and aggregation.

---

## 10. Subqueries

A subquery is a query written inside another query.

Example:

```sql
SELECT *
FROM (
    SELECT
        item,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY item
) AS t
WHERE item = 'Keyboard';
```

The inner query:

```sql
SELECT
    item,
    SUM(amount) AS total_amount
FROM orders
GROUP BY item;
```

first creates a result containing the total amount for each item.

The outer query then filters that result:

```sql
WHERE item = 'Keyboard';
```

### Filtering Aggregate Results with a Subquery

```sql
SELECT *
FROM (
    SELECT
        item,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY item
) AS t
WHERE total_amount > 300;
```

The inner query calculates `total_amount`, and the outer query filters using that calculated column.

---

## 11. Aggregate Aliases and WHERE

Consider this query:

```sql
SELECT
    item,
    SUM(amount) AS total_amount
FROM orders
WHERE total_amount > 200
GROUP BY item;
```

This is generally invalid because `total_amount` is created in the `SELECT` clause, while `WHERE` is logically processed before `SELECT`.

### Use HAVING Instead

```sql
SELECT
    item,
    SUM(amount) AS total_amount
FROM orders
GROUP BY item
HAVING SUM(amount) > 200;
```

Or use a subquery:

```sql
SELECT *
FROM (
    SELECT
        item,
        SUM(amount) AS total_amount
    FROM orders
    GROUP BY item
) AS t
WHERE total_amount > 300;
```

### Key Idea

```
WHERE  → filters rows
HAVING → filters groups / aggregate results
```

---

## 12. ORDER BY

`ORDER BY` is used to sort query results.

Example:

```sql
SELECT *
FROM orders
ORDER BY item ASC, customer_id ASC;
```

The query first sorts by `item` in ascending order.

If multiple rows have the same item, then `customer_id` is used as the second sorting condition.

```
item         → first sorting condition
customer_id  → second sorting condition
```

### ASC and DESC

Ascending:

```sql
SELECT *
FROM orders
ORDER BY salary ASC;
```

Descending:

```sql
SELECT *
FROM orders
ORDER BY salary DESC;
```

---

## 13. DISTINCT

`DISTINCT` removes duplicate combinations from the selected columns.

Example:

```sql
SELECT DISTINCT department, job_title
FROM employees;
```

Suppose the data is:

| department | job_title |
|------------|-----------|
| IT         | Developer |
| IT         | Developer |
| HR         | Manager   |

The result will be:

| department | job_title |
|------------|-----------|
| IT         | Developer |
| HR         | Manager   |

`DISTINCT` removes duplicate combinations of:

```
department + job_title
```

It does not remove duplicates independently from each column.

---

## 14. SQL Clause Summary

| Clause     | Purpose                          |
|------------|-----------------------------------|
| `SELECT`   | Choose the columns to return      |
| `FROM`     | Choose the table/source           |
| `WHERE`    | Filter individual rows            |
| `GROUP BY` | Group rows                        |
| `HAVING`   | Filter groups                     |
| `ORDER BY` | Sort the result                   |
| `LIMIT`    | Limit the number of rows          |
| `DISTINCT` | Remove duplicate combinations     |

---

## 15. Key Takeaways

### Basic Query

```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

### Logical Execution

```
FROM
WHERE
GROUP BY
HAVING
SELECT
ORDER BY
LIMIT
```

### Important Concepts

- **SELECT** → What columns do I want?
- **FROM** → Where is the data?
- **WHERE** → Which rows do I want?
- **GROUP BY** → How should I group the rows?
- **HAVING** → Which groups should I keep?
- **ORDER BY** → How should I sort the result?
- **LIMIT** → How many rows should I return?
- **DISTINCT** → Remove duplicate combinations.
- `WHERE` filters rows.
- `HAVING` filters groups.
- `=` is used for normal value comparison.
- `IS NULL` and `IS NOT NULL` are used for NULL checks.
- SQL is written in one order but logically processed in another order.

### Final Mental Model

**Written Order:**

```
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

**Logical Execution Order:**

```
FROM
WHERE
GROUP BY
HAVING
SELECT
ORDER BY
LIMIT
```

> **Remember:** We write SQL in one order, but SQL logically processes it in another order.