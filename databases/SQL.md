# SQL

The practical language for defining and querying relational databases: data definition (DDL), data manipulation (DML), the query language, views, transactions, and an end-to-end schema-creation walkthrough.

This document covers *writing SQL and building a database*. For the formal model and design theory behind these constructs (relational model, algebra, ER (entity-relationship) modeling, normalization) see `Database Design.md`. For how the engine executes SQL (planning, joins, indexes, concurrency) see `Database Internals.md`. Examples target PostgreSQL; dialect differences are noted where they matter.

---

## Table of Contents

1. SQL Overview
2. Data Definition (DDL)
3. Constraints
4. Data Manipulation (DML)
5. Queries
6. Joins
7. Aggregation and Grouping
8. Subqueries
9. Common Table Expressions and Window Functions
10. Views
11. Transactions in SQL
12. Worked Example: Building a Schema

---

## 1. SQL Overview

SQL (Structured Query Language) is the standard language for relational DBMSs (database management systems). It is **declarative**: a query states *what* result is wanted, leaving the engine to choose *how* to compute it (the optimizer, see `Database Internals.md`).

The core query form maps directly onto algebra:

```
SELECT  <columns>      -- projection (π)
FROM    <tables>       -- Cartesian product, then joins
WHERE   <condition>    -- selection (σ)
```

SQL operates on **bags** (multisets), not sets: duplicate rows are kept unless `DISTINCT` is requested. This is the main practical departure from the set-based algebra.

The language divides into:

- **DDL** (Data Definition) — `CREATE`, `ALTER`, `DROP` of schema objects.
- **DML** (Data Manipulation) — `INSERT`, `UPDATE`, `DELETE`, and `SELECT`.
- **DCL** (Data Control) — privileges: `GRANT`/`REVOKE`.
- **TCL** (Transaction Control) — transaction boundaries: `COMMIT`/`ROLLBACK`/`SAVEPOINT`.

---

## 2. Data Definition (DDL)

### CREATE TABLE

```sql
CREATE TABLE employees (
    id          SERIAL       PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    dept_id     INTEGER      REFERENCES departments(id),
    salary      NUMERIC(10,2) CHECK (salary >= 0),
    hired_at    DATE         NOT NULL DEFAULT CURRENT_DATE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

### Common data types

| Category | Types |
|---|---|
| Integer | `SMALLINT`, `INTEGER`, `BIGINT`, `SERIAL`/`BIGSERIAL` (auto-increment) |
| Exact decimal | `NUMERIC(p,s)` / `DECIMAL(p,s)` — for money; never use floats for currency |
| Floating point | `REAL`, `DOUBLE PRECISION` |
| Text | `CHAR(n)`, `VARCHAR(n)`, `TEXT` |
| Boolean | `BOOLEAN` |
| Temporal | `DATE`, `TIME`, `TIMESTAMP`, `TIMESTAMPTZ` (timezone-aware), `INTERVAL` |
| Other | `UUID`, `JSONB`, `BYTEA`, `ARRAY` |

Prefer `TIMESTAMPTZ` over `TIMESTAMP` to avoid timezone ambiguity, and `NUMERIC` over floating point for money.

### ALTER and DROP

```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
ALTER TABLE employees ALTER COLUMN phone SET NOT NULL;
ALTER TABLE employees ADD CONSTRAINT fk_dept
    FOREIGN KEY (dept_id) REFERENCES departments(id);
ALTER TABLE employees RENAME COLUMN phone TO phone_number;
ALTER TABLE employees DROP COLUMN phone_number;

DROP TABLE employees;                 -- fails if referenced
DROP TABLE IF EXISTS employees CASCADE;  -- also drops dependents
```

### Indexes

An index speeds lookups at the cost of write overhead and storage (mechanics in `Database Internals.md`).

```sql
CREATE INDEX idx_emp_dept ON employees (dept_id);
CREATE UNIQUE INDEX idx_emp_email ON employees (email);
CREATE INDEX idx_emp_dept_salary ON employees (dept_id, salary);  -- composite
CREATE INDEX idx_emp_active ON employees (dept_id) WHERE salary > 0;   -- partial
```

A composite index `(a, b)` serves queries filtering on `a` or on `a, b` (a key prefix), not on `b` alone. Index foreign keys and columns used in `WHERE`, `JOIN`, and `ORDER BY`.

---

## 3. Constraints

Constraints enforce integrity at the schema level, so invalid states are rejected regardless of the application. This is the SQL realization of the constraints in `Database Design.md`.

| Constraint | Meaning |
|---|---|
| `NOT NULL` | the column must have a value |
| `PRIMARY KEY` | unique + not null; the principal identifier |
| `UNIQUE` | a candidate key; no duplicate values (multiple NULLs allowed in PostgreSQL/standard; SQL Server allows only one) |
| `FOREIGN KEY ... REFERENCES` | referential integrity: values must exist in the referenced key |
| `CHECK (...)` | an arbitrary per-row boolean condition |
| `DEFAULT` | value used when none is supplied |

### Mapping design constraints to SQL

The conceptual constraints from `Database Design.md` translate directly:

| Design constraint | SQL |
|---|---|
| Mandatory attribute | `NOT NULL` |
| Primary identifier | `PRIMARY KEY` |
| Other identifier (candidate key) | `UNIQUE` |
| Foreign key / referential integrity | `FOREIGN KEY (x) REFERENCES R(y)` |
| Null interdependence (A and B both null or both set) | `CHECK ((A IS NULL AND B IS NULL) OR (A IS NOT NULL AND B IS NOT NULL))` |
| Inclusion (A's values ⊆ B's values) | `CHECK (A IN (SELECT B FROM R))` |
| Disjointness (A's values disjoint from B's) | `CHECK (A NOT IN (SELECT B FROM R))` |
| Cardinality `(i, j)` | `CHECK (i <= (SELECT count(*) FROM R WHERE ...) AND j >= (...))` |
| General external constraint | assertion or application-level check |

### Referential actions

A foreign key can specify what happens when the referenced row changes:

```sql
dept_id INTEGER REFERENCES departments(id)
    ON DELETE CASCADE      -- delete dependent rows
    ON UPDATE CASCADE;     -- propagate key changes
-- alternatives: ON DELETE SET NULL | SET DEFAULT | RESTRICT | NO ACTION
```

### Table-level and named constraints

Multi-column constraints are declared at table level; naming them eases later `ALTER`:

```sql
CREATE TABLE enrollments (
    student_id INTEGER NOT NULL REFERENCES students(id),
    course_id  INTEGER NOT NULL REFERENCES courses(id),
    grade      SMALLINT CHECK (grade BETWEEN 0 AND 100),
    CONSTRAINT pk_enrollment PRIMARY KEY (student_id, course_id)
);
```

---

## 4. Data Manipulation (DML)

### INSERT

```sql
INSERT INTO employees (name, email, dept_id, salary)
VALUES ('Alice', 'alice@ex.com', 3, 65000);

INSERT INTO employees (name, email) VALUES
    ('Bob',   'bob@ex.com'),
    ('Carol', 'carol@ex.com');           -- multi-row

INSERT INTO archive_employees SELECT * FROM employees WHERE hired_at < '2020-01-01';
```

### UPDATE

```sql
UPDATE employees
SET    salary = salary * 1.05
WHERE  dept_id = 3;
```

A `WHERE`-less `UPDATE` (or `DELETE`) affects every row. The single-statement `salary = salary * 1.05` is safe under concurrency because, under READ COMMITTED, the engine re-reads the latest committed row before applying the update (at higher isolation it instead aborts the transaction with a serialization failure); a read-modify-write split across a separate `SELECT` then `UPDATE` is not (see lost update in `Data Systems.md` / `Database Internals.md`).

### DELETE

```sql
DELETE FROM employees WHERE id = 42;
TRUNCATE TABLE staging;   -- fast bulk delete; not logged per row, but still transactional in PostgreSQL (rollback-able), unlike MySQL
```

### UPSERT (insert or update)

```sql
INSERT INTO inventory (sku, qty) VALUES ('A1', 10)
ON CONFLICT (sku) DO UPDATE SET qty = inventory.qty + EXCLUDED.qty;
```

---

## 5. Queries

### Structure and evaluation order

```sql
SELECT   DISTINCT col, expr AS alias
FROM     table
WHERE    row_condition
GROUP BY col
HAVING   group_condition
ORDER BY col [ASC|DESC]
LIMIT    n OFFSET m;
```

Written in that order, but evaluated roughly as: `FROM` -> `WHERE` -> `GROUP BY` -> `HAVING` -> `SELECT` -> `DISTINCT` -> `ORDER BY` -> `LIMIT`. This is why a column alias defined in `SELECT` cannot be used in `WHERE` (the alias does not yet exist) but can be used in `ORDER BY`.

### Filtering

```sql
WHERE salary BETWEEN 50000 AND 80000
  AND dept_id IN (1, 2, 3)
  AND name LIKE 'A%'            -- pattern: % any string, _ any char
  AND email IS NOT NULL;
```

### NULL semantics

`NULL` is *unknown*, not a value. Any comparison with `NULL` yields `UNKNOWN` (treated as not-true by `WHERE`), so `x = NULL` is never true — use `IS NULL` / `IS NOT NULL`. Three-valued logic: `TRUE AND UNKNOWN = UNKNOWN`, `FALSE AND UNKNOWN = FALSE`, `TRUE OR UNKNOWN = TRUE`. `NULL`s are excluded from most aggregates and require explicit handling (`COALESCE(x, default)`).

### Sorting and limiting

```sql
SELECT name, salary FROM employees
ORDER BY salary DESC, name ASC
LIMIT 10 OFFSET 20;          -- rows 21..30
```

---

## 6. Joins

A join correlates rows of multiple tables (the SQL form of the algebra joins in `Database Design.md`).

```sql
-- INNER: only matching rows
SELECT e.name, d.name AS dept
FROM   employees e
JOIN   departments d ON e.dept_id = d.id;

-- LEFT OUTER: all left rows, NULLs where no match
SELECT e.name, d.name AS dept
FROM   employees e
LEFT JOIN departments d ON e.dept_id = d.id;

-- RIGHT / FULL OUTER: symmetric / both sides preserved
-- CROSS: Cartesian product
SELECT * FROM colors CROSS JOIN sizes;

-- SELF join: a table joined to itself (needs aliases)
SELECT e.name AS emp, m.name AS manager
FROM   employees e
JOIN   employees m ON e.manager_id = m.id;
```

`INNER JOIN` drops unmatched rows on both sides; `LEFT JOIN` keeps unmatched left rows with NULLs on the right. A condition on the right table belongs in the `ON` clause (to preserve the outer semantics) or it silently turns the outer join back into an inner one when placed in `WHERE`.

---

## 7. Aggregation and Grouping

Aggregate functions collapse a set of rows into one value: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

```sql
SELECT   dept_id,
         COUNT(*)        AS headcount,
         AVG(salary)     AS avg_salary,
         MAX(salary)     AS top_salary
FROM     employees
GROUP BY dept_id
HAVING   COUNT(*) > 5            -- filter on groups, after aggregation
ORDER BY avg_salary DESC;
```

- `GROUP BY` partitions rows; one output row per group.
- Every non-aggregated `SELECT` column must appear in `GROUP BY` (PostgreSQL relaxes this when grouping by a table's primary key, on which the other columns functionally depend — see `Database Design.md`).
- `WHERE` filters rows *before* grouping; `HAVING` filters groups *after*.
- `COUNT(*)` counts rows; `COUNT(col)` counts non-NULL values; `COUNT(DISTINCT col)` counts distinct non-NULL values.

---

## 8. Subqueries

A query nested inside another.

```sql
-- Scalar subquery (returns one value)
SELECT name, salary,
       (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;

-- IN / NOT IN
SELECT name FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE region = 'EU');

-- EXISTS (correlated): true if the inner query returns any row
SELECT d.name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id);

-- Subquery in FROM (derived table)
SELECT dept_id, avg_salary
FROM (SELECT dept_id, AVG(salary) AS avg_salary
      FROM employees GROUP BY dept_id) t
WHERE avg_salary > 60000;
```

A **correlated** subquery references the outer row and is conceptually re-evaluated per outer row (the optimizer often rewrites it). Modern planners treat `IN (subquery)` and `EXISTS` as the same semi-join, so prefer `EXISTS`/`NOT EXISTS` for NULL-safety rather than performance: beware `NOT IN` with NULLs — a single NULL in the inner result makes `NOT IN` return no rows, whereas `NOT EXISTS` has no such trap.

---

## 9. Common Table Expressions and Window Functions

### CTEs

A `WITH` clause names a subquery, improving readability and enabling recursion.

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM   employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, da.avg_salary
FROM   employees e
JOIN   dept_avg da ON e.dept_id = da.dept_id
WHERE  e.salary > da.avg_salary;
```

```sql
-- Recursive CTE: walk an org hierarchy
WITH RECURSIVE chain AS (
    SELECT id, name, manager_id FROM employees WHERE id = 1
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM   employees e JOIN chain c ON e.manager_id = c.id
)
SELECT * FROM chain;
```

In PostgreSQL a non-recursive CTE is normally inlined into the enclosing query, like a derived table, so the optimizer can push predicates across the boundary. `WITH x AS MATERIALIZED (...)` forces single evaluation into a temporary result (an optimization fence — useful when the CTE calls a volatile function or is referenced several times, but it blocks predicate push-down); `NOT MATERIALIZED` forces inlining. Recursive CTEs are always materialized. This is dialect-specific — some engines materialize every CTE unconditionally.

### Window functions

A window function computes across a set of rows *related to the current row* without collapsing them (unlike `GROUP BY`).

```sql
SELECT name, dept_id, salary,
       RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_rank,
       AVG(salary)  OVER (PARTITION BY dept_id)                      AS dept_avg,
       salary - LAG(salary) OVER (ORDER BY hired_at)                 AS delta_prev
FROM employees;
```

- `PARTITION BY` defines the window groups; `ORDER BY` orders rows within a window.
- Ranking: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`.
- Offset: `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`.
- Any aggregate (`SUM`, `AVG`, ...) used with `OVER` becomes a running/windowed aggregate. Adding a frame (`ROWS BETWEEN ...`) gives running totals and moving averages.

#### Frames

A window with `ORDER BY` but no explicit frame defaults to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Under `RANGE`, all peer rows (rows equal in the `ORDER BY` key) share one frame boundary, so a running `SUM`/`AVG` assigns every tied row the same cumulative value, and `LAST_VALUE` returns the current row rather than the partition's last. Use `ROWS` for a strict row-by-row frame:

```sql
SUM(amount) OVER (ORDER BY ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  -- true running total
AVG(amount) OVER (ORDER BY ts ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)          -- 7-row moving average
```

A window with no `ORDER BY` frames the entire partition: `AVG(salary) OVER (PARTITION BY dept_id)` is the department average. Ranking functions (`RANK`, `ROW_NUMBER`, `NTILE`) ignore the frame.

---

## 10. Views

A **view** is a named query, queried like a table. It encapsulates complexity and presents a stable interface.

```sql
CREATE VIEW active_employees AS
SELECT id, name, dept_id, salary
FROM   employees
WHERE  salary > 0;

SELECT * FROM active_employees WHERE dept_id = 3;
```

A plain view stores no data — it is expanded into the underlying query at use. A **materialized view** stores the result for fast reads and must be refreshed:

```sql
CREATE MATERIALIZED VIEW dept_stats AS
SELECT dept_id, COUNT(*) AS headcount, AVG(salary) AS avg_salary
FROM   employees GROUP BY dept_id;

REFRESH MATERIALIZED VIEW dept_stats;
```

Views also recover the original tables after a logical restructuring (decomposition/merging, see `Database Design.md`). The relationship to denormalized read models is covered in `Data Systems.md`.

---

## 11. Transactions in SQL

A transaction groups statements into an atomic unit (the theory is in `Database Internals.md`).

```sql
BEGIN;                          -- or START TRANSACTION
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                         -- or ROLLBACK to undo everything
```

### Isolation levels

Set per transaction; each forbids a set of anomalies (engine-specific behavior is detailed in `Data Systems.md`):

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

| Level | Dirty read | Nonrepeatable read | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | prevented | possible | possible |
| REPEATABLE READ | prevented | prevented | possible* |
| SERIALIZABLE | prevented | prevented | prevented |

(*PostgreSQL's REPEATABLE READ also prevents phantoms via snapshot isolation.)

### Savepoints

A partial rollback point within a transaction:

```sql
BEGIN;
INSERT INTO log (msg) VALUES ('step 1');
SAVEPOINT sp1;
INSERT INTO log (msg) VALUES ('step 2');
ROLLBACK TO sp1;     -- undoes step 2, keeps step 1
COMMIT;
```

### Explicit locking

For read-modify-write across statements, lock the rows read:

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- exclusive row lock
-- compute new balance in the application
UPDATE accounts SET balance = :new WHERE id = 1;
COMMIT;
```

---

## 12. Worked Example: Building a Schema

From a logical design (`Database Design.md`) to a running schema: students enrolling in courses, a many-to-many relationship with an attribute.

```sql
-- 1. Parent entities
CREATE TABLE students (
    id         SERIAL       PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    enrolled_at DATE        NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE courses (
    id        SERIAL       PRIMARY KEY,
    code      VARCHAR(10)  UNIQUE NOT NULL,
    title     VARCHAR(200) NOT NULL,
    credits   SMALLINT     NOT NULL CHECK (credits > 0)
);

-- 2. The many-to-many relationship becomes its own table,
--    keyed by both foreign keys, carrying the relationship attribute (grade).
CREATE TABLE enrollments (
    student_id INTEGER  NOT NULL REFERENCES students(id) ON DELETE CASCADE,
    course_id  INTEGER  NOT NULL REFERENCES courses(id)  ON DELETE RESTRICT,
    grade      SMALLINT CHECK (grade BETWEEN 0 AND 100),
    taken_at   DATE     NOT NULL DEFAULT CURRENT_DATE,
    PRIMARY KEY (student_id, course_id)
);

-- 3. Indexes for the expected access patterns
CREATE INDEX idx_enroll_course ON enrollments (course_id);   -- "who took course X"

-- 4. A view for a common read
CREATE VIEW transcript AS
SELECT s.name, c.code, c.title, e.grade, e.taken_at
FROM   enrollments e
JOIN   students s ON s.id = e.student_id
JOIN   courses  c ON c.id = e.course_id;

-- 5. Populate and query
INSERT INTO students (name, email) VALUES ('Alice', 'alice@ex.com');
INSERT INTO courses (code, title, credits) VALUES ('DB101', 'Databases', 6);
INSERT INTO enrollments (student_id, course_id, grade) VALUES (1, 1, 92);

SELECT * FROM transcript WHERE name = 'Alice';
```

The pattern generalizes: parent entities become tables with their own keys; a one-to-many relationship is a foreign key in the "many" table; a many-to-many relationship becomes a table keyed by both foreign keys, with relationship attributes as columns.
