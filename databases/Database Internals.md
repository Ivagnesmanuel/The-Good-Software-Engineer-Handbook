# Database Internals

How a relational DBMS executes against a schema: the layered architecture, the buffer manager that mediates memory and disk, on-disk page and record layout, file organizations and indexes, external sorting, concurrency control, crash recovery, and query execution and optimization.

This document covers *how the engine works inside*. For the formal data model, relational algebra, ER modeling, and logical design see `Database Design.md`. For the systems view (choosing and operating stores at scale) see `Data Systems.md`. For the SQL language itself see `SQL.md`. The cost figures here use the page-based model introduced in section 4; the napkin latency numbers live in `Data Systems.md`.

---

## Table of Contents

1. DBMS Architecture
2. Buffer Management
3. Storage: Pages and Records
4. File Organizations and the Cost Model
5. External Sorting
6. Indexing
7. Concurrency Control
8. Recovery
9. Query Execution and Optimization

---

## 1. DBMS Architecture

A DBMS is a stack of cooperating managers between an SQL command and the data on disk.

```
                                SQL commands
                                      |
                                      v
        +---------------+----------------------------+-----------------+
        | Transaction   |         SQL engine         |   Recovery      |
        | manager       +----------------------------+   manager       |
        |               |     Access file manager    |                 |
        | (isolation,   +----------------------------+   (atomicity,   |
        |  consistency) |       Buffer manager       |    durability)  |
        |               +----------------------------+                 |
        |               |        Disk manager        |                 |
        +---------------+----------------------------+-----------------+
                                      |
                                      v
                                   +------+
                                   | Data |
                                   +------+

   The transaction and recovery managers are full-height side bars: they wrap
   every layer (SQL engine, access methods, buffer, disk), because a
   transaction's isolation and atomicity must hold across all of them.
```

- **SQL engine** — compiles a query into a plan and runs it (sections 9).
- **Access file manager** — maps relations to files, records to pages, and maintains indexes (sections 3–6).
- **Buffer manager** — moves pages between disk and a memory pool (section 2).
- **Disk manager** — issues physical reads/writes.
- **Transaction manager** — enforces isolation and consistency through concurrency control (section 7).
- **Recovery manager** — enforces atomicity and durability through logging (section 8).

---

## 2. Buffer Management

At the physical level a **database** is a set of **files**, each a set of fixed-size **pages** stored in physical blocks. A page is the unit of transfer between disk and memory: its size equals a block (typically 2 KB–64 KB).

The **buffer** (buffer pool) is a non-persistent memory region holding pages currently in use. It is shared by all transactions and organized into **frames**, each frame the size of one page. The **buffer manager** moves pages between disk and the pool, tracking the mapping in a **page table** that associates each resident page (by its page ID) with its frame.

Per frame it maintains:

- **pin-count** — how many transactions are currently using the page;
- **dirty bit** — whether the page has been modified since it was loaded.

### Primitive operations

- **Fix** — load a page into a frame and pin it. If already resident, increment the pin-count; otherwise choose a victim frame (by the replacement policy), write it back if dirty, read the requested page, and return the frame.
- **Unfix** — the transaction no longer needs the frame; decrement the pin-count.
- **Use** — the transaction modifies the frame; set the dirty bit.
- **Force** — synchronous write of a page to disk (the caller waits).
- **Flush** — asynchronous write, performed when the buffer manager is idle.

### Replacement policy

A victim is chosen only among frames with **pin-count = 0**. If none exist, the request queues or aborts. Among candidates:

- **LRU** — evict the least recently used (a queue of unpinned frames).
- **Clock (second-chance)** — frames in a circular list, each with a *referenced* bit. The hand advances: a referenced frame has its bit cleared and is skipped; the first unreferenced frame is evicted. Approximates LRU cheaply.

### Steal and force policies

Two orthogonal choices govern when modified pages reach disk; they determine what recovery must do (section 8).

- **Steal / no-steal** — *steal* permits writing a dirty page of an *uncommitted* transaction to disk (its victim may be dirty). No-steal forbids it.
- **Force / no-force** — *force* writes all of a transaction's pages to disk at commit. No-force lets a committed transaction's pages be written later, asynchronously.

From a recovery standpoint, **no-steal** avoids undo (no uncommitted change ever reaches disk) and **force** avoids redo (every committed change is already on disk). But no-steal pins many pages, and force generates synchronous I/O at every commit. The practical default is therefore **steal + no-force**, which is the most efficient for the buffer but requires both undo and redo in recovery.

---

## 3. Storage: Pages and Records

A **file** is a set of pages; a page holds records. Usually one file stores one relation (a DBMS *file* is the page set of one relation, distinct from an *OS file*; the engine may map many DB files into one OS file or onto raw disk). A file organization must support insert/delete/update of a record, read by record id, and scan.

A page has an address (**page ID**) and is divided into **slots**, each able to hold one record; it may carry a header with pointers to other pages. Each record has a **record id** `rid = <page id, slot number>`.

### Fixed-length records

- **Packed** — records occupy contiguous slots, with a count of records. Moving a record changes its rid, which breaks references.
- **Unpacked** — a bit array marks each slot free (0) or used (1); a record keeps its slot (stable rid) at the cost of checking the bit array.

A fixed-length record is a concatenation of fields; field `i` is found at a computable offset (base + sum of preceding field lengths).

### Variable-length records

The page uses a **slot directory**: an array (growing from the page end) of `(offset, length)` entries plus a free-space pointer. Deleting a record sets its directory entry to -1 and reclaims the space; moving a record to another page leaves a *forwarding pointer* in the old slot.

A variable-length record is stored either with field separators, or — preferably — with a leading **array of offsets**, which gives direct field access and compact NULL storage (two equal consecutive offsets denote a NULL).

A record too large for one page is split into **fragments** across pages (a **spanned** record), each fragment tagging whether it is first/last and carrying pointers to the previous/next fragment.

---

## 4. File Organizations and the Cost Model

```
file organization
|
+-- simple  (primary: records only, no separate index file)
|     +-- heap      unordered; fast scan & insert, poor search/delete
|     +-- sorted    ordered on a search key; good range, costly insert/delete
|     `-- hashed    buckets via h(k) mod N        (== static hashing)
|
`-- index-based  (data file + an auxiliary index file)
      +-- sorted index   a (key -> pointer) file over the data
      +-- tree index
      |     +-- ISAM      static tree; inserts go to overflow pages
      |     `-- B+-tree   dynamic balanced tree (the standard index)
      `-- hash index
            +-- static       fixed N buckets; overflow chains (= hashed file)
            +-- extendible   directory + global/local depth; splits one bucket
            `-- linear       no directory; NEXT/LEVEL pointers drive splitting
```

- **Heap file** — pages and records in no order; pages allocated/freed as the relation grows, tracked by a page list or a directory. Efficient for scan and insert; poor for search and delete.
- **Sorted file** — records sorted within each page on a **search key**, pages stored contiguously in key order. Better search (binary search) than heap; poor for insert/delete (shifting).
- **Hashed file** — pages grouped into **buckets**, each bucket a primary page plus overflow pages. A **hash function** maps a search-key value `k` to bucket `h(k) mod N`. Fast equality lookup; no range support; overflow chains degrade as the relation grows (static hashing).

### The page-access cost model

Since I/O dominates, cost is counted in page accesses, with:

- **B** — pages in the file;
- **R** — records per page;
- **D** — time to read/write one page;
- **C** — time to process one record.

| Organization | Scan | Equality selection | Range selection | Insert | Delete |
|---|---|---|---|---|---|
| Heap | BD | BD | BD | 2D | search + D |
| Sorted | BD | D log2 B | D (log2 B + matching pages) | search + 2BD | search + 2BD |
| Hashed | 1.25 BD | D (1 + matching pages) | 1.25 BD | 2D | search + D |

No organization dominates: heap wins on insert/scan, sorted on range, hashed on equality.

Equality costs assume no early termination (worst case); on a candidate key a heap stops after ~B/2 pages on average.

---

## 5. External Sorting

Sorting data larger than memory (**external sorting**) is page-based, not record-based: whole pages are read into the buffer and merged. It underlies sort-based operators (section 9) and sorted-file/index construction.

### Two-way merge-sort

- **Pass 0** — read each page, sort it in memory, write it back (each page is a sorted *run*).
- **Pass i** — merge pairs of runs into runs of double the length.

With `B` pages, the number of passes is `⌈log2 B⌉ + 1` and the total cost is `2B(⌈log2 B⌉ + 1)`. It uses only 3 frames (2 input, 1 output).

### Multi-pass merge-sort

With `F` free frames the algorithm reads `F` pages per run in pass 0 (fewer initial runs), then merges `Z = F - 1` runs at a time (one output frame). The number of passes drops to roughly `⌈log_{F-1} B⌉ + 1`, with cost `≈ 2B(log_{F-1} B + 1)` — far fewer passes because `F >> 2`. (The log base is the fan-in `F - 1`; it is often written `F`, since `F - 1 ≈ F` for large `F`.)

### Improvements

- **Double buffering** — a second set of frames is processed while the first set is being filled by I/O, hiding disk latency.
- **Replacement selection** — pass 0 builds longer initial runs using a priority queue: a newly read record whose key exceeds the last one written joins the current run, otherwise it is held for the next run. On random data this roughly doubles the average run length, equivalent to doubling the sort buffer.

---

## 6. Indexing

An **index** is any structure that, given the value of one or more fields (the **search key**), quickly locates the records with that value. An index-based organization has a **data file** (the records of `R`) and an **index file** of **data entries**, each mapping a search-key value to the records holding it.

### Properties of an index

- **Clustering** — an index is **clustered** when the data entries' order matches the order of records in the data file. There is **at most one clustered index** per file (the file can be sorted on only one key). A non-clustered index points at records scattered across pages.
- **Primary vs secondary** — a **primary** index is on a key that includes the primary key (no duplicates). Any other is **secondary** (may have duplicates; *unique* if its key is a candidate key).
- **Dense vs sparse** — a **dense** index has a data entry for every data-file value; a **sparse** index has one entry per data page (and must be clustered). A dense index supports an *index-only* search (answer from the index alone); a sparse index is smaller.
- **Simple vs composite** key; **single-level vs multi-level** (an index built over an index, recursively, until the top level fits in memory).

### Data-entry structures (alternatives)

1. The data entry *is* the data record (the index file is the data file).
2. The data entry is a pair `(k, rid)` — key plus a record reference.
3. The data entry is `(k, rid-list)` — key plus a list of references (compact for duplicates).

Alternative 1 is clustered by definition.

### Sorted index

An auxiliary sorted file of `(key, pointer)` entries over the data file. A **clustering (primary) sorted index** — data file sorted on the same key — supports equality and, since the file is ordered, range queries; it may be dense or sparse. A **non-clustering (secondary) sorted index** must be dense (the data file is not ordered on its key) and does not support range scans on the data file.

### Tree index

The index is a **balanced tree** of pages: internal nodes route by key, leaves hold the data entries. Every search descends from the root to a leaf, so minimizing tree **height** is the goal; height is `log_F N` where `F` is the **fan-out** (children per node) and `N` the number of leaves. Larger blocks give larger fan-out and shallower trees.

- **ISAM** (Indexed Sequential Access Method) — a static tree: leaves allocated sequentially, internal nodes built once. Inserts go to overflow pages (no rebalancing). Good only when the data is static.
- **B+-tree** — a dynamic balanced tree, the standard index. Each node of **rank** `d` holds between `d` and `2d` keys (root excepted); leaves are linked in key order (for range scans). All data entries live in the leaves.
  - **Search** — descend root to leaf, `log_F N` page accesses.
  - **Insertion** — find the leaf; if full, **split** it, pushing the middle key up; a leaf split copies the separator up, an internal split moves it up. The tree grows in breadth; it deepens only when the root splits.
  - **Deletion** — if a node underflows, **redistribute** from a sibling, or **coalesce** (merge) two nodes and remove a separator from the parent, recursing if needed.

A clustered B+-tree (alternative 1, leaves ~67% full) scans in `1.5B(D + RC)` and does equality/range search in `D log_F(1.5B) + C log2 R`. An unclustered B+-tree pays one page access per matching record.

### Hashed index

- **Static hashing** — coincides with the hashed file: a fixed `N` buckets, `h(k) mod N`. Insertions beyond capacity build long overflow chains, the central weakness.
- **Extendible hashing** — a **directory** of pointers to buckets, indexed by the top `g` bits (**global depth**) of `h(k)`; each bucket has a **local depth**. On overflow, split only the full bucket (redistributing by one more bit); double the directory only when the split bucket's local depth equalled the global depth. Avoids overflow chains by splitting precisely the right bucket.
- **Linear hashing** — avoids the directory. Primary pages are stored sequentially; a `NEXT` pointer and a round counter `LEVEL` drive splitting. A split occurs (e.g., on overflow) at the bucket `NEXT`, advancing round by round until the bucket count doubles. A family `h_i(v) = h(v) mod 2^i N` selects the function. No directory access per lookup, at the cost of some near-empty buckets.

### Comparison

```
Heap file        : best space; good scan/insert; poor search/delete
Sorted file      : good space; better search; poor insert/delete
Clustered B+-tree: low overhead; good insert/delete/search; optimal range
Static hash      : optimal equality search/insert/delete; poor scan and range
```

No organization is uniformly superior. The write-optimized alternative to these in-place organizations — the log-structured merge tree (LSM) — and the B-tree/LSM tradeoff are in `Data Systems.md`.

---

## 7. Concurrency Control

The **transaction manager** ensures isolation and consistency under concurrent execution.

### Transactions and ACID

A **transaction** is the execution of a procedure that reads and writes the database and forms a single logical unit, bracketed by a **begin** and an **end**, terminating in **commit** or **rollback**. The desirable properties are **ACID**:

- **Atomicity** — all or none of the actions take effect.
- **Consistency** — execution moves the database between valid states.
- **Isolation** — each transaction executes as if alone.
- **Durability** — committed effects survive crashes.

For concurrency analysis a transaction is abstracted to its reads, writes, and commit (main-memory operations are ignored): `T1: r1(A) r1(B) w1(A) w1(B) c1`.

### Schedules and serializability

A **schedule** over `T1..Tn` is a sequence of their actions respecting each transaction's internal order. It is **serial** if no two transactions interleave. A schedule is **serializable** if it is equivalent to *some* serial schedule. Two schedules are **equivalent** if, from every initial state, they produce the same final state.

Interleaving without control produces anomalies (the same family seen in `Data Systems.md`, here named by the conflicting operations):

- **WR (dirty read)** — a transaction reads a value written by an uncommitted transaction.
- **RW (unrepeatable read)** — a transaction re-reads a value that another has since changed.
- **WW (lost update)** — two transactions each read the same value and overwrite it in turn; the first writer's update is lost. (Write skew, a cross-element variant, is in `Data Systems.md`.)

#### View-serializability

`S` is **view-serializable (VSR)** if it is view-equivalent to a serial schedule, where *view-equivalent* means the same **reads-from** relation and the same **final writes**. Checking view-equivalence is polynomial, but checking view-serializability is **NP-complete**, so it is not used in practice. The class is also not *monotone* (closed under taking subsets of transactions).

#### Conflict-serializability

Two actions **conflict** if they belong to different transactions, touch the same element, and at least one is a write. `S` is **conflict-serializable (CSR)** if it can be turned into a serial schedule by swapping adjacent non-conflicting actions. CSR is tested with the **precedence (conflict) graph**: a node per transaction, an edge `Ti -> Tj` when an action of `Ti` precedes and conflicts with one of `Tj`.

> A schedule is conflict-serializable **iff its precedence graph is acyclic.**

The test is polynomial (build the graph, check for a cycle; a topological order gives an equivalent serial schedule). **CSR ⊂ VSR** (conflict-serializable implies view-serializable, not conversely), and CSR is the class databases actually enforce. Finer theoretical classes exist — the order- and commit-order-preserving refinements `COCSR ⊂ OCSR ⊂ CSR`, and the broader final-state class `CSR ⊂ VSR ⊂ FSR` (final-state serializability, matching only the final state of some serial schedule) — but they are not used in practice.

A scheduler could maintain the precedence graph incrementally and abort any transaction whose action introduces a cycle, but this is costly and rarely used directly. Locking and timestamps are the practical mechanisms; the production realization of incremental conflict-graph detection is serializable snapshot isolation (see `Data Systems.md`).

### Locking

A transaction must acquire a **lock** before operating on an element. With exclusive locks only: a transaction is **well-formed** if every action is bracketed by lock/unlock, and a schedule is **legal** if no element is locked by two transactions at once. Real systems use two modes:

- **Shared lock** `sl(A)` — for reading; compatible with other shared locks.
- **Exclusive lock** `xl(A)` — for writing; incompatible with everything.
- A shared lock may be **upgraded** to exclusive.

```
Compatibility:   requested
                  S    X
   held   S      yes   no
          X      no    no
```

#### Two-phase locking (2PL)

> A schedule follows **2PL** if, in every transaction, all lock operations precede all unlock operations.

**Every legal, well-formed 2PL schedule is conflict-serializable.** The converse fails — some CSR schedules are not 2PL. 2PL still admits **deadlock** (each transaction waits for a lock the other holds).

Stronger variants fix recoverability (below):

- **Strict 2PL (S2PL)** — 2PL holding all *exclusive* locks until commit/rollback. Strict and serializable.
- **Strong strict 2PL (SS2PL)** — holds *all* locks (shared and exclusive) until commit/rollback. Rigorous and serializable, and its commit order is a conflict-serialization order.

#### Deadlock management

- **Timeout** — kill a transaction waiting past a threshold `t`. Simple, but a high `t` reacts slowly and a low `t` kills too eagerly (risk of repeated/individual blocking).
- **Detection** — maintain a **wait-for graph** (`Ti -> Tj` if `Ti` waits for a lock held by `Tj`); a cycle is a deadlock, broken by killing a transaction on it.
- **Prevention** — assign each transaction a priority (typically by age, older = higher priority); two symmetric schemes avoid cycles by construction:
  - **wait-die** (non-preemptive): on conflict `Ti` waits for `Tj` only if `pr(Ti) > pr(Tj)`, otherwise `Ti` is killed (dies) and restarts later.
  - **wound-wait** (preemptive, the dual): if `pr(Ti) > pr(Tj)` then `Ti` *wounds* (aborts) `Tj` and takes the lock; otherwise `Ti` waits.

If a schedule deadlocks it is not 2PL; the converse does not hold.

### Timestamp ordering

Each transaction gets a unique **timestamp** `ts(T)` consistent with arrival order; the scheduler enforces the serialization order *to equal the timestamp order*, which is therefore conflict-serializable. Per element `X` it keeps:

- **rts(X)** — highest timestamp of any reader;
- **wts(X)** — highest timestamp of any writer;
- **commit-bit cb(X)** — whether the last writer of `X` has committed.

On `ri(X)` or `wi(X)` the scheduler compares `ts(Ti)` against `rts/wts` and either executes (updating the stamps), **blocks** (when `cb(X)` is false — to avoid dirty reads), or **rolls back and restarts** `Ti` with a new timestamp when its action arrives "too late" (out of timestamp order). The **Thomas rule** optimizes a late write that no one has read: it is simply *ignored* (the newer write already supersedes it) rather than causing a rollback. Timestamps avoid locks but can still deadlock via the commit-bit waiting.

**Multiversion timestamp** keeps several versions `X1..Xn` of each element so reads never block: a read is served the newest version whose `wts` does not exceed the reader's timestamp, preserving a coherent snapshot. Old versions are garbage-collected once no active transaction can need them. Real systems commonly run read-write transactions under (strict) 2PL and read-only transactions under a multiversion method.

|  | Waiting | Serialization order | Commit dependence | Deadlock |
|---|---|---|---|---|
| 2PL | yes | by conflicts | solved by SS2PL | risk |
| Timestamp | killed and restarted | by timestamps | buffer writes / wait cb | less probable |

### Recoverability

Serializability ignores rollbacks; with rollbacks a dirty read can let a committed transaction depend on an aborted one. Four nested classes constrain this:

- **Recoverable** — no transaction commits before every transaction it read from commits. (Otherwise a commit could depend on a later-aborted write.)
- **Avoids cascading rollback (ACR)** — every transaction reads only values written by *already-committed* transactions; blocks the dirty-read anomaly, so an abort never cascades.
- **Strict** — additionally, a value is overwritten only after the previous writer has committed/aborted (so rollback simply restores the before-image).
- **Rigorous** — for every pair of conflicting actions, the first transaction's commit precedes the second's conflicting action.

```
serial ⊂ rigorous ⊂ strict ⊂ ACR ⊂ recoverable
```

Recoverability is orthogonal to serializability: a schedule can be one without the other; every serial schedule is both.

### Concurrency control in SQL

An SQL session that does not open an explicit transaction treats each statement as a transaction. The standard names isolation levels by the anomalies they forbid (the practitioner view, with engine-specific nuance, is in `Data Systems.md`):

| Isolation level | Dirty read | Nonrepeatable read | Phantom read |
|---|---|---|---|
| Read uncommitted | possible | possible | possible |
| Read committed | prevented | possible | possible |
| Repeatable read | prevented | prevented | possible |
| Serializable | prevented | prevented | prevented |

A **phantom read** occurs when a query run twice in one transaction returns a different *set of rows* because a concurrent transaction inserted or deleted rows matching the query's predicate — distinct from a nonrepeatable read, where an already-read row's *values* change.

The standard fixes the *guarantees*, not the *mechanism* — an implementation may use locking, timestamps, or MVCC (multi-version concurrency control). Preventing phantoms specifically requires predicate or gap locking (see `Data Systems.md`).

---

## 8. Recovery

While the transaction manager handles isolation and consistency, the **recovery manager** handles atomicity and durability: beginning, committing, and rolling back transactions, and restoring a correct state after a failure. Failures are either **system failures** (crash, error, abort) or **storage-media failures** (disk loss, catastrophe).

```
        Fail                 Boot
 Normal ----> Stop  Stop ----------> Restart
   ^                                    |
   +--------- Restart completed --------+
```

### The log

The **log file** records all transactions' actions in chronological order on **stable storage** (failure-resistant, replicated). It is a sequential, failure-free file supporting append, forward scan, and backward scan. Two record kinds:

- **Transaction records** — for element `O` with **before-state BS** and **after-state AS**: `B(T)` begin, `I(T,O,AS)` insert, `D(T,O,BS)` delete, `U(T,O,BS,AS)` update, `C(T)` commit, `A(T)` abort.
- **System records** — `checkpoint` and `dump`.

A transaction's outcome is fixed when its `C(T)` (written synchronously, by force) or `A(T)` appears in the log. After a failure, **undo** is applied to uncommitted transactions (atomicity) and **redo** to committed ones whose effects may not have reached disk (durability).

- **Checkpoint** — periodically records the set of currently active transactions, flushing pages of transactions committed since the last checkpoint. It bounds how far back redo must reach.
- **Dump** — an offline full backup written to stable storage, marked by a dump record in the log.

### The two write rules

- **WAL (write-ahead log)** — a record's log entry reaches the log *before* the corresponding data page reaches disk. This makes **undo** possible: the before-image (BS) is always available to roll back an uncommitted write.
- **Commit-precedence** — all of a transaction's log records reach the log *before* its commit. This makes **redo** possible: the after-image (AS) is available to redo a committed transaction whose pages had not yet been written.

### When data reaches disk

Consistent with WAL and commit-precedence, three strategies:

- **Immediate** — data pages are written as soon as their log records are, before commit; **redo never needed** (everything committed is already on disk), but undo is.
- **Delayed** — data pages are written only after commit; **undo never needed**, but redo is.
- **Mixed** — either may happen depending on the buffer manager; **both undo and redo** are needed. This is the realistic case (it corresponds to steal + no-force, section 2).

### Restart

- **Warm restart** (system failure), assuming mixed effect:
  1. Scan **backward** to the most recent checkpoint record.
  2. Set `undo = {active transactions at the checkpoint}`, `redo = {}`.
  3. Scan **forward**: add transactions with a begin to `undo`; move those with a commit to `redo`.
  4. **Undo phase** — scan backward, undoing `undo` transactions, back to the begin of the oldest active transaction.
  5. **Redo phase** — scan forward, redoing `redo` transactions.
- **Cold restart** (disk failure):
  1. Load the most recent **dump**.
  2. Reapply the log forward from the dump.
  3. Run the warm-restart procedure.

### ARIES

ARIES (Algorithms for Recovery and Isolation Exploiting Semantics) is the standard steal + no-force recovery method, built on three principles: WAL, **repeat history** (on restart, reconstruct the exact pre-crash page state, including the effects of transactions that will then be undone), and **log the undos** so undo work is itself recoverable and idempotent.

Every log record carries a monotonically increasing **LSN** (log sequence number). Each page stores the **pageLSN** of the last log record that updated it; comparing pageLSN against a record's LSN tells recovery whether that update is already on the page (idempotent redo). Update records are chained per transaction by a **prevLSN** back-pointer.

Two in-memory tables drive recovery and are snapshotted by a **fuzzy checkpoint** (which does not flush data pages):

- **Transaction table** — one entry per active transaction, with its lastLSN.
- **Dirty page table (DPT)** — one entry per dirty buffer page, with its **recLSN** (the LSN of the oldest update not yet on disk), which bounds where redo must begin.

Restart runs three passes over the log:

1. **Analysis** — forward from the last checkpoint; rebuild the transaction table and DPT, and identify **losers** (transactions live at the crash).
2. **Redo** (repeat history) — forward from the smallest recLSN in the DPT; reapply *every* logged update (losers included) whose effect is not already on the page (pageLSN < record LSN), restoring the exact pre-crash state.
3. **Undo** — backward; roll back the losers. Each undo writes a **compensation log record (CLR)** describing the inverse action; a CLR's **UndoNextLSN** points past the action just undone, so a crash *during* recovery never re-undoes completed work.

Redo-before-undo (vs. the undo-then-redo warm restart above) lets a single forward pass repeat history uniformly, and CLRs make undo restartable.

---

## 9. Query Execution and Optimization

```
SQL query -> parse -> logical plan -> physical plan -> execute -> result
              |          (algebra,      (algorithms,
           (parse +     equivalence       cost-chosen)
            convert)     rewrites)
```

### 9.1 Iterators: materialization vs pipelining

An operator is implemented either by **materialization** (compute the whole result into a temporary relation) or by **pipelining** through an **iterator** with three methods:

- **Open** — initialize state and prepare to produce tuples;
- **GetNext** — return the next result tuple (and a `Found` flag);
- **Close** — finish after all tuples are produced.

Pipelining passes tuples directly from producer to consumer without staging them on disk. Unary operators (selection, projection) pipeline well; binary operators depend on buffer availability.

### 9.2 The operator cost model

Counts page accesses, with `B(R)` the pages of `R`, `M` the available frames. Algorithm families:

- **One-pass** — read each operand once; works when an operand fits in memory.
- **Nested-loop** — a loop within a loop; for join, a "one-and-a-half pass" (one relation read once, the other repeatedly).
- **Two-pass** — read, process, write intermediate results, read again; for data exceeding memory. Based on **sorting** or **hashing**.
- **Multi-pass** — recursive generalization of two-pass.
- **Index-based** — exploit an index.
- **Parallel** — across a shared-nothing cluster.

### 9.3 One-pass and nested-loop

One-pass unary operators (selection, projection, duplicate elimination, grouping) cost `B` and need a small data structure (hash table or tree) in memory; one-pass binary operators need the smaller operand to fit (`B(S) <= M - 2`) and cost `B(R) + B(S)`.

Nested-loop **join** of `R(X,Y)` and `S(Y,Z)`:

- **Tuple-based** — for each tuple of the outer, scan the inner. Cost `B(S) + p_S·B(S)·B(R)` (`p_S` = records per page of S) — prohibitive.
- **Page-based** — for each *page* of the outer, scan the inner. Cost `B(S) + B(S)·B(R)`.
- **Block-based** — load a block of `M - 2` outer pages at a time into a hash/tree (reserving one frame for the inner scan and one for output), scan the inner once per block. Cost `B(S) + B(R)·⌈B(S)/(M-2)⌉`. The smaller relation should be the outer (**build**) relation; the other is the **probe** relation.

### 9.4 Two-pass: sorting and hashing

- **Sort-based** — pass 1 builds sorted sub-lists (`M` pages each); pass 2 merges them while performing the operator. Unary operators need `B <= (M-1)^2` and cost `3B`. The **sort-merge join** costs `3(B(R) + B(S))` with `√(B(R)+B(S)) + 3` frames and yields sorted output.
- **Hash-based** — partition each operand into `M-1` buckets with a hash function, then process bucket by bucket with the one-pass algorithm. Unary cost `3B`; **hash join** costs `3(B(R) + B(S))`, needing `√min(B(R),B(S)) + 2` frames. The **hybrid hash join** keeps some buckets of the smaller relation resident, joining matching probe tuples immediately and spilling the rest, reducing I/O toward `(3 - 2M/B(S))(B(R)+B(S))`.

Sort-based binary operators need buffer on the *sum* of arguments and produce sorted output; hash-based need buffer on the *smaller* argument but assume even bucket sizes. Multi-pass variants extend both when even two passes do not suffice, costing `(2K-1)·B` for `K` passes.

### 9.5 Index-based algorithms

An index **conforms** to a condition it can evaluate: a tree index on `attr` conforms to `<, <=, =, >=, >`; a hash index conforms only to `=`. For a composite key, a tree index conforms to a conjunction matching a **prefix** of the key.

- **Index selection** — for `attr op value`, use a conforming index. On a primary/hash index, equality costs ~1–2 accesses; on a clustered tree index, `log_F(1.5B) + B(R)/V(R,A)`; on an unclustered index, up to one access per qualifying record (improved by sorting rids by page first).
- **Index join (index-nested-loop)** — for each tuple of `R`, probe an index on `S.Y` to find matches; cost `B(R) + T(R)·(cost per index lookup)`. If one index is sorted, a **zig-zag join** walks both indexes in step, accessing data files only at the end.

### 9.6 Parallel algorithms

On a **shared-nothing** cluster a relation `R` is split into `P` chunks at `P` nodes. Partitioning strategies: **round-robin** (even balance, must scan all), **range** (good for range predicates, risks skew), **hash** (good balance, equality/full-scan only). Selection/projection run locally with elapsed time `B(R)/P`. For binary operators and grouping, hash-partitioning on the relevant attributes co-locates matching tuples at one node, so each node runs a standard local algorithm in parallel.

### 9.7 Query parsing and the logical plan

**Parsing** has two phases: *parse* (build a parse tree, substitute views, validate against the schema, type-check) and *convert* (turn the parse tree into an extended relational-algebra expression tree, distinguishing set/bag operators, `δ` for duplicate elimination, `γ` for grouping, and sorting for ORDER BY). A query `select A.. from R.. where C` converts to `π_A(σ_C(R1 × ... × Rm))`.

The **logical plan** is improved by equivalence-preserving rewrites:

- Commutativity/associativity of join, product, union, intersection.
- **Selection pushdown** — `σ_p(R ⋈ S) = (σ_p R) ⋈ S` when `p` mentions only `R`; split conjunctions `σ_{p1∧p2} = σ_{p1}(σ_{p2})`.
- **Projection pushdown** — `π_X(π_Y(R)) = π_X(R)`; push projections down, but not so far as to make an index useless.
- Combine selection with product to form an equi-join.

Heuristic guidelines: push selections below projections and products, group selections, eliminate useless projections, relocate duplicate eliminations, turn selection+product into equi-join, and group commutative/associative operators into multi-way nodes.

### 9.8 The physical plan and statistics

Logical rewrites are statistics-independent; the **physical plan** is chosen against estimated result sizes, using statistics per relation:

- `T(R)` — tuples; `S(R)` — bytes per tuple; `B(R)` — pages; `V(R,A)` — distinct values of attribute `A`.

**Size estimation**:

- Equality selection `σ_{A=v}`: `T(R)/V(R,A)`.
- Inequality `σ_{A>=v}`: `≈ T(R)/3` (lazy) or `(Max(A)-v+1)/(Max(A)-Min(A)+1)·T(R)`.
- Natural/equi-join on `Y`: `T(R1)·T(R2)/max(V(R1,Y), V(R2,Y))`.
- Set union ≈ larger + half the smaller; intersection ≈ half the smaller; difference ≈ `T(R) - T(S)/2`.

**Choosing the plan.** The number of physical plans is enormous, so the optimizer applies heuristics: use a conforming index for `A op value`; prefer an index-nested-loop join when an argument is indexed on the join key; prefer a sort-based join when an argument is already sorted; group smaller relations first in multi-way unions/intersections.

Join **order** matters because join algorithms are asymmetric (build vs probe). The smaller estimated relation should be the left/build argument. For multi-way joins a **greedy left-deep** algorithm keeps intermediate results small: start from the pair with the smallest estimated join size, then repeatedly add the relation yielding the smallest next intermediate.

Finally the optimizer decides which intermediate results to **materialize** versus **pipeline**, and emits a physical tree whose internal nodes name the chosen algorithm, whose materialized edges are marked, and whose leaves are **scan operators**: `TableScan(R)`, `SortScan(R, L)`, `IndexScan(R, A, C)`, or an index-only `IndexScan(R, A)`.
