# PostgreSQL Cheat Sheet for Developers

A quick reference for PostgreSQL (covers 12+, with notes on newer features through 17).

---

## Table of Contents

- [Data Types](#data-types)
- [DDL - Data Definition](#ddl---data-definition)
- [DML - Data Manipulation](#dml---data-manipulation)
- [SELECT Essentials](#select-essentials)
- [Joins](#joins)
- [Aggregations & Grouping](#aggregations--grouping)
- [Subqueries & CTEs](#subqueries--ctes)
- [Window Functions](#window-functions)
- [String Functions](#string-functions)
- [Date & Time Functions](#date--time-functions)
- [Numeric & Conversion Functions](#numeric--conversion-functions)
- [Arrays](#arrays)
- [JSON & JSONB](#json--jsonb)
- [Conditional Logic](#conditional-logic)
- [UPSERT (ON CONFLICT)](#upsert-on-conflict)
- [Functions & Procedures](#functions--procedures)
- [Triggers](#triggers)
- [Indexes](#indexes)
- [Full-Text Search](#full-text-search)
- [Transactions](#transactions)
- [Error Handling (PL/pgSQL)](#error-handling-plpgsql)
- [Roles & Permissions](#roles--permissions)
- [psql Meta-Commands](#psql-meta-commands)
- [Performance Tips](#performance-tips)
- [Useful System Catalogs](#useful-system-catalogs)

---

## Data Types

| Category | Types |
|----------|-------|
| **Integer** | `SMALLINT` (2B), `INTEGER`/`INT` (4B), `BIGINT` (8B) |
| **Serial (auto-increment)** | `SMALLSERIAL`, `SERIAL`, `BIGSERIAL` *(legacy — prefer `GENERATED AS IDENTITY`)* |
| **Decimal** | `NUMERIC(p,s)` / `DECIMAL(p,s)`, `REAL` (4B float), `DOUBLE PRECISION` (8B float) |
| **Money** | `MONEY` *(locale-dependent; prefer `NUMERIC`)* |
| **Character** | `CHAR(n)`, `VARCHAR(n)`, `TEXT` |
| **Date/Time** | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ`, `INTERVAL` |
| **Boolean** | `BOOLEAN` (`TRUE`/`FALSE`/`NULL`) |
| **Binary** | `BYTEA` |
| **UUID** | `UUID` |
| **JSON** | `JSON`, `JSONB` *(prefer `JSONB` — indexed, binary)* |
| **Array** | any type with `[]` (e.g., `INTEGER[]`, `TEXT[]`) |
| **Network** | `INET`, `CIDR`, `MACADDR` |
| **Geometric** | `POINT`, `LINE`, `POLYGON`, etc. |
| **Range** | `INT4RANGE`, `TSRANGE`, `DATERANGE`, etc. |
| **Other** | `ENUM` (user-defined), `XML`, `HSTORE` (extension) |

> **Tips:** Use `TEXT` instead of `VARCHAR(n)` unless you need a hard limit — same performance, no truncation surprises. Use `TIMESTAMPTZ` over `TIMESTAMP` for anything user-facing. Use `JSONB` over `JSON` unless you need to preserve key order.

---

## DDL - Data Definition

```sql
-- Create database
CREATE DATABASE sales_db;

-- Create schema
CREATE SCHEMA sales;

-- Create table
CREATE TABLE sales.customers (
    customer_id  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name   TEXT NOT NULL,
    last_name    TEXT NOT NULL,
    email        TEXT UNIQUE,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_active    BOOLEAN NOT NULL DEFAULT TRUE,
    metadata     JSONB,
    CONSTRAINT chk_email CHECK (email ~ '^[^@]+@[^@]+\.[^@]+$')
);

-- Alter table
ALTER TABLE sales.customers ADD COLUMN phone TEXT;
ALTER TABLE sales.customers ALTER COLUMN phone TYPE VARCHAR(30);
ALTER TABLE sales.customers ALTER COLUMN phone SET NOT NULL;
ALTER TABLE sales.customers ALTER COLUMN phone DROP NOT NULL;
ALTER TABLE sales.customers RENAME COLUMN phone TO phone_number;
ALTER TABLE sales.customers DROP COLUMN phone_number;

-- Foreign key
ALTER TABLE sales.orders
    ADD CONSTRAINT fk_orders_customers
    FOREIGN KEY (customer_id) REFERENCES sales.customers(customer_id)
    ON DELETE CASCADE
    ON UPDATE NO ACTION;

-- Drop
DROP TABLE IF EXISTS sales.customers CASCADE;
DROP INDEX IF EXISTS idx_customers_email;

-- Rename table
ALTER TABLE sales.customers RENAME TO clients;

-- Inheritance / partitioning (declarative, 10+)
CREATE TABLE measurements (
    city_id INT NOT NULL,
    logdate DATE NOT NULL,
    peaktemp INT
) PARTITION BY RANGE (logdate);

CREATE TABLE measurements_2024 PARTITION OF measurements
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

---

## DML - Data Manipulation

```sql
-- Insert single row
INSERT INTO sales.customers (first_name, last_name, email)
VALUES ('Ada', 'Lovelace', 'ada@example.com');

-- Insert multiple rows
INSERT INTO sales.customers (first_name, last_name, email)
VALUES
    ('Alan', 'Turing', 'alan@example.com'),
    ('Grace', 'Hopper', 'grace@example.com');

-- Insert and return generated values
INSERT INTO sales.customers (first_name, last_name, email)
VALUES ('Linus', 'Torvalds', 'linus@example.com')
RETURNING customer_id, created_at;

-- Insert from SELECT
INSERT INTO sales.customers_archive (first_name, last_name, email)
SELECT first_name, last_name, email
FROM sales.customers
WHERE is_active = FALSE;

-- Update
UPDATE sales.customers
SET is_active = FALSE,
    modified_at = NOW()
WHERE customer_id = 42
RETURNING *;

-- Update with JOIN (FROM clause)
UPDATE sales.customers c
SET status = 'VIP'
FROM sales.orders o
WHERE o.customer_id = c.customer_id
  AND o.total > 10000;

-- Delete
DELETE FROM sales.customers
WHERE is_active = FALSE
RETURNING customer_id, email;

-- Truncate (faster, resets identity)
TRUNCATE TABLE sales.staging_data RESTART IDENTITY CASCADE;
```

---

## SELECT Essentials

```sql
-- Basic
SELECT first_name, last_name
FROM sales.customers
WHERE is_active = TRUE
ORDER BY last_name ASC, first_name ASC
LIMIT 10;

-- DISTINCT
SELECT DISTINCT country FROM sales.customers;
SELECT DISTINCT ON (country) country, city, customer_id
FROM sales.customers
ORDER BY country, customer_id;  -- one row per country

-- Pagination
SELECT *
FROM sales.customers
ORDER BY customer_id
LIMIT 20 OFFSET 40;

-- Aliases
SELECT c.first_name AS "First Name", c.last_name AS "Last Name"
FROM sales.customers AS c;

-- Filtering operators
WHERE price BETWEEN 10 AND 50
WHERE country IN ('US', 'CA', 'UK')
WHERE email LIKE '%@gmail.com'
WHERE email ILIKE '%@GMAIL.COM'        -- case-insensitive
WHERE name ~ '^[A-Z]'                  -- regex match
WHERE name ~* 'smith'                  -- case-insensitive regex
WHERE phone IS NULL
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE tags && ARRAY['urgent', 'vip']   -- arrays overlap
WHERE metadata @> '{"verified": true}' -- JSONB contains
```

---

## Joins

```sql
-- INNER: only matching rows
SELECT c.first_name, o.order_id
FROM customers c
INNER JOIN orders o ON o.customer_id = c.customer_id;

-- LEFT: all left rows, NULLs when no match
SELECT c.first_name, o.order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id;

-- RIGHT / FULL OUTER
RIGHT JOIN       -- all right rows
FULL OUTER JOIN  -- all rows from both sides

-- CROSS JOIN (Cartesian product)
SELECT c.color, s.size FROM colors c CROSS JOIN sizes s;

-- Self join
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.employee_id = e.manager_id;

-- LATERAL (correlated join — like CROSS APPLY)
SELECT c.customer_id, recent.order_date
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_date FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY order_date DESC
    LIMIT 1
) recent;
-- LEFT JOIN LATERAL ... ON TRUE keeps left row even if no match

-- USING (shorthand for equi-join on same column name)
SELECT * FROM customers c JOIN orders o USING (customer_id);
```

---

## Aggregations & Grouping

```sql
SELECT
    country,
    COUNT(*)                   AS customer_count,
    COUNT(DISTINCT city)       AS city_count,
    COUNT(*) FILTER (WHERE is_active) AS active_count,  -- conditional aggregate
    SUM(lifetime_value)        AS total_ltv,
    AVG(lifetime_value)        AS avg_ltv,
    MIN(created_at)            AS first_signup,
    MAX(created_at)            AS latest_signup,
    ARRAY_AGG(first_name ORDER BY created_at) AS names,
    STRING_AGG(first_name, ', ' ORDER BY first_name) AS names_csv,
    JSONB_AGG(first_name)      AS names_json
FROM sales.customers
WHERE is_active = TRUE
GROUP BY country
HAVING COUNT(*) > 10
ORDER BY total_ltv DESC;

-- GROUPING SETS / ROLLUP / CUBE
SELECT country, city, COUNT(*)
FROM customers
GROUP BY ROLLUP (country, city);  -- subtotals + grand total

GROUP BY GROUPING SETS ((country), (city), ());
```

> `WHERE` filters rows **before** grouping; `HAVING` filters **after**.

---

## Subqueries & CTEs

```sql
-- Scalar subquery
SELECT first_name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count
FROM customers c;

-- IN / EXISTS
SELECT * FROM customers
WHERE customer_id IN (SELECT customer_id FROM orders WHERE total > 1000);

SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- Common Table Expression
WITH top_customers AS (
    SELECT customer_id, SUM(total) AS ltv
    FROM orders
    GROUP BY customer_id
    HAVING SUM(total) > 10000
)
SELECT c.*, tc.ltv
FROM customers c
JOIN top_customers tc USING (customer_id);

-- Multiple CTEs + data-modifying CTE
WITH archived AS (
    DELETE FROM customers WHERE is_active = FALSE
    RETURNING *
)
INSERT INTO customers_archive
SELECT * FROM archived;

-- Recursive CTE (e.g., org hierarchy)
WITH RECURSIVE org_chart AS (
    SELECT employee_id, name, manager_id, 0 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON oc.employee_id = e.manager_id
)
SELECT * FROM org_chart;

-- MATERIALIZED hint (12+) - force/prevent CTE materialization
WITH top AS MATERIALIZED (...)        -- force materialize
WITH top AS NOT MATERIALIZED (...)    -- inline (default for simple cases)
```

---

## Window Functions

```sql
SELECT
    order_id,
    customer_id,
    total,
    -- Ranking
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn,
    RANK()       OVER (ORDER BY total DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY total DESC) AS dense_rnk,
    NTILE(4)     OVER (ORDER BY total) AS quartile,
    PERCENT_RANK() OVER (ORDER BY total) AS pct_rank,

    -- Aggregates as windows
    SUM(total) OVER (PARTITION BY customer_id) AS customer_total,
    AVG(total) OVER (PARTITION BY customer_id) AS customer_avg,
    SUM(total) OVER (ORDER BY order_date
                     ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
    AVG(total) OVER (ORDER BY order_date
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7,

    -- Offset
    LAG(total, 1)  OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order,
    LEAD(total, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order,
    FIRST_VALUE(total) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order,
    LAST_VALUE(total)  OVER (PARTITION BY customer_id ORDER BY order_date
                              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_order
FROM orders;

-- Named window (avoid repetition)
SELECT order_id,
       SUM(total) OVER w AS sum_total,
       AVG(total) OVER w AS avg_total
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY order_date);
```

---

## String Functions

```sql
LENGTH(str) / CHAR_LENGTH(str)    -- character count
OCTET_LENGTH(str)                 -- bytes
UPPER(str) / LOWER(str) / INITCAP(str)
TRIM(str) / LTRIM(str) / RTRIM(str)
TRIM(BOTH 'x' FROM 'xxhelloxx')   -- trim specific chars
LEFT(str, n) / RIGHT(str, n)
SUBSTRING(str FROM 2 FOR 5)       -- or SUBSTR(str, 2, 5)
SUBSTRING(str FROM '[0-9]+')      -- regex extract
POSITION('x' IN str)              -- 1-based, 0 if not found
STRPOS(str, 'x')                  -- same thing
REPLACE(str, 'old', 'new')
REGEXP_REPLACE(str, 'pat', 'new', 'g')
REGEXP_MATCHES(str, '(\d+)', 'g') -- returns set of arrays
REGEXP_SPLIT_TO_ARRAY(str, ',')
REGEXP_SPLIT_TO_TABLE(str, ',')
REPEAT(str, n)
REVERSE(str)
LPAD(str, 10, '0') / RPAD(str, 10, ' ')
CONCAT(a, b, c)                   -- NULLs become empty
CONCAT_WS(',', a, b, c)           -- with separator; NULLs skipped
str || 'suffix'                   -- concatenation operator
FORMAT('Hello %s, age %s', name, age)
SPLIT_PART('a,b,c', ',', 2)       -- 'b' (1-based)
TO_CHAR(123.45, 'FM999,990.00')   -- formatting
QUOTE_IDENT(name) / QUOTE_LITERAL(val)
```

---

## Date & Time Functions

```sql
-- Current date/time
NOW()                          -- TIMESTAMPTZ, transaction start
CURRENT_TIMESTAMP              -- same as NOW()
STATEMENT_TIMESTAMP()          -- start of current statement
CLOCK_TIMESTAMP()              -- actual current time (changes during txn)
CURRENT_DATE
CURRENT_TIME
LOCALTIMESTAMP                 -- TIMESTAMP without timezone

-- Parts (use EXTRACT or DATE_PART)
EXTRACT(YEAR FROM dt)          -- year, month, day, hour, minute, second
EXTRACT(DOW FROM dt)           -- day of week, 0=Sunday
EXTRACT(ISODOW FROM dt)        -- 1=Monday..7=Sunday
EXTRACT(DOY FROM dt)           -- day of year
EXTRACT(EPOCH FROM dt)         -- seconds since 1970
EXTRACT(QUARTER FROM dt)
DATE_PART('month', dt)         -- equivalent

-- Truncation (for grouping)
DATE_TRUNC('day', dt)          -- second, minute, hour, day, week, month, quarter, year
DATE_TRUNC('month', dt)::DATE

-- Arithmetic
dt + INTERVAL '7 days'
dt - INTERVAL '1 month'
dt + INTERVAL '1 year 2 months 3 days'
dt2 - dt1                      -- returns INTERVAL
AGE(dt2, dt1)                  -- human-friendly interval (years/months/days)
AGE(birth_date)                -- vs current date

-- Construction
MAKE_DATE(2024, 6, 15)
MAKE_TIMESTAMP(2024, 6, 15, 10, 30, 0)
MAKE_TIMESTAMPTZ(2024, 6, 15, 10, 30, 0, 'UTC')
MAKE_INTERVAL(days => 5, hours => 3)

-- Conversion / formatting
CAST('2024-06-15' AS DATE)
'2024-06-15'::DATE             -- shorthand
TO_DATE('15/06/2024', 'DD/MM/YYYY')
TO_TIMESTAMP('2024-06-15 10:30', 'YYYY-MM-DD HH24:MI')
TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS')
TO_CHAR(NOW(), 'Day, DD Month YYYY')

-- Timezone
NOW() AT TIME ZONE 'America/New_York'
'2024-06-15 10:00'::TIMESTAMP AT TIME ZONE 'UTC' AT TIME ZONE 'Europe/London'

-- Generate series (super useful)
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 day') AS day;
SELECT generate_series(1, 100) AS n;
```

---

## Numeric & Conversion Functions

```sql
ABS(x), CEIL(x), FLOOR(x), ROUND(x, n), TRUNC(x, n)
POWER(x, n), SQRT(x), CBRT(x), EXP(x), LN(x), LOG(x), LOG(b, x)
PI(), RANDOM(), RANDOM() * (max - min) + min
SIGN(x), MOD(a, b), GCD(a, b), LCM(a, b)
GREATEST(a, b, c), LEAST(a, b, c)

-- Conversion
CAST(expr AS type)
expr::type                     -- shorthand
TO_NUMBER('1,234.56', '999,999.99')

-- NULL handling
COALESCE(a, b, c, 'default')   -- first non-null
NULLIF(a, b)                   -- NULL if a = b, else a
IFNULL                         -- ❌ not in PostgreSQL, use COALESCE
ISNULL                         -- ❌ not in PostgreSQL, use IS NULL
```

---

## Arrays

```sql
-- Create
SELECT ARRAY[1, 2, 3];
SELECT '{1,2,3}'::INTEGER[];

-- Access (1-based)
SELECT (ARRAY[10, 20, 30])[2];           -- 20
SELECT arr[1:2] FROM t;                  -- slice

-- Length
ARRAY_LENGTH(arr, 1)                     -- 1 = first dimension
CARDINALITY(arr)                         -- total elements

-- Modify
ARRAY_APPEND(arr, val)                   -- arr || val
ARRAY_PREPEND(val, arr)                  -- val || arr
arr1 || arr2                             -- concatenate
ARRAY_REMOVE(arr, val)
ARRAY_REPLACE(arr, from_val, to_val)

-- Search
val = ANY(arr)                           -- membership test
arr @> ARRAY[1,2]                        -- contains
arr <@ ARRAY[1,2,3,4]                    -- contained by
arr && ARRAY[2,3]                        -- overlaps
ARRAY_POSITION(arr, val)

-- Aggregate / unnest
SELECT ARRAY_AGG(name ORDER BY id) FROM t;
SELECT UNNEST(ARRAY[1,2,3]) AS n;        -- expand to rows
SELECT UNNEST(arr) WITH ORDINALITY AS u(val, idx) FROM t;

-- Conversion
ARRAY_TO_STRING(arr, ',')
STRING_TO_ARRAY('a,b,c', ',')
```

---

## JSON & JSONB

```sql
-- Create
SELECT '{"name":"Ada","age":36}'::JSONB;
SELECT JSONB_BUILD_OBJECT('name', 'Ada', 'age', 36);
SELECT JSONB_BUILD_ARRAY(1, 'two', TRUE);
SELECT TO_JSONB(ROW(1, 'Ada')::customer);

-- Access (-> returns json/jsonb, ->> returns text)
SELECT data->'name' FROM t;              -- JSONB
SELECT data->>'name' FROM t;             -- TEXT
SELECT data->'addresses'->0->>'city';    -- nested
SELECT data#>'{addresses,0,city}';       -- path operator (JSONB)
SELECT data#>>'{addresses,0,city}';      -- path, returns text

-- Containment & existence (JSONB only — fast with GIN index)
data @> '{"verified": true}'             -- contains
data <@ '{"a":1,"b":2,"c":3}'            -- contained by
data ? 'email'                           -- key exists
data ?| ARRAY['email', 'phone']          -- any key exists
data ?& ARRAY['email', 'phone']          -- all keys exist

-- Modify
JSONB_SET(data, '{name}', '"New Name"')
JSONB_SET(data, '{address,city}', '"Paris"', TRUE)  -- create if missing
data || '{"new_key": "val"}'::JSONB                  -- merge (shallow)
data - 'key'                                         -- remove key
data #- '{address,city}'                             -- remove path

-- Expand / shred
SELECT * FROM JSONB_EACH(data);                      -- key/value pairs
SELECT * FROM JSONB_EACH_TEXT(data);
SELECT JSONB_OBJECT_KEYS(data);
SELECT * FROM JSONB_ARRAY_ELEMENTS(data->'items');
SELECT * FROM JSONB_TO_RECORDSET(data->'items')
    AS x(id INT, name TEXT, price NUMERIC);

-- Aggregate
SELECT JSONB_AGG(row_to_json(c)) FROM customers c;
SELECT JSONB_OBJECT_AGG(key, value) FROM kv_pairs;

-- Query (JSON path, 12+)
SELECT JSONB_PATH_QUERY(data, '$.items[*].price');
SELECT JSONB_PATH_EXISTS(data, '$.items ? (@.price > 100)');
```

---

## Conditional Logic

```sql
-- CASE (searched)
SELECT
    CASE
        WHEN total >= 1000 THEN 'High'
        WHEN total >= 100  THEN 'Medium'
        ELSE 'Low'
    END AS tier
FROM orders;

-- CASE (simple)
SELECT
    CASE status
        WHEN 'P' THEN 'Pending'
        WHEN 'S' THEN 'Shipped'
        ELSE 'Unknown'
    END AS status_name
FROM orders;

-- Shorthand alternatives
COALESCE(a, b, c)              -- first non-null
NULLIF(a, b)                   -- NULL if equal
GREATEST(a, b, c) / LEAST(a, b, c)
```

---

## UPSERT (ON CONFLICT)

```sql
-- Insert or do nothing on conflict
INSERT INTO customers (email, first_name)
VALUES ('ada@example.com', 'Ada')
ON CONFLICT (email) DO NOTHING;

-- Insert or update on conflict (the upsert)
INSERT INTO customers (email, first_name, updated_at)
VALUES ('ada@example.com', 'Ada', NOW())
ON CONFLICT (email) DO UPDATE
SET first_name = EXCLUDED.first_name,
    updated_at = EXCLUDED.updated_at
WHERE customers.first_name IS DISTINCT FROM EXCLUDED.first_name
RETURNING *;

-- Conflict on constraint name
ON CONFLICT ON CONSTRAINT customers_email_key DO UPDATE ...

-- MERGE (15+) - more flexible than ON CONFLICT
MERGE INTO customers c
USING new_data n ON c.customer_id = n.customer_id
WHEN MATCHED AND n.is_active = FALSE THEN DELETE
WHEN MATCHED THEN UPDATE SET first_name = n.first_name
WHEN NOT MATCHED THEN INSERT (customer_id, first_name) VALUES (n.customer_id, n.first_name);
```

---

## Functions & Procedures

```sql
-- Plain SQL function (fastest, can be inlined)
CREATE OR REPLACE FUNCTION full_name(first TEXT, last TEXT)
RETURNS TEXT
LANGUAGE SQL
IMMUTABLE
AS $$
    SELECT first || ' ' || last;
$$;

-- PL/pgSQL function with logic
CREATE OR REPLACE FUNCTION sales.get_customer_total(cust_id BIGINT, since DATE DEFAULT NULL)
RETURNS NUMERIC
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
    total NUMERIC;
BEGIN
    SELECT COALESCE(SUM(o.total), 0) INTO total
    FROM orders o
    WHERE o.customer_id = cust_id
      AND (since IS NULL OR o.order_date >= since);

    RETURN total;
END;
$$;

-- Set-returning function (TVF)
CREATE OR REPLACE FUNCTION orders_by_customer(cust_id BIGINT)
RETURNS TABLE (order_id BIGINT, order_date DATE, total NUMERIC)
LANGUAGE SQL
STABLE
AS $$
    SELECT order_id, order_date, total
    FROM orders
    WHERE customer_id = cust_id;
$$;

-- Usage
SELECT * FROM orders_by_customer(42);
SELECT full_name('Ada', 'Lovelace');

-- Procedure (11+) — supports transactions inside
CREATE OR REPLACE PROCEDURE archive_old_data()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO archive SELECT * FROM main_table WHERE created_at < NOW() - INTERVAL '1 year';
    DELETE FROM main_table WHERE created_at < NOW() - INTERVAL '1 year';
    COMMIT;  -- procedures can commit/rollback; functions cannot
END;
$$;

CALL archive_old_data();
```

> **Volatility hints:** `IMMUTABLE` (same input → same output, no side effects), `STABLE` (consistent within a query), `VOLATILE` (default; can change anytime). Mark accurately — the planner uses this for optimization.

---

## Triggers

```sql
-- Trigger function (must return TRIGGER)
CREATE OR REPLACE FUNCTION audit_orders()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO orders_audit(order_id, action, changed_at)
        VALUES (NEW.order_id, 'INSERT', NOW());
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO orders_audit(order_id, action, changed_at, old_data, new_data)
        VALUES (NEW.order_id, 'UPDATE', NOW(), to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO orders_audit(order_id, action, changed_at)
        VALUES (OLD.order_id, 'DELETE', NOW());
        RETURN OLD;
    END IF;
END;
$$;

-- Attach trigger
CREATE TRIGGER trg_orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW
EXECUTE FUNCTION audit_orders();

-- Common: auto-update updated_at column
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_set_updated_at
BEFORE UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Conditional trigger
CREATE TRIGGER trg_only_active
AFTER UPDATE ON customers
FOR EACH ROW
WHEN (OLD.status IS DISTINCT FROM NEW.status)
EXECUTE FUNCTION ...;

-- Drop / disable
DROP TRIGGER IF EXISTS trg_orders_audit ON orders;
ALTER TABLE orders DISABLE TRIGGER trg_orders_audit;
ALTER TABLE orders ENABLE  TRIGGER trg_orders_audit;
```

> Inside trigger functions: `NEW` is the new row (INSERT/UPDATE), `OLD` is the old row (UPDATE/DELETE), `TG_OP` is `'INSERT'`/`'UPDATE'`/`'DELETE'`/`'TRUNCATE'`.

---

## Indexes

```sql
-- B-tree (default; equality, range, ORDER BY, LIKE 'prefix%')
CREATE INDEX idx_customers_email ON customers(email);

-- Composite + DESC
CREATE INDEX idx_orders_customer_date
ON orders (customer_id, order_date DESC);

-- Covering index (INCLUDE, 11+)
CREATE INDEX idx_orders_lookup
ON orders (customer_id) INCLUDE (total, status);

-- Unique
CREATE UNIQUE INDEX uq_customers_email ON customers(email);

-- Partial (filtered)
CREATE INDEX idx_active_customers
ON customers(email) WHERE is_active = TRUE;

-- Expression
CREATE INDEX idx_lower_email ON customers (LOWER(email));

-- GIN (full-text, JSONB, arrays — set membership)
CREATE INDEX idx_customers_metadata ON customers USING GIN (metadata);
CREATE INDEX idx_post_tags ON posts USING GIN (tags);

-- GiST (geometric, range types, full-text)
CREATE INDEX idx_locations ON locations USING GiST (geom);

-- BRIN (huge tables with natural ordering, like timestamps)
CREATE INDEX idx_logs_created_brin ON logs USING BRIN (created_at);

-- Hash (equality only; rarely needed since B-tree handles it)
CREATE INDEX idx_session_token ON sessions USING HASH (token);

-- Concurrent build (no table lock)
CREATE INDEX CONCURRENTLY idx_customers_email ON customers(email);

-- Maintenance
REINDEX INDEX idx_customers_email;
REINDEX TABLE customers;
REINDEX TABLE CONCURRENTLY customers;  -- 12+
DROP INDEX CONCURRENTLY idx_old;
```

---

## Full-Text Search

```sql
-- tsvector + tsquery
SELECT to_tsvector('english', 'The quick brown fox jumps over the lazy dog');
SELECT to_tsquery('english', 'quick & fox');

-- Search
SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body)
   @@ plainto_tsquery('english', 'database performance');

-- Indexed full-text search
ALTER TABLE articles ADD COLUMN search_vec TSVECTOR
    GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title,'') || ' ' || coalesce(body,''))
    ) STORED;
CREATE INDEX idx_articles_fts ON articles USING GIN (search_vec);

SELECT title, ts_rank(search_vec, query) AS rank
FROM articles, plainto_tsquery('english', 'postgres tips') query
WHERE search_vec @@ query
ORDER BY rank DESC;

-- Highlight
SELECT ts_headline('english', body, plainto_tsquery('postgres')) FROM articles;
```

---

## Transactions

```sql
BEGIN;  -- or START TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;

COMMIT;  -- or ROLLBACK;

-- Savepoints
BEGIN;
INSERT INTO ...;
SAVEPOINT sp1;
INSERT INTO ...;       -- might fail
ROLLBACK TO SAVEPOINT sp1;  -- undo to savepoint, keep first insert
COMMIT;

-- Isolation levels
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;       -- default
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- READ UNCOMMITTED is accepted but behaves like READ COMMITTED

-- Read-only / deferrable
SET TRANSACTION READ ONLY;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;

-- Row-level locking
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;        -- exclusive
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT; -- error if locked
SELECT * FROM accounts WHERE id = 1 FOR UPDATE SKIP LOCKED;  -- great for queues

-- Advisory locks (application-level)
SELECT pg_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);
```

---

## Error Handling (PL/pgSQL)

```sql
DO $$
BEGIN
    INSERT INTO customers(email) VALUES ('dup@example.com');
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Email already exists';
    WHEN foreign_key_violation THEN
        RAISE NOTICE 'FK violation';
    WHEN OTHERS THEN
        RAISE NOTICE 'Error: % / %', SQLSTATE, SQLERRM;
END $$;

-- Raise custom error
RAISE EXCEPTION 'Value out of range: %', val
    USING ERRCODE = '22003',
          HINT    = 'Use a value between 0 and 100';

-- Levels: DEBUG, LOG, INFO, NOTICE, WARNING, EXCEPTION
RAISE NOTICE 'Processing customer %', cust_id;

-- Common SQLSTATE codes
-- 23505 unique_violation
-- 23503 foreign_key_violation
-- 23502 not_null_violation
-- 22001 string_data_right_truncation
-- 40001 serialization_failure (retry-able)
-- 40P01 deadlock_detected
```

---

## Roles & Permissions

```sql
-- Roles (users and groups are both roles)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';
CREATE ROLE readonly;
GRANT readonly TO app_user;

-- Privileges
GRANT CONNECT ON DATABASE sales_db TO readonly;
GRANT USAGE ON SCHEMA sales TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON sales.customers TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA sales TO app_user;

-- Default privileges (apply to future objects)
ALTER DEFAULT PRIVILEGES IN SCHEMA sales
    GRANT SELECT ON TABLES TO readonly;

-- Revoke
REVOKE INSERT ON sales.customers FROM app_user;

-- Row-level security (RLS)
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.tenant_id')::INT);
```

---

## psql Meta-Commands

```text
\l                  list databases
\c db_name          connect to database
\dn                 list schemas
\dt                 list tables (current schema)
\dt sales.*         list tables in schema
\d customers        describe table (columns, indexes, constraints)
\d+ customers       verbose describe
\di                 list indexes
\dv                 list views
\df                 list functions
\du                 list roles
\dx                 list installed extensions
\timing             toggle query timing
\x                  toggle expanded display (good for wide rows)
\e                  open editor for query
\i file.sql         execute SQL file
\copy t FROM 'file.csv' CSV HEADER  -- client-side import
\q                  quit
\?                  help
\h SELECT           SQL syntax help
\set ECHO all       echo executed commands
```

---

## Performance Tips

- **`EXPLAIN ANALYZE`** is your best friend. Use `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` for the most useful output. Look for sequential scans on large tables, mismatched row estimates, and nested loops over big sets.
- **`VACUUM` and `ANALYZE`** matter. Autovacuum usually handles it, but for heavy-write tables consider tuning per-table settings. Run `ANALYZE` after big bulk loads.
- **Sargability:** `WHERE date_trunc('day', created_at) = '2024-06-15'` defeats the index. Use `WHERE created_at >= '2024-06-15' AND created_at < '2024-06-16'` — or build an expression index on `date_trunc(...)`.
- **`EXISTS` over `IN`** for correlated existence checks; both are usually fine, but `EXISTS` is more predictable with NULLs.
- **`COUNT(*)` is not free** in Postgres — there's no stored row count. For approximate counts, use `pg_class.reltuples`.
- **Batch large `DELETE`/`UPDATE`** to keep transactions short and avoid bloat:
  ```sql
  DELETE FROM big_table WHERE id IN (
      SELECT id FROM big_table WHERE created_at < '2020-01-01' LIMIT 5000
  );
  ```
- **Use `CONCURRENTLY`** when creating/dropping indexes on production tables to avoid blocking writers.
- **Connection pooling** matters — Postgres processes are expensive. Use PgBouncer or a pool in your app.
- **`LATERAL` joins** are often faster than correlated subqueries for "top-N per group" queries.
- **Partial indexes** can be dramatically smaller and faster when you query a subset frequently.
- **JSONB GIN indexes** make `@>` containment queries fast; without one, you're scanning every row.
- **Prepared statements** help when running the same query repeatedly — Postgres caches the plan.
- **`work_mem` per session** can help individual sort/hash operations; bump it for analytical queries with `SET LOCAL work_mem = '256MB'`.

---

## Useful System Catalogs

```sql
-- Tables / columns
SELECT * FROM information_schema.tables WHERE table_schema = 'sales';
SELECT * FROM information_schema.columns WHERE table_name = 'customers';

-- Sizes
SELECT pg_size_pretty(pg_database_size('sales_db'));
SELECT pg_size_pretty(pg_total_relation_size('sales.customers'));
SELECT pg_size_pretty(pg_relation_size('sales.customers'));   -- table only
SELECT pg_size_pretty(pg_indexes_size('sales.customers'));

-- Top tables by size
SELECT schemaname, relname,
       pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;

-- Indexes on a table
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'customers';

-- Active queries
SELECT pid, usename, state, query_start, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Kill a query
SELECT pg_cancel_backend(pid);   -- polite
SELECT pg_terminate_backend(pid); -- forceful

-- Locks
SELECT * FROM pg_locks WHERE NOT granted;

-- Index usage stats (find unused indexes)
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Slow queries (requires pg_stat_statements extension)
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

*Targets PostgreSQL 12+. Notable version-specific features: `MERGE` (15+), `COMMIT`/`ROLLBACK` in procedures (11+), `INCLUDE` in indexes (11+), `GENERATED ... AS IDENTITY` (10+), declarative partitioning (10+), `MATERIALIZED` CTE hint (12+).*
