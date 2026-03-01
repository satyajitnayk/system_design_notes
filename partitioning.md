# MySQL Partitioning – System Design Revision Notes

## 1. Goal

Understand:

* When partitioning helps
* How to create partitioned tables
* How pruning works
* How aggregation behaves with partitions
* Tradeoffs vs indexing

---

# 2. Example 1 – Normal Events Table

```sql
CREATE TABLE events (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  event_type VARCHAR(50),
  amount DECIMAL(10,2),
  created_at DATETIME NOT NULL,
  INDEX idx_created_at (created_at),
  INDEX idx_user_id (user_id)
);
```

### Use Case

* Moderate dataset
* Queries mostly filter by user_id
* No large time-based deletion

### Typical Query

```sql
SELECT user_id, SUM(amount)
FROM events
WHERE created_at >= '2025-01-01'
  AND created_at < '2026-01-01'
GROUP BY user_id;
```

---

# 3. Example 2 – Partitioned Events Table (Yearly)

```sql
CREATE TABLE events_partitioned (
  id BIGINT AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  event_type VARCHAR(50),
  amount DECIMAL(10,2),
  created_at DATETIME NOT NULL,
  PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE COLUMNS(created_at) (
  PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
  PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
  PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
  PARTITION pmax VALUES LESS THAN (MAXVALUE)
);
```

---

# 4. What Partitioning Actually Does

Partitioning:

* Splits one logical table into multiple physical segments
* Each partition stores a subset of rows
* MySQL can skip partitions using pruning

It does NOT:

* Automatically parallelize queries
* Replace proper indexing
* Speed up full-table aggregations

---

# 5. Partition Pruning

Pruning works ONLY when query filters on partition key.

### Pruning Example

```sql
EXPLAIN
SELECT *
FROM events_partitioned
WHERE created_at >= '2025-01-01'
  AND created_at < '2026-01-01';
```

Expected: Only partition p2025 scanned.

### No Pruning Example

```sql
SELECT user_id, SUM(amount)
FROM events_partitioned
GROUP BY user_id;
```

Result: All partitions scanned.

---

# 6. Aggregation Across Partitions

Global aggregation:

* Scans every partition
* Partitioning does not reduce work

Manual per-partition aggregation:

```sql
SELECT user_id, SUM(amount)
FROM events_partitioned PARTITION (p2025)
GROUP BY user_id;
```

Then merge results.

Usually unnecessary unless parallelizing externally.

---

# 7. When Partitioning Helps (System Design Perspective)

Use partitioning when:

1. Time-based data (logs, events, transactions)
2. Need fast deletion of old data
3. Queries filter by time range
4. Table > 100M rows (rule of thumb)

Avoid partitioning when:

1. Small tables
2. No filtering on partition key
3. Heavy joins across tables

---

# 8. Indexing vs Partitioning

Indexing improves:

* Lookup speed
* Aggregation on indexed columns

Partitioning improves:

* Data lifecycle management
* Range pruning
* Large dataset maintenance

They solve different problems.

---

# 9. Operational Advantages

Drop old data instantly:

```sql
ALTER TABLE events_partitioned DROP PARTITION p2023;
```

Add new year:

```sql
ALTER TABLE events_partitioned
ADD PARTITION (
  PARTITION p2026 VALUES LESS THAN ('2027-01-01')
);
```

No full table rewrite.

---

# 10. Production Best Practices

* Always partition by actual query filter column
* Use RANGE COLUMNS instead of YEAR() function
* Keep partition count reasonable (< 100)
* Monitor pruning with EXPLAIN
* Combine with proper secondary indexes

---

# 11. Interview Summary (System Design Angle)

Partitioning is:

* A data management strategy
* Not a performance silver bullet
* Most useful for time-series systems

For large-scale event systems:

* Partition by time
* Index by user_id
* Pre-aggregate for analytics

---

# 12. Mental Model

Think of partitioning as:

"Multiple physical tables behind one logical table, automatically routed by key."

If your query doesn't filter by that key → you scan everything.

---

# 13. EXPLAIN vs EXPLAIN ANALYZE (Clarity Section)

## A. EXPLAIN (With Partition Pruning)

```sql
EXPLAIN
SELECT *
FROM events_partitioned
WHERE created_at >= '2025-01-01'
  AND created_at < '2026-01-01';
```

Typical Output (Pruning Happens):

```
id: 1
select_type: SIMPLE
table: events_partitioned
partitions: p2025
type: range
possible_keys: PRIMARY
key: PRIMARY
rows: 120000
Extra: Using where
```

Key Insight:

* partitions: p2025 → only one partition scanned
* rows → estimated rows scanned

---

## B. EXPLAIN (No Partition Filter)

```sql
EXPLAIN
SELECT user_id, SUM(amount)
FROM events_partitioned
GROUP BY user_id;
```

Typical Output (No Pruning):

```
id: 1
select_type: SIMPLE
table: events_partitioned
partitions: p2023,p2024,p2025,pmax
type: index
key: idx_user_id
rows: 950000
Extra: Using index
```

Key Insight:

* All partitions scanned
* Partitioning does NOT help global aggregation

---

## C. EXPLAIN ANALYZE (MySQL 8+)

```sql
EXPLAIN ANALYZE
SELECT *
FROM events_partitioned
WHERE created_at >= '2025-01-01'
  AND created_at < '2026-01-01';
```

Typical Output:

```
-> Filter: (created_at >= '2025-01-01' and created_at < '2026-01-01')
    -> Table scan on events_partitioned partition p2025
       rows=120000  cost=12000
       actual rows=118532  loops=1
```

Key Differences vs EXPLAIN:

EXPLAIN:

* Shows estimated plan
* Uses optimizer statistics

EXPLAIN ANALYZE:

* Executes query
* Shows actual rows scanned
* Shows real execution timing
* Best tool to verify pruning + performance

---

## D. How to Validate Partitioning Correctly

1. Run EXPLAIN
2. Check "partitions" column
3. Run EXPLAIN ANALYZE
4. Confirm only expected partition scanned

If more partitions appear than expected →

* Query is not aligned with partition key
* Boundaries may be wrong

---

END
