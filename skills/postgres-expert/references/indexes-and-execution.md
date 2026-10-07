# Index Engineering and Execution Plans

Detailed guide for index selection, composite ordering, index types, and reading `EXPLAIN (ANALYZE, BUFFERS)` in PostgreSQL 17.

## Reading EXPLAIN (ANALYZE, BUFFERS)

Always run `EXPLAIN` with both `ANALYZE` and `BUFFERS` in staging to see cache hits and disk reads.

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT order_id, total_amount
FROM orders
WHERE customer_id = 42 AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;
```

Key metrics to evaluate:
- **Shared Hit Blocks**: Read from PostgreSQL memory cache (`shared_buffers`). High is good.
- **Shared Read Blocks**: Read from OS disk cache or NVMe storage. Minimize this.
- **Rows Removed by Filter**: Indicates an inefficient index filter step where rows were read and discarded.
- **Sort Method (external merge vs quicksort)**: If disk sort occurs, check `work_mem` settings.

## Composite Index Column Ordering

Order composite index columns based on:
1. **Equality columns first**: Columns filtered with `=`
2. **Range / Sort columns last**: Columns filtered with `<`, `>`, `BETWEEN`, or used in `ORDER BY`

```sql
-- Query:
-- WHERE tenant_id = 12 AND created_at >= '2026-01-01' ORDER BY created_at DESC

-- Correct index:
CREATE INDEX idx_events_tenant_created 
ON events (tenant_id, created_at DESC);

-- Inefficient index (range filter prevents using subsequent columns):
CREATE INDEX idx_events_wrong 
ON events (created_at DESC, tenant_id);
```

## Covering Indexes with INCLUDE

Use covering indexes to enable Index-Only Scans without bloating the B-Tree sort keys:

```sql
-- Query fetches email for an active account:
-- SELECT email FROM accounts WHERE account_id = $1 AND is_active = true;

CREATE INDEX idx_accounts_active_covering
ON accounts (account_id)
INCLUDE (email)
WHERE is_active = true;
```

The `INCLUDE` payload columns are stored in leaf pages only, keeping branch nodes compact and traversals fast.

## Partial Indexes

Index only the subset of rows queried frequently to save RAM and write IOPS:

```sql
-- Only index unresolved queue tasks
CREATE INDEX idx_task_queue_unprocessed
ON task_queue (priority DESC, scheduled_at ASC)
WHERE processed_at IS NULL;

-- Only index soft-deleted records when active
CREATE INDEX idx_users_active_lookup
ON users (email)
WHERE deleted_at IS NULL;
```

## Specialized Index Types

- **B-Tree**: Default, general purpose for equality, range, and sort operations.
- **BRIN (Block Range Index)**: Extremely compact for append-only, naturally sorted timeseries or sequential IDs.
- **GIN (Generalized Inverted Index)**: For JSONB (`jsonb_ops` supports `@>`, `?`; `jsonb_path_ops` supports only `@>`), full-text search (`tsvector`), and array lookups (`&&`).

```sql
-- Timeseries BRIN index (uses kilobytes instead of gigabytes):
CREATE INDEX idx_audit_logs_brin 
ON audit_logs USING BRIN (created_at);

-- JSONB GIN index using jsonb_path_ops:
CREATE INDEX idx_documents_payload_gin 
ON documents USING GIN (metadata jsonb_path_ops);
```

## Advanced Indexing Strategies

### Covering Indexes with INCLUDE

To achieve an index-only scan, an index must cover all columns referenced in the `SELECT`, `WHERE`, `ORDER BY`, and `GROUP BY` clauses. Adding non-key columns to the `INCLUDE` clause allows the index to satisfy the query without visiting the heap, while keeping the B-tree lean:

```sql
-- The B-tree is organized by (customer_id, status)
-- The payload (total_amount, created_at) is simply stored in the leaf nodes
CREATE INDEX idx_orders_customer_status_inc
ON orders (customer_id, status)
INCLUDE (total_amount, created_at);
```

### Expression Indexes

Expression indexes evaluate a function or operation at insertion time and index the result. This is highly effective for case-insensitive searches or extracting JSONB properties.

```sql
-- Case-insensitive search
CREATE INDEX idx_users_email_lower
ON users (LOWER(email));

-- Indexing a specific JSONB property
CREATE INDEX idx_users_country
ON users ((preferences->>'country'));
```

When querying, the expression in the `WHERE` clause must exactly match the index expression.

### Analyzing Index Usage

Regularly audit index usage to identify unused or redundant indexes. Unused indexes consume disk space and slow down `INSERT`, `UPDATE`, and `DELETE` operations.

```sql
-- Find unused indexes
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indisunique IS FALSE
ORDER BY relname;
```

_Note on JSONB GIN indexes:_ The default `jsonb_ops` operator class supports `@>`, `?`, `?&`, and `?|`. The `jsonb_path_ops` class is more space-efficient but only supports the `@>` (contains) operator. Choose the operator class based on the queries your application executes.
