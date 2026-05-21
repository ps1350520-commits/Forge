# Database — Design, Migrations, Indexing & Query Optimization

## Schema Design Principles

### Naming conventions (be consistent — pick one and enforce it)
```sql
-- snake_case for everything (recommended)
CREATE TABLE order_items (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id      UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id    UUID NOT NULL REFERENCES products(id),
  quantity      INTEGER NOT NULL CHECK (quantity > 0),
  unit_price_cents INTEGER NOT NULL CHECK (unit_price_cents >= 0),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Rules:
-- Tables: plural nouns (orders, users, products, order_items)
-- Primary keys: always `id`
-- Foreign keys: <table_singular>_id (order_id, user_id)
-- Timestamps: created_at, updated_at (always TIMESTAMPTZ, never TIMESTAMP)
-- Soft delete: deleted_at TIMESTAMPTZ NULL (null = active)
-- Booleans: is_active, has_verified_email, can_retry
-- Money: always store as integer cents — never FLOAT for currency
```

### Normalization — when to and when not to
```
1NF: One value per cell. No repeating groups.
2NF: Every non-key column depends on the whole primary key.
3NF: Every non-key column depends only on the primary key (no transitive deps).

Denormalize intentionally when:
- Read performance is critical and joins are expensive
- Data changes infrequently (user's full name in an audit log)
- The duplicated data is truly a snapshot (order line items store price at time of purchase)
```

### Storing money — never floats
```sql
-- ❌ Float precision errors will cause financial discrepancies
price DECIMAL(10,2)  -- still risky
price FLOAT          -- never

-- ✅ Integer cents (or smallest currency unit)
price_cents INTEGER NOT NULL  -- 1999 = $19.99
-- Application layer converts: cents / 100 for display
```

### Timestamps — always timezone-aware
```sql
-- ❌ TIMESTAMP: stores local time, ambiguous on DST change
created_at TIMESTAMP DEFAULT NOW()

-- ✅ TIMESTAMPTZ: stores UTC, converts on retrieval
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- Always updated_at trigger
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

### Soft deletes vs hard deletes
```sql
-- Soft delete: add deleted_at column
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Create a view for active records
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial index for performance (only indexes non-deleted rows)
CREATE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;

-- Hard deletes: use for truly disposable data (sessions, temp data, logs)
```

---

## Indexing Strategy

### The golden rule: index columns used in WHERE, JOIN, ORDER BY, GROUP BY

```sql
-- Analyze your queries first
EXPLAIN ANALYZE SELECT * FROM orders
WHERE user_id = $1 AND status = 'pending'
ORDER BY created_at DESC;

-- If you see "Seq Scan" on a large table → you need an index
-- If you see "Index Scan" → good
-- Check "actual rows" vs "estimated rows" — large mismatch = stale statistics
-- Run: ANALYZE orders; to update statistics
```

### Types of indexes

**B-tree (default)** — Range queries, equality, ORDER BY
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

**Composite index** — Multiple columns. Column order matters: most selective first, or match query pattern
```sql
-- Query: WHERE user_id = ? AND status = ? ORDER BY created_at DESC
-- Correct composite:
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at DESC);

-- Rule: equality columns first, range/sort columns last
```

**Partial index** — Only indexes rows matching a condition. Smaller, faster.
```sql
-- Most queries only care about active/pending records
CREATE INDEX idx_orders_pending ON orders(user_id, created_at)
WHERE status = 'pending';

CREATE INDEX idx_users_unverified ON users(email)
WHERE email_verified_at IS NULL;
```

**Expression index** — Index on a computed value
```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Query must use the same expression:
SELECT * FROM users WHERE LOWER(email) = LOWER($1);
```

**GIN index** — Full-text search, JSONB, arrays
```sql
-- Full-text search
ALTER TABLE products ADD COLUMN search_vector TSVECTOR;
CREATE INDEX idx_products_fts ON products USING GIN(search_vector);

-- JSONB column
CREATE INDEX idx_orders_metadata ON orders USING GIN(metadata);
-- Query: WHERE metadata @> '{"source": "mobile"}'
```

### Index maintenance
```sql
-- Find unused indexes (wasted write overhead)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY schemaname, tablename;

-- Find missing indexes (sequential scans on large tables)
SELECT schemaname, tablename, seq_scan, seq_tup_read,
       idx_scan, seq_tup_read / seq_scan AS avg_seq_read
FROM pg_stat_user_tables
WHERE seq_scan > 0 AND seq_tup_read / seq_scan > 1000
ORDER BY seq_tup_read DESC;

-- Rebuild bloated indexes (after heavy deletes/updates)
REINDEX INDEX CONCURRENTLY idx_orders_user_id;
```

---

## Migrations — Safe Schema Changes

### The expand/contract pattern (zero-downtime migrations)

Never make breaking changes in one step. Use three phases:

**Phase 1 — Expand:** Add the new structure alongside the old. Both old and new code work.
```sql
-- Add new column (nullable so existing rows don't break)
ALTER TABLE users ADD COLUMN display_name VARCHAR(100);

-- Backfill in batches (don't lock the table)
UPDATE users SET display_name = full_name
WHERE id IN (SELECT id FROM users WHERE display_name IS NULL LIMIT 1000);
-- Run this repeatedly until backfill is complete
```

**Phase 2 — Migrate:** Deploy code that uses the new structure. Both old and new column used.
```sql
-- Once all rows are backfilled and new code is deployed:
ALTER TABLE users ALTER COLUMN display_name SET NOT NULL;
```

**Phase 3 — Contract:** Remove the old structure once new code is fully rolled out.
```sql
-- Only safe after the old column is no longer referenced anywhere in code
ALTER TABLE users DROP COLUMN full_name;
```

### Migration file conventions
```
migrations/
  001_create_users.sql
  002_add_email_index.sql
  003_add_display_name_expand.sql    ← phase 1
  004_backfill_display_name.sql      ← phase 2 (or a script)
  005_drop_full_name_contract.sql    ← phase 3 (deployed separately)
```

### Safe vs dangerous migrations

| Safe (non-blocking) | Dangerous (may lock table) |
|---|---|
| Add nullable column | Add NOT NULL column without default |
| Add index CONCURRENTLY | Add index without CONCURRENTLY |
| Add/drop constraint NOT VALID | Add constraint that validates all rows |
| Drop column (contract phase) | Rename column |
| Create table | Change column type |

```sql
-- Safe index creation (doesn't lock table)
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);

-- Safe NOT NULL addition (add default first, backfill, then add constraint)
ALTER TABLE orders ADD COLUMN currency VARCHAR(3) DEFAULT 'USD';
-- ... backfill ...
ALTER TABLE orders ALTER COLUMN currency SET NOT NULL;
ALTER TABLE orders ALTER COLUMN currency DROP DEFAULT; -- if not wanted
```

---

## Query Optimization Patterns

### Fix N+1 queries

```typescript
// ❌ N+1: 1 query for orders + N queries for users
const orders = await db.query('SELECT * FROM orders')
for (const order of orders) {
  order.user = await db.query('SELECT * FROM users WHERE id = $1', [order.user_id])
}

// ✅ Single JOIN
const orders = await db.query(`
  SELECT o.*, u.name AS user_name, u.email AS user_email
  FROM orders o
  JOIN users u ON u.id = o.user_id
  WHERE o.status = $1
`, ['pending'])

// ✅ Or batch fetch and map in application layer
const orders = await db.query('SELECT * FROM orders WHERE status = $1', ['pending'])
const userIds = [...new Set(orders.map(o => o.user_id))]
const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [userIds])
const userMap = Object.fromEntries(users.map(u => [u.id, u]))
orders.forEach(o => { o.user = userMap[o.user_id] })
```

### Cursor pagination (performant at scale)
```sql
-- ❌ Offset pagination — O(offset) cost, degrades at scale
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;

-- ✅ Cursor pagination — constant performance
SELECT * FROM orders
WHERE created_at < $1  -- cursor is the last seen value
ORDER BY created_at DESC
LIMIT 20;
-- Return last row's created_at as the next cursor
```

### Avoiding common query pitfalls
```sql
-- ❌ Function on indexed column defeats the index
SELECT * FROM users WHERE UPPER(email) = 'ALICE@EXAMPLE.COM';
-- ✅ Use expression index or store normalized value
SELECT * FROM users WHERE email = LOWER('Alice@Example.com');

-- ❌ Implicit type cast prevents index use
SELECT * FROM orders WHERE user_id = '123';  -- user_id is INTEGER
-- ✅ Match types
SELECT * FROM orders WHERE user_id = 123;

-- ❌ SELECT * on wide tables — over-fetching
SELECT * FROM users;
-- ✅ Select only what you need
SELECT id, name, email FROM users;

-- ❌ Unbounded query
SELECT * FROM audit_logs WHERE user_id = $1;
-- ✅ Always paginate
SELECT * FROM audit_logs WHERE user_id = $1 ORDER BY created_at DESC LIMIT 100;
```

### Connection pooling
```typescript
// Never create a new connection per request
// ❌ New connection per request
app.get('/users', async (req, res) => {
  const client = new pg.Client(connectionString)
  await client.connect()
  const { rows } = await client.query('SELECT * FROM users')
  await client.end()
  res.json(rows)
})

// ✅ Shared pool
const pool = new pg.Pool({ connectionString, max: 20, idleTimeoutMillis: 30000 })
app.get('/users', async (req, res) => {
  const { rows } = await pool.query('SELECT * FROM users')
  res.json(rows)
})
```

---

## Transactions — When and How

```typescript
// Use transactions when multiple operations must succeed or fail together
async function transferFunds(fromId: string, toId: string, amountCents: number) {
  const client = await pool.connect()
  try {
    await client.query('BEGIN')

    const { rows: [from] } = await client.query(
      'SELECT balance_cents FROM accounts WHERE id = $1 FOR UPDATE', // row lock
      [fromId]
    )

    if (from.balance_cents < amountCents) {
      throw new InsufficientFundsError()
    }

    await client.query(
      'UPDATE accounts SET balance_cents = balance_cents - $1 WHERE id = $2',
      [amountCents, fromId]
    )
    await client.query(
      'UPDATE accounts SET balance_cents = balance_cents + $1 WHERE id = $2',
      [amountCents, toId]
    )
    await client.query(
      'INSERT INTO transfers (from_id, to_id, amount_cents) VALUES ($1, $2, $3)',
      [fromId, toId, amountCents]
    )

    await client.query('COMMIT')
  } catch (err) {
    await client.query('ROLLBACK')
    throw err
  } finally {
    client.release()
  }
}
```
