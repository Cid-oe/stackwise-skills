# PostgreSQL Query Optimization Patterns

Advanced techniques for query optimization, execution plan tuning, and efficient set processing in PostgreSQL 17.

## Keyset (Cursor) Pagination vs OFFSET

Avoid `OFFSET` for pagination beyond page 1. `OFFSET 10000` forces the database engine to scan and discard 10,000 index tuples and heap pages.

```sql
-- Keyset pagination (always scans exactly 20 tuples):
SELECT order_id, customer_id, total_amount, created_at
FROM orders
WHERE (created_at, order_id) < ($last_seen_created_at, $last_seen_order_id)
ORDER BY created_at DESC, order_id DESC
LIMIT 20;

-- Requires supporting composite index:
CREATE INDEX idx_orders_pagination 
ON orders (created_at DESC, order_id DESC);
```

## Optimizing Common Table Expressions (CTEs)

PostgreSQL 12+ can inline simple CTEs automatically. To prevent materialization or force inlining explicitly:

```sql
-- Explicitly inline CTE (treat like subquery for index pushdown):
WITH recent_orders AS NOT MATERIALIZED (
    -- Allows planner to push down conditions into the CTE
    SELECT customer_id, SUM(total_amount) AS revenue
    FROM orders
    WHERE created_at >= now() - INTERVAL '30 days'
    GROUP BY customer_id
)
SELECT c.email, r.revenue
FROM customers c
JOIN recent_orders r ON r.customer_id = c.customer_id;
```

## Lateral Joins for Correlated Subqueries

Use `LEFT JOIN LATERAL` to retrieve top-N records per parent entity in a single query:

```sql
-- Get 3 most recent comments for every blog post:
SELECT p.post_id, p.title, c.comment_id, c.body, c.created_at
FROM posts p
LEFT JOIN LATERAL (
    SELECT comment_id, body, created_at
    FROM comments
    WHERE comments.post_id = p.post_id
    ORDER BY created_at DESC
    LIMIT 3
) c ON true;

-- Supported by index on comments (post_id, created_at DESC)
```

## Eliminating Sequential Scans on Big Tables

Check why the planner avoided an index:
1. **Low table selectivity**: If the query matches >15% of the table rows, sequential scans are naturally faster.
2. **Missing functional indexes**: Query filters `WHERE lower(email) = 'test@example.com'`; standard index on `email` cannot be used. Create `CREATE INDEX idx_users_lower_email ON users (lower(email))`.
3. **Data type mismatches and casts**: Querying `WHERE timestamp_col::date = '2023-01-01'` forces a sequential scan because the column is cast to a date. Use `WHERE timestamp_col >= '2023-01-01' AND timestamp_col < '2023-01-02'` instead to allow index usage.

## Aggregate Optimization

Large aggregations can easily spill to disk or take excessive time if not optimized. PostgreSQL uses either HashAgg (in-memory hash tables) or GroupAgg (requires sorted input).

### Increasing work_mem

If `EXPLAIN ANALYZE` shows `external merge Disk` in a sort, or a HashAgg spilling to disk, the `work_mem` configuration may be too small for the query.

```sql
-- Temporarily boost memory for a heavy reporting query
SET LOCAL work_mem = '256MB';
SELECT customer_id, count(*), sum(amount)
FROM historical_orders
GROUP BY customer_id;
```

### Grouping Sets, Rollup, and Cube

When building dashboards, do not run multiple queries to calculate grand totals and subtotals. Use `GROUPING SETS`, `ROLLUP`, or `CUBE` to compute multiple levels of aggregation in a single scan.

```sql
SELECT 
    COALESCE(department, 'ALL_DEPARTMENTS') AS dept,
    COALESCE(role, 'ALL_ROLES') AS role,
    COUNT(*) AS head_count,
    SUM(salary) AS total_payroll
FROM employees
GROUP BY ROLLUP (department, role);
```
This requires a single pass over the `employees` table, rather than separate queries for the detailed breakdown, department subtotals, and the grand total.

## Prepared Statements

Prepared statements parse and plan queries once, avoiding planning overhead for repeated execution. This is critical for high-frequency queries.

```sql
PREPARE get_user_by_email(text) AS
SELECT id, username, status FROM users WHERE email = $1;

EXECUTE get_user_by_email('test@example.com');
```

Note that pgBouncer requires session pooling or specific configurations to support prepared statements seamlessly. If transaction pooling is enabled, prepared statements must be disabled in the client OR you must use pgBouncer 1.21+ which supports prepared statement routing.
