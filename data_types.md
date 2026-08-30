# PostgreSQL Data Types
PostgreSQL provides a rich and powerful type system for storing structured, semi-structured, and unstructured data. Choosing the appropriate data type is important for data integrity, query performance, storage efficiency, and maintainability.

This tutorial covers the most commonly used PostgreSQL data types, their syntax, practical examples, and recommendations for production applications.

## Table of Contents

1. [Numeric Data Types](#1-numeric-data-types)
2. [Character Data Types](#2-character-data-types)
3. [Boolean Data Type](#3-boolean-data-type)
4. [Date and Time Data Types](#4-date-and-time-data-types)
5. [UUID](#5-uuid)
6. [JSON and JSONB](#6-json-and-jsonb)
7. [Array Data Types](#7-array-data-types)
8. [Binary Data](#8-binary-data)
9. [Enumerated Types](#9-enumerated-types)
10. [Network Address Types](#10-network-address-types)
11. [Full-Text Search Types](#11-full-text-search-types)
12. [Geometric Types](#12-geometric-types)
13. [XML](#13-xml)
14. [Bit String Types](#14-bit-string-types)
15. [Money Type](#15-money-type)
16. [Common Production Choices](#16-common-production-choices)
17. [Complete Example](#17-complete-example)
18. [Best Practices](#18-best-practices)
19. [Quick Reference](#19-quick-reference)

## 1. Numeric Data Types
## 2. Character Data Types
## 3. Boolean Data Type
## 4. Date and Time Data Types
## 5. UUID
## 6. JSON and JSONB
## 7. Array Data Types
## 8. Binary Data
## 9. Enumerated Types
## 10. Network Address Types
## 11. Full-Text Search Types
## 12. Geometric Types
## 13. XML
## 14. Bit String Types
## 15. Money Type
## 16. Common Production Choices
## 17. Complete Example
## 18. Best Practices
## 19. Quick Reference
## 1. Numeric Data Types
PostgreSQL provides several numeric types for storing integers and decimal values.

### Common Numeric Types
| Type | Storage | Use Case |
| --- | --- | --- |
| SMALLINT | 2 bytes | Small whole numbers |
| INTEGER / INT | 4 bytes | General-purpose integers |
| BIGINT | 8 bytes | Large whole numbers |
| NUMERIC / DECIMAL | Variable | Exact decimal values |
| REAL | 4 bytes | Single-precision floating point |
| DOUBLE PRECISION | 8 bytes | Double-precision floating point |
| SMALLSERIAL | 2 bytes | Auto-generated small integer |
| SERIAL | 4 bytes | Auto-generated integer |
| BIGSERIAL | 8 bytes | Auto-generated large integer |

### Integer Example
```sql
CREATE TABLE products (
id BIGINT,
stock_quantity INTEGER,
display_order SMALLINT
);
```

Use:

INTEGER for most normal integer values.
BIGINT when values may exceed the range of INTEGER.
SMALLINT when the value range is known to be small.
### Exact Decimal Values
For financial and other exact calculations, use NUMERIC or DECIMAL.

```sql
CREATE TABLE products (
id BIGINT,
price NUMERIC(10, 2)
);
```

NUMERIC(10,2) means:

10 = maximum total number of digits.
2 = number of digits after the decimal point.
For example:

99999999.99

### Floating-Point Types
REAL and DOUBLE PRECISION use floating-point representation and may introduce rounding differences.

```sql
latitude  DOUBLE PRECISION,
longitude DOUBLE PRECISION
```

For monetary values, prefer:

```sql
price NUMERIC(12, 2)
```

rather than:
price DOUBLE PRECISION

## 2. Character Data Types
PostgreSQL provides three commonly used character types:

CHAR(n) \
VARCHAR(n) \
TEXT 
### CHAR
CHAR(n) stores fixed-length strings.

```sql
country_code CHAR(2)
```

If the value is shorter than the defined length, PostgreSQL pads it with spaces.

It is useful for genuinely fixed-length values, but is less commonly required in application databases.

### VARCHAR
VARCHAR(n) stores variable-length strings with a maximum length.

```sql
username VARCHAR(50)
```

The value cannot exceed the specified length.

### TEXT
TEXT stores variable-length strings without an explicitly defined maximum length.

```sql
description TEXT
```

For most PostgreSQL applications, TEXT is an excellent default for general-purpose text.

### Recommendation
Do not automatically use VARCHAR(n) simply because other database systems commonly encourage it.

If the application does not require a database-level maximum length, this is often sufficient:

```sql
name TEXT
```

If a strict database-level limit is part of the data model, use:
```sql
username VARCHAR(50)
```

## 3. Boolean Data Type
PostgreSQL provides the BOOLEAN type for true/false values.

```sql
is_active BOOLEAN
```

Example:

```sql
CREATE TABLE users (
id BIGINT,
username TEXT,
is_active BOOLEAN
);
```

Insert values:

```sql
INSERT INTO users (id, username, is_active)
VALUES (1, 'alice', TRUE);
```

PostgreSQL also accepts several boolean representations, but using TRUE and FALSE explicitly makes SQL easier to read.

## 4. Date and Time Data Types
PostgreSQL provides several date and time types.

| Type | Description |
| --- | --- |
| DATE | Calendar date |
| TIME | Time of day |
| TIMESTAMP | Date and time without time zone |
| TIMESTAMPTZ | Timestamp with time-zone-aware semantics |
| INTERVAL | Duration of time |

### DATE
Use DATE when only the calendar date matters.

```sql
birth_date DATE
```

Example:

```sql
INSERT INTO users (birth_date)
VALUES ('1995-08-20');
```

### TIME
Use TIME when you only need a time of day.

```sql
opening_time TIME
```

### TIMESTAMP
TIMESTAMP stores date and time without time-zone information.
```sql
created_at TIMESTAMP
```

### TIMESTAMPTZ
TIMESTAMPTZ is commonly preferred for timestamps representing real-world points in time.

```sql
created_at TIMESTAMPTZ
```

Example:

```sql
created_at TIMESTAMPTZ DEFAULT NOW()
```

This is generally a good choice for:

created_at
updated_at
deleted_at
event timestamps
audit timestamps
### INTERVAL
INTERVAL represents a duration.

```sql
SELECT NOW() + INTERVAL '7 days';
```

Another example:

```sql
SELECT INTERVAL '2 hours 30 minutes';
```

### Production Recommendation
For application timestamps representing an actual moment in time, prefer:

### TIMESTAMPTZ

rather than storing local time as plain text.

## 5. UUID
UUID stands for Universally Unique Identifier.

PostgreSQL provides a native UUID type:
```sql
id UUID
```

Example:

```sql
CREATE TABLE users (
id UUID PRIMARY KEY,
name TEXT NOT NULL
);
```

UUIDs are commonly used as identifiers in distributed systems and APIs.

Example UUID:

550e8400-e29b-41d4-a716-446655440000

UUIDs can be generated using PostgreSQL-supported UUID generation functionality, depending on the PostgreSQL version and installed extensions.

For example, environments using the pgcrypto extension can use:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

```sql
CREATE TABLE users (
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
name TEXT NOT NULL
);
```

### UUID vs SERIAL/BIGSERIAL
Use BIGINT/identity columns when:

sequential numeric IDs are appropriate,
compact indexes are desirable,
IDs do not need to be generated independently across systems.
Use UUID when:

IDs are exposed externally,
multiple systems generate IDs,
distributed creation is important,
sequential IDs should not be easily guessable.
For new PostgreSQL applications, identity columns are generally preferable to the older SERIAL convenience syntax when you want auto-generated integer IDs.

Example:

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

## 6. JSON and JSONB
PostgreSQL supports both:

### JSON
### JSONB
### JSON
JSON stores JSON text while preserving its original textual representation.
```sql
metadata JSON
```

### JSONB
JSONB stores JSON in a decomposed binary representation that is generally more useful for querying and indexing.

```sql
metadata JSONB
```

Example:

```sql
CREATE TABLE users (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
name TEXT NOT NULL,
metadata JSONB
);
```

Insert JSON:

```sql
INSERT INTO users (name, metadata)
VALUES (
'Alice',
'{"role": "admin", "skills": ["PostgreSQL", "Docker"]}'
);
```

Query JSONB:

```sql
SELECT *
FROM users
WHERE metadata->>'role' = 'admin';
```

### JSONB Indexing
For frequently queried JSONB data, a GIN(Generalized Inverted Index) index can be useful:

```sql
CREATE INDEX idx_users_metadata
ON users
USING GIN (metadata);
```

### JSON vs JSONB
| Feature | JSON | JSONB |
| --- | --- | --- |
| Preserves input formatting | Yes | No |
| Binary representation | No | Yes |
| Efficient querying | Limited | Better |
| Indexing support | Limited | Strong |
| Typical application choice | Less common | Usually preferred |

### Production Recommendation
Prefer JSONB when you need to store and query semi-structured data.

However, do not use JSONB simply to avoid designing a relational schema. Frequently queried or relationally important data often belongs in normal columns and tables.

## 7. Array Data Types
PostgreSQL supports arrays of many data types.

Examples:

```sql
tags TEXT[]
scores INTEGER[]
```
Example table:

```sql
CREATE TABLE articles (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
title TEXT NOT NULL,
tags TEXT[]
);
```

Insert an array:

```sql
INSERT INTO articles (title, tags)
VALUES (
'PostgreSQL Basics',
ARRAY['postgresql', 'database', 'sql']
);
```

Query an array:

```sql
SELECT *
FROM articles
WHERE 'postgresql' = ANY(tags);
```

Arrays can be useful for naturally grouped values.

However, if you need complex relationships, filtering, joining, or independent management of each item, a separate relational table is often a better design.

## 8. Binary Data
PostgreSQL provides the BYTEA type for binary data.

```sql
file_data BYTEA
```

Example:

```sql
CREATE TABLE documents (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
file_name TEXT NOT NULL,
file_data BYTEA
);
```

BYTEA can store binary content such as:

encrypted data
small files
binary payloads
For large files, application architectures often store the object in dedicated object storage and keep only metadata or a storage key in PostgreSQL.

## 9. Enumerated Types
PostgreSQL supports custom enumerated types using ENUM.

Create an enum:

```sql
CREATE TYPE user_status AS ENUM (
'active',
'inactive',
'suspended'
);
```

Use it in a table:

```sql
CREATE TABLE users (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
username TEXT NOT NULL,
status user_status NOT NULL DEFAULT 'active'
);
```

Insert data:

```sql
INSERT INTO users (username, status)
VALUES ('alice', 'active');
```

### When to Use ENUM
ENUM can be appropriate when:

the set of values is small,
values change rarely,
the database itself should enforce the allowed values.
For frequently changing business states, a lookup/reference table or a TEXT column with a CHECK constraint may be easier to evolve.

Example:

```sql
status TEXT NOT NULL
```
CHECK (status IN ('active', 'inactive', 'suspended'))

## 10. Network Address Types
PostgreSQL provides specialized types for network addresses.

Common types include:

### INET
### CIDR
### MACADDR
### MACADDR8
### INET
INET stores IPv4 or IPv6 addresses.
```sql
ip_address INET
```

Example:

```sql
INSERT INTO access_logs (ip_address)
VALUES ('192.168.1.10');
```

Using INET is preferable to storing IP addresses as arbitrary strings when you need PostgreSQL's network-aware functionality.

## 11. Full-Text Search Types
PostgreSQL provides:

TSVECTOR
TSQUERY
These types support PostgreSQL's full-text search functionality.

Example:
```sql
search_vector TSVECTOR
```

A search vector can be generated from text:

```sql
SELECT to_tsvector(
'english',
'PostgreSQL is a powerful relational database'
);
```

A query can be created with:

```sql
SELECT plainto_tsquery(
'english',
'powerful database'
);
```

### Full-text search can be indexed for better performance.

Example:

```sql
CREATE INDEX idx_articles_search
ON articles
USING GIN (search_vector);
```

## 12. Geometric Types
PostgreSQL provides built-in geometric types, including:

POINT
LINE
LSEG
BOX
PATH
POLYGON
CIRCLE
Example:

```sql
location POINT
```

Insert a point:

```sql
INSERT INTO locations (location)
VALUES (POINT(10, 20));
```

These types are useful for certain geometric calculations. For advanced geographic and spatial workloads, PostgreSQL is also commonly used with the PostGIS extension.

## 13. XML
PostgreSQL provides an XML type:

```sql
document XML
```

Example:

```sql
CREATE TABLE documents (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
content XML
);
```

Insert XML:

```sql
INSERT INTO documents (content)
VALUES (
'<user><name>Alice</name></user>'
);
```

XML is useful when applications must integrate with systems that exchange XML documents.

## 14. Bit String Types
PostgreSQL supports:

BIT(n)
BIT VARYING(n)
Example:

```sql
permissions BIT(8)
```

Insert:

```sql
INSERT INTO security_settings (permissions)
VALUES (B'10101010');
```

These types can be useful when working directly with bit-level representations.

## 15. Money Type
PostgreSQL provides a MONEY type:

price MONEY

However, MONEY is often avoided in portable application schemas because its formatting and behavior can depend on locale settings.

For most financial application data, a fixed-precision numeric type is generally easier to control:

```sql
price NUMERIC(12, 2)
```

## 16. Common Production Choices
A practical PostgreSQL schema might look like this:

```sql
CREATE TABLE users (
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
username TEXT NOT NULL,
email TEXT NOT NULL,
age INTEGER,
balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
is_active BOOLEAN NOT NULL DEFAULT TRUE,
birth_date DATE,
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
metadata JSONB
);
```

Common choices:

| Requirement | Recommended Type |
| --- | --- |
| Normal integer | INTEGER |
| Very large integer | BIGINT |
| Auto-generated integer ID | BIGINT GENERATED ... AS IDENTITY |
| Money/precise decimal | NUMERIC(p,s) |
| General text | TEXT |
| Limited text | VARCHAR(n) when the limit is meaningful |
| True/false | BOOLEAN |
| Date only | DATE |
| Point in time | TIMESTAMPTZ |
| Unique distributed ID | UUID |
| Semi-structured data | JSONB |
| IP address | INET |
| Binary data | BYTEA |
| Duration | INTERVAL |

## 17. Complete Example
The following example demonstrates several PostgreSQL data types together:

```sql
CREATE TABLE employees (
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
employee_number BIGINT GENERATED ALWAYS AS IDENTITY,
name TEXT NOT NULL,
age INTEGER,
salary NUMERIC(12, 2),
is_active BOOLEAN NOT NULL DEFAULT TRUE,
hire_date DATE NOT NULL,
last_login_at TIMESTAMPTZ,
skills TEXT[],
metadata JSONB,
ip_address INET,
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Insert an employee:

```sql
INSERT INTO employees (
name,
age,
salary,
hire_date,
skills,
metadata,
ip_address
)
VALUES (
'Alice Johnson',
30,
75000.00,
'2024-01-15',
ARRAY['PostgreSQL', 'Docker', 'Linux'],
'{"department": "Engineering", "level": "Senior"}',
'192.168.1.100'
);
```

Query the employee:
```sql
SELECT
id,
name,
salary,
skills,
metadata,
created_at
FROM employees;
```

Query JSONB data:

```sql
SELECT *
FROM employees
WHERE metadata->>'department' = 'Engineering';
```

Query array data:

```sql
SELECT *
FROM employees
WHERE 'PostgreSQL' = ANY(skills);
```

## 18. Best Practices
### 18.1 Choose Types Based on the Data
Do not store everything as TEXT.

Prefer:

```sql
age INTEGER
```

instead of:
```sql
age TEXT
```
This gives PostgreSQL better opportunities for validation, comparison, indexing, and query optimization.

### 18.2 Use NUMERIC for Exact Financial Values
For prices, account balances, and other values where exact decimal arithmetic matters:

```sql
amount NUMERIC(12, 2)
```

Avoid floating-point types for values where exact decimal representation is required.

### 18.3 Prefer TIMESTAMPTZ for Points in Time
For application events and audit fields:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

This avoids many problems caused by treating a timestamp without time-zone context as an absolute point in time.

### 18.4 Prefer JSONB for Queryable JSON
If JSON data needs to be searched or indexed:

```sql
metadata JSONB
```

is generally more useful than:
```sql
metadata JSON
```
### 18.5 Don't Overuse JSONB
JSONB is powerful, but it should not replace relational modeling everywhere.

If you frequently query:

```sql
customer_id
product_id
status
created_at
```

these values generally belong in proper columns rather than being hidden inside JSONB.

### 18.6 Use Constraints for Data Integrity
Data types are only one part of database design.

Use constraints such as:

NOT NULL

UNIQUE

CHECK

PRIMARY KEY

FOREIGN KEY

Example:

```sql
CREATE TABLE accounts (
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
email TEXT NOT NULL UNIQUE,
balance NUMERIC(12, 2) NOT NULL
CHECK (balance >= 0)
);
```

This ensures invalid data cannot easily enter the database.

### 18.7 Prefer Identity Columns for New Schemas
For new schemas, consider:

```sql
id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

instead of relying on the older:
```sql
id SERIAL PRIMARY KEY
```

Identity columns are part of the SQL-standard approach and provide clearer schema semantics.

## 19. Quick Reference

| Category | Common PostgreSQL Types |
| :--- | :--- |
| Integer | `SMALLINT`, `INTEGER`, `BIGINT` |
| Decimal | `NUMERIC`, `DECIMAL` |
| Floating Point | `REAL`, `DOUBLE PRECISION` |
| Text | `CHAR`, `VARCHAR`, `TEXT` |
| Boolean | `BOOLEAN` |
| Date | `DATE` |
| Time | `TIME` |
| Timestamp | `TIMESTAMP`, `TIMESTAMPTZ` |
| Duration | `INTERVAL` |
| Identifier | `UUID` |
| JSON | `JSON`, `JSONB` |
| Array | `TEXT[]`, `INTEGER[]`, etc. |
| Binary | `BYTEA` |
| Enum | `ENUM` |
| Network | `INET`, `CIDR`, `MACADDR` |
| Full Text Search | `TSVECTOR`, `TSQUERY` |
| Geometry | `POINT`, `LINE`, `BOX`, `POLYGON`, `CIRCLE` |
| XML | `XML` |
| Bit | `BIT`, `BIT VARYING` |
| Currency | `MONEY` |


## Conclusion
PostgreSQL's type system allows you to model application data accurately instead of treating every value as a string.

For most production applications, a small set of types covers the majority of requirements:

### INTEGER / BIGINT
### NUMERIC
### TEXT
### BOOLEAN
### DATE
### TIMESTAMPTZ
### UUID
### JSONB
### ARRAY
### INET

The best data type depends on the meaning, range, precision, query patterns, and lifecycle of the data.
