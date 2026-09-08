# SQL Practice — Employees Table

**Topic:** Filtering, LIKE/ILIKE, NULL handling, CASE expressions, EXISTS/NOT EXISTS
**Category:** Data Engineering / SQL
**Level:** Intermediate

## Table of Contents

1. [Schema Setup](#1-schema-setup)
2. [Sample Data](#2-sample-data)
3. [Practice Questions](#3-practice-questions)
4. [Solved Queries](#4-solved-queries)
5. [Bonus: CASE Expression Patterns](#5-bonus-case-expression-patterns)

---

## 1. Schema Setup

```sql
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    department VARCHAR(50),
    salary NUMERIC(10,2),
    city VARCHAR(50),
    manager_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 2. Sample Data

```sql
INSERT INTO employees
    (first_name, last_name, email, department, salary, city, manager_id, created_at)
VALUES
    ('John',    'Smith',    'john.smith@gmail.com',     'IT',      75000, 'Bhopal', NULL, '2024-01-15 10:30:00'),
    ('Sarah',   'Johnson',  'sarah.johnson@gmail.com',  'HR',      55000, 'Indore', 1,    '2024-02-20 14:15:00'),
    ('Michael', 'Brown',    'michael.brown@yahoo.com',  'IT',      90000, 'Bhopal', 1,    '2024-03-10 09:45:00'),
    ('Emily',   'Davis',    'emily.davis@company.com',  'Finance', 65000, 'Delhi',  3,    '2024-04-05 16:20:00'),
    ('David',   'Wilson',   'david.wilson@gmail.com',   'Sales',   48000, 'Mumbai', NULL, '2024-05-12 11:10:00'),
    ('Jessica', 'Miller',   'jessica.miller@yahoo.com', 'HR',      58000, 'Bhopal', 2,    '2024-06-18 13:50:00'),
    ('Daniel',  'Moore',    'daniel.moore@gmail.com',   'IT',      82000, 'Pune',   1,    '2024-07-22 08:30:00'),
    ('Sophia',  'Taylor',   'sophia.taylor@company.com','Finance', 72000, 'Indore', 4,    '2024-08-14 15:40:00'),
    ('James',   'Anderson', NULL,                       'Sales',   51000, 'Delhi',  5,    '2024-09-03 10:05:00'),
    ('Olivia',  'Thomas',   'olivia.thomas@gmail.com',  'IT',      95000, 'Mumbai', 3,    '2024-10-11 12:25:00'),
    ('William', 'Jackson',  'william.jackson@yahoo.com','HR',      62000, 'Pune',   NULL, '2024-11-19 17:15:00'),
    ('Ava',     'White',    'ava.white@company.com',    'Sales',   47000, 'Bhopal', 5,    '2024-12-01 09:20:00'),
    ('Robert',  'Harris',   'robert.harris@gmail.com',  'Finance', 68000, 'Delhi',  4,    '2025-01-08 14:35:00'),
    ('Mia',     'Martin',   'mia.martin@yahoo.com',     'IT',      78000, 'Indore', 7,    '2025-02-16 11:45:00'),
    ('Ethan',   'Thompson', 'ethan.thompson@gmail.com', 'Sales',   55000, 'Mumbai', NULL, '2025-03-25 16:10:00');
```

---

## 3. Practice Questions

1. Find employees with salary between 50,000 and 70,000.
2. Find employees whose first name starts with `'J'`.
3. Find employees whose email contains `'gmail'`.
4. Find employees whose name contains `'a'`, regardless of uppercase/lowercase.
5. Find employees who do not have an email.
6. Find employees who do not have a manager.
7. Find employees created between January 1, 2024 and June 30, 2024.
8. Find employees created during 2025.
9. Display employee name and a salary category:
   - `>= 80,000` → `'High'`
   - `60,000–79,999` → `'Medium'`
   - `< 60,000` → `'Low'`
10. Find employees who work in IT or Finance and have salary between 60,000 and 90,000.
11. Find employees whose city starts with `'B'` using `ILIKE`.
12. Find employees whose last name ends with `'son'`.
13. Find employees who have a manager. Use `EXISTS` with a correlated subquery.
14. Find employees who do NOT have a manager. Use `NOT EXISTS`.
15. Find employees whose manager exists in the employees table. Use `EXISTS`.
16. Find departments that have at least one employee earning more than 90,000. Use `EXISTS`.
17. Find employees whose salary is NOT between 50,000 and 80,000.
18. Create a `CASE` expression:
    - `salary >= 90,000` → `'Executive'`
    - `salary >= 70,000` → `'Senior'`
    - `salary >= 50,000` → `'Mid-Level'`
    - otherwise → `'Junior'`
19. Find employees created after `'2024-06-01'` whose salary is between 60,000 and 90,000.
20. Find employees where email is `NULL` OR email does not contain `'gmail'`.

---

## 4. Solved Queries

### Q1 — Salary between 50,000 and 70,000

```sql
SELECT *
FROM employees
WHERE salary BETWEEN 50000 AND 70000;
```

### Q2 — First name starts with 'J'

```sql
SELECT *
FROM employees
WHERE first_name LIKE 'J%';
```

### Q3 — Email contains 'gmail'

```sql
SELECT *
FROM employees
WHERE email LIKE '%gmail%';
```

### Q5 — Employees with no email

```sql
SELECT *
FROM employees
WHERE email IS NULL;
```

> **Common mistake:** Writing `WHERE email = NULL` instead of `WHERE email IS NULL`. Since any comparison against `NULL` evaluates to `UNKNOWN` (not `TRUE`), `email = NULL` always returns **zero rows** — even for rows where the email genuinely is `NULL`.
>
> ```sql
> -- Returns 0 rows, always — this is a bug
> SELECT *
> FROM employees
> WHERE email = NULL;
> ```

### Q6 — Employees with no manager

```sql
SELECT *
FROM employees
WHERE manager_id IS NULL;
```

### Q7 & Q8 — Filtering by date range

For date/timestamp columns, prefer an explicit half-open range over `BETWEEN`:

```sql
SELECT *
FROM employees
WHERE created_at >= '2024-01-01'
  AND created_at < '2025-01-01';
```

> **Why this is preferred over `BETWEEN '2024-01-01' AND '2025-01-01'`:** `BETWEEN` is inclusive on both ends. Since `created_at` is a `TIMESTAMP`, `BETWEEN ... AND '2025-01-01'` only includes rows exactly at `2025-01-01 00:00:00` — it silently excludes any timestamp later that same day (e.g. `2025-01-01 09:15:00`). Using `>=` on the start date and `<` on the day *after* the end date avoids that gap and works correctly regardless of the time portion.
>
> Adjust the bounds to match each question — e.g. `>= '2024-01-01' AND < '2024-07-01'` for Q7 (Jan–Jun 2024), or `>= '2025-01-01' AND < '2026-01-01'` for Q8 (all of 2025).

### Q9 — Salary category with CASE

```sql
SELECT
    first_name,
    last_name,
    CASE
        WHEN salary >= 80000 THEN 'High'
        WHEN salary >= 60000 AND salary < 80000 THEN 'Medium'
        WHEN salary < 60000 THEN 'Low'
    END AS salary_category
FROM employees;
```

### Q10 — IT or Finance with salary range

```sql
SELECT *
FROM employees
WHERE department IN ('IT', 'Finance')
  AND salary >= 60000
  AND salary < 90000;
```

### Q13 — Employees who have a manager (EXISTS)

```sql
SELECT *
FROM employees e1
WHERE EXISTS (
    SELECT 1
    FROM employees e2
    WHERE e2.employee_id = e1.manager_id
);
```

### Q14 — Employees who do NOT have a manager (NOT EXISTS)

```sql
SELECT *
FROM employees e1
WHERE NOT EXISTS (
    SELECT 1
    FROM employees e2
    WHERE e2.employee_id = e1.manager_id
);
```

> **Note:** Q14 can also be answered more simply with `WHERE manager_id IS NULL` (see Q6). The `NOT EXISTS` version is useful practice for cases where "no manager" means *no matching row in a related table*, rather than a plain `NULL` foreign key.

---

## 5. Bonus: CASE Expression Patterns

A few general-purpose `CASE` patterns worth keeping handy, beyond the numbered exercises above.

### Bucketing a numeric column (e.g. age)

```sql
SELECT
    customer_id,
    age,
    CASE
        WHEN age < 18 THEN '0-17'
        WHEN age < 30 THEN '18-29'
        WHEN age < 50 THEN '30-49'
        WHEN age < 65 THEN '50-64'
        ELSE '65+'
    END AS age_bucket
FROM customers;
```

### Standardizing messy values (data cleaning)

```sql
CASE
    WHEN UPPER(status) IN ('Y', 'YES') THEN 'Yes'
    WHEN UPPER(status) IN ('N', 'NO')  THEN 'No'
    ELSE 'Unknown'
END AS standardized_status
```

### Flagging NULLs

```sql
CASE
    WHEN phone IS NULL THEN 'Missing'
    ELSE 'Available'
END AS phone_status
```

### Flagging NULLs *and* empty strings

```sql
CASE
    WHEN email IS NULL OR TRIM(email) = '' THEN 'Missing'
    ELSE 'Available'
END AS email_status
```

> **Why check both:** As covered in the WHERE-clause notes, `NULL` and `''` (empty string) are different things in SQL. A `NULL`-only check (`email IS NULL`) will miss rows where an email field was set to an empty string instead of left blank — so data-cleaning `CASE` expressions typically check both.