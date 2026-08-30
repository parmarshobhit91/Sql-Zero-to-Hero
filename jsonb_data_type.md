# PostgreSQL JSONB — Data Engineering Notes

**Topic:** PostgreSQL JSONB and Semi-Structured Data  
**Category:** Data Engineering / PostgreSQL  
**Level:** Beginner → Intermediate

## Table of Contents

1. [What is JSONB?](#1-what-is-jsonb)
2. [Why JSONB is Useful in Data Engineering](#2-why-jsonb-is-useful-in-data-engineering)
3. [Creating a Table with JSONB](#3-creating-a-table-with-jsonb)
4. [Inserting JSONB Data](#4-inserting-jsonb-data)
5. [Querying JSONB Data](#5-querying-jsonb-data)
6. [Important JSONB Operators](#6-important-jsonb-operators)
7. [Filtering JSONB Data](#7-filtering-jsonb-data)
8. [Checking Whether a Key Exists](#8-checking-whether-a-key-exists)
9. [Practical Data Engineering Example](#9-practical-data-engineering-example)
10. [Schema Flexibility](#10-schema-flexibility)
11. [Hybrid Data Modeling](#11-hybrid-data-modeling)
12. [When Should You Use JSONB?](#12-when-should-you-use-jsonb)
13. [When Should You Avoid JSONB?](#13-when-should-you-avoid-jsonb)
14. [Common Mistakes](#14-common-mistakes)
15. [Key Takeaways](#15-key-takeaways)
16. [Interview Questions](#16-interview-questions)
17. [Final Mental Model](#final-mental-model)

---

## 1. What is JSONB?

JSONB is a PostgreSQL data type used to store JSON (JavaScript Object Notation) data in a binary representation.

It is useful when a database needs to store semi-structured or flexible data.

For example:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1",
  "promo_code": "SUMMER2026",
  "tags": ["mobile", "app"]
}
```

Instead of creating separate columns for:

- `browser`
- `ip`
- `promo_code`
- `tags`

we can store these attributes inside a single JSONB column called `metadata`.

### Simple Definition

**JSONB = PostgreSQL's binary JSON data type used for storing and querying semi-structured data.**

---

## 2. Why JSONB is Useful in Data Engineering

In real-world Data Engineering systems, data does not always have a fixed structure.

Data may come from:

- APIs
- Event streams
- Application logs
- Microservices
- External source systems
- Different versions of applications
- Different event types

For example, one application event might contain:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1"
}
```

Another event might contain:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1",
  "device": "mobile",
  "app_version": "5.2.1"
}
```

The second event contains additional attributes.

With a traditional relational schema, we may need to modify the table structure whenever new attributes are introduced.

With JSONB, these additional attributes can be stored inside the same `metadata` column.

This provides **schema flexibility**.

---

## 3. Creating a Table with JSONB

Consider the following `orders` table:

```sql
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(12,2),
    order_date DATE,
    created_at TIMESTAMP,
    metadata JSONB
);
```

The table contains both structured and semi-structured data.

### Structured columns

- `order_id`
- `customer_id`
- `amount`
- `order_date`
- `created_at`

### Semi-structured column

- `metadata`

The `metadata` column uses the PostgreSQL `JSONB` data type.

---

## 4. Inserting JSONB Data

Let's insert an order:

```sql
INSERT INTO orders (
    order_id,
    customer_id,
    amount,
    metadata
)
VALUES (
    1001,
    55,
    149.99,
    '{"browser": "Chrome", "ip": "192.168.1.1", "promo_code": "SUMMER2026", "tags": ["mobile", "app"]}'
);
```

The inserted metadata looks like:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1",
  "promo_code": "SUMMER2026",
  "tags": ["mobile", "app"]
}
```

The relational part of the record is:

```text
order_id    = 1001
customer_id = 55
amount      = 149.99
```

The additional metadata is stored inside:

```text
metadata
```

---

## 5. Querying JSONB Data

PostgreSQL provides special operators for working with JSON and JSONB.

The two operators we will use most frequently are:

| Operator | Purpose |
|---|---|
| `->` | Extract JSON object/value |
| `->>` | Extract JSON value as text |

### 5.1 Extracting a JSON Value Using `->>`

Suppose we want to extract the browser name.

```sql
SELECT metadata ->> 'browser' AS browser_name
FROM orders;
```

Example output:

```text
browser_name
------------
Chrome
```

The expression:

```sql
metadata ->> 'browser'
```

means:

> Get the value associated with the `browser` key and return it as text.

### 5.2 Extracting Multiple JSON Fields

We can extract multiple values from the same JSONB object.

```sql
SELECT
    order_id,
    metadata ->> 'browser' AS browser_name,
    metadata ->> 'ip' AS ip_address,
    metadata ->> 'promo_code' AS promo_code
FROM orders;
```

Example output:

```text
order_id | browser_name | ip_address   | promo_code
---------+--------------+--------------+-------------
1001     | Chrome       | 192.168.1.1  | SUMMER2026
```

---

## 6. Important JSONB Operators

PostgreSQL provides several operators for working with JSONB.

| Operator | Description |
|---|---|
| `->` | Extracts a JSON object/value |
| `->>` | Extracts a JSON value as text |
| `#>` | Extracts a JSON object/value at a specified path |
| `#>>` | Extracts a value at a specified path as text |
| `?` | Checks whether a key exists |
| `?\|` | Checks whether any specified keys exist |
| `?&` | Checks whether all specified keys exist |
| `@>` | Checks whether the left JSONB value contains the right JSONB value |
| `<@` | Checks whether the left JSONB value is contained by the right JSONB value |

For basic Data Engineering work, the most important operators to remember initially are:

- `->`
- `->>`
- `?`
- `@>`

---

## 7. Filtering JSONB Data

JSONB values can be used directly inside a `WHERE` clause.

### 7.1 Filter by an Exact JSON Value

Suppose we want orders using the `SUMMER2026` promo code.

```sql
SELECT *
FROM orders
WHERE metadata ->> 'promo_code' = 'SUMMER2026';
```

The expression:

```sql
metadata ->> 'promo_code'
```

extracts the value from the JSONB object as text.

Then:

```sql
= 'SUMMER2026'
```

compares the extracted value.

### 7.2 Filtering Using LIKE

We can also use `LIKE` with values extracted from JSONB.

Consider this example:

```sql
SELECT *
FROM orders_new
WHERE metadata ->> 'promo_code' LIKE '%sale';
```

Here:

```sql
metadata ->> 'promo_code'
```

extracts the `promo_code`.

Then:

```sql
LIKE '%sale'
```

checks whether the extracted text ends with `sale`.

For example:

```text
Independence_sale
Diwali_sale
NewYear_sale
```

match the condition.

However:

```text
Rakshabandhan_sales
```

does not match because the value ends with `sales`, not `sale`.

---

## 8. Checking Whether a Key Exists

Sometimes we do not care about the value.

We only want to know whether a particular key exists.

PostgreSQL provides the `?` operator for this.

```sql
SELECT *
FROM orders
WHERE metadata ? 'promo_code';
```

This query finds orders where the JSONB object contains the key:

```text
promo_code
```

For example:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1",
  "promo_code": "SUMMER2026"
}
```

contains the `promo_code` key, so it matches the condition.

### 8.1 JSONB Containment Using `@>`

The `@>` operator can be used to check whether one JSONB value contains another JSONB value.

For example:

```sql
SELECT *
FROM orders
WHERE metadata @> '{"browser": "Chrome"}';
```

This finds rows where the metadata JSONB object contains:

```json
{
  "browser": "Chrome"
}
```

This is called a **JSONB containment check**.

---

## 9. Practical Data Engineering Example

Let's create a more realistic table.

```sql
CREATE TABLE orders_new (
    order_id BIGINT,
    item VARCHAR(100),
    customer_id INT,
    metadata JSONB
);
```

Now insert multiple records:

```sql
INSERT INTO orders_new
VALUES
(
    101,
    'Mouse',
    11,
    '{"browser": "Chrome", "ip": "192.164.1.1", "promo_code": "Rakshabandhan_sales", "tags": ["mobile", "app"]}'
),
(
    102,
    'Keyboard',
    12,
    '{"browser": "Firefox", "ip": "192.164.1.2", "promo_code": "Independence_sale", "tags": ["desktop", "web"]}'
),
(
    103,
    'Laptop',
    13,
    '{"browser": "Safari", "ip": "192.164.1.3", "promo_code": "Diwali_sale", "tags": ["mobile", "app"]}'
),
(
    104,
    'Headphones',
    14,
    '{"browser": "Edge", "ip": "192.164.1.4", "promo_code": "NewYear_sale", "tags": ["audio", "app"]}'
),
(
    105,
    'Monitor',
    15,
    '{"browser": "Chrome", "ip": "192.164.1.5", "promo_code": "Festive_offer", "tags": ["desktop", "web"]}'
),
(
    106,
    'Webcam',
    16,
    '{"browser": "Firefox", "ip": "192.164.1.6", "promo_code": "Rakshabandhan_sales", "tags": ["video", "app"]}'
);
```

### 9.1 Extract Browser and Promo Code

```sql
SELECT
    item,
    metadata ->> 'browser' AS browser_name,
    metadata ->> 'promo_code' AS promo_code
FROM orders_new;
```

Example output:

```text
item        | browser_name | promo_code
------------+--------------+----------------------
Mouse       | Chrome       | Rakshabandhan_sales
Keyboard    | Firefox      | Independence_sale
Laptop      | Safari       | Diwali_sale
Headphones  | Edge         | NewYear_sale
Monitor     | Chrome       | Festive_offer
Webcam      | Firefox      | Rakshabandhan_sales
```

### 9.2 Find Orders with Sale Promo Codes

```sql
SELECT *
FROM orders_new
WHERE metadata ->> 'promo_code' LIKE '%sale';
```

This query demonstrates an important pattern:

```text
JSONB field
     ↓
Extract value using ->>
     ↓
Treat it as text
     ↓
Apply normal SQL filtering
```

---

## 10. Schema Flexibility

One of the biggest reasons to use JSONB in Data Engineering is **schema flexibility**.

Imagine an application initially sends:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1"
}
```

Later, the application starts sending:

```json
{
  "browser": "Chrome",
  "ip": "192.168.1.1",
  "device": "mobile",
  "app_version": "5.2.1"
}
```

Later, another source may send:

```json
{
  "browser": "Firefox",
  "ip": "192.168.1.2",
  "device": "desktop",
  "country": "India",
  "language": "English"
}
```

The metadata structure can vary between records.

With a traditional relational design, every new attribute could potentially require a schema change such as:

```sql
ALTER TABLE orders
ADD COLUMN device VARCHAR(50);
```

and later:

```sql
ALTER TABLE orders
ADD COLUMN app_version VARCHAR(50);
```

This can become difficult to manage when metadata changes frequently.

With JSONB, these variable attributes can remain inside:

```text
metadata
```

without requiring an `ALTER TABLE` every time a new field is introduced.

---

## 11. Hybrid Data Modeling

A common Data Engineering design is to combine:

**Relational columns + JSONB metadata**

This is called **hybrid modeling**.

For example:

```sql
CREATE TABLE orders (
    order_id BIGINT,
    customer_id BIGINT,
    amount DECIMAL(12,2),
    order_date DATE,
    created_at TIMESTAMP,
    metadata JSONB
);
```

### Relational columns

- `order_id`
- `customer_id`
- `amount`
- `order_date`
- `created_at`

These represent important, structured business data.

They can be used for:

- Constraints
- Joins
- Aggregations
- Filtering
- Reporting
- Referential relationships

### JSONB metadata

- `browser`
- `ip`
- `promo_code`
- `tags`
- `device`
- `app_version`

These represent additional or less-standardized attributes.

They may change depending on:

- Source system
- API version
- Event type
- Application version
- Data producer

### Hybrid Modeling Concept

A useful mental model is:

```text
                    ORDERS TABLE
                         |
              +----------+----------+
              |                     |
       Structured Data        Semi-Structured Data
              |                     |
       Relational Columns          JSONB
              |                     |
       order_id                    browser
       customer_id                 ip
       amount                      promo_code
       order_date                  tags
       created_at                  device
                                   app_version
```

This gives the database both:

- Strong structure where structure matters
- Flexibility where the data is variable

---

## 12. When Should You Use JSONB?

JSONB is a good choice when the data:

- Is semi-structured.
- Has attributes that change frequently.
- Comes from APIs or external systems.
- Contains event-specific metadata.
- Has many optional attributes.
- Contains long-tail attributes that are not queried frequently.
- Needs to preserve source-system metadata.
- Does not justify creating a dedicated relational column for every attribute.

### Common Data Engineering Use Cases

JSONB can be useful for:

- API ingestion
- Event tracking
- Application logs
- User activity metadata
- IoT/event metadata
- Source-system payloads
- Audit metadata
- Application configuration
- Raw/Bronze-layer ingestion

---

## 13. When Should You Avoid JSONB?

JSONB provides flexibility, but it should not automatically replace relational modeling.

If a field is:

- Frequently queried
- Frequently filtered
- Frequently joined
- Important for business logic
- Required to have strong constraints
- Used heavily in aggregations

then a dedicated relational column may be a better choice.

For example, if `customer_id` is heavily used for joins, it is generally better modeled as:

```sql
customer_id BIGINT
```

rather than:

```json
{
  "customer_id": 55
}
```

Similarly, an important metric such as:

```text
amount
```

is generally better represented as:

```sql
amount DECIMAL(12,2)
```

rather than being stored inside JSONB.

### Practical Rule

> **Use relational columns for important, stable, frequently queried business attributes. Use JSONB for flexible or less-standardized metadata.**

---

## 14. Common Mistakes

### Mistake 1: Using Double Quotes for SQL Strings

Incorrect:

```sql
INSERT INTO orders_new
VALUES (102, "Keyboard", 12, ...);
```

In PostgreSQL, double quotes are generally used for identifiers.

Use single quotes for string values:

```sql
INSERT INTO orders_new
VALUES (102, 'Keyboard', 12, ...);
```

### Mistake 2: Invalid JSON

JSON must use valid JSON syntax.

Incorrect:

```json
{
  "browser": "Chrome",
  "promo_code": "-----
}
```

Correct:

```json
{
  "browser": "Chrome",
  "promo_code": "Rakshabandhan_sales"
}
```

### Mistake 3: Confusing `->` and `->>`

#### `->`

Returns JSON/JSONB.

```sql
metadata -> 'tags'
```

#### `->>`

Returns text.

```sql
metadata ->> 'browser'
```

This distinction becomes important when filtering and comparing values.

### Mistake 4: Putting Everything into JSONB

JSONB should not be treated as a replacement for relational modeling.

Avoid putting core business fields such as:

- `order_id`
- `customer_id`
- `amount`
- `order_date`

into JSONB just because JSONB is flexible.

Use JSONB where flexibility provides an actual benefit.

---

## 15. Key Takeaways

### Core Concept

JSONB allows PostgreSQL to store and query semi-structured JSON data efficiently.

### Important Operators

```text
->    Extract JSON
->>   Extract text
?     Check whether a key exists
@>    Check JSONB containment
```

### Most Common Pattern

```sql
metadata ->> 'key'
```

Example:

```sql
SELECT metadata ->> 'browser'
FROM orders;
```

### Filtering Pattern

```sql
SELECT *
FROM orders
WHERE metadata ->> 'promo_code' = 'SUMMER2026';
```

### Pattern Matching

```sql
SELECT *
FROM orders
WHERE metadata ->> 'promo_code' LIKE '%sale';
```

### Key Existence

```sql
SELECT *
FROM orders
WHERE metadata ? 'promo_code';
```

### Containment

```sql
SELECT *
FROM orders
WHERE metadata @> '{"browser": "Chrome"}';
```

---

## 16. Interview Questions

### Q1. What is JSONB in PostgreSQL?

JSONB is PostgreSQL's binary JSON data type. It is used to store and query semi-structured data while providing flexibility in the structure of the stored attributes.

### Q2. What is the difference between JSON and JSONB?

At a high level:

- **JSON** stores the original JSON text representation.
- **JSONB** stores JSON in a decomposed binary representation that PostgreSQL can process efficiently.

For most applications that need to query JSON data, JSONB is generally preferred.

### Q3. What is the difference between `->` and `->>`?

`->` returns a JSON/JSONB value.

```sql
metadata -> 'browser'
```

`->>` returns the value as text.

```sql
metadata ->> 'browser'
```

### Q4. How do you filter a JSONB field?

Use `->>` to extract the value and then apply a normal SQL condition.

```sql
SELECT *
FROM orders
WHERE metadata ->> 'promo_code' = 'SUMMER2026';
```

### Q5. How do you check whether a JSONB key exists?

Use the `?` operator:

```sql
SELECT *
FROM orders
WHERE metadata ? 'promo_code';
```

### Q6. Why is JSONB useful in Data Engineering?

JSONB is useful when dealing with semi-structured and frequently changing data, such as:

- API payloads
- Event data
- Application metadata
- Logs
- Data from multiple source systems

It provides schema flexibility without requiring a relational schema change for every new metadata attribute.

### Q7. Should everything be stored as JSONB?

No.

A good Data Engineering design often uses hybrid modeling:

```text
Stable + important + frequently queried data
                ↓
        Relational columns

Flexible + optional + source-specific metadata
                ↓
              JSONB
```

---

# Final Mental Model

Think of a PostgreSQL table like this:

```text
┌──────────────────────────────────────────────────────┐
│                     ORDERS                            │
├──────────┬─────────────┬──────────┬──────────────────┤
│ order_id │ customer_id │ amount   │ metadata         │
├──────────┼─────────────┼──────────┼──────────────────┤
│ 1001     │ 55          │ 149.99   │ {                │
│          │             │          │   "browser":     │
│          │             │          │   "Chrome",      │
│          │             │          │   "ip": "...",   │
│          │             │          │   "promo_code":  │
│          │             │          │   "SUMMER2026",  │
│          │             │          │   "tags": [...]  │
│          │             │          │ }                │
└──────────┴─────────────┴──────────┴──────────────────┘
```

The core business data stays structured and relational.

The changing or additional information stays inside JSONB.

That combination gives Data Engineers a practical balance between:

```text
Consistency
    +
Performance
    +
Schema Flexibility
```

## Key Principle

> **Don't choose between relational modeling and JSONB blindly. Use each where it provides the most value.**
