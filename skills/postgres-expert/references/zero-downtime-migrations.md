# Zero-Downtime PostgreSQL Migrations

Best practices for executing schema migrations safely on high-throughput PostgreSQL databases without blocking transactions or table locks.

## Lock Hierarchy and The Danger of Lock Queues

DDL operations require table locks (`ACCESS EXCLUSIVE`). When an `ACCESS EXCLUSIVE` lock request is queued behind a long-running `SELECT`, all subsequent queries to that table are blocked behind the queued lock request, causing connection pool exhaustion and application outage.

### Rule 1: Always Set Lock Timeout

Every migration script modifying production schemas must specify a short lock timeout:

```sql
SET lock_timeout = '2s';
SET statement_timeout = '30s';
```

If the lock cannot be acquired within 2 seconds, the migration aborts immediately, letting background queries continue without queueing.

## Non-Blocking Index Creation

Never run `CREATE INDEX` in transactional migration steps on large tables.

```sql
-- Safe: Builds index concurrently without holding table write locks
CREATE INDEX CONCURRENTLY idx_users_organization_id
ON users (organization_id);
```

If `CREATE INDEX CONCURRENTLY` fails, it leaves behind an `INVALID` index stub that continues to incur write overhead. Always check for and drop invalid indexes:

```sql
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE NOT indisvalid;

-- Clean up invalid index
DROP INDEX CONCURRENTLY IF EXISTS idx_users_organization_id;
```

## Adding Columns Safely

In PostgreSQL 11+, adding a column with a constant default does not rewrite the table:

```sql
-- Fast metadata-only update in PostgreSQL 11+:
ALTER TABLE orders 
ADD COLUMN is_archived BOOLEAN NOT NULL DEFAULT false;

-- For volatile defaults (e.g. gen_random_uuid()), add column nullable first:
ALTER TABLE orders ADD COLUMN token UUID;

-- Backfill in small batches to avoid locking huge numbers of rows and blooming the table:
DO $$
DECLARE
    row_count INT;
BEGIN
    LOOP
        WITH to_update AS (
            SELECT order_id
            FROM orders
            WHERE token IS NULL
            LIMIT 5000
            FOR UPDATE SKIP LOCKED
        )
        UPDATE orders
        SET token = gen_random_uuid()
        FROM to_update
        WHERE orders.order_id = to_update.order_id;

        GET DIAGNOSTICS row_count = ROW_COUNT;
        EXIT WHEN row_count = 0;
        COMMIT;
    END LOOP;
END $$;

-- Add a CHECK constraint as NOT VALID, which is fast and does not scan the table
ALTER TABLE orders ADD CONSTRAINT token_not_null CHECK (token IS NOT NULL) NOT VALID;

-- Validate the constraint in the background (takes SHARE UPDATE EXCLUSIVE, non-blocking)
ALTER TABLE orders VALIDATE CONSTRAINT token_not_null;

-- Safely swap to SET NOT NULL now that PostgreSQL knows all rows pass the check
ALTER TABLE orders ALTER COLUMN token SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT token_not_null;
```

## Adding Foreign Keys Safely

Adding a foreign key constraint normally requires scanning the entire table under `SHARE ROW EXCLUSIVE` lock. Break this into two safe steps:

```sql
-- Step 1: Add constraint without validating existing rows (takes immediate light lock)
ALTER TABLE line_items
ADD CONSTRAINT fk_line_items_order_id
FOREIGN KEY (order_id) REFERENCES orders(order_id)
NOT VALID;

-- Step 2: Validate existing rows in background (takes SHARE UPDATE EXCLUSIVE, non-blocking)
ALTER TABLE line_items
VALIDATE CONSTRAINT fk_line_items_order_id;
```

## Enum Migrations

Adding values to enums must be done safely outside a transaction block:

```sql
-- Adds a new value without rebuilding the table
ALTER TYPE order_status ADD VALUE 'refunded';
```

Note that enum values cannot be removed without rebuilding the entire type. For dynamic lists that shrink, use a lookup table instead of an enum.
