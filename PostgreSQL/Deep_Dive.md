# PostgreSQL Deep Dive

> **Goal**
>
> This guide explains PostgreSQL from a **System Design Interview** perspective.
>
> Rather than documenting every PostgreSQL feature, the focus is on understanding:
>
> - Why PostgreSQL exists.
> - How it stores and retrieves data.
> - The tradeoffs behind its design.
> - When PostgreSQL is the right choice.

---

# 1. Why PostgreSQL?

Before understanding PostgreSQL internals, it's important to understand **what problem PostgreSQL was built to solve**.

Imagine you're building one of the following systems:

- Banking Application
- Amazon
- Airline Reservation
- Inventory Management
- HR Portal
- Hospital Management

Although these applications are different, they all share one requirement:

> **The data must always remain correct.**

Consider a simple bank transfer.

Initially:

```
Account A : ₹10,000

Account B : ₹5,000
```

After transferring ₹1000,

```
Account A : ₹9,000

Account B : ₹6,000
```

Both updates must succeed together.

The following situation should never happen:

```
Account A : ₹9,000

Account B : ₹5,000
```

Money has disappeared.

Preventing situations like this is one of PostgreSQL's primary responsibilities.

---

## PostgreSQL Philosophy

Every database optimizes for something.

PostgreSQL optimizes for **correctness**.

Its philosophy can be summarized as:

```
Correctness

↓

Transactions

↓

Consistency

↓

Powerful Queries

↓

Performance

↓

Scalability
```

Notice that scalability comes last.

This doesn't mean PostgreSQL cannot scale.

It means PostgreSQL refuses to compromise correctness simply to maximize throughput.

---

## Where PostgreSQL Excels

PostgreSQL is an excellent choice for applications that require:

- ACID transactions
- Strong consistency
- Complex SQL
- Relationships between data
- Joins
- Foreign keys
- Reporting

Typical applications include:

- Banking
- ERP
- E-commerce
- Booking Systems
- Payroll
- User Management

---

## Where PostgreSQL Struggles

PostgreSQL begins to struggle when applications require:

- Millions of writes per second
- Horizontal write scalability
- Massive event ingestion
- Global low-latency writes

Examples include:

- WhatsApp
- Twitter
- IoT platforms
- Logging systems

In these architectures PostgreSQL is often still present, but usually as the **system of record**, while other databases handle caching, search or high-volume writes.

---

> **💡 Common Confusion**
>
> **If PostgreSQL has only one writable primary, how do banks handle millions of customers?**
>
> Banks do **not** use one giant PostgreSQL database.
>
> Instead, they split the system into multiple services such as Accounts, Loans, Payments and Cards. Each service owns its own PostgreSQL database. High-volume services can then be sharded across multiple PostgreSQL clusters.
>
> PostgreSQL provides strong consistency **within** a database, while the application architecture handles scaling across databases.

---

# 2. Storage Engine

Every database must answer one fundamental question:

> **How should data be stored on disk?**

PostgreSQL separates storage into two independent structures.

```
                 PostgreSQL

          ┌──────────┴──────────┐

          ▼                     ▼

       Heap                B+Tree Index
```

Each structure has a different responsibility.

| Component | Responsibility |
|-----------|----------------|
| Heap | Stores table rows |
| B+Tree | Finds rows quickly |

This separation is one of PostgreSQL's most important design decisions.

---

# 3. Heap

The Heap stores the **actual table data**.

Suppose we have the following table.

| ID | Name |
|----|------|
| 1 | Alice |
| 2 | Bob |
| 3 | Charlie |

Many people assume PostgreSQL stores these rows in sorted order.

It doesn't.

Instead, rows are placed wherever free space exists.

Conceptually, the Heap might look like this.

```
Heap

Page 1

----------------

ID 8

Deepak

----------------

Page 2

----------------

ID 2

Bob

----------------

Page 3

----------------

ID 15

Alice
```

Notice that IDs are completely unordered.

This is intentional.

---

## Why is the Heap Unordered?

Imagine the Heap was sorted by ID.

```
1

2

3

4

5
```

Now insert:

```
3.5
```

Everything after 3 would have to move.

Moving large amounts of data on disk is expensive.

Instead, PostgreSQL simply places the new row into any page with available space.

The index is then updated to point to the new location.

This makes inserts and updates much faster.

---

## Mental Model

Think of the Heap as a warehouse.

Boxes are placed on any shelf with free space.

Workers don't care whether Shelf 5 contains IDs larger than Shelf 4.

Their only goal is to store the box quickly.

Finding a box later is the job of the warehouse catalog.

The catalog is the index.

---

> **💡 Common Confusion**
>
> **If the Heap is unordered, how can PostgreSQL execute `ORDER BY ID`?**
>
> The Heap is never responsible for ordering.
>
> PostgreSQL walks the B+Tree in sorted order.
>
> The ordering comes entirely from the index.

---

# 4. Pages

The Heap is divided into fixed-size blocks called **Pages**.

The default page size is **8 KB**.

```
Heap

├── Page 1

├── Page 2

├── Page 3

├── Page 4
```

Pages are the smallest unit PostgreSQL reads from or writes to disk.

Even if a query needs only one row, PostgreSQL loads the entire page into memory.

Reading larger contiguous blocks is significantly faster than performing many tiny disk reads.

---

# 5. B+Tree Indexes

If the Heap is unordered, how does PostgreSQL quickly find a row?

The answer is the **B+Tree**.

Instead of storing complete rows, a B+Tree stores:

```
Search Key

↓

Pointer to Heap Row
```

For example,

```
ID = 15

↓

Heap Page 3

↓

Offset 8
```

When a query searches for ID 15, PostgreSQL first searches the B+Tree.

The B+Tree tells PostgreSQL exactly where the row lives inside the Heap.

The row is then fetched from that location.

---

## Why Doesn't the Index Store the Entire Row?

Suppose a table has five indexes.

If every index stored the complete row, updating a single row would require updating the row inside:

- Heap
- Index 1
- Index 2
- Index 3
- Index 4
- Index 5

Instead, PostgreSQL stores the row only once inside the Heap.

Indexes store lightweight pointers.

This reduces storage and keeps index maintenance efficient.

---

> **💡 Common Confusion**
>
> **Why doesn't PostgreSQL use a Hash Index by default?**
>
> Hash indexes are excellent for exact lookups like:
>
> ```sql
> WHERE id = 15;
> ```
>
> However, they cannot efficiently answer:
>
> ```sql
> WHERE id BETWEEN 10 AND 100;
> ```
>
> or
>
> ```sql
> ORDER BY id;
> ```
>
> B+Trees keep keys sorted, making them suitable for exact lookups, range scans and ordered traversal.

---

# Part 1 Summary

By the end of this chapter you should understand the following ideas.

- PostgreSQL prioritizes correctness over scalability.
- Table rows are stored in an unordered Heap.
- The Heap is divided into 8 KB pages.
- B+Tree indexes store pointers to Heap rows.
- Separating the Heap from indexes allows PostgreSQL to achieve a good balance between read and write performance.

These concepts form the foundation for understanding how PostgreSQL executes queries, which is covered in the next chapter.



# Part 2 — Query Planner & Read Path

At this point we understand how PostgreSQL stores data.

```
Heap

↓

Stores Rows

B+Tree

↓

Stores Pointers
```

The next question is:

> **How does PostgreSQL actually execute a query?**

Suppose we execute:

```sql
SELECT *
FROM users
WHERE id = 15;
```

Finding the row is not as simple as immediately searching the B+Tree.

PostgreSQL first decides **how** to execute the query.

This decision is made by the **Query Planner**.

---

# 6. Query Planner

The Query Planner is the brain of PostgreSQL.

Its job is simple:

> Find the cheapest way to execute a query.

Notice the wording.

Not

> Find the fastest.

Instead,

> Find the lowest estimated cost.

The planner considers many possible execution strategies and estimates how expensive each one will be.

It then selects the cheapest plan.

---

## Example

Suppose the table contains

```
Users

10 rows
```

Query

```sql
SELECT *
FROM users;
```

Should PostgreSQL use an index?

No.

Reading all 10 rows sequentially is cheaper than traversing an index and then visiting the Heap for every row.

---

Now imagine

```
Users

100 Million rows
```

Query

```sql
SELECT *
FROM users
WHERE id = 15;
```

Scanning every row would be extremely expensive.

The planner now prefers an Index Scan.

Notice something important.

**The same SQL query can produce different execution plans depending on the data.**

---

## How does the Planner decide?

The planner estimates costs using statistics collected about the table.

Examples include:

- Number of rows
- Number of pages
- Distribution of values
- Number of distinct values
- Selectivity of predicates

For example,

suppose 99% of users are from India.

Query

```sql
WHERE country = 'India'
```

Using an index may actually be slower because almost the entire table must still be read.

A Sequential Scan is often cheaper.

---

> **💡 Common Confusion**
>
> **Why doesn't PostgreSQL always use an index?**
>
> An index is not automatically faster.
>
> Using an index requires:
>
> 1. Traversing the B+Tree.
> 2. Following pointers into the Heap.
> 3. Reading many different pages.
>
> If a query returns a large percentage of the table, reading the table sequentially is usually faster.

---

# 7. Sequential Scan

The simplest execution strategy is a Sequential Scan.

Imagine the table contains

```
1 Million rows
```

Query

```sql
SELECT *
FROM users;
```

The planner simply starts at the first page.

```
Page 1

↓

Page 2

↓

Page 3

↓

...

↓

Page N
```

Every row is examined.

Rows matching the query are returned.

Sequential Scans are surprisingly fast because modern disks and operating systems are optimized for reading large contiguous blocks.

---

## When is a Sequential Scan used?

Typical situations include:

- Small tables
- Returning most rows
- No useful index exists

---

## Example

```sql
SELECT *
FROM employees
WHERE salary > 0;
```

If almost every employee satisfies the condition,

using an index provides little benefit.

A Sequential Scan is cheaper.

---

# 8. Index Scan

Suppose we execute

```sql
SELECT *
FROM users
WHERE id = 15;
```

The planner chooses an Index Scan.

Execution proceeds as follows.

```
Planner

↓

B+Tree

↓

Heap Pointer

↓

Heap Page

↓

Row
```

The B+Tree quickly identifies the Heap location.

The Heap is then accessed to retrieve the complete row.

This process is extremely efficient when only a small number of rows are required.

---

## Example

Suppose the B+Tree contains

```
15

↓

Page 8

↓

Offset 12
```

PostgreSQL immediately jumps to

```
Heap

↓

Page 8

↓

Row 12
```

No other pages need to be scanned.

---

## Why is Heap access still necessary?

Remember,

the index stores only

```
Key

↓

Pointer
```

The complete row still resides in the Heap.

Therefore,

after locating the pointer,

PostgreSQL must fetch the row.

---

> **💡 Common Confusion**
>
> **Why not store the entire row inside the index?**
>
> Because indexes would become extremely large.
>
> Tables often have multiple indexes.
>
> Storing the complete row in every index would waste storage and significantly slow down writes.

---

# 9. Composite Indexes

Indexes may contain multiple columns.

Suppose our query is

```sql
SELECT title
FROM posts
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

The best index is

```
(user_id, created_at DESC)
```

Notice the order.

The planner first uses

```
user_id
```

to locate the relevant portion of the index.

Within that portion,

rows are already ordered by

```
created_at DESC
```

No additional sorting is required.

---

## Why Column Order Matters

Suppose the index is

```
(created_at, user_id)
```

Searching by

```
user_id
```

becomes much less efficient.

The planner may ignore the index completely.

When designing composite indexes,

always think about

```
WHERE

↓

ORDER BY
```

The filtering columns should generally appear before the sorting columns.

---

# 10. Covering Indexes

Suppose we execute

```sql
SELECT title, created_at
FROM posts
WHERE user_id = 42
ORDER BY created_at DESC
LIMIT 20;
```

The index

```
(user_id, created_at DESC)
```

still requires Heap lookups to retrieve

```
title
```

Instead,

PostgreSQL allows additional columns to be stored inside the index.

```
(user_id, created_at DESC)

INCLUDE(title)
```

Now every required column already exists inside the index.

The Heap no longer needs to be accessed.

This is called an **Index Only Scan**.

---

## Why are Index Only Scans Faster?

Because random Heap access is eliminated.

The planner reads only the index pages.

Less disk I/O generally means better performance.

---

> **💡 Common Confusion**
>
> **Does every query automatically become an Index Only Scan if all columns exist in the index?**
>
> No.
>
> PostgreSQL must also verify that the indexed entry is visible to the current transaction.
>
> If visibility information isn't sufficient, PostgreSQL still performs Heap lookups.
>
> Therefore, an Index Only Scan is an optimization that PostgreSQL uses only when it is safe.

---

# 11. Read Path

We can now understand the complete read path.

Suppose we execute

```sql
SELECT *
FROM users
WHERE id = 15;
```

Execution proceeds as follows.

```
Client

↓

SQL Query

↓

Query Planner

↓

Choose Execution Plan

↓

Index Scan

↓

B+Tree

↓

Heap

↓

Return Row
```

If the planner decides an index is not beneficial,

the path becomes

```
Client

↓

SQL Query

↓

Query Planner

↓

Sequential Scan

↓

Heap

↓

Return Rows
```

The planner is therefore responsible for selecting the most efficient path before any data is actually read.

---

# Part 2 Summary

By the end of this chapter you should understand:

- PostgreSQL does not blindly use indexes.
- The Query Planner chooses the lowest-cost execution plan.
- Sequential Scans are often faster than Index Scans for large result sets.
- Index Scans use B+Trees to locate Heap rows.
- Composite indexes should match the access pattern.
- Covering indexes can eliminate Heap lookups.
- Every query follows the path:

```
Planner

↓

Execution Plan

↓

Heap / Index

↓

Result
```

These concepts explain how PostgreSQL retrieves data efficiently. In the next chapter, we'll examine how PostgreSQL guarantees correctness using transactions, ACID properties, and MVCC.


# Part 3 — Transactions, ACID & MVCC

So far we've learned:

```
Storage

↓

Heap

↓

B+Tree

↓

Query Planner
```

Everything we've discussed answers one question:

> **How does PostgreSQL store and retrieve data?**

Now we answer a much harder question:

> **How does PostgreSQL allow thousands of users to modify data simultaneously without corrupting it?**

This is the purpose of **transactions**.

---

# 12. Transactions

A transaction is a group of operations that PostgreSQL treats as a **single logical unit**.

Either every operation succeeds,

or none of them succeed.

Example:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

COMMIT;
```

This transfer consists of two updates.

From the application's perspective,

both updates represent one business operation.

If one succeeds,

both must succeed.

If one fails,

both must fail.

---

## Why do we need transactions?

Suppose the server crashes after the first UPDATE.

```
Account A

₹10,000

↓

₹9,000
```

```
Account B

₹5,000
```

Money has disappeared.

Without transactions,

the database becomes inconsistent.

Transactions prevent this.

---

# ACID

Every PostgreSQL transaction guarantees four properties.

## Atomicity

Either all operations succeed,

or none succeed.

Think of a light switch.

It is either

```
ON
```

or

```
OFF
```

There is no halfway state.

---

## Consistency

Every committed transaction leaves the database in a valid state.

For example,

suppose the balance cannot become negative.

```
Balance >= 0
```

A transaction violating this rule is rejected.

---

## Isolation

Multiple users can execute transactions simultaneously,

but each transaction behaves as if it were running alone.

We'll see how PostgreSQL achieves this using MVCC.

---

## Durability

Once PostgreSQL reports

```
COMMIT
```

the data will survive a crash.

Even if power is lost immediately afterwards,

the committed transaction will be recovered.

This is made possible by the Write Ahead Log (WAL),

which we'll study in the next chapter.

---

> **💡 Common Confusion**
>
> **Is ACID a PostgreSQL feature?**
>
> No.
>
> ACID is a property of transactional databases.
>
> PostgreSQL is one implementation that guarantees ACID.

---

# 13. Isolation Levels

Imagine two users updating the same account.

```
Account Balance

₹1000
```

User A

```
Withdraw ₹200
```

User B

```
Withdraw ₹300
```

Should User B see User A's changes immediately?

The answer depends on the isolation level.

PostgreSQL supports multiple isolation levels,

allowing applications to choose between

- stronger guarantees

or

- higher concurrency.

For system design interviews,

it's enough to remember:

Higher isolation

↓

Fewer anomalies

↓

More coordination

↓

Lower concurrency

---

# 14. MVCC

MVCC stands for

**Multi-Version Concurrency Control.**

It is one of PostgreSQL's most important features.

Instead of overwriting rows,

PostgreSQL creates **new versions**.

This allows readers and writers to work simultaneously.

---

## The Problem

Suppose the table contains

```
ID

Name

1

Deepak
```

User A executes

```sql
UPDATE users
SET name = 'Deepak Kumar'
WHERE id = 1;
```

At exactly the same time,

User B executes

```sql
SELECT *
FROM users
WHERE id = 1;
```

What should User B see?

If PostgreSQL overwrote the row immediately,

User B might observe half-completed changes.

If PostgreSQL locked the table,

every read would wait.

Neither solution scales well.

---

## PostgreSQL's Solution

Instead of modifying the existing row,

PostgreSQL creates a new row version.

Before:

```
ID

Name

1

Deepak
```

After UPDATE:

```
Version 1

Deepak
```

```
Version 2

Deepak Kumar
```

Both versions now exist.

The old version is **not immediately deleted**.

---

## Which version is returned?

Every transaction sees its own consistent snapshot of the database.

Suppose

```
Transaction A

started first
```

It continues seeing

```
Deepak
```

Meanwhile,

a newer transaction sees

```
Deepak Kumar
```

Different transactions can therefore see different row versions,

while both remain correct.

This is the essence of MVCC.

---

## Why is MVCC useful?

Without MVCC,

reads and writes frequently block each other.

With MVCC,

```
Reader

↓

Old Version
```

```
Writer

↓

New Version
```

Both proceed simultaneously.

This dramatically improves concurrency.

---

## Mental Model

Imagine editing a Google Doc.

Instead of changing the document you're reading,

Google silently creates a new version.

People already viewing the old version continue reading it.

New readers receive the latest version.

PostgreSQL works similarly.

---

> **💡 Common Confusion**
>
> **If PostgreSQL keeps creating new row versions, how does it know which one to return?**
>
> Every row stores metadata describing the transaction that created it and (if applicable) the transaction that replaced it.
>
> When a query runs, PostgreSQL compares this metadata with the current transaction's snapshot to determine which version is visible.
>
> Applications never choose the version manually.
>
> PostgreSQL performs this visibility check automatically.

---

> **💡 Common Confusion**
>
> **Why doesn't PostgreSQL simply overwrite the existing row?**
>
> Overwriting would force readers and writers to block each other.
>
> MVCC allows readers to continue using the old version while writers create a new version.
>
> The tradeoff is that old row versions accumulate and must eventually be cleaned up by VACUUM.

---

# Part 3 Summary

By the end of this chapter you should understand:

- A transaction groups multiple operations into one logical unit.
- PostgreSQL guarantees ACID properties.
- Different isolation levels balance consistency and concurrency.
- PostgreSQL uses MVCC instead of overwriting rows.
- Updates create new row versions.
- Readers and writers usually do not block each other.
- Old row versions remain until VACUUM removes them.

These concepts explain **how PostgreSQL maintains correctness under concurrent access**. In the next chapter, we'll follow a write from the moment it arrives until it is safely stored on disk, introducing the Shared Buffer Cache, WAL, Commit, Background Writer and Checkpoints.


# Part 4 — Write Path, WAL & Checkpoints

So far we've learned:

```
Read Path

Client

↓

Planner

↓

Index

↓

Heap

↓

Result
```

Now let's understand what happens when a client modifies data.

Suppose a client executes

```sql
UPDATE users
SET name = 'Deepak Kumar'
WHERE id = 15;
```

A common misconception is that PostgreSQL immediately writes this change to disk.

It doesn't.

Instead, the write passes through several stages before the data is permanently stored.

Understanding this write path explains why PostgreSQL provides both high performance and strong durability.

---

# 15. The Complete Write Path

Every write roughly follows this path.

```
Client

↓

Shared Buffer Cache

↓

Write Ahead Log (WAL)

↓

COMMIT

↓

Background Writer

↓

Heap File

↓

Checkpoint
```

Each stage has a specific purpose.

---

# 16. Shared Buffer Cache

The Shared Buffer Cache is PostgreSQL's in-memory working area.

Instead of modifying the Heap file directly,

PostgreSQL first loads the required page into memory.

Suppose the Heap contains

```
Page 42

-----------------

ID 15

Deepak
```

The page is copied into memory.

```
Shared Buffer Cache

Page 42

-----------------

ID 15

Deepak
```

The UPDATE now modifies the in-memory page.

```
Shared Buffer Cache

Page 42

-----------------

ID 15

Deepak Kumar
```

Notice something.

The Heap file on disk has **not** changed yet.

Only memory has changed.

---

## Dirty Pages

Once a page in memory differs from the copy stored on disk,

it becomes a **Dirty Page**.

```
Disk

↓

Deepak
```

```
Memory

↓

Deepak Kumar
```

The page is now "dirty."

Dirty simply means

> The in-memory copy contains newer data than the copy stored on disk.

---

## Why not write directly to disk?

Disk operations are slow.

Suppose the same row is updated

10 times in one minute.

Without buffering,

PostgreSQL would perform

```
Write

Write

Write

Write

...
```

Instead,

all updates occur in memory,

and only the latest version is eventually written to disk.

This dramatically reduces disk I/O.

---

> **💡 Common Confusion**
>
> **If PostgreSQL keeps changes only in memory, won't a crash lose data?**
>
> Yes.
>
> This is exactly why PostgreSQL has the Write Ahead Log (WAL).
>
> The WAL protects changes before the Heap is updated.

---

# 17. Write Ahead Log (WAL)

The WAL is one of PostgreSQL's most important components.

Its purpose is simple:

> Never lose a committed transaction.

Whenever PostgreSQL modifies a page,

it also records the modification inside the WAL.

Think of the WAL as a diary.

Before PostgreSQL permanently changes the database,

it first writes down what it intends to do.

Example:

```
WAL

Transaction 102

UPDATE users

ID 15

Deepak

↓

Deepak Kumar
```

This record is much smaller than rewriting an entire database page.

---

## Why is it called "Write Ahead"?

Because the log reaches disk **before** the actual database page.

The sequence is always

```
Update Memory

↓

Write WAL

↓

Commit

↓

Write Heap Later
```

Never

```
Write Heap

↓

Write WAL
```

This ordering guarantees durability.

---

## Why is WAL Fast?

Heap pages exist throughout the data file.

Updating them requires random disk writes.

Random writes are expensive.

The WAL, however, is append-only.

```
Old WAL

A

B

C

↓

Append

D
```

Appending to the end of a file is much faster than modifying pages scattered throughout the disk.

---

# 18. COMMIT

Suppose the application executes

```sql
COMMIT;
```

Many engineers assume PostgreSQL now writes the updated Heap page.

It doesn't.

Instead,

PostgreSQL performs only one critical operation.

```
Flush WAL

↓

Disk
```

Once the WAL reaches disk,

the transaction is considered durable.

The client immediately receives

```
COMMIT Successful
```

Even though

the Heap page is still only in memory.

---

## Why is this safe?

Suppose power fails immediately after COMMIT.

```
Heap

Old Version
```

```
WAL

New Version
```

During recovery,

PostgreSQL simply replays the WAL,

bringing the Heap back to its correct state.

This is why WAL is written before the Heap.

---

> **💡 Common Confusion**
>
> **Why doesn't PostgreSQL wait until the Heap page is written before returning COMMIT?**
>
> Because writing Heap pages is relatively slow.
>
> Once the WAL is safely stored on disk, PostgreSQL already has enough information to recover the transaction after a crash.
>
> Waiting for Heap writes would significantly increase transaction latency.

---

# 19. Background Writer

At this point,

the updated page still exists only in memory.

```
Memory

↓

Dirty Page
```

Eventually,

PostgreSQL copies the page back to the Heap.

```
Dirty Page

↓

Heap File
```

This work is performed asynchronously by the Background Writer.

The client does not wait for it.

---

## Why delay Heap writes?

Suppose a page receives

20 updates.

Without buffering,

```
Update

↓

Write

Update

↓

Write

Update

↓

Write
```

Twenty disk writes.

Instead,

```
20 Updates

↓

1 Heap Write
```

This process is called **write batching**.

It is one of PostgreSQL's most important performance optimizations.

---

# 20. Checkpoints

Eventually,

dirty pages accumulate inside memory.

PostgreSQL cannot keep them there forever.

Periodically,

a **Checkpoint** occurs.

During a checkpoint,

dirty pages are flushed from memory to disk.

```
Dirty Pages

↓

Heap File
```

Once this completes,

the corresponding WAL entries are no longer required for crash recovery.

Future crashes only need to replay WAL records written after the latest checkpoint.

---

## Why are Checkpoints Necessary?

Imagine PostgreSQL has been running for three months.

Without checkpoints,

crash recovery would require replaying

three months of WAL.

Instead,

checkpoints establish a new recovery starting point.

Crash recovery becomes much faster.

---

## Background Writer vs Checkpoint

Many engineers confuse these two processes.

The Background Writer

- continuously writes some dirty pages to disk,
- reducing sudden spikes in disk activity.

The Checkpoint

- ensures all pages modified before that checkpoint are safely stored,
- allowing PostgreSQL to discard older WAL during recovery.

Both write dirty pages,

but their responsibilities are different.

---

> **💡 Common Confusion**
>
> **Do Background Writer and Checkpoint perform the same task?**
>
> No.
>
> Both write dirty pages to disk, but they exist for different reasons.
>
> The Background Writer smooths normal write activity.
>
> Checkpoints establish recovery boundaries and reduce crash recovery time.

---

# Part 4 Summary

The complete PostgreSQL write path is

```
Client

↓

Shared Buffer Cache

↓

Dirty Page

↓

WAL

↓

COMMIT

↓

Background Writer

↓

Heap File

↓

Checkpoint
```

The key ideas to remember are:

- Updates first modify memory.
- Modified pages become Dirty Pages.
- WAL reaches disk before the Heap.
- COMMIT waits only for WAL.
- Heap pages are written later.
- Background Writer batches writes.
- Checkpoints reduce crash recovery time.

These mechanisms allow PostgreSQL to provide both high performance and strong durability.

The next chapter covers what happens after a crash, how PostgreSQL recovers using WAL, why old row versions accumulate, and how VACUUM keeps the database healthy.


# Part 5 — Recovery, VACUUM, Replication & Scaling

So far we've learned how PostgreSQL stores data, executes queries and processes writes.

This final chapter answers the remaining questions:

- What happens if PostgreSQL crashes?
- What happens to the old row versions created by MVCC?
- How does PostgreSQL scale?
- When should PostgreSQL be used?

---

# 21. Crash Recovery

Suppose a transaction commits successfully.

```
Client

↓

UPDATE

↓

COMMIT
```

Immediately afterwards,

the machine loses power.

What happens?

Did we lose the transaction?

No.

Remember the write path.

```
Shared Buffer

↓

WAL

↓

COMMIT
```

The Heap page may never have reached disk.

However,

the WAL was safely written before PostgreSQL acknowledged the commit.

When PostgreSQL starts again,

it performs **Crash Recovery**.

---

## Recovery Process

PostgreSQL first locates the most recent checkpoint.

```
Checkpoint

↓

Read WAL

↓

Replay Changes

↓

Database Restored
```

Instead of scanning the entire database,

it simply replays WAL records generated after the latest checkpoint.

Every committed transaction is reconstructed.

Any uncommitted transaction is discarded.

The database returns to a consistent state.

---

## Why Recovery Is Fast

Imagine replaying

```
3 months
```

of WAL.

That would take a very long time.

Checkpoints solve this problem.

Instead of replaying everything,

PostgreSQL only replays WAL generated after the latest checkpoint.

---

> **💡 Common Confusion**
>
> **If the Heap wasn't updated, how can PostgreSQL recover the latest data?**
>
> Because the WAL contains every committed modification.
>
> Recovery simply repeats those modifications until the Heap reaches the same state it had before the crash.

---

# 22. VACUUM

Earlier we learned that PostgreSQL uses MVCC.

Instead of overwriting rows,

it creates new versions.

Example.

Initially,

```
ID 15

Deepak
```

After an UPDATE,

```
Version 1

Deepak
```

```
Version 2

Deepak Kumar
```

The old version still exists.

If PostgreSQL never removed old versions,

the table would grow forever.

This is the purpose of VACUUM.

---

## What VACUUM Does

VACUUM periodically scans tables.

For each old row version,

it asks:

```
Can any active transaction still see this row?
```

If the answer is

```
No
```

the row becomes eligible for removal.

The occupied space can now be reused for future inserts.

---

## Why VACUUM Exists

Without VACUUM,

tables would continuously grow,

even if the logical amount of data stayed constant.

This phenomenon is called

```
Table Bloat
```

The same problem also affects indexes,

creating

```
Index Bloat
```

Regular VACUUM prevents unnecessary growth and keeps queries efficient.

---

## Mental Model

Imagine editing a Word document.

Every edit creates a draft.

Once everyone has finished reviewing the old drafts,

they can be thrown away.

VACUUM performs this cleanup automatically.

---

> **💡 Common Confusion**
>
> **Why doesn't PostgreSQL delete old rows immediately after an UPDATE?**
>
> Because other transactions may still be reading the old version.
>
> PostgreSQL waits until it is certain that no active transaction can access that version before removing it.

---

# 23. Replication

A single PostgreSQL server eventually reaches its limits.

One common scaling strategy is replication.

```
          Primary

        /    |    \

Replica Replica Replica
```

The Primary accepts all writes.

Replicas continuously copy changes from the Primary.

---

## How Replication Works

Remember the WAL?

```
Primary

↓

Generate WAL
```

Instead of using the WAL only for crash recovery,

PostgreSQL also streams it to replicas.

```
Primary

↓

WAL

↓

Replica

↓

Replay WAL
```

The replica performs the same modifications locally,

producing an identical copy of the Primary.

---

## Why Replicas Are Useful

Replicas improve

### High Availability

If the Primary fails,

a replica can be promoted.

---

### Read Scaling

Suppose

```
1000 Reads/sec

50 Writes/sec
```

Reads can be distributed across replicas.

```
Application

↓

Load Balancer

↓

Replica 1

Replica 2

Replica 3
```

This significantly increases read throughput.

---

## Limitation

Replicas cannot increase write throughput.

Every write still reaches

```
One Primary
```

This is one of PostgreSQL's biggest scaling limitations.

---

> **💡 Common Confusion**
>
> **If I add ten replicas, do writes become ten times faster?**
>
> No.
>
> Replicas improve read throughput and availability.
>
> All writes still pass through the Primary.

---

# 24. Partitioning

As tables grow,

they become harder to manage.

Suppose we have

```
Orders

2 Billion Rows
```

Searching,

backups,

maintenance,

and indexing become increasingly expensive.

Partitioning divides one large table into multiple smaller tables.

Example.

```
Orders

↓

2024

2025

2026
```

Now,

a query requesting

```
Orders from 2026
```

does not need to examine the older partitions.

PostgreSQL skips them entirely.

This optimization is called

```
Partition Pruning
```

---

## Partitioning vs Sharding

These concepts are often confused.

### Partitioning

One PostgreSQL database.

One server.

One logical table divided into smaller pieces.

---

### Sharding

Multiple PostgreSQL databases.

Multiple servers.

The application decides which server owns the data.

Partitioning improves manageability.

Sharding improves scalability.

---

# 25. Scaling PostgreSQL

A common interview question is:

> How do you scale PostgreSQL?

The typical progression is:

```
Better Queries

↓

Indexes

↓

Vertical Scaling

↓

Read Replicas

↓

Partitioning

↓

Caching

↓

Sharding
```

Notice that sharding appears last.

Sharding dramatically increases operational complexity.

It should be introduced only when simpler solutions are no longer sufficient.

---

# 26. When Should You Use PostgreSQL?

Choose PostgreSQL when your application requires:

- ACID transactions
- Strong consistency
- SQL
- Joins
- Reporting
- Data integrity

Typical examples include:

- Banking
- E-commerce
- HR Systems
- Inventory
- Booking Platforms

---

Avoid PostgreSQL as the only database when the primary challenge is:

- Massive write throughput
- Horizontal write scaling
- Full-text search
- Ultra-low latency caching

These problems are often solved using specialized systems alongside PostgreSQL.

---

# PostgreSQL Mental Map

```
                     PostgreSQL

                          │

          ┌───────────────┴───────────────┐

          ▼                               ▼

      Read Path                     Write Path

Planner                         Shared Buffer

↓

Index                           Dirty Page

↓

Heap                            WAL

↓

Result                          Commit

                                 ↓

                           Background Writer

                                 ↓

                            Checkpoint

                                 ↓

                           Crash Recovery

                                 ↓

                               VACUUM
```

---

# Interview Summary

If you remember only a few ideas from PostgreSQL, remember these.

### Storage

- Heap stores rows.
- Heap is unordered.
- Pages are 8 KB.
- B+Tree stores pointers to Heap rows.

---

### Reads

```
Planner

↓

Sequential Scan

or

Index Scan

↓

Heap

↓

Result
```

The Query Planner chooses the cheapest execution plan.

---

### Writes

```
Shared Buffer

↓

Dirty Page

↓

WAL

↓

COMMIT

↓

Background Writer

↓

Heap

↓

Checkpoint
```

COMMIT waits only for the WAL.

Heap pages are written later.

---

### MVCC

- Updates create new row versions.
- Readers and writers rarely block each other.
- VACUUM removes obsolete versions.

---

### Scaling

- Vertical Scaling
- Read Replicas
- Partitioning
- Sharding (last resort)

---

# Final Takeaway

PostgreSQL is not the fastest database.

It is not the easiest database to scale horizontally.

It was never designed to be.

PostgreSQL's strength lies in providing a reliable, consistent and feature-rich relational database that can safely power business-critical applications.

Understanding PostgreSQL is less about memorizing its internal components and more about understanding the tradeoffs behind its design.

Once those tradeoffs become clear, choosing PostgreSQL—or deciding not to use it—becomes a system design decision rather than a technology preference.
