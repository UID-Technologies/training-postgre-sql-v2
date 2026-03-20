
# 🧩 **Lab 13 – PostgreSQL Performance Tuning, Indexing & Monitoring (Hands-on Lab)**

---

## 🎯 **Objectives**

By the end of this lab, learners will be able to:

* Interpret **query execution plans** using `EXPLAIN` and `EXPLAIN ANALYZE`.
* Explain and compare **Sequential Scan vs Index Scan** behavior.
* Use **VACUUM**, **ANALYZE**, and **Autovacuum** to maintain performance.
* Design basic **indexing strategies** for high-volume workloads.
* Apply **table partitioning** concepts to large datasets.
* Monitor active queries and locks using **pg_stat_activity** and `pg_locks`.

---

## 🧭 **Introduction & Concept Overview**

PostgreSQL’s performance depends on:

- How the **planner** executes queries (execution plans, scans, use of indexes).
- How well **statistics and storage** are maintained (VACUUM/ANALYZE/autovacuum).
- How **indexes and partitions** are designed for real workloads.
- How effectively you **monitor** running queries and locks.

This lab walks through each of these, using a synthetic `sales` workload to simulate a large table.

| Area                 | Description                                                                                                               |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Execution Plans**  | `EXPLAIN` shows the plan; `EXPLAIN ANALYZE` executes and measures it.                                                    |
| **Sequential Scan**  | Reads the entire table – often fine for small tables, expensive for large ones.                                          |
| **Index Scan**       | Uses an index to fetch matching rows directly.                                                                           |
| **VACUUM / ANALYZE** | Reclaims dead tuples and updates planner statistics.                                                                     |
| **Autovacuum**       | Automatic background maintenance based on activity thresholds.                                                            |
| **Partitioning**     | Splits large tables into smaller child tables for faster access and maintenance.                                        |
| **pg_stat_activity** | Shows current backend activity, query text, and, together with `pg_locks`, helps diagnose blocking.                      |

---

## 🧰 **Step 0 – Setup (make this lab independent)**

Continue with your running container or start a new one:

```bash
docker exec -it postgres-container bash
psql -U postgres
```

Create and connect to a new database:

```sql
CREATE DATABASE performancelab;
\c performancelab;
```

---

# 🧪 **Lab Exercises**

---

## 🔹 **Step 1 – Create Sample Table and Populate Data**

```sql
CREATE TABLE sales (
  id SERIAL PRIMARY KEY,
  product TEXT,
  category TEXT,
  quantity INT,
  price NUMERIC(10,2),
  sale_date DATE
);

-- Insert 1 million rows (simulated for real test)
INSERT INTO sales (product, category, quantity, price, sale_date)
SELECT
  'Product_' || (random()*100)::INT,
  CASE WHEN random() < 0.5 THEN 'Electronics' ELSE 'Clothing' END,
  (random()*10)::INT + 1,
  (random()*5000)::NUMERIC(10,2),
  NOW() - ((random()*365)::INT * INTERVAL '1 day')
FROM generate_series(1,1000000);
```

✅ **Observation:** A large table is created to simulate heavy workload.

---

## 🔹 **Step 2 – View Execution Plans (EXPLAIN / ANALYZE)**

```sql
EXPLAIN SELECT * FROM sales WHERE category='Electronics';

EXPLAIN ANALYZE SELECT * FROM sales WHERE category='Electronics';
```

✅ **Observation:**

* Output shows `Seq Scan on sales` because no index exists.
* `EXPLAIN ANALYZE` includes actual execution time.

---

## 🔹 **Step 3 – Create Index and Compare**

```sql
CREATE INDEX idx_sales_category ON sales(category);

EXPLAIN ANALYZE SELECT * FROM sales WHERE category='Electronics';
```

✅ **Observation:**
Now you should see `Index Scan using idx_sales_category` with lower execution time.

---

## 🔹 **Step 4 – Sequential vs Index Scan Comparison**

```sql
SET enable_seqscan = off;    -- Force index scan
EXPLAIN ANALYZE SELECT * FROM sales WHERE category='Clothing';

SET enable_seqscan = on;     -- Restore default
EXPLAIN ANALYZE SELECT * FROM sales WHERE category='Clothing';
```

✅ **Observation:**
Disabling sequential scans forces the planner to use the index even if it’s not optimal.

---

## 🔹 **Step 5 – VACUUM and ANALYZE**

Simulate updates and cleanup:

```sql
UPDATE sales SET price = price * 1.05 WHERE category='Electronics';

VACUUM sales;     -- Reclaim space
ANALYZE sales;    -- Refresh statistics
```

✅ **Observation:**
`ANALYZE` helps the optimizer choose better plans; `VACUUM` prevents table bloat.

### Check Autovacuum Status

```sql
SHOW autovacuum;
SELECT relname, last_vacuum, last_autovacuum FROM pg_stat_user_tables;
```

---

## 🔹 **Step 6 – Composite Index and Performance Measurement**

Create a multi-column index for more selective filtering:

```sql
CREATE INDEX idx_sales_category_date ON sales(category, sale_date);

EXPLAIN ANALYZE
SELECT * FROM sales
WHERE category='Electronics' AND sale_date > NOW() - INTERVAL '30 days';
```

✅ **Observation:**
Planner now performs a faster index scan using the composite index.

---

## 🔹 **Step 7 – Partitioning Large Table**

### 1️⃣ Create Parent Table

```sql
CREATE TABLE sales_partitioned (
  id SERIAL,
  product TEXT,
  category TEXT,
  quantity INT,
  price NUMERIC(10,2),
  sale_date DATE
) PARTITION BY RANGE (sale_date);
```

### 2️⃣ Create Monthly Partitions

```sql
CREATE TABLE sales_2025_01 PARTITION OF sales_partitioned
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE sales_2025_02 PARTITION OF sales_partitioned
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');


-- Continue...

CREATE TABLE sales_2025_11 PARTITION OF sales_partitioned
FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE sales_2025_12 PARTITION OF sales_partitioned
FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');
```

### 3️⃣ Insert Data and Test

```sql
INSERT INTO sales_partitioned (product, category, quantity, price, sale_date)
SELECT product, category, quantity, price, sale_date
FROM sales WHERE sale_date > '2025-01-01';

EXPLAIN ANALYZE
SELECT * FROM sales_partitioned WHERE sale_date BETWEEN '2025-01-15' AND '2025-01-31';
```

✅ **Observation:**
Planner only scans the relevant partition (partition pruning).

---

## 🔹 **Step 8 – Monitoring Active and Blocked Sessions**

```sql
SELECT pid, usename, state, query, wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle';

-- Monitor locks
SELECT relation::regclass AS table_name, mode, granted, pid
FROM pg_locks
WHERE NOT granted;
```

✅ **Observation:**
These views help detect long-running or blocked queries.

---

## 🔹 **Step 9 – Identify Slow Queries and Optimize**

### 1️⃣ Enable Query Timing Logs

```sql
ALTER SYSTEM SET log_min_duration_statement = 500; -- log queries slower than 500 ms
SELECT pg_reload_conf();
```

### 2️⃣ Run Slow Query Example

```sql
EXPLAIN ANALYZE SELECT * FROM sales ORDER BY price DESC;
```

Add an index to optimize:

```sql
CREATE INDEX idx_sales_price ON sales(price DESC);
EXPLAIN ANALYZE SELECT * FROM sales ORDER BY price DESC;
```

✅ **Observation:**
Execution time decreases, and the plan now uses `Index Scan Backward`.

---

## 🧾 **Summary**

- **Execution plans**: always inspect `EXPLAIN` / `EXPLAIN ANALYZE` before tuning.
- **Sequential vs Index Scan**: indexes help selective queries on large tables; Seq Scan can still be best for small or broad scans.
- **Maintenance**: `VACUUM`, `ANALYZE`, and autovacuum keep tables lean and statistics fresh.
- **Indexing strategy**: choose single vs composite indexes based on real predicates and sorting needs.
- **Partitioning**: range partitioning combined with pruning can significantly reduce scanned data for large tables.
- **Monitoring**: `pg_stat_activity`, `pg_locks`, and query logging reveal slow or blocked queries.

---

## ✅ **Deliverables**

Each learner should submit:

1. `EXPLAIN ANALYZE` output screenshots (or captured text) **before and after** indexing `sales(category)` and `sales(price)`.
2. Evidence of `VACUUM` / `ANALYZE` on `sales` (commands plus any relevant statistics).
3. At least one example of **partition pruning** on `sales_partitioned`.
4. A sample `pg_stat_activity` + `pg_locks` query showing how they would diagnose a blocked or long-running query.
5. A short written note (3–5 bullets) summarizing which optimization had the biggest effect and why.

---

## 🧩 **Practice Challenges**

1. Create a **covering index** (`INCLUDE` clause, if supported) and test performance on a query that selects extra columns.
2. Build a **BRIN index** on the `sale_date` column and compare its performance and size to a BTREE index.
3. Configure a **custom autovacuum threshold** for `sales` and observe its behavior in `pg_stat_user_tables`.
4. Use `pg_stat_statements` (if enabled) to identify top slow queries and test an optimization on one of them.
5. Simulate **deadlocks** using two sessions updating rows in opposite order and inspect them with `pg_locks`.

---

## 🧹 **Cleanup (optional, to avoid conflicts with future labs)**

When you are done with this lab, you can drop the performance database:

```sql
DROP DATABASE IF EXISTS performancelab;
```


