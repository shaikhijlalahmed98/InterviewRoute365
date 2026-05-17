# 🗄️ SQL Locking & Isolation Cheatsheet

Quick reference for database concurrency questions.

---

## Lock Types

### 🔵 Shared Lock (S-Lock)
- Taken on **read**
- Multiple transactions can hold simultaneously
- Blocks writers until released
- Released at end of transaction (READ_COMMITTED) or each statement (READ_UNCOMMITTED)

### 🔴 Exclusive Lock (X-Lock)
- Taken on **write** (UPDATE/DELETE/INSERT)
- Only ONE transaction at a time
- Blocks readers AND writers
- Released only at COMMIT/ROLLBACK

### 🟡 Update Lock (U-Lock)
- Intermediate: "I might update this"
- Compatible with S-Locks, incompatible with U-Locks
- Prevents deadlocks during read-then-update patterns
- Converts to X-Lock if actual update happens

---

## Isolation Levels (Weakest → Strongest)

### 1. READ UNCOMMITTED
- Can see other transactions' uncommitted changes ("dirty reads")
- Almost never used
- **Use case**: Reporting where slight inaccuracy is acceptable

### 2. READ COMMITTED ⭐ (Default for SQL Server, Oracle, PostgreSQL)
- Only sees committed data
- Non-repeatable reads possible (same query, different result later)
- **Use case**: 90% of applications

### 3. REPEATABLE READ (Default for MySQL InnoDB)
- Rows read in transaction stay frozen
- Phantom reads possible (new matching rows can appear)
- **Use case**: Re-reading same row multiple times in transaction

### 4. SERIALIZABLE
- Transactions appear to run one-by-one
- Strongest, slowest
- **Use case**: Banking, critical financial

---

## Phenomena Matrix

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------------|-----------|---------------------|--------------|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible |
| READ COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible |
| REPEATABLE READ | ❌ Prevented | ❌ Prevented | ✅ Possible |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented |

---

## Optimistic vs Pessimistic Locking

### Pessimistic
**Philosophy**: "I'll lock first, ask questions later"
```sql
BEGIN TRANSACTION;
SELECT * FROM accounts WHERE id=5 FOR UPDATE;  -- LOCK
-- do work
UPDATE accounts SET balance=... WHERE id=5;
COMMIT;
```
- ✅ No conflicts possible
- ❌ Throughput suffers, deadlock risk

### Optimistic
**Philosophy**: "No conflict probably; check at commit"
```sql
-- Read with version
SELECT *, version FROM accounts WHERE id=5;  -- version=3

-- Update with version check
UPDATE accounts SET balance=..., version=4
WHERE id=5 AND version=3;
-- 0 rows = conflict, retry
```
- ✅ High throughput, no locks
- ❌ Failed transactions retry

**Choose pessimistic when**: high contention, short transactions
**Choose optimistic when**: low contention, web apps, microservices

---

## Index Types

### B-Tree Index (Default)
- Balanced tree, log(n) lookup
- Best for: equality, range, sort
- Used by: Primary keys, UNIQUE, most queries

### Composite Index
```sql
CREATE INDEX idx_compound ON users(country, city, age);
```
**Leftmost-prefix rule**: Index used only if leftmost columns in WHERE:
- ✅ `WHERE country=?`
- ✅ `WHERE country=? AND city=?`
- ✅ `WHERE country=? AND city=? AND age=?`
- ❌ `WHERE city=?` (skipped country)
- ❌ `WHERE age=?` (skipped country and city)

### Filtered Index (SQL Server) / Partial Index (PostgreSQL)
```sql
CREATE INDEX idx_active ON users(email)
WHERE is_deleted = 0;
```
- Indexes only matching rows
- Space efficient
- Use case: Soft-delete tables, status filters

### Covering Index
Index includes ALL columns query needs:
```sql
CREATE INDEX idx_cover ON orders(user_id) INCLUDE (status, total);

-- Query satisfied entirely from index, no table lookup
SELECT status, total FROM orders WHERE user_id = 5;
```

### Unique Index
- Enforces uniqueness + fast lookup
- Auto-created by `UNIQUE` constraint
- Race-condition safe (DB-level enforcement)

---

## Atomic UPDATE Pattern (TOCTOU Fix)

**Problem**: Check-then-act race condition.

```sql
-- ❌ BAD: TOCTOU bug
SELECT used FROM tokens WHERE id=5;  -- returns false
-- App checks "not used" ✓
UPDATE tokens SET used=1 WHERE id=5;
-- Concurrent request did same → double consumption!
```

```sql
-- ✅ GOOD: atomic single statement
UPDATE tokens SET used=1
WHERE id=5 AND used=0;

-- Check rowcount: 1 = success, 0 = conflict
```

**Why it works**: SQL engine takes row lock during UPDATE. Concurrent requests serialize on lock, only one's WHERE matches.

---

## Write-Ahead Log (WAL)

**Purpose**: Durability guarantee (the "D" in ACID)

**Mechanism**:
1. Transaction modifies data in memory
2. COMMIT triggered
3. DB writes change to **log file first** (sequential, fast)
4. Log write confirmed → COMMIT acknowledged to client
5. (Async, later) Actual data file updated

**Crash recovery**:
1. DB restart
2. Replay WAL from last checkpoint
3. Committed-but-not-flushed transactions re-apply
4. Database back to consistent state

**Why it's fast**: Sequential write to single file beats random writes to data files.

---

## Connection Pool Tuning

**Default for HikariCP** (Spring's default):
- `maximumPoolSize = 10`

**Rule of thumb**: `pool_size = (cpu_cores * 2) + effective_disk_count`

**Too small** → connection wait queue → API slow
**Too large** → DB overwhelmed, context-switching overhead

**Monitor**: Connection wait time, active connections, idle connections.

---

## Common SQL Anti-Patterns

### ❌ SELECT *
Brings unnecessary columns, breaks covering indexes, network waste.

### ❌ N+1 Query
```java
List<Order> orders = ...;
for (Order o : orders) {
    o.getCustomer();  // separate query per order!
}
```
**Fix**: Eager fetch, JOIN FETCH, or batch loading.

### ❌ Function on Indexed Column
```sql
WHERE UPPER(email) = 'ADMIN@...'  -- index unused!
WHERE YEAR(created_at) = 2025     -- index unused!
```
**Fix**: Functional index, or normalize at write time.

### ❌ Implicit Type Conversion
```sql
WHERE user_id = '5'  -- if user_id is INT, casts every row!
```
**Fix**: Match parameter type to column type.

### ❌ OR Clauses on Different Columns
```sql
WHERE name = ? OR email = ?  -- index seek impossible
```
**Fix**: UNION ALL of two indexed queries.
