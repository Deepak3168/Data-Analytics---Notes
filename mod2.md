# Module 2: SQL — Asking Questions From Data

> **Goal of this module:** Go from "I've never written a query" to "I can answer real business questions from a database": filter, summarize, combine tables, rank, calculate running totals, clean messy data, and write queries that run fast.

In Module 1 you learned how to *get* data. Most of the world's business data sits in **relational databases**, and **SQL** is the language you use to ask those databases questions. If Python is your Swiss army knife, SQL is your scalpel for data.

---

## Table of Contents

0. [Setup: The Practice Database](#0-setup-the-practice-database)
1. [Tables, Rows, Columns, and How SQL Thinks](#1-tables-rows-columns-and-how-sql-thinks)
2. [SELECT: Choosing Columns](#2-select-choosing-columns)
3. [WHERE: Filtering Rows](#3-where-filtering-rows)
4. [ORDER BY, LIMIT, DISTINCT](#4-order-by-limit-distinct)
5. [Aggregations](#5-aggregations)
6. [GROUP BY and HAVING](#6-group-by-and-having)
7. [CASE: If/Else Inside SQL](#7-case-ifelse-inside-sql)
8. [NULL Handling](#8-null-handling)
9. [String and Date Functions](#9-string-and-date-functions)
10. [Joins](#10-joins)
11. [Subqueries and CTEs](#11-subqueries-and-ctes)
12. [Window Functions](#12-window-functions)
13. [Ranking](#13-ranking)
14. [Running Totals and Moving Averages](#14-running-totals-and-moving-averages)
15. [Date-Based Analysis](#15-date-based-analysis)
16. [SQL Data Cleaning](#16-sql-data-cleaning)
17. [Query Performance Basics](#17-query-performance-basics)
18. [Common Mistakes](#18-common-mistakes)
19. [Practice Exercises (with Solutions)](#19-practice-exercises-with-solutions)
20. [Cheat Sheet and Glossary](#20-cheat-sheet-and-glossary)

---

# 0. Setup: The Practice Database

To learn SQL you must **type queries and see results**. Every example in this document uses the same small "shop" database, so you can check your answers against the outputs shown.

## 0.1 Which SQL database are we using?

We use **SQLite**: it needs no installation of a server, it's built into Python, and its SQL is 90% identical to PostgreSQL and MySQL. Where the dialects differ (mostly date and string functions), notes are given in [Section 9](#94-dialect-cheat-table).

**Three ways to practice (pick one):**

1. **Python** (you already know it from Module 1): see 0.3.
2. **DB Browser for SQLite** (free GUI): open a new database, paste the setup script in "Execute SQL", then run queries.
3. **Any online SQLite playground**: paste the setup script, then run queries.

> You need SQLite version **3.25 or newer** for window functions (Sections 12–14). Any recent Python already includes that.

## 0.2 The setup script

Save this as `shop_setup.sql` and run it once.

```sql
DROP VIEW  IF EXISTS order_lines;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS customers;

CREATE TABLE customers (
    customer_id  INTEGER PRIMARY KEY,
    name         TEXT NOT NULL,
    city         TEXT,
    email        TEXT,
    signup_date  TEXT,          -- SQLite stores dates as 'YYYY-MM-DD' text
    referred_by  INTEGER        -- customer_id of the person who referred them
);

CREATE TABLE products (
    product_id    INTEGER PRIMARY KEY,
    product_name  TEXT NOT NULL,
    category      TEXT,
    price         INTEGER
);

CREATE TABLE orders (
    order_id     INTEGER PRIMARY KEY,
    customer_id  INTEGER,
    product_id   INTEGER,
    quantity     INTEGER,
    order_date   TEXT,
    status       TEXT,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
    FOREIGN KEY (product_id)  REFERENCES products(product_id)
);

INSERT INTO customers VALUES
(1, 'Asha',   'Hyderabad', 'asha@example.com',   '2025-01-15', NULL),
(2, 'Ravi',   'Bengaluru', 'ravi@example.com',   '2025-02-20', 1),
(3, 'Meena',  'Chennai',   NULL,                 '2025-03-05', 1),
(4, 'Kiran',  'Hyderabad', 'kiran@example.com',  '2025-03-18', 2),
(5, 'Sneha',  'Mumbai',    'sneha@example.com',  '2025-04-02', NULL),
(6, 'Arjun',  'Bengaluru', NULL,                 '2025-05-11', 5),
(7, 'Divya',  'Hyderabad', 'divya@example.com',  '2025-06-25', 4),
(8, 'Farhan', 'Delhi',     'farhan@example.com', '2025-07-30', NULL);

INSERT INTO products VALUES
(1, 'Laptop',     'Electronics', 55000),
(2, 'Headphones', 'Electronics',  2500),
(3, 'Notebook',   'Stationery',    120),
(4, 'Pen Pack',   'Stationery',     80),
(5, 'Desk Chair', 'Furniture',    7500),
(6, 'Backpack',   'Accessories',  1800);

INSERT INTO orders VALUES
( 1, 1, 1,  1, '2025-02-01', 'delivered'),
( 2, 1, 2,  2, '2025-02-15', 'delivered'),
( 3, 2, 3, 10, '2025-03-01', 'delivered'),
( 4, 2, 6,  1, '2025-03-20', 'cancelled'),
( 5, 3, 5,  1, '2025-04-10', 'delivered'),
( 6, 4, 2,  1, '2025-04-12', 'delivered'),
( 7, 4, 4,  5, '2025-05-03', 'delivered'),
( 8, 5, 1,  1, '2025-05-20', 'returned'),
( 9, 5, 3,  4, '2025-06-08', 'delivered'),
(10, 6, 6,  2, '2025-06-15', 'delivered'),
(11, 1, 5,  1, '2025-07-01', 'delivered'),
(12, 3, 2,  1, '2025-07-19', 'delivered'),
(13, 7, 1,  1, '2025-08-05', 'delivered'),
(14, 7, 4, 10, '2025-08-22', 'pending'),
(15, 2, 2,  2, '2025-09-02', 'delivered'),
(16, 4, 6,  1, '2025-09-14', 'delivered');

-- A saved query ("view") that joins all three tables. Explained in Section 10.
CREATE VIEW order_lines AS
SELECT o.order_id, o.customer_id, c.name, c.city, o.order_date, o.status,
       p.product_id, p.product_name, p.category,
       o.quantity, p.price,
       o.quantity * p.price AS line_total
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN products  p ON p.product_id  = o.product_id;
```

## 0.3 Running SQL from Python

```python
import sqlite3
import pandas as pd

conn = sqlite3.connect(":memory:")            # a temporary database in RAM
conn.executescript(open("shop_setup.sql").read())

def q(sql):
    """Run a SELECT and return a nice DataFrame."""
    return pd.read_sql_query(sql, conn)

print(q("SELECT * FROM customers"))
```

## 0.4 The shop database at a glance

**customers** (8 rows)

| customer_id | name | city | email | signup_date | referred_by |
|---|---|---|---|---|---|
| 1 | Asha | Hyderabad | asha@example.com | 2025-01-15 | NULL |
| 2 | Ravi | Bengaluru | ravi@example.com | 2025-02-20 | 1 |
| 3 | Meena | Chennai | NULL | 2025-03-05 | 1 |
| 4 | Kiran | Hyderabad | kiran@example.com | 2025-03-18 | 2 |
| 5 | Sneha | Mumbai | sneha@example.com | 2025-04-02 | NULL |
| 6 | Arjun | Bengaluru | NULL | 2025-05-11 | 5 |
| 7 | Divya | Hyderabad | divya@example.com | 2025-06-25 | 4 |
| 8 | Farhan | Delhi | farhan@example.com | 2025-07-30 | NULL |

**products** (6 rows)

| product_id | product_name | category | price |
|---|---|---|---|
| 1 | Laptop | Electronics | 55000 |
| 2 | Headphones | Electronics | 2500 |
| 3 | Notebook | Stationery | 120 |
| 4 | Pen Pack | Stationery | 80 |
| 5 | Desk Chair | Furniture | 7500 |
| 6 | Backpack | Accessories | 1800 |

**orders** (16 rows)

| order_id | customer_id | product_id | quantity | order_date | status |
|---|---|---|---|---|---|
| 1 | 1 | 1 | 1 | 2025-02-01 | delivered |
| 2 | 1 | 2 | 2 | 2025-02-15 | delivered |
| 3 | 2 | 3 | 10 | 2025-03-01 | delivered |
| 4 | 2 | 6 | 1 | 2025-03-20 | cancelled |
| 5 | 3 | 5 | 1 | 2025-04-10 | delivered |
| 6 | 4 | 2 | 1 | 2025-04-12 | delivered |
| 7 | 4 | 4 | 5 | 2025-05-03 | delivered |
| 8 | 5 | 1 | 1 | 2025-05-20 | returned |
| 9 | 5 | 3 | 4 | 2025-06-08 | delivered |
| 10 | 6 | 6 | 2 | 2025-06-15 | delivered |
| 11 | 1 | 5 | 1 | 2025-07-01 | delivered |
| 12 | 3 | 2 | 1 | 2025-07-19 | delivered |
| 13 | 7 | 1 | 1 | 2025-08-05 | delivered |
| 14 | 7 | 4 | 10 | 2025-08-22 | pending |
| 15 | 2 | 2 | 2 | 2025-09-02 | delivered |
| 16 | 4 | 6 | 1 | 2025-09-14 | delivered |

Notice a few things on purpose: **Meena and Arjun have no email (NULL)**, **Farhan has never ordered**, and orders have different **statuses**. Real data is like this, and these quirks will teach you important lessons.

---

# 1. Tables, Rows, Columns, and How SQL Thinks

## 1.1 The vocabulary (a quick refresher)

- **Table**: a collection of similar things (`customers`, `orders`).
- **Row** (record): one thing (one customer).
- **Column** (field): one property of every row (`city`).
- **Primary key**: a column whose value is unique for each row (`customer_id`). It identifies the row.
- **Foreign key**: a column pointing at another table's primary key (`orders.customer_id` → `customers.customer_id`). This is how tables are *linked*.
- **Schema**: the blueprint (which tables, which columns, which types).

**Analogy:** A table is a spreadsheet tab. A database is the whole workbook, but with strict rules and the ability to link the tabs together.

## 1.2 Common data types

| Type | Meaning | Example |
|---|---|---|
| `INTEGER` | Whole numbers | `16` |
| `REAL` / `DECIMAL` | Decimals | `2.75` |
| `TEXT` / `VARCHAR` | Text | `'Hyderabad'` |
| `DATE` / `TIMESTAMP` | Dates/times (in SQLite, stored as text `'YYYY-MM-DD'`) | `'2025-09-14'` |
| `BOOLEAN` | True/false | `TRUE` |

## 1.3 Declarative thinking: describe *what*, not *how*

In Python you write step-by-step *how*. In SQL you describe *what* you want and the database figures out *how*:

> "Give me the names of customers in Hyderabad."

```sql
SELECT name FROM customers WHERE city = 'Hyderabad';
```

## 1.4 The four families of SQL commands

| Family | Purpose | Commands |
|---|---|---|
| **DQL** (query) | Read data | `SELECT` ← *this module's focus* |
| **DML** (manipulate) | Change rows | `INSERT`, `UPDATE`, `DELETE` |
| **DDL** (define) | Change structure | `CREATE`, `ALTER`, `DROP` |
| **DCL** (control) | Permissions | `GRANT`, `REVOKE` |

As an analyst, 95% of your time is `SELECT`.

## 1.5 The most important idea: the order SQL *runs* your query

You **write** a query in this order:

```
SELECT ...  FROM ...  WHERE ...  GROUP BY ...  HAVING ...  ORDER BY ...  LIMIT ...
```

But SQL **executes** it in this order:

| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` / `JOIN` | Pick the table(s) and combine them |
| 2 | `WHERE` | Filter individual rows |
| 3 | `GROUP BY` | Bucket rows into groups |
| 4 | `HAVING` | Filter groups |
| 5 | `SELECT` | Choose/calculate the output columns (aliases are created here) |
| 6 | `DISTINCT` | Remove duplicate output rows |
| 7 | `ORDER BY` | Sort |
| 8 | `LIMIT` | Keep the first N rows |

This explains a lot of "why doesn't this work?" moments. Example: you **cannot** use a column alias created in `SELECT` inside `WHERE`, because `WHERE` runs *before* `SELECT`. Keep this table in mind for the whole module.

## 1.6 Formatting conventions

- SQL keywords are case-insensitive (`select` = `SELECT`), but we write them in UPPERCASE for readability.
- Text values use **single quotes**: `'Hyderabad'`. Double quotes are for column/table names.
- Statements end with a semicolon `;`.
- Comments: `-- single line` or `/* multi-line */`.

---

# 2. SELECT: Choosing Columns

`SELECT` says **which columns** you want. `FROM` says **which table**.

## 2.1 Everything, or specific columns

```sql
SELECT * FROM customers;          -- * means "all columns"
```

```sql
SELECT name, city FROM customers;
```

| name | city |
|---|---|
| Asha | Hyderabad |
| Ravi | Bengaluru |
| Meena | Chennai |
| Kiran | Hyderabad |
| Sneha | Mumbai |
| Arjun | Bengaluru |
| Divya | Hyderabad |
| Farhan | Delhi |

> **Good habit:** In real work avoid `SELECT *` (it's slow on wide tables and breaks when columns change). Name the columns you need. `SELECT *` is fine for a quick peek.

## 2.2 Aliases: renaming columns with AS

```sql
SELECT name AS customer_name, city AS home_city
FROM customers
LIMIT 3;
```

| customer_name | home_city |
|---|---|
| Asha | Hyderabad |
| Ravi | Bengaluru |
| Meena | Chennai |

Table aliases shorten queries (very useful with joins later):

```sql
SELECT c.name, c.city FROM customers AS c LIMIT 2;   -- AS is optional: "customers c"
```

## 2.3 Calculated columns

You can compute new columns from existing ones.

```sql
SELECT product_name,
       price,
       ROUND(price * 1.18, 2) AS price_with_gst
FROM products;
```

| product_name | price | price_with_gst |
|---|---|---|
| Laptop | 55000 | 64900.0 |
| Headphones | 2500 | 2950.0 |
| Notebook | 120 | 141.6 |
| Pen Pack | 80 | 94.4 |
| Desk Chair | 7500 | 8850.0 |
| Backpack | 1800 | 2124.0 |

Common operators: `+  -  *  /  %`. **Beware integer division**: in SQLite and PostgreSQL, `7 / 2` = `3` (not `3.5`). Fix by making one side decimal: `7 / 2.0` = `3.5`.

## 2.4 Literals and constants

```sql
SELECT name, 'customer' AS record_type, 1 AS one FROM customers LIMIT 2;
```

| name | record_type | one |
|---|---|---|
| Asha | customer | 1 |
| Ravi | customer | 1 |

## 2.5 SELECT without FROM (quick calculator)

```sql
SELECT 2 + 3 AS five, 10 / 4 AS int_div, 10 / 4.0 AS real_div;
```

| five | int_div | real_div |
|---|---|---|
| 5 | 2 | 2.5 |

---

# 3. WHERE: Filtering Rows

`WHERE` keeps only the rows that satisfy a condition. Think of it as a **bouncer**: only rows that pass get in.

## 3.1 Comparison operators

| Operator | Meaning |
|---|---|
| `=` | equals (single `=`, not `==`) |
| `<>` or `!=` | not equal |
| `<`  `>`  `<=`  `>=` | less/greater than (or equal) |

```sql
SELECT customer_id, name FROM customers WHERE city = 'Hyderabad';
```

| customer_id | name |
|---|---|
| 1 | Asha |
| 4 | Kiran |
| 7 | Divya |

```sql
SELECT product_name, price FROM products WHERE price > 5000;
```

| product_name | price |
|---|---|
| Laptop | 55000 |
| Desk Chair | 7500 |

## 3.2 AND, OR, NOT

```sql
SELECT product_name, price
FROM products
WHERE category = 'Electronics' AND price < 10000;
```

| product_name | price |
|---|---|
| Headphones | 2500 |

```sql
SELECT order_id, status FROM orders WHERE status = 'cancelled' OR status = 'returned';
```

| order_id | status |
|---|---|
| 4 | cancelled |
| 8 | returned |

```sql
SELECT order_id, status FROM orders WHERE NOT status = 'delivered';
```

Returns orders 4, 8, 14 (cancelled, returned, pending).

### ⚠️ AND runs before OR — use parentheses!

```sql
-- Meant: "Bengaluru or Chennai customers who have an email"
SELECT name, city FROM customers
WHERE city = 'Bengaluru' OR city = 'Chennai' AND email IS NOT NULL;
```

SQL reads it as `Bengaluru OR (Chennai AND has email)`:

| name | city |
|---|---|
| Ravi | Bengaluru |
| Arjun | Bengaluru |

Arjun has **no** email but got in! Add parentheses:

```sql
SELECT name, city FROM customers
WHERE (city = 'Bengaluru' OR city = 'Chennai') AND email IS NOT NULL;
```

| name | city |
|---|---|
| Ravi | Bengaluru |

**Rule:** whenever you mix AND and OR, use parentheses.

## 3.3 IN: match any value in a list

```sql
SELECT name, city FROM customers WHERE city IN ('Bengaluru', 'Chennai');
```

| name | city |
|---|---|
| Ravi | Bengaluru |
| Meena | Chennai |
| Arjun | Bengaluru |

`NOT IN` does the opposite. (Careful with NULLs: see Section 8.)

## 3.4 BETWEEN: inclusive ranges

```sql
SELECT order_id, order_date
FROM orders
WHERE order_date BETWEEN '2025-06-01' AND '2025-07-31';
```

| order_id | order_date |
|---|---|
| 9 | 2025-06-08 |
| 10 | 2025-06-15 |
| 11 | 2025-07-01 |
| 12 | 2025-07-19 |

`BETWEEN a AND b` is the same as `>= a AND <= b` (both ends included).

## 3.5 LIKE: pattern matching on text

| Wildcard | Meaning |
|---|---|
| `%` | any number of characters (including none) |
| `_` | exactly one character |

```sql
SELECT name FROM customers WHERE name LIKE 'A%';     -- starts with A
```
→ Asha, Arjun

```sql
SELECT name FROM customers WHERE name LIKE '%a';     -- ends with a
```
→ Asha, Meena, Sneha, Divya

```sql
SELECT name FROM customers WHERE name LIKE '%an%';   -- contains "an"
```
→ Kiran, Farhan

```sql
SELECT name FROM customers WHERE name LIKE '_ira_';  -- exactly 5 letters, middle three are "ira"
```
→ Kiran (`K` + `ira` + `n`)

> **Dialect note:** In SQLite, `LIKE` ignores upper/lower case for English letters. In PostgreSQL, `LIKE` is case-sensitive; use `ILIKE` for case-insensitive.

## 3.6 Filtering on dates (as text in SQLite)

```sql
SELECT order_id, order_date FROM orders WHERE order_date >= '2025-09-01';
```

| order_id | order_date |
|---|---|
| 15 | 2025-09-02 |
| 16 | 2025-09-14 |

Because dates are stored as `YYYY-MM-DD`, alphabetical order equals chronological order. That's why this works.

## 3.7 Combining several conditions (a realistic filter)

*"Delivered orders of Electronics with at least 2 units."* Use the view from setup:

```sql
SELECT order_id, name, product_name, quantity
FROM order_lines
WHERE status = 'delivered'
  AND category = 'Electronics'
  AND quantity >= 2;
```

| order_id | name | product_name | quantity |
|---|---|---|---|
| 2 | Asha | Headphones | 2 |
| 15 | Ravi | Headphones | 2 |

---

# 4. ORDER BY, LIMIT, DISTINCT

## 4.1 ORDER BY: sorting

`ASC` (ascending, default) or `DESC` (descending).

```sql
SELECT product_name, price FROM products ORDER BY price DESC;
```

| product_name | price |
|---|---|
| Laptop | 55000 |
| Desk Chair | 7500 |
| Headphones | 2500 |
| Backpack | 1800 |
| Notebook | 120 |
| Pen Pack | 80 |

**Sort by several columns** (second column breaks ties in the first):

```sql
SELECT category, product_name, price
FROM products
ORDER BY category ASC, price DESC;
```

| category | product_name | price |
|---|---|---|
| Accessories | Backpack | 1800 |
| Electronics | Laptop | 55000 |
| Electronics | Headphones | 2500 |
| Furniture | Desk Chair | 7500 |
| Stationery | Notebook | 120 |
| Stationery | Pen Pack | 80 |

**Sort by an alias or a position:**

```sql
SELECT product_name, price * 2 AS double_price FROM products ORDER BY double_price DESC LIMIT 2;
```

| product_name | double_price |
|---|---|
| Laptop | 110000 |
| Desk Chair | 15000 |

(Unlike `WHERE`, `ORDER BY` runs after `SELECT`, so it *can* use aliases.)

> Without `ORDER BY`, SQL guarantees **no particular row order**. Never rely on "it usually comes out sorted".

## 4.2 LIMIT and OFFSET

`LIMIT n` keeps the first *n* rows. `OFFSET k` skips *k* rows first (used for paging).

```sql
-- Top 3 most expensive products
SELECT product_name, price FROM products ORDER BY price DESC LIMIT 3;
```

| product_name | price |
|---|---|
| Laptop | 55000 |
| Desk Chair | 7500 |
| Headphones | 2500 |

```sql
-- "Page 2" (skip 3, take 3)
SELECT product_name, price FROM products ORDER BY price DESC LIMIT 3 OFFSET 3;
```

| product_name | price |
|---|---|
| Backpack | 1800 |
| Notebook | 120 |
| Pen Pack | 80 |

**Second most expensive product** (a classic interview question):

```sql
SELECT product_name, price FROM products ORDER BY price DESC LIMIT 1 OFFSET 1;
```
→ Desk Chair, 7500

> Dialect note: SQL Server uses `SELECT TOP 3 ...`; Oracle (older) uses `FETCH FIRST 3 ROWS ONLY`. PostgreSQL, MySQL, SQLite use `LIMIT`.

## 4.3 DISTINCT: remove duplicate rows

```sql
SELECT city FROM customers;             -- 8 rows, Hyderabad appears 3 times
SELECT DISTINCT city FROM customers;    -- unique cities only
```

| city |
|---|
| Hyderabad |
| Bengaluru |
| Chennai |
| Mumbai |
| Delhi |

```sql
SELECT DISTINCT status FROM orders;
```

| status |
|---|
| delivered |
| cancelled |
| returned |
| pending |

`DISTINCT` applies to the **whole row** of selected columns:

```sql
SELECT DISTINCT category, price > 5000 AS is_expensive FROM products ORDER BY category;
```

| category | is_expensive |
|---|---|
| Accessories | 0 |
| Electronics | 1 |
| Electronics | 0 |
| Furniture | 1 |
| Stationery | 0 |

(Electronics appears twice because the pair `(Electronics, 1)` differs from `(Electronics, 0)`. Stationery collapses to one row since both products give `(Stationery, 0)`.)

**Counting unique values:**

```sql
SELECT COUNT(DISTINCT city) AS unique_cities FROM customers;   -- 5
```

> **Warning:** Don't use `DISTINCT` as a bandage for a wrong join that produced duplicates. Find out *why* duplicates appeared (Section 10.8).

---

# 5. Aggregations

**Aggregate functions** take many rows and squash them into **one value**.

| Function | Returns |
|---|---|
| `COUNT(*)` | number of rows |
| `COUNT(col)` | number of rows where `col` is **not NULL** |
| `COUNT(DISTINCT col)` | number of unique non-NULL values |
| `SUM(col)` | total |
| `AVG(col)` | average |
| `MIN(col)` / `MAX(col)` | smallest / largest |

**Analogy:** a funnel. Many rows go in the top; one summary number comes out.

## 5.1 Basic aggregates

```sql
SELECT COUNT(*)                AS total_orders,
       SUM(quantity)           AS total_units,
       ROUND(AVG(quantity), 2) AS avg_units,
       MIN(quantity)           AS min_units,
       MAX(quantity)           AS max_units
FROM orders;
```

| total_orders | total_units | avg_units | min_units | max_units |
|---|---|---|---|---|
| 16 | 44 | 2.75 | 1 | 10 |

## 5.2 COUNT(*) vs COUNT(column)

```sql
SELECT COUNT(*) AS all_customers, COUNT(email) AS with_email FROM customers;
```

| all_customers | with_email |
|---|---|
| 8 | 6 |

`COUNT(email)` skips NULLs, so Meena and Arjun aren't counted. **Aggregates ignore NULL** (except `COUNT(*)`).

## 5.3 Aggregates with WHERE

```sql
SELECT COUNT(*) AS delivered_orders FROM orders WHERE status = 'delivered';   -- 13
```

```sql
SELECT SUM(line_total) AS delivered_revenue
FROM order_lines
WHERE status = 'delivered';
```

| delivered_revenue |
|---|
| 147480 |

`WHERE` filters rows **first**, then the aggregate runs on what's left.

## 5.4 Common trap: mixing aggregates and normal columns

```sql
SELECT name, COUNT(*) FROM customers;   -- ⚠️ SQLite allows it but the result is meaningless;
                                        --    PostgreSQL/MySQL(strict) throw an error
```

If you select a normal column next to an aggregate, you must tell SQL how to group: that's `GROUP BY`, next.

---

# 6. GROUP BY and HAVING

## 6.1 GROUP BY: one summary row per group

**Analogy:** sort a pile of receipts into bins by month, then total each bin.

```sql
SELECT status, COUNT(*) AS orders, SUM(quantity) AS units
FROM orders
GROUP BY status
ORDER BY status;
```

| status | orders | units |
|---|---|---|
| cancelled | 1 | 1 |
| delivered | 13 | 32 |
| pending | 1 | 10 |
| returned | 1 | 1 |

**Rule:** every column in `SELECT` must either be in `GROUP BY` or wrapped in an aggregate function.

```sql
SELECT category,
       COUNT(*)                AS products,
       ROUND(AVG(price), 2)    AS avg_price
FROM products
GROUP BY category
ORDER BY category;
```

| category | products | avg_price |
|---|---|---|
| Accessories | 1 | 1800.0 |
| Electronics | 2 | 28750.0 |
| Furniture | 1 | 7500.0 |
| Stationery | 2 | 100.0 |

### A business question: revenue by category (delivered only)

```sql
SELECT category,
       COUNT(*)        AS orders,
       SUM(line_total) AS revenue
FROM order_lines
WHERE status = 'delivered'
GROUP BY category
ORDER BY revenue DESC;
```

| category | orders | revenue |
|---|---|---|
| Electronics | 6 | 125000 |
| Furniture | 2 | 15000 |
| Accessories | 2 | 5400 |
| Stationery | 3 | 2080 |

### Grouping by more than one column

```sql
SELECT category, status, COUNT(*) AS orders
FROM order_lines
GROUP BY category, status
ORDER BY category, status;
```

| category | status | orders |
|---|---|---|
| Accessories | cancelled | 1 |
| Accessories | delivered | 2 |
| Electronics | delivered | 6 |
| Electronics | returned | 1 |
| Furniture | delivered | 2 |
| Stationery | delivered | 3 |
| Stationery | pending | 1 |

## 6.2 HAVING: filtering groups

`WHERE` filters **rows before** grouping. `HAVING` filters **groups after** grouping (so it can use aggregates).

```sql
-- Cities with more than one customer
SELECT city, COUNT(*) AS customers
FROM customers
GROUP BY city
HAVING COUNT(*) > 1
ORDER BY city;
```

| city | customers |
|---|---|
| Bengaluru | 2 |
| Hyderabad | 3 |

```sql
-- Customers with 3 or more orders
SELECT customer_id, COUNT(*) AS n_orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 3
ORDER BY customer_id;
```

| customer_id | n_orders |
|---|---|
| 1 | 3 |
| 2 | 3 |
| 4 | 3 |

## 6.3 WHERE and HAVING together

*"Among delivered orders only, which customers bought 5+ units in total?"*

```sql
SELECT customer_id, SUM(quantity) AS units
FROM orders
WHERE status = 'delivered'          -- row filter, runs first
GROUP BY customer_id
HAVING SUM(quantity) >= 5           -- group filter, runs after grouping
ORDER BY units DESC;
```

| customer_id | units |
|---|---|
| 2 | 12 |
| 4 | 7 |

| | WHERE | HAVING |
|---|---|---|
| Filters | individual rows | groups |
| Runs | before GROUP BY | after GROUP BY |
| Can use aggregates? | No | Yes |

---

# 7. CASE: If/Else Inside SQL

`CASE` lets you create a column whose value depends on conditions, like `if / elif / else`.

```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

SQL checks the `WHEN`s top to bottom and returns the **first** match.

## 7.1 Labeling rows

```sql
SELECT product_name,
       price,
       CASE
           WHEN price >= 10000 THEN 'Premium'
           WHEN price >= 1000  THEN 'Mid'
           ELSE 'Budget'
       END AS tier
FROM products;
```

| product_name | price | tier |
|---|---|---|
| Laptop | 55000 | Premium |
| Headphones | 2500 | Mid |
| Notebook | 120 | Budget |
| Pen Pack | 80 | Budget |
| Desk Chair | 7500 | Mid |
| Backpack | 1800 | Mid |

## 7.2 Bucketing and then counting

```sql
SELECT CASE
           WHEN quantity >= 5 THEN 'Bulk'
           WHEN quantity >= 2 THEN 'Multiple'
           ELSE 'Single'
       END AS order_type,
       COUNT(*) AS orders
FROM orders
GROUP BY order_type
ORDER BY orders DESC;
```

| order_type | orders |
|---|---|
| Single | 9 |
| Multiple | 4 |
| Bulk | 3 |

## 7.3 Conditional aggregation (the analyst's favorite trick)

Put `CASE` **inside** an aggregate to count or sum only certain rows, producing several columns in one pass (a "pivot").

```sql
SELECT COUNT(*) AS total,
       SUM(CASE WHEN status = 'delivered' THEN 1 ELSE 0 END) AS delivered,
       SUM(CASE WHEN status <> 'delivered' THEN 1 ELSE 0 END) AS not_delivered
FROM orders;
```

| total | delivered | not_delivered |
|---|---|---|
| 16 | 13 | 3 |

Revenue by category, one column per status:

```sql
SELECT category,
       SUM(CASE WHEN status = 'delivered' THEN line_total ELSE 0 END) AS delivered_rev,
       SUM(CASE WHEN status IN ('cancelled','returned') THEN line_total ELSE 0 END) AS lost_rev
FROM order_lines
GROUP BY category
ORDER BY category;
```

| category | delivered_rev | lost_rev |
|---|---|---|
| Accessories | 5400 | 1800 |
| Electronics | 125000 | 55000 |
| Furniture | 15000 | 0 |
| Stationery | 2080 | 0 |

## 7.4 CASE in ORDER BY (custom sort order)

```sql
SELECT order_id, status
FROM orders
ORDER BY CASE status
             WHEN 'pending'   THEN 1
             WHEN 'delivered' THEN 2
             ELSE 3
         END,
         order_id
LIMIT 4;
```

| order_id | status |
|---|---|
| 14 | pending |
| 1 | delivered |
| 2 | delivered |
| 3 | delivered |

---

# 8. NULL Handling

**NULL means "unknown / missing"**. It is *not* zero, *not* an empty string, *not* the word "null". It's the absence of a value. NULL causes more SQL bugs than anything else, so learn its rules.

## 8.1 Rule 1: NULL is never equal to anything (even NULL)

```sql
SELECT name FROM customers WHERE email = NULL;      -- ❌ returns NOTHING
SELECT name FROM customers WHERE email IS NULL;     -- ✅
```

| name |
|---|
| Meena |
| Arjun |

`IS NOT NULL` finds rows that have a value.

## 8.2 Rule 2: comparisons with NULL are "unknown", and WHERE drops unknowns

```sql
SELECT name FROM customers WHERE email <> 'ravi@example.com';
```

| name |
|---|
| Asha |
| Kiran |
| Sneha |
| Divya |
| Farhan |

Only **5** rows come back, not 7. Meena and Arjun (NULL email) silently vanished because `NULL <> 'ravi@...'` is unknown, not true. To include them:

```sql
SELECT name FROM customers WHERE email <> 'ravi@example.com' OR email IS NULL;   -- 7 rows
```

## 8.3 Rule 3: math with NULL gives NULL

```sql
SELECT 5 + NULL AS a, NULL * 10 AS b, 'Hi ' || NULL AS c;
```
→ all three are `NULL`.

## 8.4 Rule 4: aggregates skip NULLs

```sql
SELECT COUNT(*) AS customers, COUNT(referred_by) AS were_referred FROM customers;
```

| customers | were_referred |
|---|---|
| 8 | 5 |

## 8.5 COALESCE: replace NULL with a default

`COALESCE(a, b, c, ...)` returns the first non-NULL argument.

```sql
SELECT name, COALESCE(email, 'no email on file') AS contact
FROM customers
WHERE customer_id IN (1, 3, 6);
```

| name | contact |
|---|---|
| Asha | asha@example.com |
| Meena | no email on file |
| Arjun | no email on file |

```sql
-- Treat missing referrer as 0
SELECT name, COALESCE(referred_by, 0) AS referrer_id FROM customers LIMIT 3;
```

| name | referrer_id |
|---|---|
| Asha | 0 |
| Ravi | 1 |
| Meena | 1 |

## 8.6 NULLIF: turn a value *into* NULL (great for avoiding divide-by-zero)

`NULLIF(a, b)` returns NULL if `a = b`, else `a`.

```sql
SELECT 100.0 / NULLIF(0, 0) AS safe_division;   -- NULL instead of an error (PostgreSQL would error on 100/0)
```

Also used in cleaning: `NULLIF(phone, '')` turns empty strings into real NULLs (Section 16).

## 8.7 IFNULL / NVL / ISNULL: the dialect versions of COALESCE

| Database | Two-argument version |
|---|---|
| SQLite / MySQL | `IFNULL(x, default)` |
| Oracle | `NVL(x, default)` |
| SQL Server | `ISNULL(x, default)` |
| All | `COALESCE(x, default)` ← use this, it's standard |

## 8.8 The NOT IN + NULL trap (a famous bug)

*"Which customers have never referred anyone?"*

```sql
-- ❌ Looks right, returns ZERO rows
SELECT name FROM customers
WHERE customer_id NOT IN (SELECT referred_by FROM customers);
```

The subquery's result includes NULL (from Asha, Sneha, Farhan). `x NOT IN (1, 2, NULL)` is never true, so nothing passes. Fix by removing NULLs:

```sql
SELECT name FROM customers
WHERE customer_id NOT IN (SELECT referred_by FROM customers WHERE referred_by IS NOT NULL);
```

| name |
|---|
| Meena |
| Arjun |
| Divya |
| Farhan |

Or use `NOT EXISTS` (Section 11).

## 8.9 NULLs when sorting

SQLite puts NULLs **first** in ascending order. PostgreSQL puts them **last**. Be explicit when it matters:

```sql
SELECT name, email FROM customers ORDER BY email IS NULL, email;   -- non-NULL first, NULLs last (works in SQLite/MySQL)
-- PostgreSQL: ORDER BY email NULLS LAST
```

## 8.10 Detecting NULLs (data quality check)

```sql
SELECT COUNT(*) AS total_rows,
       SUM(CASE WHEN email IS NULL THEN 1 ELSE 0 END) AS missing_email,
       SUM(CASE WHEN city  IS NULL THEN 1 ELSE 0 END) AS missing_city
FROM customers;
```

| total_rows | missing_email | missing_city |
|---|---|---|
| 8 | 2 | 0 |

---

# 9. String and Date Functions

Function names vary a bit across databases. Examples here use **SQLite**; the dialect table at the end of this section shows the PostgreSQL/MySQL equivalents.

## 9.1 String functions

| Function | Purpose | Example → Result |
|---|---|---|
| `UPPER(x)` / `LOWER(x)` | change case | `UPPER('asha')` → `ASHA` |
| `LENGTH(x)` | number of characters | `LENGTH('Asha')` → `4` |
| `TRIM(x)` | remove spaces at both ends | `TRIM('  hi ')` → `hi` |
| `SUBSTR(x, start, len)` | cut part of text (start at 1) | `SUBSTR('Hyderabad',1,3)` → `Hyd` |
| `REPLACE(x, old, new)` | replace text | `REPLACE('98-76','-','')` → `9876` |
| `INSTR(x, find)` | position of `find` (0 if none) | `INSTR('a@b.com','@')` → `2` |
| `x \|\| y` | concatenate | `'A' \|\| 'B'` → `AB` |

```sql
SELECT name,
       UPPER(city)  AS city_upper,
       LENGTH(name) AS name_length
FROM customers
LIMIT 3;
```

| name | city_upper | name_length |
|---|---|---|
| Asha | HYDERABAD | 4 |
| Ravi | BENGALURU | 4 |
| Meena | CHENNAI | 5 |

**Concatenation:**

```sql
SELECT name || ' (' || city || ')' AS label FROM customers LIMIT 3;
```

| label |
|---|
| Asha (Hyderabad) |
| Ravi (Bengaluru) |
| Meena (Chennai) |

**Extracting the email domain** (position of `@`, then take everything after it):

```sql
SELECT email,
       SUBSTR(email, INSTR(email, '@') + 1) AS domain
FROM customers
WHERE email IS NOT NULL
LIMIT 2;
```

| email | domain |
|---|---|
| asha@example.com | example.com |
| ravi@example.com | example.com |

**First letter + first 3 letters:**

```sql
SELECT name, SUBSTR(name, 1, 1) AS initial, SUBSTR(name, 1, 3) AS short_name
FROM customers LIMIT 3;
```

| name | initial | short_name |
|---|---|---|
| Asha | A | Ash |
| Ravi | R | Rav |
| Meena | M | Mee |

**Searching inside text:**

```sql
SELECT product_name FROM products WHERE LOWER(product_name) LIKE '%pack%';   -- Pen Pack, Backpack
```

## 9.2 Date functions (SQLite)

SQLite has no real date type; it stores `'YYYY-MM-DD'` text and provides functions to work with it.

| Function | Purpose | Example |
|---|---|---|
| `date(x)` | normalize to date | `date('2025-02-15')` |
| `date(x, '+7 days')` | add/subtract time | `date('2025-02-01','+7 days')` → `2025-02-08` |
| `date(x, 'start of month')` | first day of month | `2025-02-15` → `2025-02-01` |
| `strftime('%Y', x)` | year | `'2025'` |
| `strftime('%m', x)` | month (`01`–`12`) | `'02'` |
| `strftime('%Y-%m', x)` | year-month | `'2025-02'` |
| `strftime('%d', x)` | day of month | `'15'` |
| `strftime('%w', x)` | weekday (0=Sunday) | |
| `julianday(x)` | day number (for subtraction) | |
| `date('now')` | today | changes daily |

```sql
SELECT order_id,
       order_date,
       strftime('%Y-%m', order_date)      AS month,
       date(order_date, '+7 days')        AS expected_delivery,
       date(order_date, 'start of month') AS month_start
FROM orders
LIMIT 3;
```

| order_id | order_date | month | expected_delivery | month_start |
|---|---|---|---|---|
| 1 | 2025-02-01 | 2025-02 | 2025-02-08 | 2025-02-01 |
| 2 | 2025-02-15 | 2025-02 | 2025-02-22 | 2025-02-01 |
| 3 | 2025-03-01 | 2025-03 | 2025-03-08 | 2025-03-01 |

**Difference between two dates** (in days) = subtract `julianday` values:

```sql
SELECT julianday('2025-03-01') - julianday('2025-02-01') AS days_between;   -- 28.0
```

**Converting month text to a number:**

```sql
SELECT order_id, CAST(strftime('%m', order_date) AS INTEGER) AS month_number FROM orders LIMIT 3;
```

| order_id | month_number |
|---|---|
| 1 | 2 |
| 2 | 2 |
| 3 | 3 |

## 9.3 Using functions in WHERE and GROUP BY

```sql
-- Orders placed in July or later
SELECT order_id, order_date FROM orders WHERE CAST(strftime('%m', order_date) AS INTEGER) >= 7;
-- (performance tip in Section 17: filtering with a range is usually faster than wrapping the column in a function)
```

```sql
-- Customers who signed up in each month
SELECT strftime('%Y-%m', signup_date) AS signup_month, COUNT(*) AS new_customers
FROM customers
GROUP BY signup_month
ORDER BY signup_month;
```

| signup_month | new_customers |
|---|---|
| 2025-01 | 1 |
| 2025-02 | 1 |
| 2025-03 | 2 |
| 2025-04 | 1 |
| 2025-05 | 1 |
| 2025-06 | 1 |
| 2025-07 | 1 |

## 9.4 Dialect cheat table

| Task | SQLite | PostgreSQL | MySQL |
|---|---|---|---|
| Concatenate | `a \|\| b` | `a \|\| b` or `CONCAT(a,b)` | `CONCAT(a,b)` |
| Substring | `SUBSTR(x,1,3)` | `SUBSTRING(x,1,3)` or `SUBSTR` | `SUBSTRING(x,1,3)` |
| Find position | `INSTR(x,'@')` | `POSITION('@' IN x)` | `INSTR(x,'@')` |
| Year-month | `strftime('%Y-%m',d)` | `TO_CHAR(d,'YYYY-MM')` | `DATE_FORMAT(d,'%Y-%m')` |
| Truncate to month | `date(d,'start of month')` | `DATE_TRUNC('month', d)` | `DATE_FORMAT(d,'%Y-%m-01')` |
| Days between | `julianday(a)-julianday(b)` | `a - b` (dates) | `DATEDIFF(a,b)` |
| Add 7 days | `date(d,'+7 days')` | `d + INTERVAL '7 days'` | `DATE_ADD(d, INTERVAL 7 DAY)` |
| Today | `date('now')` | `CURRENT_DATE` | `CURDATE()` |
| Proper case | (none) | `INITCAP(x)` | (none) |

---

# 10. Joins

## 10.1 Why joins exist

Databases split information across tables to avoid repeating data. Customer details live in `customers`; purchases live in `orders`. To answer "what did **Asha** buy?" you must **join** the tables using the linking column.

**Analogy:** Two phone books. One maps *names → ID numbers*; the other maps *ID numbers → purchases*. A join follows the ID from one to the other.

## 10.2 Anatomy of a join

```sql
SELECT o.order_id, c.name, o.order_date
FROM orders AS o
INNER JOIN customers AS c
        ON c.customer_id = o.customer_id;
```

- `FROM orders AS o` — the first table (alias `o`).
- `JOIN customers AS c` — the second table (alias `c`).
- `ON ...` — the **matching rule**: "a row from `orders` matches a row from `customers` when their `customer_id` values are equal".
- Prefix columns with the alias (`c.name`) so SQL knows which table you mean.

## 10.3 The join types

```
INNER JOIN         LEFT JOIN          RIGHT JOIN         FULL OUTER JOIN
  (A ∩ B)          (all A + matches)  (all B + matches)  (everything)

  ( A (▓) B )      ( A▓(▓) B )        ( A (▓)▓B )        ( A▓(▓)▓B )
```

| Join | Returns |
|---|---|
| **INNER JOIN** | Only rows with a match in **both** tables |
| **LEFT JOIN** | **All rows from the left** table; matched right values, or NULL when no match |
| **RIGHT JOIN** | All rows from the right table (rarely used; swap the tables and use LEFT) |
| **FULL OUTER JOIN** | All rows from both; NULLs where there's no match |
| **CROSS JOIN** | Every row paired with every row (a product; no `ON`) |
| **Self join** | A table joined to itself |

> SQLite supports RIGHT and FULL joins from version 3.39. LEFT is what you'll use 95% of the time.

## 10.4 INNER JOIN in action

```sql
SELECT o.order_id, c.name, o.order_date
FROM orders o
INNER JOIN customers c ON c.customer_id = o.customer_id
ORDER BY o.order_id
LIMIT 5;
```

| order_id | name | order_date |
|---|---|---|
| 1 | Asha | 2025-02-01 |
| 2 | Asha | 2025-02-15 |
| 3 | Ravi | 2025-03-01 |
| 4 | Ravi | 2025-03-20 |
| 5 | Meena | 2025-04-10 |

## 10.5 Joining three tables

```sql
SELECT o.order_id,
       c.name,
       p.product_name,
       o.quantity,
       p.price,
       o.quantity * p.price AS line_total
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN products  p ON p.product_id  = o.product_id
WHERE o.order_id <= 5
ORDER BY o.order_id;
```

| order_id | name | product_name | quantity | price | line_total |
|---|---|---|---|---|---|
| 1 | Asha | Laptop | 1 | 55000 | 55000 |
| 2 | Asha | Headphones | 2 | 2500 | 5000 |
| 3 | Ravi | Notebook | 10 | 120 | 1200 |
| 4 | Ravi | Backpack | 1 | 1800 | 1800 |
| 5 | Meena | Desk Chair | 1 | 7500 | 7500 |

(`JOIN` alone means `INNER JOIN`.) This exact query is saved as the view `order_lines` in the setup script, so you can now write `SELECT * FROM order_lines` instead of repeating the joins.

## 10.6 Joins + GROUP BY = real analysis

**Revenue per customer (delivered orders):**

```sql
SELECT c.name, SUM(o.quantity * p.price) AS total_spent
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
JOIN products  p ON p.product_id  = o.product_id
WHERE o.status = 'delivered'
GROUP BY c.customer_id, c.name
ORDER BY total_spent DESC;
```

| name | total_spent |
|---|---|
| Asha | 67500 |
| Divya | 55000 |
| Meena | 10000 |
| Ravi | 6200 |
| Kiran | 4700 |
| Arjun | 3600 |
| Sneha | 480 |

**Units sold per product:**

```sql
SELECT p.product_name, SUM(o.quantity) AS units_sold
FROM orders o
JOIN products p ON p.product_id = o.product_id
WHERE o.status = 'delivered'
GROUP BY p.product_id, p.product_name
ORDER BY units_sold DESC, p.product_name;
```

| product_name | units_sold |
|---|---|
| Notebook | 14 |
| Headphones | 6 |
| Pen Pack | 5 |
| Backpack | 3 |
| Desk Chair | 2 |
| Laptop | 2 |

## 10.7 LEFT JOIN: keeping rows that have no match

Farhan has never ordered. An INNER JOIN would drop him. A LEFT JOIN keeps him:

```sql
SELECT c.name, COUNT(o.order_id) AS n_orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.name
ORDER BY n_orders DESC, c.name;
```

| name | n_orders |
|---|---|
| Asha | 3 |
| Kiran | 3 |
| Ravi | 3 |
| Divya | 2 |
| Meena | 2 |
| Sneha | 2 |
| Arjun | 1 |
| Farhan | 0 |

> **Why `COUNT(o.order_id)` and not `COUNT(*)`?** For Farhan, the LEFT JOIN creates one row with NULLs in the `o` columns. `COUNT(*)` would count that row (giving 1); `COUNT(o.order_id)` counts only non-NULL matches (giving 0).

### Anti-join: "find rows with NO match"

```sql
SELECT c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

| name |
|---|
| Farhan |

This "LEFT JOIN … WHERE right.key IS NULL" pattern answers: customers with no orders, products never sold, users who never logged in.

## 10.8 Join pitfalls

**1. Forgetting the `ON` condition** creates a cross join (every row × every row):

```sql
SELECT COUNT(*) FROM customers CROSS JOIN products;   -- 8 × 6 = 48 rows
```

**2. Fan-out (duplicates from one-to-many):** joining a customer to their orders repeats the customer once per order. If you then `SUM` a customer-level number, you'd count it multiple times. *Always check row counts before and after a join.*

```sql
SELECT COUNT(*) FROM customers;                                                        -- 8
SELECT COUNT(*) FROM customers c JOIN orders o ON o.customer_id = c.customer_id;      -- 16
```

**3. Ambiguous columns:** `customer_id` exists in both tables. Prefix it: `c.customer_id`.

**4. Filtering a LEFT JOIN in `WHERE`** can silently turn it into an INNER JOIN. To keep unmatched rows, put the condition in the `ON`:

```sql
SELECT c.name, COUNT(o.order_id) AS delivered_orders
FROM customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
      AND o.status = 'delivered'          -- ✅ filter inside ON keeps every customer
GROUP BY c.customer_id, c.name
ORDER BY c.customer_id;
```

| name | delivered_orders |
|---|---|
| Asha | 3 |
| Ravi | 2 |
| Meena | 2 |
| Kiran | 3 |
| Sneha | 1 |
| Arjun | 1 |
| Divya | 1 |
| Farhan | 0 |

(Had we written `WHERE o.status = 'delivered'` instead, Farhan would disappear.)

## 10.9 Self join: a table joined to itself

`customers.referred_by` points to another row in the same table. To show *who referred whom*, join the table to itself with two different aliases:

```sql
SELECT c.name AS customer, r.name AS referred_by
FROM customers c
LEFT JOIN customers r ON r.customer_id = c.referred_by
ORDER BY c.customer_id;
```

| customer | referred_by |
|---|---|
| Asha | NULL |
| Ravi | Asha |
| Meena | Asha |
| Kiran | Ravi |
| Sneha | NULL |
| Arjun | Sneha |
| Divya | Kiran |
| Farhan | NULL |

## 10.10 UNION: stacking results (not a join, but related)

`UNION` stacks rows from two queries with the same columns (removing duplicates); `UNION ALL` keeps duplicates and is faster.

```sql
SELECT name AS label, 'customer' AS kind FROM customers WHERE city = 'Delhi'
UNION ALL
SELECT product_name, 'product' FROM products WHERE category = 'Furniture';
```

| label | kind |
|---|---|
| Farhan | customer |
| Desk Chair | product |

---

# 11. Subqueries and CTEs

Sometimes one query needs the answer to another query first. There are two tools: **subqueries** (a query inside a query) and **CTEs** (named steps).

## 11.1 Scalar subquery (returns one value)

*"Which orders have a quantity above the average quantity?"* (average is 2.75)

```sql
SELECT order_id, quantity
FROM orders
WHERE quantity > (SELECT AVG(quantity) FROM orders)
ORDER BY order_id;
```

| order_id | quantity |
|---|---|
| 3 | 10 |
| 7 | 5 |
| 9 | 4 |
| 14 | 10 |

The inner query runs first, yielding `2.75`; the outer query then filters.

## 11.2 Subquery with IN (returns a list)

*"Customers who ever ordered a Laptop."*

```sql
SELECT name
FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders WHERE product_id = 1);
```

| name |
|---|
| Asha |
| Sneha |
| Divya |

## 11.3 EXISTS / NOT EXISTS

`EXISTS` asks "is there at least one matching row?" It's often faster and safer than `IN`/`NOT IN` (no NULL trap).

```sql
-- Customers who have NEVER ordered
SELECT name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

| name |
|---|
| Farhan |

## 11.4 Subquery in FROM (derived table)

*"What is the average total spend per customer?"* First compute each customer's total, then average those totals.

```sql
SELECT ROUND(AVG(total_spent), 2) AS avg_customer_spend
FROM (
    SELECT customer_id, SUM(line_total) AS total_spent
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY customer_id
) AS customer_totals;
```

| avg_customer_spend |
|---|
| 21068.57 |

A subquery in `FROM` **must have an alias** (`customer_totals`).

## 11.5 Correlated subquery (refers to the outer row)

*"Orders where the quantity is above that same customer's own average."*

```sql
SELECT o.order_id, o.customer_id, o.quantity
FROM orders o
WHERE o.quantity > (
    SELECT AVG(o2.quantity)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id      -- links inner query to the current outer row
)
ORDER BY o.order_id;
```

| order_id | customer_id | quantity |
|---|---|---|
| 2 | 1 | 2 |
| 3 | 2 | 10 |
| 7 | 4 | 5 |
| 9 | 5 | 4 |
| 14 | 7 | 10 |

A correlated subquery conceptually re-runs for **every outer row**, so it can be slow on big tables. Window functions (Section 12) usually do the same job faster.

## 11.6 CTEs: `WITH` (subqueries with names)

A **Common Table Expression** names a step so the query reads top to bottom like a recipe.

```sql
WITH customer_spend AS (
    SELECT customer_id, SUM(line_total) AS total_spent
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY customer_id
)
SELECT c.name, cs.total_spent
FROM customer_spend cs
JOIN customers c ON c.customer_id = cs.customer_id
WHERE cs.total_spent > (SELECT AVG(total_spent) FROM customer_spend)
ORDER BY cs.total_spent DESC;
```

| name | total_spent |
|---|---|
| Asha | 67500 |
| Divya | 55000 |

Same result as nested subqueries, but far easier to read, debug, and reuse (`customer_spend` is used twice).

### Multiple CTEs (chain steps with commas)

*"Categories whose delivered revenue is above the average category revenue."*

```sql
WITH category_revenue AS (
    SELECT category, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY category
),
avg_revenue AS (
    SELECT AVG(revenue) AS avg_rev FROM category_revenue
)
SELECT cr.category, cr.revenue
FROM category_revenue cr, avg_revenue ar
WHERE cr.revenue > ar.avg_rev;
```

| category | revenue |
|---|---|
| Electronics | 125000 |

(Average category revenue = 147480 / 4 = 36870; only Electronics exceeds it.)

## 11.7 Recursive CTE (a peek)

A CTE can refer to itself, useful for hierarchies (org charts, referral chains) and generating sequences.

```sql
WITH RECURSIVE nums(n) AS (
    SELECT 1                       -- starting row
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 5   -- repeat until n = 5
)
SELECT n FROM nums;
```

| n |
|---|
| 1 |
| 2 |
| 3 |
| 4 |
| 5 |

**Who is in Asha's referral chain?**

```sql
WITH RECURSIVE chain(customer_id, name, depth) AS (
    SELECT customer_id, name, 0 FROM customers WHERE name = 'Asha'
    UNION ALL
    SELECT c.customer_id, c.name, chain.depth + 1
    FROM customers c
    JOIN chain ON c.referred_by = chain.customer_id
)
SELECT name, depth FROM chain ORDER BY depth, name;
```

| name | depth |
|---|---|
| Asha | 0 |
| Meena | 1 |
| Ravi | 1 |
| Kiran | 2 |
| Divya | 3 |

## 11.8 Subquery vs CTE: which to use?

| Situation | Choose |
|---|---|
| One small, simple inner value | Subquery |
| Multi-step logic; readability matters; reuse a result | CTE |
| Hierarchies | Recursive CTE |
| Need to save it permanently | `CREATE VIEW` |

---

# 12. Window Functions

## 12.1 The big idea

`GROUP BY` **collapses** rows: 100 rows in, 5 group-rows out. A **window function** calculates over a group of related rows **but keeps every row**. It adds a new column next to each row.

**Analogy:** You're in a classroom. `GROUP BY` says "give me the class average" (one number). A window function writes the class average *next to every student's name*, so you can compare each student to it.

## 12.2 Syntax

```sql
function_name(...) OVER (
    PARTITION BY column_to_split_into_groups     -- optional: restart per group
    ORDER BY column_that_defines_order           -- optional (needed for ranking, running totals)
    ROWS BETWEEN ... AND ...                     -- optional: which rows to include (the "frame")
)
```

- `OVER ()` with nothing inside = the window is the whole result set.
- `PARTITION BY` = "restart for each group" (like `GROUP BY`, but without collapsing).
- `ORDER BY` inside `OVER` = the order in which the window function looks at rows.

## 12.3 Your first window: each row next to the grand total

First, per-customer delivered spend (use a CTE), and each customer's **share of the total**:

```sql
WITH customer_spend AS (
    SELECT name, SUM(line_total) AS total_spent
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY name
)
SELECT name,
       total_spent,
       SUM(total_spent) OVER ()                                   AS grand_total,
       ROUND(100.0 * total_spent / SUM(total_spent) OVER (), 1)   AS pct_of_total
FROM customer_spend
ORDER BY total_spent DESC;
```

| name | total_spent | grand_total | pct_of_total |
|---|---|---|---|
| Asha | 67500 | 147480 | 45.8 |
| Divya | 55000 | 147480 | 37.3 |
| Meena | 10000 | 147480 | 6.8 |
| Ravi | 6200 | 147480 | 4.2 |
| Kiran | 4700 | 147480 | 3.2 |
| Arjun | 3600 | 147480 | 2.4 |
| Sneha | 480 | 147480 | 0.3 |

All 7 rows survive and each carries the grand total. With `GROUP BY` that would be impossible in one step.

## 12.4 PARTITION BY: a separate window per group

*"Show each product's delivered revenue next to its category total."*

```sql
WITH product_rev AS (
    SELECT category, product_name, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY category, product_name
)
SELECT category,
       product_name,
       revenue,
       SUM(revenue) OVER (PARTITION BY category) AS category_total
FROM product_rev
ORDER BY category, revenue DESC;
```

| category | product_name | revenue | category_total |
|---|---|---|---|
| Accessories | Backpack | 5400 | 5400 |
| Electronics | Laptop | 110000 | 125000 |
| Electronics | Headphones | 15000 | 125000 |
| Furniture | Desk Chair | 15000 | 15000 |
| Stationery | Notebook | 1680 | 2080 |
| Stationery | Pen Pack | 400 | 2080 |

## 12.5 Other useful window functions

| Function | What it does |
|---|---|
| `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE(n)` | ranking (Section 13) |
| `SUM/AVG/COUNT/MIN/MAX() OVER (...)` | running or group aggregates (Section 14) |
| `LAG(col, n)` / `LEAD(col, n)` | value from n rows before / after |
| `FIRST_VALUE(col)` / `LAST_VALUE(col)` | first/last value in the window |

**LAG example (previous order date per customer):**

```sql
SELECT name, order_id, order_date,
       LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS previous_order_date
FROM order_lines
WHERE customer_id IN (1, 2)
ORDER BY customer_id, order_date;
```

| name | order_id | order_date | previous_order_date |
|---|---|---|---|
| Asha | 1 | 2025-02-01 | NULL |
| Asha | 2 | 2025-02-15 | 2025-02-01 |
| Asha | 11 | 2025-07-01 | 2025-02-15 |
| Ravi | 3 | 2025-03-01 | NULL |
| Ravi | 4 | 2025-03-20 | 2025-03-01 |
| Ravi | 15 | 2025-09-02 | 2025-03-20 |

**FIRST_VALUE example (each order vs the customer's first product):**

```sql
SELECT name, order_date, product_name,
       FIRST_VALUE(product_name) OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS first_product
FROM order_lines
WHERE customer_id = 1
ORDER BY order_date;
```

| name | order_date | product_name | first_product |
|---|---|---|---|
| Asha | 2025-02-01 | Laptop | Laptop |
| Asha | 2025-02-15 | Headphones | Laptop |
| Asha | 2025-07-01 | Desk Chair | Laptop |

## 12.6 GROUP BY vs window: at a glance

| | GROUP BY | Window function |
|---|---|---|
| Rows returned | one per group | all original rows |
| Use when | you want a summary table | you want row-level detail *plus* group context |
| Can combine? | Yes: window functions run *after* GROUP BY (that's why we used a CTE above) | |

> Window functions run after `WHERE`, `GROUP BY`, and `HAVING`. That is why you can't filter on them directly in `WHERE`; wrap them in a CTE or subquery and filter outside (see Top-N below).

---

# 13. Ranking

Ranking functions number rows according to an `ORDER BY`. They differ in how they treat **ties**.

| Function | Ties get… | Example sequence for values 10, 10, 5, 4 |
|---|---|---|
| `ROW_NUMBER()` | different numbers (arbitrary among ties) | 1, 2, 3, 4 |
| `RANK()` | same rank, then **skips** numbers | 1, 1, 3, 4 |
| `DENSE_RANK()` | same rank, **no skipping** | 1, 1, 2, 3 |

## 13.1 The three side by side

```sql
SELECT order_id,
       quantity,
       ROW_NUMBER() OVER (ORDER BY quantity DESC, order_id) AS row_num,
       RANK()       OVER (ORDER BY quantity DESC)           AS rnk,
       DENSE_RANK() OVER (ORDER BY quantity DESC)           AS dense_rnk
FROM orders
ORDER BY quantity DESC, order_id
LIMIT 8;
```

| order_id | quantity | row_num | rnk | dense_rnk |
|---|---|---|---|---|
| 3 | 10 | 1 | 1 | 1 |
| 14 | 10 | 2 | 1 | 1 |
| 7 | 5 | 3 | 3 | 2 |
| 9 | 4 | 4 | 4 | 3 |
| 2 | 2 | 5 | 5 | 4 |
| 10 | 2 | 6 | 5 | 4 |
| 15 | 2 | 7 | 5 | 4 |
| 1 | 1 | 8 | 8 | 5 |

Look at orders 3 and 14 (both quantity 10): `RANK` gives both 1 and then jumps to 3; `DENSE_RANK` gives both 1 and continues with 2.

## 13.2 Ranking customers by spend

```sql
WITH customer_spend AS (
    SELECT name, SUM(line_total) AS total_spent
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY name
)
SELECT name, total_spent,
       RANK() OVER (ORDER BY total_spent DESC) AS spend_rank
FROM customer_spend
ORDER BY spend_rank;
```

| name | total_spent | spend_rank |
|---|---|---|
| Asha | 67500 | 1 |
| Divya | 55000 | 2 |
| Meena | 10000 | 3 |
| Ravi | 6200 | 4 |
| Kiran | 4700 | 5 |
| Arjun | 3600 | 6 |
| Sneha | 480 | 7 |

## 13.3 Ranking inside groups (PARTITION BY)

```sql
WITH product_rev AS (
    SELECT category, product_name, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY category, product_name
)
SELECT category, product_name, revenue,
       ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
FROM product_rev
ORDER BY category, rn;
```

| category | product_name | revenue | rn |
|---|---|---|---|
| Accessories | Backpack | 5400 | 1 |
| Electronics | Laptop | 110000 | 1 |
| Electronics | Headphones | 15000 | 2 |
| Furniture | Desk Chair | 15000 | 1 |
| Stationery | Notebook | 1680 | 1 |
| Stationery | Pen Pack | 400 | 2 |

The numbering **restarts at 1** in each category.

## 13.4 Top-N per group (the most common ranking pattern)

*"The best-selling product in each category."* You can't write `WHERE rn = 1` in the same query that defines `rn`, so wrap it in a CTE:

```sql
WITH product_rev AS (
    SELECT category, product_name, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY category, product_name
),
ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rn
    FROM product_rev
)
SELECT category, product_name, revenue
FROM ranked
WHERE rn = 1
ORDER BY category;
```

| category | product_name | revenue |
|---|---|---|
| Accessories | Backpack | 5400 |
| Electronics | Laptop | 110000 |
| Furniture | Desk Chair | 15000 |
| Stationery | Notebook | 1680 |

Change `rn = 1` to `rn <= 2` for the top 2 per category.

**Which ranking function for Top-N?** Use `ROW_NUMBER` for "exactly N rows"; use `RANK`/`DENSE_RANK` when ties should all be included.

## 13.5 Latest record per customer

*"Each customer's most recent delivered order."*

```sql
WITH ranked AS (
    SELECT name, order_id, order_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id
                              ORDER BY order_date DESC, order_id DESC) AS rn
    FROM order_lines
    WHERE status = 'delivered'
)
SELECT name, order_id, order_date
FROM ranked
WHERE rn = 1
ORDER BY name;
```

| name | order_id | order_date |
|---|---|---|
| Arjun | 10 | 2025-06-15 |
| Asha | 11 | 2025-07-01 |
| Divya | 13 | 2025-08-05 |
| Kiran | 16 | 2025-09-14 |
| Meena | 12 | 2025-07-19 |
| Ravi | 15 | 2025-09-02 |
| Sneha | 9 | 2025-06-08 |

## 13.6 NTILE: split into equal-sized buckets

```sql
WITH customer_spend AS (
    SELECT name, SUM(line_total) AS total_spent
    FROM order_lines WHERE status = 'delivered' GROUP BY name
)
SELECT name, total_spent,
       NTILE(4) OVER (ORDER BY total_spent DESC) AS quartile
FROM customer_spend
ORDER BY total_spent DESC;
```

| name | total_spent | quartile |
|---|---|---|
| Asha | 67500 | 1 |
| Divya | 55000 | 1 |
| Meena | 10000 | 2 |
| Ravi | 6200 | 2 |
| Kiran | 4700 | 3 |
| Arjun | 3600 | 3 |
| Sneha | 480 | 4 |

(7 customers into 4 buckets: sizes 2, 2, 2, 1.) Quartile 1 = your top customers.

---

# 14. Running Totals and Moving Averages

First, build the **monthly revenue** result we'll reuse (delivered orders only):

```sql
SELECT strftime('%Y-%m', order_date) AS month,
       SUM(line_total)               AS revenue
FROM order_lines
WHERE status = 'delivered'
GROUP BY month
ORDER BY month;
```

| month | revenue |
|---|---|
| 2025-02 | 60000 |
| 2025-03 | 1200 |
| 2025-04 | 10000 |
| 2025-05 | 400 |
| 2025-06 | 4080 |
| 2025-07 | 10000 |
| 2025-08 | 55000 |
| 2025-09 | 6800 |

## 14.1 Running total (cumulative sum)

```sql
WITH monthly AS (
    SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY month
)
SELECT month,
       revenue,
       SUM(revenue) OVER (ORDER BY month
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM monthly
ORDER BY month;
```

| month | revenue | running_total |
|---|---|---|
| 2025-02 | 60000 | 60000 |
| 2025-03 | 1200 | 61200 |
| 2025-04 | 10000 | 71200 |
| 2025-05 | 400 | 71600 |
| 2025-06 | 4080 | 75680 |
| 2025-07 | 10000 | 85680 |
| 2025-08 | 55000 | 140680 |
| 2025-09 | 6800 | 147480 |

The last running total equals the overall delivered revenue: `147480`. That's a nice sanity check.

**The frame explained:** `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` means "from the very first row up to the current row". Frame options:

| Frame | Meaning |
|---|---|
| `UNBOUNDED PRECEDING` | the first row of the partition |
| `n PRECEDING` | n rows before the current row |
| `CURRENT ROW` | this row |
| `n FOLLOWING` | n rows after |
| `UNBOUNDED FOLLOWING` | the last row |

## 14.2 Running total per group (PARTITION BY)

*"Cumulative spending for each customer over time."*

```sql
SELECT name, order_date, line_total,
       SUM(line_total) OVER (PARTITION BY customer_id
                             ORDER BY order_date, order_id
                             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_spend
FROM order_lines
WHERE status = 'delivered' AND name IN ('Asha', 'Ravi')
ORDER BY name, order_date;
```

| name | order_date | line_total | cumulative_spend |
|---|---|---|---|
| Asha | 2025-02-01 | 55000 | 55000 |
| Asha | 2025-02-15 | 5000 | 60000 |
| Asha | 2025-07-01 | 7500 | 67500 |
| Ravi | 2025-03-01 | 1200 | 1200 |
| Ravi | 2025-09-02 | 5000 | 6200 |

The running total **restarts** for each customer.

## 14.3 Month-over-month change with LAG

```sql
WITH monthly AS (
    SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY month
)
SELECT month,
       revenue,
       LAG(revenue) OVER (ORDER BY month)                       AS prev_month,
       revenue - LAG(revenue) OVER (ORDER BY month)             AS change,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month))
                   / LAG(revenue) OVER (ORDER BY month), 1)     AS pct_change
FROM monthly
ORDER BY month;
```

| month | revenue | prev_month | change | pct_change |
|---|---|---|---|---|
| 2025-02 | 60000 | NULL | NULL | NULL |
| 2025-03 | 1200 | 60000 | -58800 | -98.0 |
| 2025-04 | 10000 | 1200 | 8800 | 733.3 |
| 2025-05 | 400 | 10000 | -9600 | -96.0 |
| 2025-06 | 4080 | 400 | 3680 | 920.0 |
| 2025-07 | 10000 | 4080 | 5920 | 145.1 |
| 2025-08 | 55000 | 10000 | 45000 | 450.0 |
| 2025-09 | 6800 | 55000 | -48200 | -87.6 |

The first month has no previous month, so `LAG` returns NULL. You can give it a default: `LAG(revenue, 1, 0)`.

`LEAD` is the mirror image (looks at the *next* row). `LAG(revenue, 12)` would compare with the same month last year, on data that spans multiple years.

## 14.4 Moving (rolling) average

A 3-month moving average smooths spikes:

```sql
WITH monthly AS (
    SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY month
)
SELECT month,
       revenue,
       ROUND(AVG(revenue) OVER (ORDER BY month
                                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3m
FROM monthly
ORDER BY month;
```

| month | revenue | moving_avg_3m |
|---|---|---|
| 2025-02 | 60000 | 60000.0 |
| 2025-03 | 1200 | 30600.0 |
| 2025-04 | 10000 | 23733.33 |
| 2025-05 | 400 | 3866.67 |
| 2025-06 | 4080 | 4826.67 |
| 2025-07 | 10000 | 4826.67 |
| 2025-08 | 55000 | 23026.67 |
| 2025-09 | 6800 | 23933.33 |

`2 PRECEDING AND CURRENT ROW` = the current month plus the two before it. The first two rows average fewer than 3 months.

## 14.5 Each month's share of the year-to-date total

```sql
WITH monthly AS (
    SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
    FROM order_lines WHERE status = 'delivered' GROUP BY month
)
SELECT month, revenue,
       ROUND(100.0 * revenue / SUM(revenue) OVER (), 1) AS pct_of_period
FROM monthly
ORDER BY month;
```

| month | revenue | pct_of_period |
|---|---|---|
| 2025-02 | 60000 | 40.7 |
| 2025-03 | 1200 | 0.8 |
| 2025-04 | 10000 | 6.8 |
| 2025-05 | 400 | 0.3 |
| 2025-06 | 4080 | 2.8 |
| 2025-07 | 10000 | 6.8 |
| 2025-08 | 55000 | 37.3 |
| 2025-09 | 6800 | 4.6 |

---

# 15. Date-Based Analysis

Business questions are usually about *time*: trends, seasonality, growth, retention. Here are the standard patterns.

## 15.1 Trend by month (recap)

Use `strftime('%Y-%m', date)` (SQLite), `DATE_TRUNC('month', date)` (PostgreSQL), or `DATE_FORMAT(date, '%Y-%m')` (MySQL), then `GROUP BY` it. See Section 14 for the full example with the monthly revenue table.

## 15.2 Quarterly analysis

A quarter is `(month + 2) / 3` using integer division:

| Month | 1–3 | 4–6 | 7–9 | 10–12 |
|---|---|---|---|---|
| Quarter | 1 | 2 | 3 | 4 |

```sql
SELECT 'Q' || ((CAST(strftime('%m', order_date) AS INTEGER) + 2) / 3) AS quarter,
       COUNT(*)        AS orders,
       SUM(line_total) AS delivered_revenue
FROM order_lines
WHERE status = 'delivered'
GROUP BY quarter
ORDER BY quarter;
```

| quarter | orders | delivered_revenue |
|---|---|---|
| Q1 | 3 | 61200 |
| Q2 | 5 | 14480 |
| Q3 | 5 | 71800 |

Sanity check: 3 + 5 + 5 = 13 delivered orders, and 61200 + 14480 + 71800 = 147480, which matches the total from Section 5.3. (Order 4 in March was cancelled, so Q1 has only orders 1, 2 and 3.)

## 15.3 Filtering a date range correctly

Use a **half-open range**: `>= start` and `< next_start`. It works for both dates and timestamps and never misses the last day.

```sql
-- All of June 2025
SELECT order_id, order_date
FROM orders
WHERE order_date >= '2025-06-01' AND order_date < '2025-07-01';
```

| order_id | order_date |
|---|---|
| 9 | 2025-06-08 |
| 10 | 2025-06-15 |

Why not `BETWEEN '2025-06-01' AND '2025-06-30 '`? If your column holds timestamps like `2025-06-30 14:00:00`, that row is *after* `'2025-06-30'` and gets missed.

**Relative dates ("last 30 days")** use "now":

```sql
WHERE order_date >= date('now', '-30 days')      -- SQLite
-- PostgreSQL: WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
```
(Results change every day, so this repo's fixed sample data won't return rows for it.)

## 15.4 Time between events

*"How many days after signing up did each customer place their first order?"*

```sql
WITH first_order AS (
    SELECT customer_id, MIN(order_date) AS first_order_date
    FROM orders
    GROUP BY customer_id
)
SELECT c.name,
       c.signup_date,
       f.first_order_date,
       CAST(julianday(f.first_order_date) - julianday(c.signup_date) AS INTEGER) AS days_to_first_order
FROM customers c
JOIN first_order f ON f.customer_id = c.customer_id
ORDER BY c.customer_id;
```

| name | signup_date | first_order_date | days_to_first_order |
|---|---|---|---|
| Asha | 2025-01-15 | 2025-02-01 | 17 |
| Ravi | 2025-02-20 | 2025-03-01 | 9 |
| Meena | 2025-03-05 | 2025-04-10 | 36 |
| Kiran | 2025-03-18 | 2025-04-12 | 25 |
| Sneha | 2025-04-02 | 2025-05-20 | 48 |
| Arjun | 2025-05-11 | 2025-06-15 | 35 |
| Divya | 2025-06-25 | 2025-08-05 | 41 |

Averaging that last column gives 211 / 7 = **30.14 days** on average from signup to first order.

## 15.5 Days between consecutive orders (LAG + dates)

```sql
SELECT name, order_date,
       CAST(julianday(order_date)
            - julianday(LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)) AS INTEGER) AS days_since_prev
FROM order_lines
WHERE customer_id = 1
ORDER BY order_date;
```

| name | order_date | days_since_prev |
|---|---|---|
| Asha | 2025-02-01 | NULL |
| Asha | 2025-02-15 | 14 |
| Asha | 2025-07-01 | 136 |

## 15.6 Repeat vs one-time customers

```sql
SELECT CASE WHEN n_orders = 1 THEN 'One-time' ELSE 'Repeat' END AS customer_type,
       COUNT(*) AS customers
FROM (
    SELECT customer_id, COUNT(*) AS n_orders
    FROM orders
    GROUP BY customer_id
) t
GROUP BY customer_type;
```

| customer_type | customers |
|---|---|
| One-time | 1 |
| Repeat | 6 |

## 15.7 Filling gaps: months with zero sales

`GROUP BY` only returns months that **have** data. To show every month (including empty ones), generate a calendar with a recursive CTE, then `LEFT JOIN`:

```sql
WITH RECURSIVE months(m) AS (
    SELECT '2025-01-01'
    UNION ALL
    SELECT date(m, '+1 month') FROM months WHERE m < '2025-10-01'
),
monthly AS (
    SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
    FROM order_lines WHERE status = 'delivered' GROUP BY month
)
SELECT strftime('%Y-%m', months.m) AS month,
       COALESCE(monthly.revenue, 0) AS revenue
FROM months
LEFT JOIN monthly ON monthly.month = strftime('%Y-%m', months.m)
ORDER BY month;
```

In our data January (2025-01) and October (2025-10) come back with revenue `0`, while Feb–Sep show the values from Section 14. Charts look wrong when gaps are silently missing, so this pattern matters.

## 15.8 Weekday vs weekend

```sql
SELECT CASE strftime('%w', order_date)
           WHEN '0' THEN 'Sunday'  WHEN '1' THEN 'Monday'   WHEN '2' THEN 'Tuesday'
           WHEN '3' THEN 'Wednesday' WHEN '4' THEN 'Thursday' WHEN '5' THEN 'Friday'
           WHEN '6' THEN 'Saturday' END AS weekday,
       COUNT(*) AS orders
FROM orders
GROUP BY strftime('%w', order_date)
ORDER BY strftime('%w', order_date);
```

Run it and inspect the pattern in your own results. Weekday analysis is a classic first look at seasonality.

## 15.9 A standard checklist for time analysis

1. Decide the **grain** (day, week, month, quarter).
2. Use consistent time zones and date formats.
3. Use half-open date ranges.
4. Fill gaps if you'll chart it.
5. Compare **like with like** (month vs same month last year, not vs a partial month).
6. Sanity-check totals against the raw table.

---

# 16. SQL Data Cleaning

Real data is messy: extra spaces, inconsistent capitalization, empty strings that should be NULL, duplicates, text that should be numbers. SQL can clean it, which is great because it's fast and reproducible.

## 16.1 A messy sample table

```sql
DROP TABLE IF EXISTS raw_contacts;
CREATE TABLE raw_contacts (
    id        INTEGER PRIMARY KEY,
    full_name TEXT,
    email     TEXT,
    phone     TEXT,
    city      TEXT
);

INSERT INTO raw_contacts VALUES
(1, '  asha kumar ', 'ASHA@Example.com',  '98765-43210', 'hyderabad '),
(2, 'RAVI SHARMA',   ' ravi@example.com', '9876543211',  'Bengaluru'),
(3, 'Meena Iyer',    NULL,                '',            'CHENNAI'),
(4, 'Asha Kumar',    'asha@example.com',  '9876543210',  'Hyderabad'),
(5, 'Kiran',         'N/A',               '9876543212',  'Hyderabad');
```

| id | full_name | email | phone | city |
|---|---|---|---|---|
| 1 | `'  asha kumar '` | ASHA@Example.com | 98765-43210 | `'hyderabad '` |
| 2 | RAVI SHARMA | `' ravi@example.com'` | 9876543211 | Bengaluru |
| 3 | Meena Iyer | NULL | `''` | CHENNAI |
| 4 | Asha Kumar | asha@example.com | 9876543210 | Hyderabad |
| 5 | Kiran | N/A | 9876543212 | Hyderabad |

Problems: stray spaces, mixed case, phone with a dash, an empty-string phone, `'N/A'` as a fake email, and rows 1 and 4 are the **same person**.

## 16.2 Step-by-step cleaning toolbox

### Trim whitespace and fix case

```sql
SELECT id,
       LOWER(TRIM(full_name)) AS name_clean,
       LOWER(TRIM(email))     AS email_lower
FROM raw_contacts;
```

| id | name_clean | email_lower |
|---|---|---|
| 1 | asha kumar | asha@example.com |
| 2 | ravi sharma | ravi@example.com |
| 3 | meena iyer | NULL |
| 4 | asha kumar | asha@example.com |
| 5 | kiran | n/a |

**Capitalize the first letter only** (SQLite has no `INITCAP`):

```sql
SELECT id,
       UPPER(SUBSTR(TRIM(city), 1, 1)) || LOWER(SUBSTR(TRIM(city), 2)) AS city_clean
FROM raw_contacts;
```

| id | city_clean |
|---|---|
| 1 | Hyderabad |
| 2 | Bengaluru |
| 3 | Chennai |
| 4 | Hyderabad |
| 5 | Hyderabad |

(PostgreSQL: `INITCAP(TRIM(city))`.)

### Remove unwanted characters

```sql
SELECT id, phone, REPLACE(REPLACE(TRIM(phone), '-', ''), ' ', '') AS phone_digits
FROM raw_contacts;
```

Row 1's `98765-43210` becomes `9876543210`.

### Turn empty strings and placeholders into real NULLs

Empty strings, `'N/A'`, `'-'`, `'unknown'` are all "missing" but SQL doesn't know that.

```sql
SELECT id,
       NULLIF(TRIM(phone), '')  AS phone_null,
       CASE WHEN LOWER(TRIM(email)) IN ('', 'n/a', 'na', 'null', '-') THEN NULL
            ELSE LOWER(TRIM(email)) END AS email_clean
FROM raw_contacts;
```

| id | phone_null | email_clean |
|---|---|---|
| 1 | 98765-43210 | asha@example.com |
| 2 | 9876543211 | ravi@example.com |
| 3 | NULL | NULL |
| 4 | 9876543210 | asha@example.com |
| 5 | 9876543212 | NULL |

### Fix data types

```sql
SELECT CAST('42' AS INTEGER) + 1 AS num,       -- 43
       CAST('3.5' AS REAL) * 2   AS dec,       -- 7.0
       CAST(42 AS TEXT) || ' units' AS txt;    -- '42 units'
```

### Standardize a date format

Suppose dates arrive as `DD/MM/YYYY` text and you want ISO `YYYY-MM-DD`:

```sql
SELECT '15/02/2025' AS raw_date,
       SUBSTR('15/02/2025', 7, 4) || '-' || SUBSTR('15/02/2025', 4, 2) || '-' || SUBSTR('15/02/2025', 1, 2) AS iso_date;
```

| raw_date | iso_date |
|---|---|
| 15/02/2025 | 2025-02-15 |

(SQLite: `SUBSTR(x, start, len)`; the day is characters 1–2, month 4–5, year 7–10.)

## 16.3 Put it all together in a cleaning CTE

```sql
WITH cleaned AS (
    SELECT id,
           LOWER(TRIM(full_name))                                           AS name_clean,
           CASE WHEN LOWER(TRIM(email)) IN ('', 'n/a') THEN NULL
                ELSE LOWER(TRIM(email)) END                                 AS email_clean,
           NULLIF(REPLACE(TRIM(phone), '-', ''), '')                        AS phone_clean,
           UPPER(SUBSTR(TRIM(city), 1, 1)) || LOWER(SUBSTR(TRIM(city), 2))  AS city_clean
    FROM raw_contacts
)
SELECT * FROM cleaned ORDER BY id;
```

| id | name_clean | email_clean | phone_clean | city_clean |
|---|---|---|---|---|
| 1 | asha kumar | asha@example.com | 9876543210 | Hyderabad |
| 2 | ravi sharma | ravi@example.com | 9876543211 | Bengaluru |
| 3 | meena iyer | NULL | NULL | Chennai |
| 4 | asha kumar | asha@example.com | 9876543210 | Hyderabad |
| 5 | kiran | NULL | 9876543212 | Hyderabad |

## 16.4 Finding duplicates

```sql
WITH cleaned AS (
    SELECT id, LOWER(TRIM(email)) AS email_clean FROM raw_contacts
)
SELECT email_clean, COUNT(*) AS copies
FROM cleaned
WHERE email_clean IS NOT NULL AND email_clean <> 'n/a'
GROUP BY email_clean
HAVING COUNT(*) > 1;
```

| email_clean | copies |
|---|---|
| asha@example.com | 2 |

## 16.5 Removing duplicates with ROW_NUMBER

Number the rows inside each "same person" group, then keep number 1:

```sql
WITH cleaned AS (
    SELECT id,
           LOWER(TRIM(full_name))                     AS name_clean,
           NULLIF(REPLACE(TRIM(phone), '-', ''), '')  AS phone_clean
    FROM raw_contacts
),
numbered AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY name_clean, phone_clean ORDER BY id) AS rn
    FROM cleaned
)
SELECT id FROM numbered WHERE rn = 1 ORDER BY id;
```

| id |
|---|
| 1 |
| 2 |
| 3 |
| 5 |

Row 4 is dropped (it's a duplicate of row 1). 

> **Why not partition by `email_clean`?** Rows 3 and 5 both have NULL emails, and `PARTITION BY` treats all NULLs as one group, so row 5 (a different person) would be wrongly deleted. **Choose duplicate keys that are reliably filled in.**

## 16.6 Save the clean result as a new table

Never edit raw data in place; create a new clean table so you can always redo it.

```sql
CREATE TABLE contacts_clean AS
WITH cleaned AS (
    SELECT id,
           LOWER(TRIM(full_name))                                           AS name_clean,
           CASE WHEN LOWER(TRIM(email)) IN ('', 'n/a') THEN NULL
                ELSE LOWER(TRIM(email)) END                                 AS email_clean,
           NULLIF(REPLACE(TRIM(phone), '-', ''), '')                        AS phone_clean,
           UPPER(SUBSTR(TRIM(city), 1, 1)) || LOWER(SUBSTR(TRIM(city), 2))  AS city_clean
    FROM raw_contacts
),
numbered AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY name_clean, phone_clean ORDER BY id) AS rn
    FROM cleaned
)
SELECT id, name_clean, email_clean, phone_clean, city_clean
FROM numbered
WHERE rn = 1;
```

## 16.7 UPDATE and DELETE (use with care)

If you *must* change data in place: **(1) write a `SELECT` with the same `WHERE` first and check the rows, (2) use a transaction, (3) keep a backup.**

```sql
-- 1) Preview
SELECT * FROM raw_contacts WHERE TRIM(phone) = '';

-- 2) Change inside a transaction
BEGIN;
UPDATE raw_contacts SET phone = NULL WHERE TRIM(phone) = '';
-- check the result; if it's wrong: ROLLBACK;  if it's right:
COMMIT;
```

`UPDATE` or `DELETE` without `WHERE` changes **every row**.

## 16.8 Data quality checks (run these on every new table)

```sql
-- 1. Orphan rows: orders pointing to a customer that doesn't exist (should return 0 rows)
SELECT o.order_id
FROM orders o
LEFT JOIN customers c ON c.customer_id = o.customer_id
WHERE c.customer_id IS NULL;

-- 2. Impossible values (should return 0 rows)
SELECT * FROM orders WHERE quantity <= 0;
SELECT * FROM products WHERE price < 0;

-- 3. Duplicate primary-key-like columns (should return 0 rows)
SELECT email, COUNT(*) FROM customers WHERE email IS NOT NULL GROUP BY email HAVING COUNT(*) > 1;

-- 4. Unexpected category values
SELECT DISTINCT status FROM orders;    -- expect: delivered, cancelled, returned, pending

-- 5. Missing values summary (see 8.10)

-- 6. Date sanity: any dates in the future or too old?
SELECT MIN(order_date), MAX(order_date) FROM orders;    -- 2025-02-01 .. 2025-09-14
```

## 16.9 Cleaning rules of thumb

1. Keep the raw table untouched; write cleaned output to a new table/view.
2. Clean in the order: **trim → case → replace → NULL-ify → cast → dedupe**.
3. Document every rule (what counts as "missing"? which duplicates to keep?).
4. Re-run counts before/after (`COUNT(*)`, `COUNT(DISTINCT ...)`) to prove nothing was lost accidentally.

---

# 17. Query Performance Basics

For small tables everything is fast. On millions of rows, how you write a query matters a lot. The following habits cover 90% of performance for analysts.

## 17.1 Think like the database: scan vs seek

**Analogy:** Finding "Photosynthesis" in a textbook. You can flip through every page (**full table scan**) or look it up in the book's **index** and jump straight to the right page (**index seek**).

A database **index** is a sorted lookup structure on a column (like a book's index). Queries that filter or join on an indexed column can skip most of the table.

## 17.2 See what the database plans to do: EXPLAIN

```sql
EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE customer_id = 1;
```

Without an index the plan says something like:

```
SCAN orders
```

Now create an index:

```sql
CREATE INDEX idx_orders_customer ON orders(customer_id);

EXPLAIN QUERY PLAN
SELECT * FROM orders WHERE customer_id = 1;
```

```
SEARCH orders USING INDEX idx_orders_customer (customer_id=?)
```

`SCAN` = read every row. `SEARCH ... USING INDEX` = jump directly. (Exact wording varies by version. In PostgreSQL use `EXPLAIN` or `EXPLAIN ANALYZE`; look for "Seq Scan" vs "Index Scan".)

## 17.3 Which columns to index

- Columns used in `WHERE` filters on big tables.
- Columns used in `JOIN ... ON` (foreign keys!).
- Columns used in `ORDER BY` / `GROUP BY` on big tables.

```sql
CREATE INDEX idx_orders_product   ON orders(product_id);
CREATE INDEX idx_orders_date      ON orders(order_date);
CREATE INDEX idx_orders_cust_date ON orders(customer_id, order_date);   -- composite index
```

**Trade-offs:** indexes take disk space and make `INSERT/UPDATE/DELETE` slower. Don't index everything. Primary keys are indexed automatically.

**Composite index rule:** an index on `(customer_id, order_date)` helps filters on `customer_id` alone, or on both, but not on `order_date` alone (the *leftmost* column must be used).

## 17.4 Write "sargable" conditions

*Sargable* = "Search ARGument ABLE" = the database can use an index. Wrapping the column in a function usually breaks it.

```sql
-- ❌ Function on the column: index on order_date can't be used
SELECT * FROM orders WHERE strftime('%Y', order_date) = '2025';

-- ✅ Range on the raw column: index can be used
SELECT * FROM orders WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01';
```

```sql
-- ❌ Leading wildcard: cannot use an index
WHERE name LIKE '%sha'
-- ✅ Trailing wildcard: can use an index
WHERE name LIKE 'As%'
```

```sql
-- ❌ Math on the column
WHERE price * 2 > 10000
-- ✅ Move the math to the constant
WHERE price > 5000
```

## 17.5 Habits that make queries cheaper

| Habit | Why |
|---|---|
| **Select only the columns you need** (avoid `SELECT *`) | less data to read and send |
| **Filter early** (`WHERE` before joining big tables; pre-aggregate in a CTE/subquery) | fewer rows flow through later steps |
| **Use `LIMIT` while exploring** | see a sample, don't pull millions of rows |
| **Prefer `UNION ALL` over `UNION`** when duplicates are impossible or fine | `UNION` sorts to remove duplicates |
| **Prefer `EXISTS` over `IN` on huge subqueries** | it can stop at the first match |
| **Avoid `DISTINCT` as a crutch** | it forces extra sorting; fix the join instead |
| **Avoid correlated subqueries on big tables** | they run once per outer row; use a join or window function |
| **Join on indexed, same-type columns** | fast lookups, no implicit conversions |
| **Aggregate before you join** when you only need totals | avoids multiplying rows |

## 17.6 Example: aggregate first, join second

Less efficient at scale (joins every order row to customers, then groups):

```sql
SELECT c.name, SUM(o.quantity) AS units
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.name;
```

Often better: shrink `orders` to one row per customer first, then join the small result:

```sql
WITH units_per_customer AS (
    SELECT customer_id, SUM(quantity) AS units
    FROM orders
    GROUP BY customer_id
)
SELECT c.name, u.units
FROM units_per_customer u
JOIN customers c ON c.customer_id = u.customer_id;
```

(Modern optimizers sometimes do this automatically, but knowing the idea helps when they don't.)

## 17.7 Views vs materialized views

- **View**: a saved query (like `order_lines`). No data stored; runs each time. Great for simplifying, not for speed.
- **Materialized view** (PostgreSQL, Oracle): stores the *result*, so reads are fast, but you must refresh it. Ideal for heavy dashboards.

## 17.8 A performance debugging routine

1. Run `EXPLAIN` on the slow query. Look for full scans on big tables.
2. Check row counts at each step (`SELECT COUNT(*)`) to spot join explosions.
3. Add a `WHERE` that shrinks the data as early as possible.
4. Add an index on the filter/join column and re-run `EXPLAIN`.
5. Test on real volume; a query that's instant on 16 rows tells you nothing about 16 million.
6. Ask before running heavy queries on a production database, and use a read replica if available.

---

# 18. Common Mistakes

| Mistake | What happens | Fix |
|---|---|---|
| `WHERE col = NULL` | Returns nothing | `WHERE col IS NULL` |
| Mixing `AND`/`OR` without parentheses | Wrong rows | Add parentheses |
| `NOT IN` with a subquery that can return NULL | Returns zero rows | `NOT EXISTS` or filter NULLs out |
| Selecting non-aggregated columns without `GROUP BY` | Error or meaningless result | Add them to `GROUP BY` or aggregate them |
| Using an alias in `WHERE` | "no such column" | Repeat the expression, or use a CTE |
| `HAVING` where `WHERE` belongs | Slower; sometimes errors | Row filters go in `WHERE` |
| `COUNT(*)` after a LEFT JOIN | Counts NULL rows as 1 | `COUNT(right_table.key)` |
| Filtering the right table of a LEFT JOIN in `WHERE` | Silently becomes INNER JOIN | Put the condition in `ON` |
| Joining without checking row counts | Duplicates/fan-out inflate sums | Compare counts before/after |
| Integer division (`7/2 = 3`) | Wrong averages/percentages | Multiply by `1.0` or use `/ 2.0` |
| `BETWEEN` on timestamps | Misses the last day | Half-open ranges |
| Forgetting `ORDER BY` with `LIMIT` | "Random" rows | Always sort before limiting |
| `RANK` vs `ROW_NUMBER` confusion | Missing/extra rows in Top-N | Choose by tie behavior |
| `UPDATE`/`DELETE` without `WHERE` | Changes every row | Preview with `SELECT`, use transactions |
| `SELECT *` in production queries | Slow, fragile | List columns |
| Assuming row order without `ORDER BY` | Inconsistent results | Sort explicitly |

---

# 19. Practice Exercises (with Solutions)

Use the shop database from Section 0. Try each yourself before looking at the solution.

## Beginner

**1.** List the names of customers in Hyderabad, sorted alphabetically.

<details><summary>Solution</summary>

```sql
SELECT name FROM customers WHERE city = 'Hyderabad' ORDER BY name;
```
→ Asha, Divya, Kiran
</details>

**2.** Show product name and price for products costing between 100 and 3000, cheapest first.

<details><summary>Solution</summary>

```sql
SELECT product_name, price FROM products
WHERE price BETWEEN 100 AND 3000
ORDER BY price;
```
→ Notebook 120, Backpack 1800, Headphones 2500
</details>

**3.** How many orders were cancelled or returned?

<details><summary>Solution</summary>

```sql
SELECT COUNT(*) AS problem_orders FROM orders WHERE status IN ('cancelled', 'returned');
```
→ 2
</details>

**4.** List customers who have no email, showing name and city.

<details><summary>Solution</summary>

```sql
SELECT name, city FROM customers WHERE email IS NULL;
```
→ Meena (Chennai), Arjun (Bengaluru)
</details>

**5.** How many different products have been ordered?

<details><summary>Solution</summary>

```sql
SELECT COUNT(DISTINCT product_id) AS products_ordered FROM orders;
```
→ 6
</details>

## Intermediate

**6.** For each city, count *delivered* orders, showing only cities with 3 or more.

<details><summary>Solution</summary>

```sql
SELECT city, COUNT(*) AS delivered_orders
FROM order_lines
WHERE status = 'delivered'
GROUP BY city
HAVING COUNT(*) >= 3
ORDER BY delivered_orders DESC;
```
→ Hyderabad 7, Bengaluru 3
</details>

**7.** List customers who have never ordered a Laptop.

<details><summary>Solution</summary>

```sql
SELECT name FROM customers
WHERE customer_id NOT IN (SELECT customer_id FROM orders WHERE product_id = 1);
```
→ Ravi, Meena, Kiran, Arjun, Farhan. (Safe here because `orders.customer_id` has no NULLs.)
</details>

**8.** Find customers who bought from at least 3 different categories.

<details><summary>Solution</summary>

```sql
SELECT name, COUNT(DISTINCT category) AS categories
FROM order_lines
GROUP BY name
HAVING COUNT(DISTINCT category) >= 3
ORDER BY name;
```
→ Kiran 3, Ravi 3
</details>

**9.** Assign each customer a tier by delivered spend: Gold (≥ 50000), Silver (≥ 5000), Bronze (otherwise).

<details><summary>Solution</summary>

```sql
SELECT name,
       SUM(line_total) AS total_spent,
       CASE WHEN SUM(line_total) >= 50000 THEN 'Gold'
            WHEN SUM(line_total) >= 5000  THEN 'Silver'
            ELSE 'Bronze' END AS tier
FROM order_lines
WHERE status = 'delivered'
GROUP BY customer_id, name
ORDER BY total_spent DESC;
```
→ Asha 67500 Gold · Divya 55000 Gold · Meena 10000 Silver · Ravi 6200 Silver · Kiran 4700 Bronze · Arjun 3600 Bronze · Sneha 480 Bronze
</details>

**10.** Which month had the highest delivered revenue?

<details><summary>Solution</summary>

```sql
SELECT strftime('%Y-%m', order_date) AS month, SUM(line_total) AS revenue
FROM order_lines
WHERE status = 'delivered'
GROUP BY month
ORDER BY revenue DESC
LIMIT 1;
```
→ 2025-02, 60000
</details>

## Advanced

**11.** Show each customer's most recent delivered order. *(Solved in Section 13.5; try it from scratch.)*

**12.** For each month, show delivered revenue and the percentage change from the previous month. *(See Section 14.3.)*

**13.** Rank products within their category by units sold (delivered) and show only the top product per category.

<details><summary>Solution</summary>

```sql
WITH units AS (
    SELECT category, product_name, SUM(quantity) AS units
    FROM order_lines
    WHERE status = 'delivered'
    GROUP BY category, product_name
),
ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY units DESC) AS rn
    FROM units
)
SELECT category, product_name, units
FROM ranked
WHERE rn = 1
ORDER BY category;
```
→ Accessories Backpack 3 · Electronics Headphones 6 · Furniture Desk Chair 2 · Stationery Notebook 14
</details>

**14.** Using `raw_contacts` from Section 16, produce a clean, de-duplicated table where emails are lowercase, empty/`N/A` emails are NULL, and phones contain digits only. *(See Section 16.6.)*

**15.** *Challenge:* For each customer, compute the number of days between their first and last order (only customers with 2+ orders).

<details><summary>Solution</summary>

```sql
SELECT name,
       MIN(order_date) AS first_order,
       MAX(order_date) AS last_order,
       CAST(julianday(MAX(order_date)) - julianday(MIN(order_date)) AS INTEGER) AS days_active,
       COUNT(*) AS orders
FROM order_lines
GROUP BY customer_id, name
HAVING COUNT(*) >= 2
ORDER BY days_active DESC;
```
→ Ravi (2025-03-01 → 2025-09-14?) *Run it and check:* Ravi's orders are on 03-01, 03-20, 09-02 → 185 days; Asha 02-01 → 07-01 → 150 days; Kiran 04-12 → 09-14 → 155 days; Meena 04-10 → 07-19 → 100 days; Divya 08-05 → 08-22 → 17 days; Sneha 05-20 → 06-08 → 19 days.
</details>

---

# 20. Cheat Sheet and Glossary

## 20.1 Query template

```sql
WITH step1 AS (...),                 -- optional named steps
     step2 AS (...)
SELECT   col, AGG(col2) AS alias,    -- what to show
         window_fn() OVER (...)      -- optional window columns
FROM     table1 t1
JOIN     table2 t2 ON t2.key = t1.key
WHERE    row_condition               -- filters rows
GROUP BY col                         -- makes groups
HAVING   AGG(col2) > n               -- filters groups
ORDER BY alias DESC                  -- sorts
LIMIT    10;                         -- keeps top rows
```

## 20.2 Order SQL executes

`FROM/JOIN → WHERE → GROUP BY → HAVING → window functions → SELECT → DISTINCT → ORDER BY → LIMIT`

## 20.3 Quick reference

| Need | Use |
|---|---|
| Rename column | `AS` |
| Filter rows | `WHERE` |
| Filter groups | `HAVING` |
| Match list / range / pattern | `IN` / `BETWEEN` / `LIKE` |
| Missing values | `IS NULL`, `COALESCE`, `NULLIF` |
| If/else | `CASE WHEN … THEN … ELSE … END` |
| Unique values | `DISTINCT`, `COUNT(DISTINCT x)` |
| Combine tables sideways | `JOIN` (`INNER`, `LEFT`, …) |
| Stack tables vertically | `UNION [ALL]` |
| Inner steps | Subquery / CTE (`WITH`) |
| Row-level + group context | `… OVER (PARTITION BY … ORDER BY …)` |
| Number rows | `ROW_NUMBER`, `RANK`, `DENSE_RANK` |
| Running total | `SUM(x) OVER (ORDER BY d ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` |
| Compare to previous row | `LAG(x)` |
| Moving average | `AVG(x) OVER (ORDER BY d ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` |
| Speed up | filter early, select needed columns, `CREATE INDEX`, `EXPLAIN` |

## 20.4 Join cheat

| I want… | Use |
|---|---|
| Only matching rows | `INNER JOIN` |
| All from A, plus matches from B | `A LEFT JOIN B` |
| Rows in A with **no** match in B | `A LEFT JOIN B … WHERE B.key IS NULL` |
| Table related to itself | Self join with two aliases |

## 20.5 Glossary

| Term | Meaning |
|---|---|
| **Aggregate** | Function that summarizes many rows into one value (`SUM`, `COUNT`) |
| **Alias** | A temporary name for a column or table (`AS`) |
| **CTE** | Common Table Expression: a named step defined with `WITH` |
| **Correlated subquery** | Inner query that depends on the current outer row |
| **Cross join** | Every row of A paired with every row of B |
| **Dialect** | A database's own variant of SQL (SQLite, PostgreSQL, MySQL…) |
| **Fan-out** | Duplicates created by joining one row to many |
| **Foreign key** | Column that references another table's primary key |
| **Frame** | The set of rows a window function looks at (`ROWS BETWEEN …`) |
| **Grain** | What one row represents (one order? one customer-month?) |
| **Index** | A sorted lookup structure that speeds up searches |
| **NULL** | Missing/unknown value |
| **Partition** | A group of rows within a window (`PARTITION BY`) |
| **Primary key** | Column(s) uniquely identifying each row |
| **Query plan** | The database's strategy for running a query (`EXPLAIN`) |
| **Sargable** | Written so an index can be used |
| **Schema** | Blueprint of tables, columns, and types |
| **Subquery** | A query nested inside another query |
| **View** | A saved query you can use like a table |
| **Window function** | Calculation across related rows that keeps every row |

---

## ✅ Module 2 Self-Check

Before Module 3, you should be able to say "yes" to each:

- [ ] I can explain tables, rows, columns, primary keys, and foreign keys.
- [ ] I know the *execution* order of a query and why aliases don't work in `WHERE`.
- [ ] I can filter with `WHERE` (`AND/OR`, `IN`, `BETWEEN`, `LIKE`), sort with `ORDER BY`, and use `LIMIT` and `DISTINCT`.
- [ ] I can summarize with aggregates, `GROUP BY`, and `HAVING`, and explain `WHERE` vs `HAVING`.
- [ ] I can build labels and pivots with `CASE`.
- [ ] I handle NULLs with `IS NULL`, `COALESCE`, `NULLIF`, and I know the `NOT IN` trap.
- [ ] I can use string and date functions and know they differ by database.
- [ ] I can `INNER JOIN` and `LEFT JOIN` (including finding rows with no match) without creating duplicates.
- [ ] I can write subqueries and CTEs and choose between them.
- [ ] I can use window functions for ranking, Top-N per group, running totals, `LAG`, and moving averages.
- [ ] I can analyze data over time (monthly/quarterly trends, time between events, gap filling).
- [ ] I can clean messy text/duplicates in SQL and run data quality checks.
- [ ] I can read a query plan, add an index, and write index-friendly conditions.

**Next up: Module 3.** With data acquired and queried, you'll move on to the next stage of the analytics workflow.