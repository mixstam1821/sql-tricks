# SQL Tricks Showcase

A collection of practical, real-world SQL patterns — the kind that come up
constantly in data engineering work, not textbook trivia.

**Dialect:** PostgreSQL 13+ (noted inline where a feature is Postgres-specific).
Each trick is self-contained: copy the setup block once, then run any section.

---

## Table of contents

1. [Sample schema & data](#0-sample-schema--data)
2. [Running totals & moving averages](#1-window-functions-running-total--moving-average)
3. [ROW_NUMBER vs RANK vs DENSE_RANK](#2-ranking-tricks)
4. [Top N per group](#3-top-n-per-group)
5. [Deduplication without losing data](#4-deduplication-without-delete--group-by)
6. [LAG / LEAD](#5-lag--lead)
7. [Gaps and islands](#6-gaps-and-islands)
8. [Pivot with conditional aggregation](#7-pivot-with-conditional-aggregation)
9. [NTILE quartiles](#8-ntile-bucket-into-quartiles)
10. [Recursive CTEs](#9-recursive-cte-date-series--hierarchy)
11. [Filling gaps in a report](#10-filling-gaps-in-a-report)
12. [UPSERT](#11-upsert-insert--on-conflict)
13. [Percent of total & cumulative %](#12-percent-of-total--cumulative-distribution)
14. [Safe division](#13-safe-division-helper)
15. [EXISTS vs IN](#14-exists-vs-in-vs-join)
16. [String aggregation](#15-string-aggregation)
17. [LATERAL joins](#16-lateral-joins-per-row-subqueries)
18. [ARRAY_AGG and unnesting](#17-array_agg-and-unnest)

---

## 0. Sample schema & data

```sql
DROP TABLE IF EXISTS orders CASCADE;
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer    TEXT NOT NULL,
    region      TEXT NOT NULL,
    order_date  DATE NOT NULL,
    amount      NUMERIC(10, 2) NOT NULL,
    status      TEXT NOT NULL DEFAULT 'completed'
);

INSERT INTO orders (customer, region, order_date, amount, status) VALUES
    ('Alice',  'EU', '2026-01-03', 120.00, 'completed'),
    ('Alice',  'EU', '2026-01-10',  85.50, 'completed'),
    ('Alice',  'EU', '2026-02-01',  40.00, 'refunded'),
    ('Bob',    'EU', '2026-01-05', 300.00, 'completed'),
    ('Bob',    'EU', '2026-01-06', 300.00, 'completed'),   -- accidental duplicate
    ('Bob',    'EU', '2026-03-01',  75.25, 'completed'),
    ('Carla',  'US', '2026-01-15', 500.00, 'completed'),
    ('Carla',  'US', '2026-01-20', 220.00, 'completed'),
    ('Carla',  'US', '2026-02-18', 130.00, 'cancelled'),
    ('Dimitri','US', '2026-01-08',  60.00, 'completed'),
    ('Dimitri','US', '2026-02-09',  60.00, 'completed'),
    ('Dimitri','US', '2026-02-10',  90.00, 'completed');
```

---

## 1. Window functions: running total & moving average

**Why it matters:** answer "how did this metric evolve" without a self-join.

```sql
SELECT
    customer,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total,
    ROUND(AVG(amount) OVER (
        PARTITION BY customer ORDER BY order_date
        ROWS BETWEEN 1 PRECEDING AND CURRENT ROW
    ), 2) AS moving_avg_2
FROM orders
ORDER BY customer, order_date;
```

---

## 2. Ranking tricks

**Why it matters:** a classic interview gotcha — know the tie-breaking behavior.

```sql
SELECT
    customer,
    amount,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num,   -- always unique
    RANK()       OVER (ORDER BY amount DESC) AS rank_,     -- ties share rank, gaps after
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank -- ties share rank, no gaps
FROM orders;
```

---

## 3. Top N per group

**Why it matters:** extremely common ("2 most recent orders per customer").

```sql
SELECT *
FROM (
    SELECT
        o.*,
        ROW_NUMBER() OVER (PARTITION BY customer ORDER BY order_date DESC) AS rn
    FROM orders o
) ranked
WHERE rn <= 2
ORDER BY customer, order_date DESC;
```

---

## 4. Deduplication without DELETE + GROUP BY

**Why it matters:** the safe way to preview and remove duplicates without
collapsing the whole table through a `GROUP BY`.

```sql
WITH duplicates AS (
    SELECT
        order_id,
        ROW_NUMBER() OVER (
            PARTITION BY customer, order_date, amount, status
            ORDER BY order_id
        ) AS rn
    FROM orders
)
-- Preview first (always check before deleting!):
SELECT o.*
FROM orders o
JOIN duplicates d ON o.order_id = d.order_id
WHERE d.rn > 1;

-- Then delete, keeping the first occurrence:
DELETE FROM orders
WHERE order_id IN (
    SELECT order_id FROM (
        SELECT order_id,
               ROW_NUMBER() OVER (
                   PARTITION BY customer, order_date, amount, status
                   ORDER BY order_id
               ) AS rn
        FROM orders
    ) d
    WHERE d.rn > 1
);
```

---

## 5. LAG / LEAD

**Why it matters:** compare a row to the previous/next one — e.g. days since
last order, per customer.

```sql
SELECT
    customer,
    order_date,
    order_date - LAG(order_date) OVER (
        PARTITION BY customer ORDER BY order_date
    ) AS days_since_last_order
FROM orders
ORDER BY customer, order_date;
```

---

## 6. Gaps and islands

**Why it matters:** find consecutive runs of activity (streaks). The trick:
subtract a row number from a date-based sequence — equal results land in the
same contiguous "island".

```sql
WITH monthly_activity AS (
    SELECT DISTINCT
        customer,
        DATE_TRUNC('month', order_date)::date AS month
    FROM orders
),
numbered AS (
    SELECT
        customer,
        month,
        ROW_NUMBER() OVER (PARTITION BY customer ORDER BY month) AS rn
    FROM monthly_activity
),
islands AS (
    SELECT
        customer,
        month,
        (month - (rn * INTERVAL '1 month'))::date AS island_key
    FROM numbered
)
SELECT
    customer,
    MIN(month) AS streak_start,
    MAX(month) AS streak_end,
    COUNT(*)   AS consecutive_months
FROM islands
GROUP BY customer, island_key
ORDER BY customer, streak_start;
```

---

## 7. Pivot with conditional aggregation

**Why it matters:** turn rows into columns without a pivot extension —
portable across databases.

```sql
SELECT
    customer,
    SUM(amount) FILTER (WHERE status = 'completed') AS completed_total,
    SUM(amount) FILTER (WHERE status = 'refunded')  AS refunded_total,
    SUM(amount) FILTER (WHERE status = 'cancelled') AS cancelled_total
FROM orders
GROUP BY customer
ORDER BY customer;
```

Portable equivalent (works even without `FILTER`, e.g. MySQL/SQLite):

```sql
SELECT
    customer,
    SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS completed_total,
    SUM(CASE WHEN status = 'refunded'  THEN amount ELSE 0 END) AS refunded_total
FROM orders
GROUP BY customer;
```

---

## 8. NTILE: bucket into quartiles

**Why it matters:** cohort/segment analysis — e.g. spend quartiles.

```sql
WITH customer_totals AS (
    SELECT customer, SUM(amount) AS total_spend
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer
)
SELECT
    customer,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend DESC) AS spend_quartile
FROM customer_totals;
```

---

## 9. Recursive CTE: date series & hierarchy

**Why it matters:** fill in missing dates, or walk a manager/org hierarchy —
things a plain `GROUP BY` can't do.

```sql
-- 9a. Date spine via recursion:
WITH RECURSIVE date_spine AS (
    SELECT DATE '2026-01-01' AS day
    UNION ALL
    SELECT day + INTERVAL '1 day'
    FROM date_spine
    WHERE day < DATE '2026-01-10'
)
SELECT day::date FROM date_spine;

-- 9b. generate_series is the idiomatic Postgres way to do the same thing:
SELECT gs::date AS day
FROM generate_series('2026-01-01'::date, '2026-01-10'::date, '1 day') AS gs;

-- 9c. Recursive CTE walking an employee/manager hierarchy:
CREATE TABLE employees (
    emp_id     INT PRIMARY KEY,
    name       TEXT,
    manager_id INT REFERENCES employees(emp_id)
);
INSERT INTO employees VALUES
    (1, 'CEO Chris',   NULL),
    (2, 'VP Vasia',    1),
    (3, 'Mgr Marios',  2),
    (4, 'IC Irini',    3),
    (5, 'IC Iason',    3);

WITH RECURSIVE org_chart AS (
    SELECT emp_id, name, manager_id, 0 AS depth
    FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, oc.depth + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.emp_id
)
SELECT REPEAT('  ', depth) || name AS org_tree, depth
FROM org_chart
ORDER BY depth, org_tree;
```

---

## 10. Filling gaps in a report

**Why it matters:** no sales on a day ≠ the day doesn't exist. Combine a date
spine with a `LEFT JOIN` so every day appears in the output, even at zero.

```sql
WITH date_spine AS (
    SELECT gs::date AS day
    FROM generate_series('2026-01-01'::date, '2026-01-10'::date, '1 day') AS gs
)
SELECT
    ds.day,
    COALESCE(SUM(o.amount), 0) AS daily_revenue
FROM date_spine ds
LEFT JOIN orders o ON o.order_date = ds.day AND o.status = 'completed'
GROUP BY ds.day
ORDER BY ds.day;
```

---

## 11. UPSERT (INSERT ... ON CONFLICT)

**Why it matters:** atomic "insert or update" — avoids race conditions from a
check-then-write pattern.

```sql
CREATE TABLE customer_stats (
    customer    TEXT PRIMARY KEY,
    order_count INT NOT NULL DEFAULT 0,
    last_seen   DATE
);

INSERT INTO customer_stats (customer, order_count, last_seen)
VALUES ('Alice', 1, '2026-01-03')
ON CONFLICT (customer)
DO UPDATE SET
    order_count = customer_stats.order_count + 1,
    last_seen   = GREATEST(customer_stats.last_seen, EXCLUDED.last_seen);
```

---

## 12. Percent of total & cumulative distribution

**Why it matters:** "what share of revenue does each customer represent, and
where's the 80/20 cutoff" — common in Pareto-style analysis.

```sql
SELECT
    customer,
    SUM(amount) AS total_spend,
    ROUND(100.0 * SUM(amount) / SUM(SUM(amount)) OVER (), 1) AS pct_of_total,
    ROUND(
        SUM(SUM(amount)) OVER (ORDER BY SUM(amount) DESC)
        / SUM(SUM(amount)) OVER () * 100.0,
    1) AS cumulative_pct
FROM orders
WHERE status = 'completed'
GROUP BY customer
ORDER BY total_spend DESC;
```

---

## 13. Safe division helper

**Why it matters:** `NULLIF` prevents a single zero-denominator from crashing
an entire report.

```sql
SELECT
    customer,
    COUNT(*) FILTER (WHERE status = 'completed') AS completed_orders,
    COUNT(*) FILTER (WHERE status = 'refunded')  AS refunded_orders,
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE status = 'refunded')
        / NULLIF(COUNT(*), 0),
    1) AS refund_rate_pct
FROM orders
GROUP BY customer;
```

---

## 14. EXISTS vs IN vs JOIN

**Why it matters:** `EXISTS` short-circuits on first match and is NULL-safe —
often the safest default for "customers who have at least one X".

```sql
-- EXISTS: stops at first match, NULL-safe
SELECT DISTINCT o.customer
FROM orders o
WHERE EXISTS (
    SELECT 1 FROM orders r
    WHERE r.customer = o.customer AND r.status = 'refunded'
);

-- IN: fine here, but breaks silently if the subquery can return NULL
SELECT DISTINCT customer
FROM orders
WHERE customer IN (
    SELECT customer FROM orders WHERE status = 'refunded'
);
```

---

## 15. String aggregation

**Why it matters:** roll up multiple rows into one delimited string per group
— handy for quick audit trails or export formatting.

```sql
SELECT
    customer,
    STRING_AGG(status, ', ' ORDER BY order_date) AS status_history
FROM orders
GROUP BY customer
ORDER BY customer;
```

---

## 16. LATERAL joins (per-row subqueries)

**Why it matters:** the real answer to "top N per group" when N needs its own
correlated logic (e.g. per-customer aggregates that reference the outer row).
Also the backbone of calling a set-returning function once per row.

```sql
-- For each customer, the single largest order plus how it compares to their average
SELECT
    c.customer,
    top_order.amount AS biggest_order,
    c.avg_amount,
    ROUND(top_order.amount - c.avg_amount, 2) AS diff_from_avg
FROM (
    SELECT customer, AVG(amount) AS avg_amount
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer
) c
CROSS JOIN LATERAL (
    SELECT amount
    FROM orders o
    WHERE o.customer = c.customer AND o.status = 'completed'
    ORDER BY amount DESC
    LIMIT 1
) top_order;
```

---

## 17. ARRAY_AGG and UNNEST

**Why it matters:** collapse rows into an array for compact storage/output
(`ARRAY_AGG`), or explode an array back into rows (`UNNEST`) — the two
together are how you round-trip normalized data through a denormalized shape.

```sql
-- Collapse: one row per customer, all their order amounts as an array
SELECT
    customer,
    ARRAY_AGG(amount ORDER BY order_date) AS amounts
FROM orders
GROUP BY customer;

-- Explode: turn an array literal back into rows
SELECT unnest(ARRAY[120.00, 85.50, 40.00]) AS amount;
```

---

## Cleanup

```sql
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS employees CASCADE;
DROP TABLE IF EXISTS customer_stats CASCADE;
```
