# SQL Data Types, Structs, Casting, NULL & Three-Valued Logic

## Table of Contents

1. [STRUCT / Composite Data](#1-struct--composite-data)
2. [Inserting STRUCT Data](#2-inserting-struct-data)
3. [Querying STRUCT Data](#3-querying-struct-data)
4. [STRUCT vs PostgreSQL Composite Types](#4-struct-vs-postgresql-composite-types)
5. [SQL Casting](#5-sql-casting)
6. [PostgreSQL Casting Shorthand](#6-postgresql-casting-shorthand)
7. [TRY_CAST and SAFE_CAST](#7-try_cast-and-safe_cast)
8. [Casting in Production](#8-casting-in-production)
9. [NULL in SQL](#9-null-in-sql)
10. [Three-Valued Logic](#10-three-valued-logic)
11. [NULL and Comparisons](#11-null-and-comparisons)
12. [COALESCE](#12-coalesce)
13. [COALESCE in PostgreSQL](#13-coalesce-in-postgresql)
14. [Production Best Practices](#14-production-best-practices)
15. [Quick Reference](#15-quick-reference)

---

# 1. STRUCT / Composite Data

Some SQL database systems support a **STRUCT** data type, which allows multiple related fields to be stored together as a nested object.

For example, a customer can have:

* `customer_id`
* `name`
* `address`

  * `city`
  * `country`

Conceptually:

```text
Customer
├── customer_id
├── name
└── address
    ├── city
    └── country
```

A STRUCT is useful when data naturally contains nested attributes.

---

## Example

The following syntax is commonly seen in systems such as **Google BigQuery**:

```sql
CREATE TABLE customers (
    customer_id INTEGER,
    name STRING,
    address STRUCT<
        city STRING,
        country STRING
    >
);
```

The `address` column contains two nested fields:

```text
address.city
address.country
```

---

# 2. Inserting STRUCT Data

A STRUCT value can be inserted using a constructor appropriate to the database system.

For example, in BigQuery:

```sql
INSERT INTO customers
VALUES (
    1,
    'shobhit',
    STRUCT('MP', 'India')
);
```

The resulting logical record is:

```text
customer_id: 1
name: shobhit
address:
    city: MP
    country: India
```

> **Important:** SQL syntax differs between database systems. `STRING`, `STRUCT`, and STRUCT constructors are not portable SQL syntax.

---

# 3. Querying STRUCT Data

Nested fields can be accessed using dot notation.

```sql
SELECT
    name,
    address.city,
    address.country
FROM customers;
```

Result:

```text
name      address.city    address.country
--------  --------------  ---------------
shobhit   MP              India
```

The important concept is:

```sql
address.city
```

means:

> Access the `city` field inside the `address` STRUCT.

---

# 4. STRUCT vs PostgreSQL Composite Types

PostgreSQL does **not** use the BigQuery-style `STRUCT` type.

PostgreSQL has a related concept called a **composite type**.

For example:

```sql
CREATE TYPE address_type AS (
    city TEXT,
    country TEXT
);
```

Then:

```sql
CREATE TABLE customers (
    customer_id INTEGER,
    name TEXT,
    address address_type
);
```

You can insert a composite value like:

```sql
INSERT INTO customers
VALUES (
    1,
    'shobhit',
    ROW('MP', 'India')
);
```

You can access fields using PostgreSQL's composite-field syntax:

```sql
SELECT
    name,
    (address).city,
    (address).country
FROM customers;
```

### Important distinction

| Concept             | BigQuery       | PostgreSQL         |
| ------------------- | -------------- | ------------------ |
| Nested structure    | `STRUCT`       | Composite type     |
| String type         | `STRING`       | `TEXT` / `VARCHAR` |
| Struct constructor  | `STRUCT(...)`  | `ROW(...)`         |
| Nested field access | `address.city` | `(address).city`   |

For most relational application databases, however, deeply nested structures should be used carefully. Often a separate table is a better relational design.

---

# 5. SQL Casting

**Casting** means converting a value from one data type to another.

General SQL syntax:

```sql
CAST(expression AS data_type)
```

For example:

```sql
SELECT CAST('123' AS INTEGER);
```

Result:

```text
123
```

---

## Casting String to Integer

```sql
SELECT CAST('123' AS INTEGER);
```

Result:

```text
123
```

---

## Casting String to Decimal

```sql
SELECT CAST('100.50' AS DECIMAL(10, 2));
```

Result:

```text
100.50
```

`DECIMAL(10, 2)` means:

```text
10 = total number of digits
2  = digits after decimal point
```

Therefore:

```text
100.50
```

contains:

```text
3 digits before decimal
2 digits after decimal
```

---

## Casting String to Date

```sql
SELECT CAST('2026-08-28' AS DATE);
```

Result:

```text
2026-08-28
```

---

# 6. PostgreSQL Casting Shorthand

PostgreSQL provides another convenient casting syntax using `::`.

Instead of:

```sql
SELECT CAST('123' AS INTEGER);
```

you can write:

```sql
SELECT '123'::INTEGER;
```

Both perform the same conversion.

---

## Examples

### String → Integer

```sql
SELECT '123'::INTEGER;
```

### String → Numeric

```sql
SELECT '100.50'::NUMERIC(10, 2);
```

### String → Date

```sql
SELECT '2026-08-28'::DATE;
```

### Integer → Text

```sql
SELECT 123::TEXT;
```

### Timestamp → Date

```sql
SELECT NOW()::DATE;
```

The `::` syntax is **PostgreSQL-specific shorthand** and should not be assumed to work across all SQL databases.

---

# 7. TRY_CAST and SAFE_CAST

Normal `CAST()` can fail when the input cannot be converted.

For example:

```sql
SELECT CAST('hello' AS INTEGER);
```

This produces a conversion error because:

```text
hello
```

is not a valid integer.

In production data pipelines, invalid or dirty data is common.

For this reason, some SQL engines provide safer casting functions.

---

## TRY_CAST

Some SQL systems support:

```sql
TRY_CAST(expression AS data_type)
```

Instead of throwing an error for an invalid conversion, `TRY_CAST` generally returns:

```text
NULL
```

Example:

```sql
SELECT TRY_CAST('123' AS INTEGER);
```

Result:

```text
123
```

Invalid value:

```sql
SELECT TRY_CAST('hello' AS INTEGER);
```

Result:

```text
NULL
```

This is extremely useful when processing external or unreliable data.

> **Important:** `TRY_CAST` is not universally available with identical behavior in every database. Always check the SQL engine you are using.

---

# 8. SAFE_CAST

Some SQL engines provide:

```sql
SAFE_CAST(expression AS data_type)
```

A common example is **BigQuery**.

Example:

```sql
SELECT SAFE_CAST('123' AS INT64);
```

Result:

```text
123
```

Invalid conversion:

```sql
SELECT SAFE_CAST('hello' AS INT64);
```

Result:

```text
NULL
```

The key idea is:

```text
CAST
  ↓
Invalid input → Error

SAFE_CAST / TRY_CAST
  ↓
Invalid input → NULL
```

---

## TRY_CAST vs SAFE_CAST

| Function      | Commonly associated with                 | Invalid conversion |
| ------------- | ---------------------------------------- | ------------------ |
| `CAST()`      | Standard SQL                             | Error              |
| `TRY_CAST()`  | SQL Server, Snowflake, DuckDB and others | Usually `NULL`     |
| `SAFE_CAST()` | BigQuery                                 | `NULL`             |
| `::`          | PostgreSQL                               | Error              |

These functions are **database-specific**, so don't assume that a function available in one database exists in another.

---

# 9. Casting in Production

Casting becomes particularly important in:

* Data Engineering
* ETL pipelines
* ELT pipelines
* Data Warehouses
* Data Lakes
* API ingestion
* CSV ingestion
* Log processing
* Streaming pipelines

External data frequently arrives as strings.

For example:

```text
customer_id = "123"
age         = "25"
salary      = "75000.50"
hire_date   = "2026-08-28"
```

But the target database may require:

```text
customer_id → INTEGER
age         → INTEGER
salary      → NUMERIC
hire_date   → DATE
```

A transformation layer can perform these conversions.

---

## Example

```sql
SELECT
    CAST(customer_id AS INTEGER) AS customer_id,
    CAST(age AS INTEGER) AS age,
    CAST(salary AS NUMERIC(12, 2)) AS salary,
    CAST(hire_date AS DATE) AS hire_date
FROM raw_customers;
```

For dirty data, a safe conversion can be preferable:

```sql
SELECT
    TRY_CAST(customer_id AS INTEGER) AS customer_id,
    TRY_CAST(age AS INTEGER) AS age,
    TRY_CAST(salary AS NUMERIC(12, 2)) AS salary,
    TRY_CAST(hire_date AS DATE) AS hire_date
FROM raw_customers;
```

Invalid values can then become `NULL` instead of stopping the entire transformation.

---

## Production Pattern

A common data-engineering pattern is:

```text
Raw Data
   ↓
Validation
   ↓
Safe Casting
   ↓
Clean Data
   ↓
Production Tables
```

For example:

```text
"123"       → 123
"100.50"    → 100.50
"2026-08-28" → 2026-08-28
"unknown"   → NULL
```

The invalid records can then be monitored or sent to a quarantine/error table.

---

# 10. NULL in SQL

`NULL` represents **missing, unknown, or unavailable information**.

It does not mean:

```text
0
```

It does not mean:

```text
''
```

It does not necessarily mean:

```text
FALSE
```

It represents the absence of a known value.

---

## Example

```sql
CREATE TABLE customers (
    customer_id INTEGER,
    name TEXT,
    age INTEGER
);
```

Insert:

```sql
INSERT INTO customers
VALUES
    (1, 'Alice', 25),
    (2, 'Bob', NULL);
```

Here:

```text
Alice → age = 25
Bob   → age = NULL
```

For Bob, the age is unknown or missing.

---

# 11. Three-Valued Logic

One of the most important concepts in SQL is that SQL does not use only:

```text
TRUE
FALSE
```

SQL uses **three-valued logic**:

```text
TRUE
FALSE
UNKNOWN
```

`NULL` is responsible for the `UNKNOWN` result in many expressions.

---

## Example

Suppose:

```text
age = NULL
```

Now execute:

```sql
SELECT age = 25;
```

The result is not:

```text
FALSE
```

It is:

```text
UNKNOWN
```

Why?

Because SQL does not know whether the missing age is actually 25.

---

## Three-Valued Logic Example

Consider:

```sql
NULL = 10
```

Result:

```text
UNKNOWN
```

Similarly:

```sql
NULL > 10
```

Result:

```text
UNKNOWN
```

And:

```sql
NULL <> 10
```

Result:

```text
UNKNOWN
```

---

# 12. NULL and Comparisons

A very common mistake is:

```sql
SELECT *
FROM customers
WHERE age = NULL;
```

This does **not** correctly find NULL values.

The expression:

```sql
age = NULL
```

evaluates to `UNKNOWN`.

Use:

```sql
WHERE age IS NULL
```

instead.

---

## Finding Non-NULL Values

Use:

```sql
WHERE age IS NOT NULL;
```

Example:

```sql
SELECT *
FROM customers
WHERE age IS NULL;
```

```sql
SELECT *
FROM customers
WHERE age IS NOT NULL;
```

---

---
## NULLIF
Returns NULL if a = b
```text
NULLIF(a,b)
```

example:
```text
NULLIF(revenue, 0)
```
---

# 13. NULL and WHERE

A `WHERE` clause keeps rows only when its condition evaluates to:

```text
TRUE
```

Rows where the condition evaluates to:

```text
FALSE
```

or:

```text
UNKNOWN
```

are not returned.

Example:

```sql
SELECT *
FROM customers
WHERE age > 18;
```

If:

```text
Alice → age = 25
Bob   → age = NULL
Charlie → age = 15
```

Then:

```text
Alice   → TRUE     → returned
Bob     → UNKNOWN  → not returned
Charlie → FALSE    → not returned
```

Therefore, `NULL` values can behave differently from ordinary values during filtering.

---

# 14. COALESCE

`COALESCE` is used to replace `NULL` with another value.

Syntax:

```sql
COALESCE(value, replacement)
```

Example:

```sql
SELECT COALESCE(age, 0)
FROM customers;
```

If:

```text
age = 25
```

result:

```text
25
```

If:

```text
age = NULL
```

result:

```text
0
```

---

## Multiple COALESCE Values

`COALESCE` can accept multiple arguments:

```sql
COALESCE(value1, value2, value3, ...)
```

It returns the first non-NULL value.

Example:

```sql
SELECT COALESCE(
    phone_number,
    email,
    'No contact information'
)
FROM customers;
```

Logic:

```text
phone_number exists
        ↓
     return phone

phone_number NULL
        ↓
     check email

email NULL
        ↓
return 'No contact information'
```

---

# 15. COALESCE in PostgreSQL

`COALESCE` is supported by PostgreSQL and is commonly used when working with nullable columns.

Example:

```sql
SELECT
    name,
    COALESCE(age, 0) AS age
FROM customers;
```

You can also use it in calculations.

Without `COALESCE`:

```sql
SELECT salary + bonus
FROM employees;
```

If `bonus` is `NULL`, the result can also become:

```text
NULL
```

Using:

```sql
SELECT salary + COALESCE(bonus, 0)
FROM employees;
```

means:

```text
NULL bonus → 0
```

So:

```text
salary + 0
```

can be calculated normally.

---

# 16. NULL, COALESCE and Aggregations

SQL aggregate functions have specific NULL behavior.

For example:

```sql
SELECT AVG(salary)
FROM employees;
```

`AVG()` generally ignores NULL values.

Suppose:

```text
salary
------
100
200
NULL
```

Then:

```sql
AVG(salary)
```

is:

```text
150
```

not:

```text
100
```

However, replacing NULL before aggregation changes the meaning:

```sql
SELECT AVG(COALESCE(salary, 0))
FROM employees;
```

Now the calculation includes the missing salary as zero.

This can produce a different result.

Therefore, don't blindly use `COALESCE(..., 0)` in analytical queries. Decide what NULL actually means in the business context.

---

# 17. NULL with AND and OR

Three-valued logic becomes especially important with:

```sql
AND
OR
NOT
```

## AND

| A       | B       | A AND B |
| ------- | ------- | ------- |
| TRUE    | TRUE    | TRUE    |
| TRUE    | FALSE   | FALSE   |
| TRUE    | UNKNOWN | UNKNOWN |
| FALSE   | UNKNOWN | FALSE   |
| UNKNOWN | UNKNOWN | UNKNOWN |

---

## OR

| A       | B       | A OR B  |
| ------- | ------- | ------- |
| TRUE    | FALSE   | TRUE    |
| TRUE    | UNKNOWN | TRUE    |
| FALSE   | FALSE   | FALSE   |
| FALSE   | UNKNOWN | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN |

---

## NOT

```text
NOT TRUE    → FALSE
NOT FALSE   → TRUE
NOT UNKNOWN → UNKNOWN
```

This is why SQL's NULL behavior can sometimes appear surprising if you think only in terms of TRUE/FALSE logic.

---

# 18. PostgreSQL NULL-Specific Operators

PostgreSQL provides additional useful operators.

## IS DISTINCT FROM

Normal comparison:

```sql
NULL = NULL
```

produces:

```text
UNKNOWN
```

PostgreSQL provides:

```sql
NULL IS NOT DISTINCT FROM NULL
```

which evaluates to:

```text
TRUE
```

`IS DISTINCT FROM` treats NULL as a comparable value.

Example:

```sql
SELECT *
FROM customers
WHERE age IS DISTINCT FROM 25;
```

This is useful when you need NULL-safe comparisons.

---

# 19. Production Best Practices

## 19.1 Don't Treat NULL as Zero Automatically

This:

```sql
COALESCE(balance, 0)
```

is appropriate only when a missing balance logically means zero.

Otherwise, it can hide data-quality problems.

---

## 19.2 Use IS NULL

Correct:

```sql
WHERE email IS NULL;
```

Incorrect:

```sql
WHERE email = NULL;
```

---

## 19.3 Use Safe Casting for Untrusted Data

When ingesting external data, consider the safe-casting capabilities of your database:

```sql
TRY_CAST(...)
```

or:

```sql
SAFE_CAST(...)
```

when supported.

This prevents malformed records from unnecessarily breaking an entire transformation.

---

## 19.4 Validate NULLs Before Production

For important columns:

```sql
customer_id
email
created_at
```

consider enforcing:

```sql
NOT NULL
```

Example:

```sql
CREATE TABLE customers (
    customer_id BIGINT PRIMARY KEY,
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

This moves data-quality enforcement into the database.

---

## 19.5 Don't Hide Data Quality Problems

This:

```sql
COALESCE(value, 0)
```

can be useful for presentation and calculations.

But during data-quality processing, you may want to preserve the original `NULL` and identify why it is missing.

A good pipeline can separate:

```text
Raw value
    ↓
Validation
    ↓
Casting
    ↓
NULL handling
    ↓
Business rules
    ↓
Final table
```

---

# 20. Quick Reference

## STRUCT

```sql
-- BigQuery-style example
STRUCT<
    city STRING,
    country STRING
>
```

Access nested fields:

```sql
address.city
```

PostgreSQL equivalent concept:

```sql
CREATE TYPE address_type AS (
    city TEXT,
    country TEXT
);
```

---

## CAST

Standard syntax:

```sql
CAST('123' AS INTEGER);
```

PostgreSQL shorthand:

```sql
'123'::INTEGER;
```

---

## TRY_CAST

Where supported:

```sql
TRY_CAST('123' AS INTEGER);
```

Invalid value:

```sql
TRY_CAST('hello' AS INTEGER);
```

Typically returns:

```text
NULL
```

---

## SAFE_CAST

BigQuery:

```sql
SAFE_CAST('123' AS INT64);
```

Invalid value:

```sql
SAFE_CAST('hello' AS INT64);
```

Returns:

```text
NULL
```

---

## NULL

Check NULL:

```sql
WHERE column IS NULL;
```

Check non-NULL:

```sql
WHERE column IS NOT NULL;
```

Do not use:

```sql
WHERE column = NULL;
```

---

## COALESCE

Replace NULL:

```sql
COALESCE(column, default_value)
```

Example:

```sql
COALESCE(age, 0)
```

Multiple fallback values:

```sql
COALESCE(phone, email, 'No contact')
```

---

# 21. Mental Model

Keep these concepts in mind:

```text
STRUCT
  ↓
Nested / grouped data

CAST
  ↓
Convert one data type to another

TRY_CAST / SAFE_CAST
  ↓
Convert safely
Invalid conversion → NULL

NULL
  ↓
Missing / unknown value

Three-Valued Logic
  ↓
TRUE
FALSE
UNKNOWN

COALESCE
  ↓
First non-NULL value
```

The most important production lesson is:

```text
Raw Data
   ↓
Validate
   ↓
Cast
   ↓
Handle NULL carefully
   ↓
Apply business rules
   ↓
Store clean typed data
```

SQL data types, casting, and NULL handling are not just syntax topics. They directly affect **data quality, correctness, ETL reliability, query behavior, and production database design**.
