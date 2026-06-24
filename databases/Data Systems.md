# Data Systems

A quick-reference guide covering the data plane of a backend system: modeling, storage engines, indexes, transactions, replication, partitioning, caching, queues, streams, and object storage. Concept-first, with concrete products (Postgres, Redis, Kafka, S3, Cassandra) named as examples — the goal is understanding tradeoffs deep enough to choose well.

For the algorithmic mechanisms behind distributed replication and consensus, see `Distributed Systems.md`. For architectural patterns that compose these pieces, see `System Design.md`. For the relational foundations and schema design (relational model, algebra, ER modeling, normalization) see `Database Design.md`; for how a single DBMS works internally (buffer management, indexes, concurrency control, recovery, query processing) see `Database Internals.md`; for the SQL language itself see `SQL.md`.

---

## Table of Contents

1. Data Systems — Overview
2. Storage Hierarchy and Latency Numbers
3. Data Modeling
4. Storage Engines (B-tree vs LSM)
5. Indexes
6. Transactions, Isolation, and MVCC
7. Replication (Applied)
8. Partitioning and Sharding
9. Schema Evolution and Migrations
10. Caching
11. Queues and Streams
12. Object Storage
13. Search
14. Time-Series and Analytical Stores
15. Reliability Patterns (Outbox, CDC, Idempotency)
16. Operating Data Systems

---

## 1. Data Systems — Overview

A backend system's data plane is rarely one thing. A typical request touches several stores, each chosen for a different read or write pattern:

```
                              +-----------------+
                              |   Edge cache    |  CDN, milliseconds, immutable
                              +-----------------+
                                       |
                              +-----------------+
                              |  App cache      |  Redis, sub-ms reads
                              +-----------------+
                                       |
                              +-----------------+
                              |  Primary DB     |  Postgres, ACID, source of truth
                              +-----------------+
                                       |
                              +--------+--------+
                              |                 |
                       +------v-----+     +-----v-------+
                       |  Search    |     |  Analytics  |
                       |  index     |     |  warehouse  |
                       |  (ES, OS)  |     |  (BQ, etc)  |
                       +------------+     +-------------+
                              |                 |
                       +------v-------------------v------+
                       |    Object storage (S3 / GCS)    |  durable, infinite, slow
                       +---------------------------------+

Plus, between any two:  queues / streams (Kafka, NATS, SQS)

  ES/OS = Elasticsearch / OpenSearch;  BQ = BigQuery
```

Each store solves a problem the others cannot solve well:

| Need | Right store |
|------|-------------|
| Transactional source of truth, complex queries, foreign keys | Relational (Postgres, MySQL) |
| Sub-millisecond reads, ephemeral data | In-memory KV (Redis, Memcached) |
| Schemaless documents, single-table fetches | Document (MongoDB, DynamoDB) |
| Write-heavy, time-series, append-mostly | Wide-column / LSM (log-structured merge-tree; Cassandra, ScyllaDB, RocksDB) |
| Full-text search, faceting, fuzzy match | Search (Elasticsearch, OpenSearch, Meilisearch) |
| OLAP — columnar scans, aggregations | Warehouse (BigQuery, Snowflake, ClickHouse, DuckDB) |
| Immutable blobs, large objects, archives | Object storage (S3, GCS, R2) |
| Decoupling producers and consumers | Queue (SQS, RabbitMQ) or stream (Kafka) |

**The first design question is rarely "which database?" It is "what are the access patterns?"** Read-heavy vs write-heavy, point-lookup vs range scan, single-record vs aggregate, latency-sensitive vs throughput-sensitive. The right store falls out of that.

---

## 2. Storage Hierarchy and Latency Numbers

Reasoning about performance requires rough numbers held in mind. These do not need to be exact; they need to be within an order of magnitude.

```
Operation                                   Time               Relative
---------                                   ----               --------
L1 cache reference                          0.5 ns                 1 x
Branch mispredict                           5   ns                10 x
L2 cache reference                          7   ns                14 x
Mutex lock/unlock                           25  ns                50 x
Main memory reference                       100 ns               200 x
Compress 1 KB with snappy                   2,000 ns           4,000 x     (2 us)
Send 1 KB over 1 Gbps                       10,000 ns         20,000 x     (10 us)
Read 4 KB from SSD                          150,000 ns                     (150 us)
Read 1 MB sequentially from memory          250,000 ns                     (250 us)
Round trip within same datacenter           500,000 ns                     (0.5 ms)
Read 1 MB sequentially from SSD             1,000,000 ns                   (1 ms)
HDD seek                                    10,000,000 ns                  (10 ms)
Read 1 MB sequentially from HDD             20,000,000 ns                  (20 ms)
TCP round trip, CA to Netherlands           150,000,000 ns                 (150 ms)
```

**Implications for system design:**

- Memory is ~200x slower than L1 cache, but ~1500x faster than a random SSD read.
- A single SSD read takes longer than 1500 cache misses; a single same-datacenter round-trip takes longer than three SSD reads.
- "Same datacenter" round trips (0.5 ms) are 300x faster than cross-continent (150 ms).
- **Sequential is dramatically faster than random** on every layer, but the gap is biggest on HDD (~100x) and smallest in RAM (~3x).

These numbers shape every design choice: caching tiers exist because each layer is an order of magnitude slower. Batching exists because per-operation overhead dominates at low latency tiers. Database storage engines are designed around the sequential/random asymmetry of their target medium.

### Disk Types

| Medium | Random IOPS | Sequential | Latency |
|--------|------|-----|------|
| HDD (7200 RPM) | ~100 | ~150 MB/s | 10 ms |
| SATA SSD | ~80,000 | ~500 MB/s | 0.1 ms |
| NVMe SSD | ~500,000+ | 3-7 GB/s | <0.1 ms |
| DRAM | N/A | ~25 GB/s | 100 ns |

HDDs are now relegated to bulk/cold storage. SSD is the default for OLTP; NVMe for high-performance OLTP or analytics scratch.

---

## 3. Data Modeling

Data modeling is the act of choosing how facts about the world are represented as records, with the access pattern in mind. The right model makes queries cheap and changes easy; the wrong model makes everything painful.

### Database Families

Modeling always happens *within* a database family, and the family constrains which models are possible. The family is therefore the first choice; the remaining subsections cover how to model inside each. (Section 1 maps a need to a store; this classifies the families themselves.)

| Family | Data model | Strong at | Weak at | Examples |
|--------|-----------|-----------|---------|----------|
| Relational | Typed rows in related tables, joined on keys | Complex ad-hoc queries, transactions, integrity | Deeply nested data, horizontal write scaling | Postgres, MySQL |
| Key-value | Opaque value addressed by a key | Point lookups, caching, sessions | Any access not by the key | Redis, Memcached |
| Document | Self-contained JSON-like documents | Hierarchical entities read as a unit, flexible schema | Cross-document joins, multi-document transactions | MongoDB, Couchbase, Postgres JSONB |
| Wide-column | Rows by partition + clustering key, sparse columns | Write-heavy load, known access patterns at scale | Ad-hoc queries, joins | Cassandra, ScyllaDB, Bigtable |
| Graph | Nodes and edges, both with properties | Relationship traversal, pathfinding | Bulk scans, aggregation | Neo4j, Neptune |
| Time-series | Timestamped points tagged with dimensions | Append-mostly metrics, time-range queries, retention | High-cardinality tags, in-place updates | Prometheus, InfluxDB, TimescaleDB |
| Search | Inverted index over documents | Full-text, relevance ranking, faceting | Serving as source of truth, transactions | Elasticsearch, OpenSearch |

**An orthogonal axis: OLTP (Online Transaction Processing) vs OLAP (Online Analytical Processing).** Independent of family, a store is tuned either for many small transactions (OLTP, row-oriented — most of the above) or for large analytical scans and aggregations (OLAP, columnar — BigQuery, Snowflake, ClickHouse, Redshift). Row-vs-columnar layout is a storage-engine property (section 4); the analytical/columnar stores are covered in section 14. A SQL interface appears in both camps, which is why "SQL vs NoSQL" is a poor primary distinction — the data model and the OLTP/OLAP axis are the ones that matter.

**Multi-model reality.** The boundaries are not rigid. Postgres is relational but also a competent document store (JSONB) and key-value store; Redis adds streams and search through modules; many engines now span several families. The family names the model an engine is *optimized* for, not a hard limit.

### Normalization (Relational)

Decomposing data into multiple tables so that each fact lives in exactly one place. When one fact is stored in many rows the copies drift apart, producing update, insertion, and deletion anomalies; normalization removes them by giving every fact a single home.

```
Unnormalized:                          Normalized (3NF):
+----------+-------+----------+         users(id, name, email)
| order_id | user  | email    |         orders(id, user_id, total)
+----------+-------+----------+         order_items(id, order_id, product, qty)
| 1        | alice | a@ex.com |
| 2        | alice | a@ex.com |   email duplicated on every order row ->
+----------+-------+----------+   one change must touch many rows (anomaly)
```

The working rule is "normalize to 3NF absent a specific reason not to," captured by the maxim that every non-key column must depend on *the key, the whole key, and nothing but the key*. The full treatment — functional dependencies, 1NF through BCNF, 4NF/5NF, and the anomalies each form removes — is in `Database Design.md`. Denormalization (below) is the more common deliberate deviation at the systems level.

### Denormalization

Deliberate duplication for read performance. Trades write complexity and storage for fewer joins per read.

```
Order summary table (denormalized for display):
+----------+--------------+------------+---------+---------------------+
| order_id | user_name    | user_email | total   | items_summary       |
+----------+--------------+------------+---------+---------------------+
| 1        | alice        | a@ex.com   | 20      | "widget x2"         |
+----------+--------------+------------+---------+---------------------+

Reading order details for display: one row, no joins.
Updating alice's email: must update many rows.
```

**When to denormalize:**

- The same join happens on every read of a high-traffic page.
- Writes are rare compared to reads.
- Eventual consistency between copies is acceptable.

Denormalization is usually wrong in OLTP and usually right in materialized views, search indexes, and analytical projections. The relational source of truth stays normalized; downstream projections are denormalized as needed.

### Document / Embedding vs Referencing

Document stores (MongoDB, Postgres JSONB, DynamoDB) allow nested data to be embedded directly in a record.

```
Embed:
{
  "_id": "order_1",
  "user": { "id": "user_1", "name": "alice", "email": "a@ex.com" },
  "items": [
    { "product": "widget", "qty": 2 },
    { "product": "gadget", "qty": 1 }
  ]
}

Reference:
{
  "_id": "order_1",
  "user_id": "user_1",
  "item_ids": ["item_1", "item_2"]
}
```

**Embed when:**

- The nested data is owned by the parent and rarely accessed independently.
- Bounded size (no unbounded lists like "all messages in a chat").
- Read together as a unit.

**Reference when:**

- The nested entity has its own identity / is accessed independently.
- The list grows without bound.
- Many parents share the same child.

A useful heuristic: if it would be unreasonable for the nested item to outlive the parent, embed.

### Modeling by Access Pattern (Wide-Column / KV)

In Cassandra, DynamoDB, Bigtable, the row is keyed by a partition key (which physically locates it) plus an optional sort key. Queries are efficient only on the keys; secondary indexes are limited or expensive.

The model is designed around the query, not the entity:

```
"Get the last 10 messages in conversation X":

PRIMARY KEY (conversation_id, message_ts DESC)

conversation_id  | message_ts          | sender   | text
-----------------|---------------------|----------|------
conv_42          | 2025-01-15 09:01:23 | alice    | hello
conv_42          | 2025-01-15 09:01:25 | bob      | hi
conv_42          | 2025-01-15 09:01:50 | alice    | how's it going?

Fetching the last 10 is a contiguous range scan on a single partition.
Fast at any scale.

"Get all conversations user X participated in" — different access pattern.
Requires a separate table indexed by (user_id, conv_id), populated by the
write path. This is normal in Cassandra/DynamoDB modeling.
```

**The wide-column mental model:** one query = one partition. Each query that doesn't naturally fit gets its own table, kept in sync at write time. This is the opposite of relational where one model serves many queries.

### Graph Modeling

When relationships are the queries — "shortest path", "common friends", "transitive ownership" — a graph database (Neo4j, Neptune, ArangoDB) outperforms recursive SQL.

```
Nodes:  User, Post, Comment
Edges:  FOLLOWS, AUTHORED, LIKED

Query: "Posts liked by people I follow, that I haven't liked"

MATCH (me:User {id: 'me'})-[:FOLLOWS]->(friend:User)-[:LIKED]->(post:Post)
WHERE NOT (me)-[:LIKED]->(post)
RETURN post
ORDER BY post.created_at DESC
LIMIT 20
```

Most apps don't need a dedicated graph DB. But when the queries are "all paths between X and Y up to N hops" relational SQL becomes painful and graph engines become essential.

### Time-Series Modeling

Time-series workloads (metrics, events, sensor data) are append-mostly with time-bounded queries. Dedicated stores (InfluxDB, TimescaleDB, Prometheus, VictoriaMetrics) optimize for this:

- Append-only writes, no in-place updates.
- Storage organized by time chunks for efficient retention.
- Downsampling and rollups.
- Tag/dimension indexing optimized for low cardinality.

Modeling tip: keep tag cardinality low (10s-1000s of unique values per tag, not millions). High-cardinality tags (user IDs, request IDs) break time-series engines and belong in a different store.

---

## 4. Storage Engines (B-tree vs LSM)

Underneath every database is a storage engine — the code that decides how data is laid out on disk and how reads and writes are served. Two dominant families: B-trees (read-optimized, in-place) and LSM trees (write-optimized, log-structured).

### B-Tree Storage (Postgres, MySQL InnoDB, SQLite, most relational databases)

The B-tree (or B+tree) is the classical data structure for indexed storage. Records are stored in fixed-size pages; pages are organized in a balanced tree keyed by the index key.

```
                +---------+---------+
                |   K=20  |   K=50  |
                +----+----+----+----+
                     |        |
        +------------+        +-----------+
        v                                 v
+------+------+                   +------+------+
| K=5 | K=10  |                   | K=30 | K=40 |
+------+------+                   +------+------+
   |      |                          |       |
   v      v                          v       v
(leaf pages with full row data and pointers to next/prev leaves)
```

**Read:** O(log n) — traverse from root to leaf. With typical fanout (~100-1000), even billion-row tables are 4-5 levels deep. Most internal pages fit in memory; only leaf pages typically require a disk read.

**Write:** locate the right leaf, update or insert in place. If the page is full, split it (cascading splits up the tree). Page splits cause write amplification and fragmentation.

**Update in place:** Postgres uses multi-version concurrency control (MVCC): it writes a new row version rather than overwriting. MySQL InnoDB writes new versions but maintains an undo log for old versions. The page-level mechanics (fanout, splits, fill factor, B+tree vs B-tree) are in `Database Internals.md`; this section stays at the engine-choice altitude.

**Properties:**

- Reads are predictable and fast.
- Writes pay the cost of finding the right page and possibly splitting.
- Index updates require updating both the table and each affected index — every write touches multiple B-trees.
- Random writes scattered across many pages cause poor locality.

**Write-Ahead Log (WAL):** Before changing a page in the buffer pool, the engine appends the change to a sequential log. On crash, replay the WAL to reconstruct any unwritten changes. WAL is sequential I/O (fast) even when the data writes are random (slow).

### LSM Trees (Cassandra, ScyllaDB, RocksDB, LevelDB, HBase)

The Log-Structured Merge tree optimizes for write throughput by **never doing in-place updates**. Writes accumulate in memory and are flushed sequentially to disk in immutable files.

```
Writes go to:
   MemTable (in-memory, sorted, e.g., skiplist)
        |
        | flush when full
        v
   SSTable level 0 (immutable file on disk, sorted)

In background:
   Compaction merges multiple SSTables into one,
   resolving duplicates and tombstones, into higher levels.

Level 0:  [SST] [SST] [SST] [SST]            <- newest, may overlap
Level 1:  [SST----][SST----][SST----]        <- sorted, no overlap
Level 2:  [SST--------][SST--------]         <- 10x larger than L1
Level 3:  [SST------------][SST---]          <- 10x larger than L2
...
```

**Write path:**
1. Append to WAL.
2. Insert into MemTable (in-memory sorted structure).
3. When MemTable hits size threshold, freeze it and flush sequentially to disk as a new SSTable.

All writes are sequential. No random I/O on the write path. Extremely fast.

**Read path:** A key might be in the MemTable, or any of the SSTables. Worst case, every level must be checked. Mitigated by:

- **Bloom filters** per SSTable — probabilistic check; skips files that definitely don't contain the key.
- **Block indexes** per SSTable — locate the right block without scanning the whole file.
- **Compaction** — merges and discards old versions, reducing the number of files to check.

**Tradeoffs:**

| Property | B-Tree | LSM |
|----------|--------|-----|
| Write throughput | Lower (random I/O) | Much higher (sequential) |
| Read latency | Lower (single path) | Higher (multiple files) |
| Write amplification | Moderate (page splits + WAL) | Often *higher* (level compaction rewrites data ~10-30×); the LSM win is sequential, not less, I/O |
| Read amplification | ~1 (root to leaf) | Variable (depends on level count) |
| Space amplification | Low | Higher (stale versions until compacted) |
| Compaction pauses | None | Significant (background but resource-heavy) |

**Why this matters:** Postgres is a B-tree-style engine — excellent for OLTP, mixed read/write. Cassandra is an LSM engine — excellent for write-heavy, time-series, append patterns. RocksDB is an LSM library embedded in many systems (MyRocks, CockroachDB, TiKV). The choice between engines often determines whether a write-heavy workload will keep up.

### Where the WAL Fits

In both families, a WAL provides durability:

```
Application writes commit:
  1. Append entry to WAL on disk     <-- sequential, must hit disk
  2. Update in-memory data structures
  3. (Eventually) flush data pages to disk

On crash:
  1. Replay WAL from last checkpoint
  2. In-memory state reconstructed
```

`fsync()` after WAL write is what makes it durable. Skipping fsync trades durability for speed. Group commit (batch many WAL writes into one fsync) reclaims much of the latency.

---

## 5. Indexes

An index is an auxiliary data structure that speeds up lookups at the cost of write overhead and storage. Most performance issues — and most over-fixes — start here.

### Index Types

| Type | Structure | Best for |
|------|-----------|----------|
| B-tree | Balanced tree | Equality, range, sort, prefix |
| Hash | Hash table | Exact equality only |
| GIN (Postgres) | Inverted index | Array/JSON containment, full-text |
| GiST (Postgres) | Generalized search tree | Geometry, full-text, fuzzy |
| BRIN (Postgres) | Block range | Very large tables with naturally ordered data |
| Bitmap | Bit per row per value | Low-cardinality columns, analytical queries |
| Trigram (`pg_trgm`) | n-gram index | Substring/LIKE/regex search |

For most OLTP workloads, **B-tree indexes solve 95% of cases.** Specialty indexes solve specific access patterns the B-tree can't.

### What an Index Does

```
Table: users (50M rows)
+----+---------+----------+----------+
| id | email   | name     | country  |
+----+---------+----------+----------+
| 1  | a@x.com | alice    | US       |
| 2  | b@x.com | bob      | UK       |
| 3  | c@x.com | carol    | US       |
...

Without an index on email:
  SELECT * FROM users WHERE email = 'b@x.com'
  -> the id primary-key B-tree is sorted by id, not email,
     so it gives no path to an email -> sequential scan of
     50M heap rows, checking each email -> seconds

With a B-tree index on email:
  CREATE INDEX ON users(email);   -- a second, separate tree keyed by email
  SELECT * FROM users WHERE email = 'b@x.com'
  -> walk the email B-tree to the entry -> get the row's heap
     location -> 1 row fetch -> ~1 ms
```

An index is per-column, not per-table. Each index — including the one backing the primary key — is its own B-tree, keyed and sorted by that index's columns. A query can only use an index whose leading column it actually filters on, so an index on `id` does nothing for a lookup by `email`; you need an index on the column you filter by.

This differs from MySQL/InnoDB, where the table itself is physically a B-tree on the primary key (index-organized). In Postgres the table is always an unordered heap, and every index — PK included — is a separate structure pointing into that heap. A `UNIQUE` constraint on a column also builds a B-tree on it, so a unique column is already indexed without an explicit `CREATE INDEX`.

### Composite Indexes

A multi-column index is ordered by the columns in declaration order — leftmost-prefix rule.

```
CREATE INDEX idx_user_country_active ON users(country, active, last_login);

Can use the index for:
  WHERE country = 'US'                                    YES
  WHERE country = 'US' AND active = true                  YES
  WHERE country = 'US' AND active = true AND last_login > ...  YES (range on last column)
  WHERE active = true                                      NO (not the leftmost column)
  WHERE country = 'US' AND last_login > ...                PARTIAL (only country)
```

Order matters. Put equality-predicate columns before range columns (`>`, `<`, `BETWEEN`), and note that a column is usable only if every column before it in the index is also constrained. "Most selective first" can misfire: a selective *range* column placed ahead of an equality column defeats the rest of the index.

### Covering Indexes

If an index contains all the columns the query needs, the engine can answer the query from the index alone (index-only scan) without fetching the row.

```sql
CREATE INDEX idx_covering ON orders(user_id, created_at) INCLUDE (total, status);

SELECT total, status FROM orders WHERE user_id = 42 AND created_at > '2025-01-01';
-- Answered entirely from the index. No table heap access.
```

This makes hot read paths dramatically faster, but bloats the index.

### Partial Indexes

Index only rows that match a predicate. Smaller index, faster on the queries it covers.

```sql
CREATE INDEX idx_active_orders ON orders(user_id, created_at) WHERE status = 'active';

SELECT * FROM orders WHERE status = 'active' AND user_id = 42;
-- Uses the partial index. Most rows excluded entirely.
```

### Expression / Functional Indexes

Index the result of an expression so queries on the same expression are fast.

```sql
CREATE INDEX idx_lower_email ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- Uses the expression index.
```

### Index Anti-Patterns

- **Indexing every column "just in case."** Each index slows writes and consumes space. Add indexes for actual queries.
- **Redundant indexes.** An index on `(a, b)` also covers `WHERE a = ...`. Don't add a separate index on just `a`.
- **Indexing low-cardinality columns alone.** An index on `gender` or `is_active` alone is rarely useful; combine with selective columns.
- **Forgetting indexes on FK columns.** Foreign key constraints don't auto-create indexes in every DB. Joins and cascades become slow.
- **Indexing `created_at` alone in append-heavy tables.** Every insert hits the rightmost leaf — a hot spot. Use a composite that distributes writes.

---

## 6. Transactions, Isolation, and MVCC

A transaction groups multiple operations into an atomic unit. Either all of them happen, or none do.

### ACID

| Property | Meaning |
|----------|---------|
| Atomicity | All or nothing. A failed transaction leaves no partial changes. |
| Consistency | The DB transitions between valid states (constraints, foreign keys, etc.). Application-level concept. |
| Isolation | Concurrent transactions appear to run sequentially. |
| Durability | Once committed, changes survive crashes. |

Atomicity comes from the WAL. Durability comes from `fsync()`. Consistency is mostly the application's responsibility. **Isolation** is where the interesting tradeoffs live.

### Isolation Levels

The SQL standard defines four levels, named by the anomalies they prevent.

| Level | Dirty read | Lost update | Non-repeatable read | Phantom read | Write skew |
|-------|-----------|-------------|---------------------|--------------|------------|
| Read Uncommitted | possible | possible | possible | possible | possible |
| Read Committed | prevented | possible | possible | possible | possible |
| Repeatable Read | prevented | prevented† | prevented | possible* | possible |
| Serializable | prevented | prevented | prevented | prevented | prevented |

*Postgres's Repeatable Read is really **snapshot isolation**: each transaction reads from a consistent MVCC snapshot, which also prevents phantom reads (despite the standard not requiring it) but still permits **write skew**. MySQL InnoDB's RR uses gap locks for the same anti-phantom effect.

†Lost-update prevention at Repeatable Read depends on the mechanism. Postgres aborts the second writer with a serialization error (first-committer-wins). MySQL InnoDB makes an in-SQL `UPDATE ... SET x = x + 1` safe by re-reading the current row, but a read-modify-write split across a separate `SELECT` then `UPDATE` still needs `SELECT ... FOR UPDATE`. Read Committed prevents neither.

**Anomaly definitions:**

```
Dirty read:
  T1 writes (uncommitted), T2 reads T1's data, T1 rolls back.
  T2 saw data that never existed.

Lost update:
  T1 reads X=10, T2 reads X=10, T1 writes X=11, T2 writes X=11.
  T2 overwrote T1's update without ever seeing it. The classic read-modify-write race.

Non-repeatable read:
  T1 reads a row, T2 updates and commits, T1 reads again -- different value.

Phantom read:
  T1 reads a set with a predicate, T2 inserts a row matching, T1 re-reads -- new rows appear.

Write skew:
  T1 and T2 each read overlapping data, then write based on what they read.
  Neither modified what the other read, but the combined result is inconsistent.
  Example: "at least one doctor on call" -- two doctors each check the count
  (sees 2 doctors), each goes off call, ends up with 0.
```

**Default isolation levels in practice:**

| Database | Default |
|----------|---------|
| Postgres | Read Committed |
| MySQL InnoDB | Repeatable Read |
| Oracle | Read Committed |
| SQL Server | Read Committed |

Most applications run on Read Committed and never hit anomalies because their workload doesn't expose them. Hot paths that update counters or implement quotas need Serializable or explicit locking.

### MVCC (Multi-Version Concurrency Control)

Used by Postgres, Oracle, MySQL InnoDB. Instead of locking on read, each transaction sees a **snapshot** of the database as of a specific point in time.

```
Row R, over time, exists as multiple physical versions:

  version  xmin (created by)  xmax (deleted by)   value
  R.v1     tx 5               tx 12               "old"
  R.v2     tx 12              (none, still live)  "new"

T1 takes its snapshot at time 10  -> sees the world as of tx <=10 committed
T2 (tx 12) updates R at time 12   -> writes R.v2, stamps R.v1 as deleted by 12

T1 reads R:
  - R.v2 created by tx 12 -> tx 12 not in T1's snapshot -> invisible
  - R.v1 created by tx 5 (committed, in snapshot), deleted by tx 12
    (not in snapshot, so the deletion "hasn't happened" yet) -> visible
  => T1 reads "old". T2's update is invisible to T1.

T1 commits. Once no live snapshot can still need R.v1, vacuum reclaims it.
```

Every row version carries visibility metadata: **xmin** = the transaction ID that created it, **xmax** = the transaction ID that deleted it (or marked it superseded by an update), 0 if still live. A version is visible to a transaction's snapshot when **both** hold:

- its **xmin is committed and within the snapshot** (the creating transaction finished before the snapshot was taken), and
- its **xmax is not visible to the snapshot** — either unset, or belonging to a transaction that hadn't committed as of the snapshot, so as far as this reader is concerned the version was never deleted.

It is *not* a plain numeric `xmin <= T < xmax` comparison: "within the snapshot" depends on which transactions had already committed when the snapshot was taken, not merely on whether an ID is smaller. An in-progress transaction with a low ID is still invisible.

**Implications:**

- Readers don't block writers; writers don't block readers.
- Writers block other writers on the same row (lock on the row's most recent version).
- "Updates" are really inserts of new versions; old versions are vacuumed later.
- **Bloat:** if vacuum can't keep up (or is blocked by a long-running transaction), tables and indexes accumulate dead versions. Performance degrades. Vacuum tuning is operational concern #1 in Postgres.

### Locking (the alternative)

MySQL InnoDB uses MVCC for reads but takes locks for writes. SQL Server uses 2PL (two-phase locking) by default (S/X locks). Locks block; MVCC doesn't. The formal treatment — 2PL variants, serializability, and wait-for-graph deadlock detection — is in `Database Internals.md`; here the focus is production-facing behavior (intent/gap locks, deadlock retry in app code).

Lock types. The core idea is a compatibility matrix: which locks can be held on the same object at the same time.

```
held \ requested    S           X
       S            compatible  blocked
       X            blocked     blocked
```

- **Shared (S, read):** many transactions can hold S on the same row at once, so concurrent readers coexist. An S lock blocks a writer (X), because the readers must see a stable value while they hold it.
- **Exclusive (X, write):** only one transaction can hold X on a row, and it blocks every other lock (S or X). A writer therefore excludes both other writers and (under pure 2PL) readers.
- **Intent locks (IS/IX):** taken at the table level *before* row locks, to signal "row-level locks exist underneath." Without them, a transaction wanting to lock the whole table would have to scan every row to check for conflicts; instead it just checks the table's intent lock. They make table-level and row-level locking coexist efficiently.
- **Gap locks:** lock the *range between* index entries, not just existing rows, so a competing transaction cannot insert a new row into the gap. This is what prevents phantoms (rows that appear on re-running a range query) at Repeatable Read / Serializable.

**Deadlocks** happen when two transactions each hold a lock the other needs, so neither can proceed — a cycle in the "who waits for whom" graph. The database periodically checks this wait-for graph; when it finds a cycle it picks a victim, aborts it (releasing its locks so the other can finish), and returns a deadlock error. The victim's work is rolled back, so **application code must catch the error and retry the whole transaction.**

```
time  T1                         T2
----  -------------------------  -------------------------
 1    UPDATE A  -- gets X on A
 2                               UPDATE B  -- gets X on B
 3    UPDATE B  -- wants X on B, T2 holds it -> T1 waits
 4                               UPDATE A  -- wants X on A, T1 holds it -> T2 waits

      T1 waits for T2, T2 waits for T1 -> cycle.
      DB aborts one (say T2) with a deadlock error; T1 then proceeds.
```

Mitigation: **always acquire locks in the same order** across the codebase (e.g. always touch row A before row B). If every transaction locks in one consistent order, no cycle can form. Keeping transactions short also shrinks the window in which a cycle can occur.

### Serializable Snapshot Isolation (SSI)

Postgres's Serializable level uses MVCC plus runtime detection of "serialization anomalies." If the engine sees that the actual concurrent execution would not be equivalent to any serial order, it aborts one transaction with a serialization failure.

**Application contract:** any transaction in Serializable may fail with a retry error. Application code must catch and retry. Repeatable transactions only — no side effects until commit succeeds.

---

## 7. Replication

Replication keeps copies of data on multiple machines, for high availability, read scaling, and disaster recovery. This section focuses on how production data systems (Postgres, MySQL, Cassandra) apply replication — the operational concerns. The underlying mechanism layer lives in `Distributed Systems.md`: the leader/multi-leader/leaderless topology taxonomy and conflict resolution in section 12, quorum math (`W + R > N`) in section 13. The two are the same picture at different altitudes.

### Leader-Follower (Master-Replica) Replication

The most common pattern. One leader accepts writes; followers replicate the WAL/oplog and serve reads.

```
         Writes
            |
            v
    +--------------------------+
    |          Leader          |  authoritative copy
    +--------------------------+
       |         |          |
       |         |          |
       v         v          v
  +--------+ +--------+ +--------+
  |Follower| |Follower| |Follower|
  +--------+ +--------+ +--------+
         Reads (possibly stale)
```

**Sync vs async:**

- **Synchronous:** leader waits for at least one follower to acknowledge before committing. Stronger durability; latency = max(leader, slowest sync follower).
- **Asynchronous:** leader commits locally, replicates in background. Lower latency; risk of data loss on leader failure.
- **Semi-sync:** wait for N of M followers (configurable). Common compromise.

Postgres supports physical (block-level) and logical (row-level) replication. MySQL supports statement, row, and mixed binlog formats.

**Failover:** when the leader dies, a follower is promoted to take its place. The hard parts: picking the most up-to-date follower (the one with the least lag), losing writes that were committed on the old leader but not yet replicated (the async data-loss window), and avoiding **split-brain** — two nodes both believing they are leader and accepting divergent writes. Done manually or by an orchestrator (Patroni, Orchestrator, cloud-managed failover); fencing the old leader is what prevents split-brain.

### Replication Lag

**Replication lag** is the delay between a write committing on the leader and that same write becoming visible on a follower. It exists because (asynchronous) replication is a pipeline: the leader commits and acknowledges the client *first*, then ships the change over the network, and only afterward does the follower apply it to its own copy. During that gap the follower is serving an older version of the data — a follower is therefore always at least slightly behind, and a read routed to it may not reflect writes that have already committed on the leader.

Lag is a *duration*, not a yes/no state: usually milliseconds, but it grows under write bursts, slow networks, or a follower that cannot keep up. This is the operational face of **eventual consistency** — stop writing and every follower eventually converges to the leader's state; keep writing and they trail behind by the current lag.

**Three read-consistency anomalies.** Lag does not just mean "slightly stale" — reading during the lag window can break three specific guarantees an application might assume. They are the practical face of the consistency models in `Distributed Systems.md`:

- **Read-your-writes:** after *you* write, *you* should see your own write. Broken when your follow-up read hits a follower that lacks the write — you edit your profile, the page reloads, and it shows the old value.
- **Monotonic reads:** you should never see time run *backward*. Broken when two successive reads hit different followers at different lag — a comment appears, you refresh, and it is gone.
- **Consistent prefix reads:** you should never see an effect before its cause. Broken when causally related writes (A *then* B) replicate at different speeds across partitions — an observer sees the answer before the question.

**Mitigations:**

- **Pin reads to the leader after a write** — route a session's reads to the leader for a short window (or while it still has writes pending), so it reads its own writes.
- **Pin a user to one replica** — hash the user to a fixed follower so their view only ever moves forward (gives monotonic reads).
- **Wait for catch-up** — block the read until the follower's applied position passes the write's log sequence number (LSN) or token.
- **Keep causally related writes together** — in one partition, or enforce causal consistency (`Distributed Systems.md`), for consistent-prefix.
- **Read from the leader for sensitive paths** — accept the throughput hit where staleness is unacceptable.

**Measuring lag.** Lag is reported in time or bytes. Postgres exposes write/flush/replay LSNs in `pg_stat_replication` (`pg_wal_lsn_diff` gives byte lag); MySQL reports `Seconds_Behind_Source`. Time lag is what users feel; byte lag predicts how long catch-up will take. Alert before lag exceeds the window your read-routing assumes.

**Causes.** Write bursts the follower cannot apply fast enough; single-threaded apply (MySQL replication was historically serial — parallel apply via `replica_parallel_workers` mitigates it); long-running read queries on a follower conflicting with apply (Postgres trades this off via `hot_standby_feedback` and `max_standby_streaming_delay`); and network saturation between leader and follower.

### Multi-Leader (Master-Master) Replication

Multiple leaders, each accepting writes, replicating bidirectionally. Used for cross-datacenter active-active deployments, multi-region writes, and offline-first sync. The hard part is conflict resolution (`Distributed Systems.md`). Rare and operationally expensive — avoid if leader-follower suffices.

### Leaderless Replication (Dynamo-style)

No designated leader; every replica accepts writes. Used by Cassandra, DynamoDB, Riak. Consistency is tunable per query via quorum reads and writes (the `W + R > N` math is in `Distributed Systems.md`). Background **read repair and anti-entropy** reconcile divergent replicas (Merkle trees compare ranges).

### Replication Topology Examples

```
Single-region Postgres:
  app -> leader -> sync replica -> async replica (read-only)

Cross-region failover (MySQL):
  region-1: leader
  region-2: async standby (for DR)

Cassandra (3 DCs):
  DC1: 3 nodes  --gossip--+
  DC2: 3 nodes  ----------+--- ring of 9, each row replicated 3 times
  DC3: 3 nodes  ----------+    (configurable replica factor per DC)

DynamoDB Global Tables:
  Region 1 <---active-active---> Region 2
  Last-write-wins on conflict, decided by wall-clock write timestamp.
```

### Backup vs Replication

Replication is **not** backup. Replication faithfully copies all data — including an accidental DROP TABLE. Backups protect against logical errors (bad migrations, accidental deletes, malicious actions); replication protects against hardware failure.

**Backup tiers:**

- **Logical:** SQL dump (pg_dump, mysqldump). Slow restore, version-independent, can selectively restore tables.
- **Physical:** filesystem-level (snapshots, pg_basebackup). Fast restore, version-locked.
- **Continuous (WAL archive):** keep all WAL since the base backup. Point-in-time recovery — restore to any timestamp.

**Test restore regularly.** Untested backups are not backups.

---

## 8. Partitioning and Sharding

A single machine's capacity is finite — in storage, memory, write throughput, or connection count. When data or load exceeds what one node can hold or serve, the data is split into parts that live on different nodes, so capacity scales roughly with the number of nodes. This is **scaling out** (more machines) as opposed to **scaling up** (a bigger machine); partitioning is the mechanism that makes scaling out possible for a single logical dataset.

Partitioning is orthogonal to replication, and production systems use both: **partitioning** splits *distinct* data across nodes (each node holds a different subset), while **replication** copies the *same* data to multiple nodes (each replica holds the same subset). Partitioning adds capacity; replication adds availability and read throughput. A typical deployment partitions the dataset into shards, then replicates each shard across a few nodes.

**Terms (often used interchangeably, with subtle differences):**

- **Partitioning:** dividing data into parts. Can be **horizontal** (split rows — the usual meaning, each partition holds a subset of rows with the same schema) or **vertical** (split columns — wide tables broken into narrower ones). Can stay within one machine (declarative table partitions in Postgres, which prune irrelevant partitions at query time) or span machines.
- **Sharding:** horizontal partitioning *across machines*, where each shard is an independent database that is independently routable — the application or a router decides which shard a key lives on. The term is most common in the NoSQL and application-managed-sharding world (MongoDB, Vitess, Citus); the relational literature usually says "horizontal partition."
- **Partition key (shard key):** the column(s) whose value determines which partition a row lands in. This single choice dominates everything downstream — see *Partition Keys* below.

### Strategies

**Hash partitioning:** hash a key, modulo the number of shards, route to the shard.

```
shard = hash(user_id) % N
```

- Even distribution.
- Range queries scatter across all shards.
- Adding/removing shards reshuffles everything unless using consistent hashing.

**Range partitioning:** assign contiguous ranges of the key to shards.

```
shard A: user_id 0          ... 1,000,000
shard B: user_id 1,000,001  ... 2,000,000
shard C: user_id 2,000,001  ... 3,000,000
```

- Range queries efficient (single shard).
- Hot spots if data is not uniformly distributed (e.g., monotonically increasing IDs concentrate writes on the latest shard).

**Directory / lookup-based:** maintain a metadata table mapping keys to shards.

- Flexible (any mapping).
- Adds a lookup hop or requires caching.
- Used by Vitess, some custom sharding layers.

**Geographic / functional:** partition by region or by service.

- Users in EU on EU shard for residency.
- "Orders service" data separate from "users service" data (often combined with sharding within each).

### Consistent Hashing

Standard hash partitioning fails when a shard is added — almost every key remaps. Consistent hashing solves this: only ~1/N keys move when a shard is added.

```
Hash space (ring 0 to 2^64):

   shard A at position 100
   shard B at position 5,000,000
   shard C at position 12,000,000

   Key X hashes to position 4,000,000 -> belongs to shard B
   (the next shard clockwise from key's position).

   Adding shard D at position 8,000,000:
   Only keys in range (5M, 8M] migrate from C to D.
   Keys in other ranges stay where they were.
```

**Virtual nodes:** each physical shard owns many positions on the ring, smoothing distribution and making rebalancing more granular. The full mechanism — ring construction, virtual nodes, rebalancing — is detailed in `Distributed Systems.md`.

### The Hard Parts

**Cross-shard queries:** a query that needs data from multiple shards must fan out, gather, and merge. Expensive at high cardinality. Avoid if possible by choosing the partition key so common queries hit one shard.

**Secondary indexes must themselves be partitioned, two ways:**

- **Local (document-partitioned):** each shard indexes only its own rows. Writes stay on one shard (cheap), but a query on the indexed column must scatter to *every* shard and gather — the index does not narrow which shards to hit. (Cassandra secondary indexes, Elasticsearch shards, DynamoDB LSI.)
- **Global (term-partitioned):** the index is partitioned by the indexed value, independently of the base-row partition key. A read hits only the shard(s) holding that term (efficient), but a single write may update an index entry on a different shard — making the write cross-shard and the index asynchronous. (DynamoDB GSI.)

Local optimizes writes and penalizes reads; global does the reverse.

**Cross-shard transactions:** distributed transactions — two-phase commit (2PC) or Sagas — are slow and operationally painful — the mechanics and trade-offs live in `Distributed Systems.md` (Distributed Transactions). Most sharded systems avoid them by designing for single-shard transactions.

**Hot keys / hot partitions:** one key or range gets disproportionate traffic. Mitigations: salt the key, split the hot range, cache aggressively at the application layer.

**Re-sharding:** splitting or merging shards as scale changes. Online resharding is hard; almost every sharding system has stories about painful migrations.

### Partition Keys: the Most Important Decision

Pick the key that satisfies these properties, in order:

1. **Even distribution.** Avoid skew.
2. **Common queries hit one shard.** Match the key to the dominant read patterns.
3. **Transactional needs.** Multi-row transactions must usually be within one shard.

A common pattern: `tenant_id` as the partition key for multi-tenant SaaS. Each tenant's data is co-located; tenants are independent. Most queries are scoped to a tenant anyway.

---

## 9. Schema Evolution and Migrations

Databases outlive code. Every running production schema is the result of a long chain of changes, and every change must be deployable without taking the system down.

### Expand / Contract (Parallel Change)

The pattern for safe schema migrations:

```
Goal: rename column `name` to `full_name`.

Stage 1 (Expand):
  - Add `full_name` column (nullable).
  - Backfill: copy `name` -> `full_name`.
  - Deploy code that writes to both columns and reads from `full_name` (fallback to `name`).

Stage 2 (Migrate clients):
  - All readers/writers on the new code.
  - Verify `full_name` is fully populated.

Stage 3 (Contract):
  - Stop writing to `name`.
  - Drop `name` column.
```

This sequence works because at every step, both old and new code can run simultaneously.

### Online Schema Changes

DDL (data-definition language) operations vary in how much they block:

| Operation | Postgres | MySQL InnoDB |
|-----------|----------|--------------|
| Add nullable column | Instant (metadata only) | Instant (since 8.0) |
| Add NOT NULL column with default | Fast (PG 11+ — instant for some cases) | Rewrites table |
| Add column with default expression | Rewrites table | Rewrites table |
| Drop column | Instant (metadata) | Rewrites table |
| Add index | `CREATE INDEX CONCURRENTLY` (non-blocking) | Online DDL (mostly non-blocking) |
| Alter column type | Often rewrites table | Often rewrites table |
| Rename column | Instant | Instant |
| Add foreign key | Locks both tables | Locks both tables |

**Postgres-specific dangers:**

- `CREATE INDEX` (without CONCURRENTLY) locks writes.
- `ALTER TABLE ADD COLUMN ... DEFAULT non_const_expr` rewrites the whole table.
- Long transactions hold dependency locks and block DDL.

For large tables, online schema change tools (pt-online-schema-change for MySQL, pg_repack for Postgres) work by building a new copy in the background and atomic-swapping.

### Versioning the Schema

Migration tools (Flyway, Liquibase, Alembic, ActiveRecord migrations, sqlx-migrate, goose) record applied versions in a `schema_migrations` table. Migrations run forward only in production; rollbacks happen via new forward migrations, not by un-running old ones.

**Migrations should be idempotent** — re-running them must be safe. **Never edit a migration that has been applied to any environment** — instead, create a new migration to correct it.

### Forward-Only Compatibility

In a multi-instance deployment, old and new versions of the code run simultaneously during rollouts. **The old code must work against the new schema, and the new code must work against the old schema, for at least one deployment cycle.**

This forces the expand-then-contract discipline. Breaking schema changes in one deploy = downtime or rollback impossibility.

---

## 10. Caching

Caches reduce latency and load on slower stores by holding recent or popular data in faster storage. Almost every production system has multiple cache layers.

### Cache Hierarchy

```
                Browser cache
                       |
                   CDN edge           <-- global, immutable
                       |
               API gateway cache
                       |
               In-process cache       <-- per-instance, microseconds
                       |
               Distributed cache      <-- Redis/Memcached, sub-ms
                       |
               Database query cache   <-- DB internal
                       |
               Source of truth        <-- disk
```

Each layer is faster but smaller and more local. Each adds an invalidation problem.

The layers, by type:

- **Browser / client cache** — lives on the end-user device, governed by HTTP headers (`Cache-Control`, `ETag`, `Expires`). Private to one user and zero network cost, but uncontrollable once shipped — you cannot force-evict a client, so it suits immutable, content-hashed assets.
- **CDN / edge cache** — a shared cache replicated to points of presence near users. Serves static assets and cacheable responses globally, offloading the origin and cutting round-trip time. Invalidated by explicit purge or by versioned (content-hashed) URLs.
- **Reverse-proxy / gateway cache** — Varnish, Nginx, or the API gateway caching whole HTTP responses keyed by URL plus selected headers. Shared across all users; sits in front of the application with no app code involved.
- **In-process (local) cache** — a map inside the application process (Caffeine, Guava, `lru-cache`). Microsecond access, no serialization or network hop, but *per-instance*: N instances hold N copies that can diverge, and it competes with the app for heap.
- **Distributed (remote) cache** — Redis/Memcached over the network: one shared copy every instance sees, large capacity, sub-millisecond — at the cost of a network hop and an extra failure domain. See *Common Cache Stores* below.
- **Database cache** — internal to the store: the buffer pool (hot pages held in RAM), the query-plan/statement cache, and materialized views. Effectively free and always consistent, but the smallest lever you directly control.

Two axes cut across all of them: **private vs shared** (a browser or in-process cache serves one user/instance; a CDN, gateway, or distributed cache is shared — higher hit rate, but invalidation becomes everyone's problem) and **near vs far** (nearer caches are faster but smaller and harder to keep coherent). The chain trades latency against consistency at every hop.

### Strategies

**Cache-aside (lazy loading):**

```
read(key):
  v = cache.get(key)
  if v is None:
    v = db.get(key)
    cache.set(key, v, ttl)
  return v

write(key, value):
  db.put(key, value)
  cache.delete(key)        <-- or update
```

Most common. Application controls the cache. Stale data possible briefly between db.put and cache.delete.

**Write-through:**

```
write(key, value):
  cache.set(key, value)
  db.put(key, value)
```

Cache always consistent with DB. Higher write latency. Requires cache to be reliable (if cache write fails, what?).

**Write-back (write-behind):**

```
write(key, value):
  cache.set(key, value)
  queue_async_db_write(key, value)
```

Lowest write latency. Risk: cache failure loses unwritten changes. Used in narrow cases (high write throughput, durability tolerant of small loss).

**Read-through:**

The cache itself fetches from the DB on miss. Same code path on every read. Cleaner application code; ties cache to a specific data source.

### Invalidation

Once the source data changes, the cached copy is stale. **Invalidation** is keeping cached entries correct as the underlying data moves — the counterpart to the write strategies above, and one of the genuinely hard problems in caching.

**Strategies:**

- **TTL (time-to-live):** simplest. Acceptable staleness window. Watch for thundering herd at expiry.
- **Explicit invalidation on write:** application deletes/updates cache after DB write. Possible to miss invalidations (process crash between DB write and cache invalidate).
- **Event-driven invalidation:** DB or change-data-capture emits events; cache listens and invalidates. Decouples writers and cache.
- **Versioned keys:** include a version in the cache key. Invalidation = bump the version.

### Eviction

Invalidation removes entries that are *wrong*; **eviction** removes entries because the cache is *full*. The two are independent concerns — invalidation is about correctness, eviction about capacity — and a bounded cache needs both. When the cache hits its memory limit, the eviction policy decides which entry to drop, and that choice largely sets the hit rate.

- **LRU (least recently used):** evict the entry untouched for longest. The default for most caches; good when recent access predicts future access (temporal locality). Usually *approximated* (Clock / second-chance, or random sampling) rather than tracked exactly — the buffer-pool version is in `Database Internals.md`.
- **LFU (least frequently used):** evict the entry with the fewest hits. Better for stable hot sets, but a one-time scan can pollute it and once-popular keys linger, so production caches use aged/windowed variants (e.g. **W-TinyLFU** in Caffeine).
- **FIFO:** evict in insertion order, ignoring access. Simple but blind to hotness; rarely the best choice.
- **Random:** evict a randomly chosen entry. O(1), no per-entry metadata, and surprisingly competitive — Redis approximates LRU/LFU by sampling a handful of random keys rather than maintaining a global order.
- **TTL-based:** drop whatever expires first. Usually combined with the above rather than used alone.

**Redis `maxmemory-policy`** makes the choice concrete: `noeviction` (reject writes when full — a full cache becomes write errors), `allkeys-{lru,lfu,random}` (consider every key), and `volatile-{lru,lfu,random,ttl}` (consider only keys that carry a TTL). The `allkeys-*` policies let the cache shed cold data to stay within budget; `volatile-*` confines eviction to entries you have explicitly marked as expendable.

No policy helps once the **working set exceeds capacity** — every useful key is evicted before it is reused (thrashing). The fix is then more memory or a hotter/smaller key space, not a cleverer policy. A key dropped here — by eviction or by a short TTL — is exactly what triggers the next problem.

### Cache Stampede / Thundering Herd

When a hot key expires, every request that needed it hits the database simultaneously.

```
key X has TTL 0
1000 requests/sec for key X
At expiry: 1000 concurrent DB queries fire to refill the cache.
```

Mitigations:

- **Probabilistic early expiration:** refresh before TTL with some probability proportional to closeness to expiry. Spreads the regeneration.
- **Lock / single-flight:** only one request regenerates; others wait or serve stale.
- **Stale-while-revalidate:** serve stale data for a short window while one request refreshes.

### Common Cache Stores

**Redis:**

- In-memory KV with rich data structures (strings, lists, sets, sorted sets, hashes, streams, geo, hyperloglog).
- Persistence options (RDB point-in-time snapshots, AOF append-only log) — but performance is the primary use case, not durability.
- Single-threaded event loop (commands serialized). Predictable but hot keys saturate a core.
- Pub/sub, Lua scripting, pipelines.
- Cluster mode for horizontal scaling.
- Sentinel for HA failover.

**Memcached:**

- Pure string KV. Multi-threaded, simpler.
- No persistence, no replication, no data structures.
- Faster than Redis for raw string GET/SET at high core counts.
- Used when the requirement is simply a fast distributed hash map.

**In-process (Caffeine, Guava, lru-cache):**

- Faster than network cache (no RTT) but per-instance.
- Memory pressure on app heaps.
- Inconsistency between instances unless invalidations are pub/sub'd.

### What to Cache

| Good caching candidates | Bad caching candidates |
|------------------------|------------------------|
| Read-heavy, write-rare | Write-heavy data |
| Expensive to compute | Cheap to compute |
| Tolerates staleness | Must always be fresh |
| Skewed distribution (some hot keys) | Uniform random access |
| Stable identity | High-cardinality, low-reuse |

When caching is used to mask a slow query, the correct fix is often the query itself (or an added index), not a cache.

---

## 11. Queues and Streams

Async messaging decouples producers from consumers, smooths bursts, and lets the system continue working when downstream is slow. Two flavors that look similar but are fundamentally different.

### Queues (Task Queues)

Each message is delivered to **one** consumer. Once processed, it's gone.

```
Producer --> [Queue] --> Worker 1
                     --> Worker 2
                     --> Worker 3   (each message goes to exactly one)
```

Used for: background jobs, sending emails, image processing, async work.

**Examples:** AWS SQS, RabbitMQ, BullMQ (Redis), Sidekiq (Redis), Celery (RabbitMQ/Redis).

**Delivery semantics:**

- **At-least-once:** default. Workers may see the same message twice (e.g., worker dies after processing, before ack).
- **At-most-once:** rare. Drops messages on failure.
- **Exactly-once:** no transport delivers true exactly-once; what is achieved is "effectively once" via deduplication or idempotency.

**Consumer responsibilities:**

- **Acknowledge** after successful processing. Until ack, the message is "in flight" and will be redelivered if visibility timeout expires.
- **Idempotency:** since retries happen, processing must be safe to repeat.
- **Dead-letter queue (DLQ):** messages that fail repeatedly go here for inspection.

### Streams (Log-Based Messaging)

Messages are **appended to an immutable log**; consumers track their own read position (offset). Multiple consumer groups read the same log independently.

```
Producer --> [ m0  | m1  | m2  | m3  | m4  | m5  | ... ]   append-only log
                     ^           ^           ^
                     |           |           |
                consumer-A  consumer-B  consumer-C
                (offset 1)  (offset 3)  (offset 5)

Each consumer tracks its own offset and reads at its own pace.
The log is retained (days, weeks, or forever), so any consumer can rewind and replay.
```

**Examples:** Kafka, AWS Kinesis, GCP Pub/Sub, NATS JetStream, Redis Streams, Pulsar.

**Key properties:**

- **Ordering within partition.** Messages with the same key go to the same partition, preserving order for that key.
- **Replay:** consumers can rewind to any offset, reprocess history.
- **Multiple consumer groups:** the same log feeds analytics, billing, and notifications independently.
- **Backpressure shifted to consumer pace.** Producers append at their own rate; slow consumers fall behind but don't block production.

**Use streams for:**

- Event sourcing / change feeds.
- Analytics / data pipelines.
- Audit logs.
- Multi-consumer fanout.
- Any case where history must be retained.

### Queues vs Streams: Choosing

| Need | Use |
|------|-----|
| One worker should do this once | Queue |
| Multiple unrelated systems must process every event | Stream |
| Order matters globally | Single-partition queue (slow) — or stream with key |
| Replay history | Stream |
| Drop old messages on retention pressure | Stream (Kafka log compaction or time retention) |
| Variable per-message visibility timeout | Queue |
| Strict at-most-once with acks | Queue with manual ack |

### Delivery Semantics in Practice

**Exactly-once is "effectively once" plus idempotency.** Even Kafka's transactional producer / exactly-once semantics only provides exactly-once *within Kafka*. End-to-end exactly-once requires the consumer to be idempotent or to atomically update its sink and offset together.

```
Idempotency keys (the most common pattern):
  Producer attaches a unique key per logical event.
  Consumer checks a "processed" store before doing the work.
  If seen, skip; if not, do the work and record the key.

  Now retries are safe regardless of delivery semantics.
```

### Backpressure

What happens when consumers fall behind?

- **Queues:** messages pile up; eventually queue limit hits and producers are throttled or rejected.
- **Streams:** lag grows; old messages eventually expire if not consumed within retention.

Monitoring queue depth / consumer lag is essential. Service-level objectives (SLOs) typically include "lag < N seconds."

### Kafka Concepts (Briefly)

```
Cluster -- N brokers
  Topic -- logical stream
    Partition -- ordered shard
      Replicas -- N copies on different brokers
        ISR (In-Sync Replicas) -- subset of replicas that are caught up
          Leader -- handles reads/writes for this partition

Producer:
  partition_key -> hash -> partition
  acks=all      -> wait for ISR ack

Consumer Group:
  Partitions distributed across members
  Each partition consumed by exactly one member in the group
```

The mechanism behind Kafka's replication and leader election (ZooKeeper or KRaft) is consensus — covered in `Distributed Systems.md`.

---

## 12. Object Storage

Object storage (S3, GCS, Azure Blob, R2) stores arbitrary files — called *objects* — addressed by name. It is neither a database nor a POSIX filesystem: there is no query language, no append, no locking, and no real directory tree. The interface is essentially `PUT(key, bytes)` and `GET(key)`. This minimal model is what lets it scale to effectively unlimited capacity, which is why it is the default home for large or unstructured data: images, videos, backups, logs, ML datasets, and data lake files.

### Model

An object store is two levels deep: a **bucket** (a named, top-level container) holds **objects**, each identified by a unique **key**.

```
Bucket
  Object: key -> bytes + metadata
    Key:      arbitrary string ("year=2025/month=01/data.parquet")
    Body:     the bytes themselves
    Metadata: content-type, custom headers
    Version:  optional (only if versioning is enabled)
```

The namespace is **flat**: there are no real directories. A key may contain `/`, so `year=2025/month=01/data.parquet` *looks* like a path, but the entire string is just one opaque key. The folder hierarchy shown in consoles and SDKs is a presentation convenience computed from key prefixes, not a stored structure.

### Properties

These differ sharply from a filesystem and are the source of most surprises:

- **Durability far exceeds availability.** Objects are replicated or erasure-coded across many devices, so the probability of losing stored data is vanishingly small — but *reachability* (being able to serve a request right now) is a weaker, separate guarantee. Data can be safe yet temporarily unreachable.
- **Objects are immutable.** A `PUT` replaces the whole object; there is no append or in-place edit. Large objects are uploaded in chunks via *multipart upload*, but the result is still one whole object.
- **No locking.** Concurrent writes to the same key resolve as last-writer-wins; the store does not coordinate writers.
- **Read-after-write consistency.** Modern object stores return the latest version on any read following a successful write.
- **Cost shape matters more than absolute price.** Three independent components — stored volume, request count, and *egress* bandwidth — and egress (data leaving the provider, especially cross-region or to the internet) typically dominates read-heavy workloads. Design to minimize data movement, not just bytes stored.

### Storage Tiers

Providers offer multiple storage classes that trade lower storage cost for higher retrieval cost and latency. The spectrum runs from *hot* (instant access, cheap to read) through *cold/archive* (very cheap to store, slow and costly to retrieve — minutes to hours). Match the tier to access frequency, and use **lifecycle policies** to migrate objects down the spectrum automatically as they age.

### Patterns

**Direct upload from client:** Rather than streaming an upload through the application server (consuming its bandwidth and making it a bottleneck), the server issues a short-lived *presigned URL* and the client uploads straight to the store.

```
Client            App                 Store
  |                |                   |
  |--request------>|                   |   client asks for an upload URL
  |                |--presign--------> |   app generates a signed URL
  |  <--URL--------|                   |
  |                                    |
  |---PUT object directly------------->|   client uploads to the store, not the app
```

This saves bandwidth, removes the app from the data path, and scales with the store rather than the server.

**Static site / CDN origin:** Object store plus a CDN is the default for hosting static assets — durable and globally cached.

**Data lake:** Store columnar files (Parquet/ORC/Avro) partitioned by date and other dimensions, then query them in place with engines such as Trino, Spark, or DuckDB. This decouples storage from compute, so each scales independently.

**Backup target:** The standard destination for database backups, point-in-time recovery archives, and application snapshots. Cross-region replication provides disaster recovery.

### Gotchas

- **Throughput scales per key prefix.** Request rate is partitioned by key prefix, so spreading keys across multiple prefixes parallelizes load. (Deliberately randomizing key names is not required on modern object stores; using more prefixes is what buys throughput.)
- **Versioning and delete markers.** With versioning enabled, deleting an object only hides it behind a delete marker; prior versions persist (and keep incurring storage) until explicitly purged.
- **Object lock / WORM** (write-once-read-many) prevents modification or deletion for a retention period, used for compliance.
- **Accidental public exposure.** An object can be made public through several independent mechanisms (bucket policy, ACLs, IAM). Account-level "block public access" controls override these with a hard deny — prefer them.

---

## 13. Search

When users need full-text, fuzzy matching, faceted filters, relevance ranking, or geospatial queries, the relational DB falls short. Dedicated search engines (Elasticsearch, OpenSearch, Meilisearch, Typesense, Apache Solr) solve this.

### How Inverted Indexes Work

A search index is fundamentally an **inverted index**: instead of "document -> words," it's "word -> documents."

```
Document 1: "the quick brown fox"
Document 2: "the lazy dog"
Document 3: "the quick dog"

Inverted index:
  the   -> [1, 2, 3]
  quick -> [1, 3]
  brown -> [1]
  fox   -> [1]
  lazy  -> [2]
  dog   -> [2, 3]

Query "quick dog":
  intersect postings(quick) and postings(dog) -> [3]
```

Adding positions, scores, and relevance ranking (TF-IDF, BM25) yields a search engine.

### What Search Engines Add

- **Analyzers and tokenizers:** lowercase, stemming, stop words, language-specific tokenization, n-grams for partial matches.
- **Relevance scoring:** BM25 by default; tunable per field.
- **Faceting and aggregations:** group by category, count by attribute, range buckets.
- **Geo queries:** distance, bounding box, polygon.
- **Fuzzy/edit-distance matching:** "tpyo" matches "typo".
- **Highlighting:** surfaces matched portions in results.

### Architecture Pattern

```
     Source of truth
     (Postgres)
           |
   Change stream (CDC / events)
           |
           v
     Search index
    (Elasticsearch)
           |
         Reads
           |
      (search API)
```

The DB is the source of truth. The search index is a derived projection, kept eventually consistent via a stream of changes. **Never make the search index the source of truth** — they prioritize search quality over durability, and reindexes are routine.

### When You Don't Need Elasticsearch

- Postgres `pg_trgm` extension handles substring/fuzzy queries.
- Postgres full-text search (`tsvector`, `tsquery`) handles many small-to-medium full-text needs.
- Meilisearch and Typesense are simpler alternatives for smaller scales.
- DuckDB / SQLite FTS for embedded search.

Adopting Elasticsearch comes with significant operational overhead. Use Postgres extensions first; graduate to Elasticsearch when relevance, scale, or specific features demand it.

---

## 14. Time-Series and Analytical Stores

OLTP databases (Postgres, MySQL) optimize for transactional reads and writes: many small queries, point lookups, low latency. Analytical workloads (OLAP) are different: few queries, scanning billions of rows, aggregating over columns.

### Columnar Storage

Analytical engines store data **by column**, not by row.

```
Row store:
  Page 1: [r1: id=1, name=alice, age=30, country=US]
          [r2: id=2, name=bob,   age=25, country=UK]
          [r3: id=3, name=carol, age=35, country=US]

Column store:
  Page 1 (id):      [1, 2, 3, ...]
  Page 2 (name):    [alice, bob, carol, ...]
  Page 3 (age):     [30, 25, 35, ...]
  Page 4 (country): [US, UK, US, ...]
```

**Why this matters for OLAP:**

- Queries scan only the columns referenced. `SELECT AVG(age) FROM users` reads one column, not 50.
- Columns of similar data compress well (run-length, dictionary, delta encoding). 5-10x compression is typical.
- Vectorized execution: process batches of column values with SIMD (single instruction, multiple data) instructions.

**The cost:** point-lookup of a single full row is slower (reads from many column files). Updates and deletes are expensive (rewrite or mark-deleted). OLAP is read-mostly.

### Examples

| System | Type | Notes |
|--------|------|-------|
| BigQuery | Cloud OLAP | Serverless, separation of storage and compute, SQL |
| Snowflake | Cloud OLAP | Compute warehouses (clusters) attached to shared storage |
| Redshift | Cloud OLAP | AWS, fixed clusters (Redshift Serverless now exists) |
| ClickHouse | OLAP (self-host) | Very fast, MergeTree storage, excellent for logs/events |
| DuckDB | Embedded OLAP | SQLite for analytics; in-process, no server |
| Druid / Pinot | Real-time OLAP | Streaming ingestion + interactive aggregations |
| TimescaleDB | Postgres extension | Hypertables, time-bucketing, retention policies |
| InfluxDB | Time-series | Tag-indexed, retention/downsampling built in |
| Prometheus | Metrics | Pull model, tag-indexed, time-series for monitoring |

### Lakehouse Architecture

Data lake (object storage with Parquet/ORC files) + transactional metadata layer (Apache Iceberg, Delta Lake, Hudi) = ACID transactions, time travel, schema evolution on top of cheap object storage.

```
Storage:        S3 / GCS / ADLS
                  |
Format:         Parquet files
                  |
Metadata:       Iceberg / Delta / Hudi
                  |
Compute (query):
  Trino / Presto / Spark / DuckDB / Snowflake (external tables)
```

The pattern decouples storage (cheap, durable, vendor-neutral) from compute (scalable, ephemeral). Same data queried by analysts in Snowflake, by ML pipelines in Spark, by ad-hoc queries in DuckDB.

### OLTP -> OLAP Pipeline

```
Source: Postgres OLTP
          |
          | CDC (Debezium / Postgres logical replication)
          v
       Kafka
          |
          | Stream processor (Flink, Spark Streaming, ksqlDB)
          v
   Object storage / Warehouse / Lakehouse
          |
          v
    BI / Analytics / ML
```

The OLTP database is never queried directly for analytics — that would compete with transactional load. CDC propagates changes to a dedicated analytical stack.

---

## 15. Reliability Patterns (Outbox, CDC, Idempotency)

The hard problem with multi-system writes: **reliably performing "write to the database AND emit an event" without losing one or the other.** (For the delivery-semantics theory behind this — at-most/at-least/exactly-once and idempotency — see `Distributed Systems.md`; this section is the data-pipeline implementation.)

Naive approach:

```python
def create_order(order):
    db.insert(order)            # commits
    queue.send(order_created)   # what if this fails?
```

Order is in the database but the event was lost. Now downstream systems (email, analytics, inventory) never see this order.

Or the reverse:

```python
def create_order(order):
    queue.send(order_created)   # sent
    db.insert(order)            # what if this fails?
```

Event was sent but the order doesn't exist. Consumers act on a ghost.

### Transactional Outbox

Write the event to an `outbox` table in the same DB transaction as the business write. A separate process polls the outbox and emits to the queue.

```sql
BEGIN;
INSERT INTO orders (...) VALUES (...);
INSERT INTO outbox (event_type, payload, ts) VALUES ('order_created', ...);
COMMIT;
```

A relay process tails the outbox and publishes:

```
outbox table:
  id | event_type    | payload | emitted_at
  1  | order_created | {...}   | NULL          <-- relay picks up
  2  | order_paid    | {...}   | 2025-01-15
```

The relay reads unpublished rows, sends to the queue, marks them emitted. At-least-once delivery — combined with idempotent consumers, effectively exactly-once.

### CDC (Change Data Capture)

Instead of an outbox table, the relay reads the database's WAL (Postgres logical replication, MySQL binlog, MongoDB oplog) and converts every change to an event.

```
Postgres WAL --> Debezium --> Kafka topic "orders.changes"
                              {op: "insert", before: null, after: {...}}
                              {op: "update", before: {...}, after: {...}}
                              {op: "delete", before: {...}, after: null}
```

Every change is captured automatically. No outbox table. The cost: events follow row changes, not business events ("order paid" is just `update orders set status='paid'`). For business event semantics, outbox is clearer.

CDC is the standard mechanism for OLTP -> data lake / search index / analytics pipelines.

### Idempotent Consumers

Outbox and CDC both deliver at-least-once, so consumers must dedupe to reach effectively-once. The consumer records an idempotency key (or event ID) per processed message and skips replays, returning the cached result. "Exactly-once delivery" at the messaging layer alone is rarely sufficient — the sink must either process idempotently or commit its result atomically with the consumer offset.

---

## 16. Operating Data Systems

The data layer is where most production incidents happen. A few patterns worth knowing.

### Capacity Planning

- **Storage:** size of working set vs RAM. If hot data doesn't fit in RAM, every read hits disk and latency multiplies.
- **IOPS:** burst vs sustained. SSDs have IOPS budgets; bursts are cheap, sustained traffic exposes the real limit.
- **Connections:** databases have connection limits. Web apps usually need connection pooling (PgBouncer, ProxySQL) to fit thousands of app instances into hundreds of DB connections.
- **Replication lag budget:** define what's acceptable (seconds vs minutes) and monitor.

### Connection Pooling

Each DB connection costs memory and an OS process/thread on the server. Web app instances tend to over-provision. PgBouncer (Postgres) and ProxySQL (MySQL) multiplex thousands of app connections onto a small DB-side pool.

```
3000 app threads <-- PgBouncer (3000 in, 100 out) --> Postgres (100 connections)
```

Modes:

- **Session pooling:** one DB connection per app session.
- **Transaction pooling:** DB connection assigned only for a transaction. Most efficient. Some features (prepared statements, advisory locks, SET LOCAL) break.

### Slow Query Triage

Most performance problems come from a handful of bad queries.

```
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Look for:

- Sequential scans on large tables (missing index).
- Repeated similar queries that could be batched (N+1).
- Long-running idle-in-transaction connections (blocks vacuum).
- Lock contention (`pg_locks`, `pg_stat_activity` with `wait_event_type`).

### N+1 Queries

The pattern that kills web app performance:

```python
orders = Order.query.all()              # 1 query
for order in orders:
    print(order.user.name)              # 1 query per order
                                        # = 1 + N queries
```

Fix by eager loading / join / batch loading.

```python
orders = Order.query.options(joinedload('user')).all()  # 1 query
```

### Backups, Restores, DR Tests

- **Automated daily backups** + continuous WAL archive for point-in-time recovery.
- **Cross-region replicated** for regional failures.
- **Test restores regularly** — a backup never restored is theoretical.
- **DR runbook with measured RTO/RPO** (recovery time / point objectives). Run drills.

### Monitoring Signals That Matter

```
Saturation:
  - Connection pool usage
  - Replication lag
  - Disk queue depth
  - CPU at the database

Errors:
  - Query errors per minute
  - Deadlocks per minute
  - Failed connections

Latency:
  - p50/p99 query time per query template
  - Lock wait time
  - WAL flush time

Throughput:
  - Transactions per second
  - Bytes read/written
  - Cache hit rate
```

USE method (Utilization, Saturation, Errors) and RED method (Rate, Errors, Duration) — see `Observability.md`.

### Tools Worth Knowing

```
Postgres:
  psql                 client
  pgcli                better client (autocompletion, syntax highlighting)
  pg_dump / pg_restore logical backup/restore
  pg_basebackup        physical backup
  pgbench              load testing
  pg_stat_statements   query statistics
  pgBadger             log analyzer
  pg_repack            online table reorganization
  PgBouncer            connection pooler

MySQL:
  mysql                client
  mysqldump            logical backup
  Percona Toolkit      pt-online-schema-change, pt-archiver, etc.
  ProxySQL             connection pooler, query routing

Redis:
  redis-cli            client + DEBUG, LATENCY, INFO commands
  redis-benchmark      load testing
  rdbtools             RDB file analysis

Kafka:
  kafka-console-consumer/producer
  kafkacat (kcat)      Swiss army knife
  ksqlDB               SQL on streams
  Kafka UI tools       AKHQ, Conduktor, Redpanda Console

Object storage:
  aws s3 / gsutil / az  cloud-native CLIs
  rclone                multi-cloud sync
  s3cmd, mc             alternatives

Cross-DB:
  Datagrip / DBeaver    GUI clients
  Atlas / Flyway / Liquibase  migrations
  Debezium              CDC
```

### Operational Discipline

- **Never run DDL or ad-hoc DELETE/UPDATE in production without a transaction wrapper and a tested rollback path.**
- **Schema changes go through the same code review and deployment pipeline as code.**
- **Long-running transactions are a liability** — they hold locks, block vacuum, and pin replication slots.
- **Add observability before scale, not after.** When the incident hits, the dashboards must already be in place.
- **Failover regularly in staging.** Untested failover often doesn't work.
