# PostgreSQL Deep Dive

## 1. Why PostgreSQL?

Problem solved

- Strong consistency
- Complex SQL
- ACID transactions
- Reliable storage

---

## 2. Storage

- Heap
- Pages
- B+Tree Index

Key idea:

Heap stores rows.

Indexes store pointers.

---

## 3. Read Path

Planner

↓

Index Scan / Sequential Scan

↓

Heap

↓

Return rows

Planner chooses the cheapest execution plan based on statistics.

---

## 4. Transactions

ACID properties

Isolation levels

MVCC

---

## 5. MVCC

PostgreSQL never overwrites rows.

Every update creates a new row version.

Readers see the appropriate version according to transaction visibility.

Old versions become dead tuples.

---

## 6. WAL

Every committed modification is first written to the Write Ahead Log.

Purpose

- Durability
- Crash Recovery
- Replication

---

## 7. Checkpoints

Heap pages remain dirty in memory after commit.

Checkpoint periodically flushes dirty pages to disk.

Crash recovery only replays WAL after the latest checkpoint.

---

## 8. VACUUM

Removes obsolete row versions.

Reclaims space.

Prevents table bloat.

---

## 9. Replication

Primary streams WAL to replicas.

Replica replays WAL to reach the same state.

Read scaling.

High availability.

---

## 10. Partitioning

Range

List

Hash

Partition pruning allows PostgreSQL to skip irrelevant partitions.

---

## 11. Scaling Limits

Scale Up

↓

Read Replicas

↓

Partitioning

↓

Caching

↓

Sharding

↓

Distributed Database

PostgreSQL excels at correctness but eventually reaches write scalability limits due to its single-primary architecture.
