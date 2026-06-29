# PostgreSQL Interview Questions

## Explain the PostgreSQL write path.

Planner

↓

B+Tree

↓

Heap

↓

Row Lock

↓

MVCC

↓

WAL

↓

Commit

↓

Checkpoint

↓

VACUUM

---

## Why does PostgreSQL use MVCC?

To allow readers and writers to proceed concurrently without blocking each other.

---

## Why does PostgreSQL need WAL?

To guarantee durability, enable crash recovery, and support replication.

---

## Why doesn't PostgreSQL always use an index?

Because the planner chooses the lowest-cost execution plan based on estimated statistics.

---

## Why is VACUUM necessary?

MVCC creates obsolete row versions that must eventually be removed to reclaim space.

---

## Difference between Partitioning and Sharding?

Partitioning keeps data on one PostgreSQL instance.

Sharding distributes data across multiple database servers.
