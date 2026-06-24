# The Good Software Engineer Handbook

A comprehensive repository of software engineering knowledge covering system design, architectural patterns, distributed systems, security, networking, low-level systems, programming languages, and engineering best practices, derived from an M.Sc. in Computer Engineering and over 5 years of industry experience.

## Contents

### Languages
- [C](languages/C.md) — types, pointers, dynamic memory, structs/unions, preprocessor, compilation units, function pointers, bitwise ops, pthreads, build tools
- [Rust](languages/Rust.md) — ownership/borrowing, lifetimes, traits, generics, Option/Result, iterators, smart pointers, concurrency, async, unsafe, macros, Cargo
- [Golang](languages/Golang.md) — types, methods/interfaces, goroutines, channels, select, context, error handling, generics, embedding, modules, testing, toolchain

### Foundations
- [Operating Systems](foundations/Operating%20Systems.md) — processes, threads, virtual memory, syscalls, IPC, signals, compilation, linking
- [Concurrency](foundations/Concurrency.md) — memory models, locks, lock-free, atomics, shared-memory vs CSP vs actors vs async
- [Performance Optimization](foundations/Performance%20Optimization.md) — methodology, profiling, memory hierarchy, data layout, CPU/SIMD, I/O, compiler flags, tail latency
- [Algorithms and Data Structures](foundations/Algorithms%20and%20Data%20Structures.md) — complexity, core data structures, sorting, searching, graphs, dynamic programming

### Data and Databases
- [Data Systems](databases/Data%20Systems.md) — storage engines, indexes, transactions, replication, partitioning, caching, queues, streams, object storage
- [Database Design](databases/Database%20Design.md) — relational model, integrity constraints, relational algebra, ER modeling, logical design, normalization
- [Database Internals](databases/Database%20Internals.md) — buffer management, page/record layout, file organizations, indexing, concurrency control, recovery, query processing
- [SQL](databases/SQL.md) — DDL, constraints, DML, queries, joins, aggregation, subqueries, CTEs, window functions, views, transactions

### Distributed Systems and Architecture
- [Distributed Systems](architecture/Distributed%20Systems.md) — failure models, consistency, CAP, consensus, quorums, replication, exactly-once, coordination, CRDTs
- [System Design](architecture/System%20Design.md) — requirements, estimation, service shapes, event-driven patterns, scaling, load balancing, SLOs, building blocks

### Linux and Networking
- [Networking Fundamentals](networking/Networking%20Fundamentals.md) — OSI/TCP-IP, protocols, system-design networking
- [Networking linux-commands](networking/Networking%20linux-commands.md) — commands for inspecting and debugging the network

### Security
- [Cryptography](security/Cryptography.md) — primitives: ciphers, hashes, MACs, signatures, key exchange, TLS

### Operations
- [Observability](operations/Observability.md) — metrics, logs, traces, SLO/error budgets, USE/RED, alerting, dashboards, profiling in prod, SRE

---

*Disclaimer:* Portions of these notes were drafted with the assistance of AI tooling. If you find an error, please open an issue.
