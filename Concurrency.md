# Concurrency

A quick-reference guide covering concurrency from the programmer's perspective: memory models, locks and lock-free structures, atomics, the canonical concurrency models (shared-memory, CSP, actors, async), and the principal hazards (races, deadlocks, livelocks).

---

## Table of Contents

0. Orientation — Layers and Languages *(reference tables)*
1. Concurrency vs Parallelism
2. The Hardware View — Caches and Memory Models
3. Atomics
4. Locks
5. Higher-Level Synchronization
6. Lock-Free and Wait-Free
7. The Shared-Memory Model
8. CSP (Communicating Sequential Processes)
9. Actor Model
10. Async / Event Loop
11. Concurrency Hazards
12. Patterns
13. Anti-Patterns and Pitfalls

---

## 0. Orientation — Layers and Languages

Two questions account for most of the confusion in the concurrency literature: *which layer of the stack provides a given guarantee*, and *how a particular language exposes it*. The two tables below serve as a reference; the sections that follow elaborate on each entry. Consult them whenever a primitive's ownership or a language's idiom is unclear.

### Who provides what — the concurrency stack

Every concurrency primitive is assembled from four layers. The governing principle: **uncontended fast paths remain in user space (hardware instructions alone); execution enters the kernel only when a thread must block or be woken.** This explains why atomics require no syscall whereas a contended mutex does.

| Layer | What it owns | Examples |
|-------|--------------|----------|
| **Hardware (CPU)** | Atomic instructions, cache coherence, and the per-ISA memory-ordering rules. No software can subdivide these. | `CMPXCHG`, `LOCK XADD` (x86); `LDXR`/`STXR` load-linked/store-conditional (ARM); MESI coherence; store buffers |
| **Kernel / OS** | Scheduling threads onto cores, and *blocking and waking* them. Only reached on contention or real I/O waits. | futex (Linux), `epoll`/`kqueue`/`io_uring`, context switches, `clone()` behind thread creation, priority inheritance |
| **Runtime / standard library** | Ergonomic wrappers over hardware+kernel, plus the machinery of the language's concurrency model. | pthreads, `std::mutex`, the Go scheduler + goroutines + channels, tokio's executor, the JVM, Python's GIL |
| **Your code** | The *protocol*: what state is shared, who may access it, and in what order. The lower layers supply the tools; correctness remains the responsibility of this layer. | CAS retry loops, chosen lock order, channels-vs-mutex decisions, ABA mitigations |

A primitive typically spans several layers. Consider a mutex: the uncontended lock and unlock are a hardware atomic (no kernel involvement); under contention the runtime invokes a futex so that the kernel can park the waiting thread; and *which* data the mutex protects is determined by the application's contract.

### How each language approaches concurrency

The hardware is identical across all of them; languages differ in the model they make idiomatic, whether they permit true parallelism, and the degree to which they guard against programmer error.

| Language | Default model | Real parallelism? | Sync style | Safety net |
|----------|---------------|-------------------|------------|------------|
| **C / C++** | Shared-memory threads | Yes | Manual locks; C11/C++ atomics | None — discipline + ThreadSanitizer |
| **Rust** | Shared-memory threads; async | Yes | `Mutex<T>` owns its data; atomics; channels | Compile-time: `Send`/`Sync` + borrow checker make data races *impossible* in safe code |
| **Go** | CSP (goroutines + channels); shared memory available | Yes (M:N scheduler) | Channels preferred; `sync.Mutex` available | Runtime race detector (`-race`); no compile-time prevention |
| **Java / JVM** | Shared-memory threads + `java.util.concurrent` | Yes | `synchronized`, locks, atomics, concurrent collections | JMM + tooling; no compile-time race prevention |
| **JavaScript / Node** | Single-threaded event loop (async) | No, within a loop (Workers are separate) | `async`/`await`; no shared memory | No data races on JS objects (one thread) |
| **Python** | Threads (GIL-limited) + asyncio | No for threads (use `multiprocessing`) | `Lock`, asyncio, queues | GIL serializes bytecode; race *conditions* remain |

Sections 7–10 expand on each model (shared-memory, CSP, async); §11's language note revisits the categories of race that remain possible in each language.

---

## 1. Concurrency vs Parallelism

These terms are often conflated. They are not the same.

```
Concurrency:  multiple tasks in progress at the same time, possibly interleaved
              on a single CPU. About structure -- expressing a program as
              independent activities that can be scheduled.

Parallelism:  multiple tasks executing simultaneously on multiple CPUs.
              About execution -- actually doing work in parallel.
```

A web server handling 10,000 requests is concurrent. On one core, only one runs at a time, scheduled in turn. On 16 cores, up to 16 run in true parallel.

You can have concurrency without parallelism (single-threaded async I/O) and parallelism without concurrency (a single batch job split into independent CPU tasks).

```
Concurrency        Parallelism      Both
+----------+       +----------+     +----------+
| task A   |       | task A   |     | task A   |
+----------+       +----------+     +----------+
| task B   |       | task B   |     | task B   |
+----------+       +----------+     +----------+
                                     
single CPU,        multi CPU,        multi CPU,
interleaved        no interleaving   interleaved within each
```

Good concurrent design enables parallelism but is valuable even on one core (responsiveness, I/O hiding, abstraction).

---

## 2. The Hardware View — Caches and Memory Models

Multiple CPUs sharing memory is more subtle than it appears. The hardware introduces caches, write buffers, and reordering; software must be designed for the hardware's actual behavior.

### Cache Hierarchy

```
CPU core
   |
   +-- L1 cache (per-core, ~32 KB, 1-4 cycles)
   |
   +-- L2 cache (per-core, ~256 KB-1 MB, ~10 cycles)
   |
   +-- L3 cache (shared across cores in a socket, ~8-64 MB, ~30 cycles)
   |
Main memory (DDR, ~100 ns = ~300 cycles)
```

Each level is faster and smaller. Reads and writes touch the cache first; the cache fills from lower levels on miss.

### Cache Coherence

When multiple cores have the same memory in their private caches, what happens when one writes? Hardware **cache coherence protocols** (MESI, MOESI) ensure each core eventually sees the new value, by invalidating or updating other cores' copies.

```
Core 0 has X=10 in L1.
Core 1 has X=10 in L1.

Core 0 writes X=11.
  -> Core 0 invalidates Core 1's copy (sends invalidation message).
  -> Core 1's next read of X gets a cache miss, fetches X=11.
```

Coherence makes shared memory work but isn't free. Cache lines (typically 64 bytes) are the unit of coherence — operations on different variables that happen to live in the same line cause **false sharing**, which destroys performance.

```
struct Counter {
    int alice_counter;   // bytes 0-3
    int bob_counter;     // bytes 4-7    same cache line as alice's!
};

Thread A: increments alice_counter constantly.
Thread B: increments bob_counter constantly.

Even though they touch different fields, every increment invalidates
the other core's cache line. Performance collapses.

Fix: pad to cache-line boundaries.
struct Counter {
    int alice_counter;
    char pad[60];        // pad to 64 bytes
    int bob_counter;
};
```

The cleaner modern idiom is to align the entire struct: `struct _Alignas(64) Counter { atomic_long count; };` (C++: `alignas(64)`). False sharing reappears in §11 as a named hazard, but the mechanism and the fix are documented here.

### Memory Reordering

Modern CPUs do not execute memory operations in source-code order. They reorder for performance — as long as a single thread sees a consistent view, the hardware is free to:

- Issue writes to cache in a different order than the program wrote.
- Buffer writes (store buffer) and complete later.
- Issue reads ahead of preceding writes if the addresses don't alias.

The compiler also reorders for optimization. Result: **another thread can observe writes in an order different from what the writer "did."**

```
Thread A:
  x = 1;
  flag = 1;

Thread B (sees flag == 1):
  read x;        // can this see x == 0?
```

Yes: on x86 this reordering is usually prevented, but on ARM and POWER it is permitted unless memory barriers or atomics are used.

### Memory Models

A memory model specifies what reorderings are allowed and how to enforce ordering when needed.

| Architecture | Model |
|--------------|-------|
| x86 / x86_64 | Total Store Order (TSO) — mostly sequential, with some store buffering |
| ARM, ARM64, POWER | Weak — extensive reordering |
| RISC-V | Weak, with extensions |

Language memory models abstract over hardware:

| Language | Memory model |
|----------|--------------|
| Java | JMM — happens-before, volatile, locks |
| C11, C++11+ | Atomics with ordering parameters (seq_cst, acquire, release, relaxed) |
| Rust | Same as C++ atomics, plus stronger compile-time safety |
| Go | Happens-before via channels, sync.Mutex, sync/atomic |

The language memory model is the contract the program is written against; the compiler and hardware cooperate to implement it correctly.

### Happens-Before

The foundational relation for reasoning about concurrency: if A happens-before B, all of A's effects are visible to B.

Established by:

- Sequential program order within a single thread.
- Lock release happens-before subsequent lock acquire.
- Volatile/atomic write happens-before subsequent read (with appropriate ordering).
- Thread.start() happens-before any action in the started thread.
- Send on a channel happens-before corresponding receive.

If two memory accesses are not related by happens-before, and at least one is a write, they constitute a **data race**, which is undefined behavior in most languages.

---

## 3. Atomics

An atomic operation is one that either completes entirely or not at all from the perspective of other threads. No partial state is observable.

### Where atomics live in the stack

Atomics are the clearest illustration of the layer map from §0: **the OS is not involved at all** — there are no syscalls, no blocking, and no scheduler. An atomic is simply a hardware instruction exposed under a callable name:

| Layer | For atomics specifically |
|-------|--------------------------|
| **CPU / hardware** | The atomic instruction itself — `LOCK XADD`, `CMPXCHG`, ARM `LDXR`/`STXR`. One instruction, uninterruptible by other cores. |
| **Compiler + standard library** | The callable names — `atomic_fetch_add`, `atomic_load`, `atomic_compare_exchange_weak`. Each compiles to *one* instruction (not a function call, not a syscall). The compiler also emits fences and suppresses reordering to honor your `memory_order_*`. |
| **Your code** | Multi-step logic on top: the CAS retry loop, ABA mitigations, the lock-free structure. |

A practical reading rule for this section: anything named `atomic_*` is a **provided primitive** that compiles to a CPU instruction; everything surrounding it — the loop, the variables, the recomputation — is **application code**. No runtime machinery sits in between.

### Atomic Reads and Writes

For machine-word-sized variables (typically int32, int64, pointer), the CPU can read and write atomically *in a single instruction* — a hardware guarantee, not something the programmer implements. Atomicity of an individual access is not sufficient, however: the *ordering* of accesses relative to one another must also be controlled, which is the role of the `memory_order_*` arguments below, enforced by the compiler.

```c
#include <stdatomic.h>

// C11 atomic, sequentially-consistent (strongest):
atomic_int counter = 0;
atomic_fetch_add(&counter, 1);                   // atomic increment
int value = atomic_load(&counter);               // atomic read

// Relaxed (no ordering, just atomicity):
atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);

// Acquire-release pair:
atomic_store_explicit(&data, 42, memory_order_release);
// some other thread:
int x = atomic_load_explicit(&data, memory_order_acquire);
```

### Compare-and-Swap (CAS)

The fundamental atomic primitive for lock-free programming. CAS answers one question atomically: *"is this location still the value I last saw? If so, replace it."* This lets a thread make an update conditional on nobody else having touched the location in the meantime — without holding a lock.

It takes three arguments:
- `addr` — the location to update
- `expected` — the value presumed to be present (passed by pointer)
- `new` — the value to store if the comparison holds

This is a **provided primitive** (one CPU instruction, e.g. x86 `CMPXCHG`). The block below is not implemented by the programmer; it describes what the hardware performs atomically when the primitive is called:

```
atomic_compare_exchange(addr, expected, new):   // hardware performs all of this as one step
  if *addr == expected:        // location unchanged since it was read
    *addr = new                // commit the update
    return true
  else:
    *expected = *addr          // report the value currently present
    return false               // another thread won the race; the update did not occur
```

The compare and the swap happen as one indivisible step — no other thread can slip in between them. This is the building block under nearly every lock-free data structure.

Because a CAS can fail (another thread won the race), the standard pattern is a **retry loop**: read the current value, compute the new value from it, and CAS. On failure, `expected` is refreshed with the value actually present, and the computation is repeated.

Here `atomic_load` and `atomic_compare_exchange_weak` are **provided primitives** (each one CPU instruction); the loop, the variables, and the recomputation are **application code**:

```c
// Lock-free counter (illustrative -- atomic_fetch_add does this in one instruction):
atomic_int counter = 0;

void increment(void) {
    int old = atomic_load(&counter);          // primitive: read current value
    int desired = old + 1;                    // application code: compute the update
    // application code: loop until the primitive (CAS) commits.
    // On failure, compare_exchange_weak overwrites `old` with the
    // observed value, and the loop recomputes and retries.
    while (!atomic_compare_exchange_weak(&counter, &old, old + 1)) {  // primitive
        // old holds the current value here; retry with the fresh reading.
    }
}
```

**weak vs strong:** `compare_exchange_weak` may fail *spuriously* (return false even when `*addr == expected`), so it is only safe inside a retry loop, but it compiles to cheaper code on some architectures (e.g. ARM's load-linked/store-conditional). Use `compare_exchange_strong` when not looping, where a false negative would be a bug.

**ABA problem:** CAS only checks that the value is equal, not that it never changed. A thread reads value A and prepares an update based on A. Meanwhile another thread changes A → B → A. The CAS still succeeds because the bits match, even though the underlying state churned in between — which corrupts structures like lock-free stacks where the "A" is a reused pointer to a freed-and-reallocated node. Mitigations: version/tag counters (bump a counter on every change so A-with-tag-1 ≠ A-with-tag-2), hazard pointers, and epoch-based reclamation.

### Memory Ordering Levels (C11 / C++)

| Order | Meaning |
|-------|---------|
| relaxed | Atomic but no ordering relative to other operations |
| consume | Ordered wrt operations that depend on the loaded value (rarely useful) |
| acquire | All subsequent reads/writes are ordered after this load |
| release | All previous reads/writes are ordered before this store |
| acq_rel | Acquire + release (read-modify-write) |
| seq_cst | Sequential consistency — there exists a total order all threads agree on |

**Default rule of thumb:** begin with `seq_cst` (the safest, and the default in most languages). Drop to weaker orderings only when profiling shows a benefit and the weaker ordering has been proven correct.

### Common Atomic Operations

```c
atomic_load, atomic_store
atomic_fetch_add, atomic_fetch_sub, atomic_fetch_and, atomic_fetch_or, atomic_fetch_xor
atomic_exchange                  (swap)
atomic_compare_exchange_weak     (may spuriously fail; cheaper -- use in a retry loop)
atomic_compare_exchange_strong   (never spuriously fails)
```

### Language Note: Atomics Beyond C

C exposes atomics as free functions on plain types. Higher-level languages wrap the same hardware primitives differently:

| Language | API | Notes |
|----------|-----|-------|
| C11 | `<stdatomic.h>`, `atomic_int`, `atomic_fetch_add` | free functions; ordering via `_explicit` variants |
| C++ | `std::atomic<T>`, `.fetch_add()`, `.load(order)` | member functions; `T` can be any trivially-copyable type |
| Rust | `AtomicUsize`, `.fetch_add(v, Ordering)` | ordering is a required argument — no accidental default to `seq_cst` |
| Go | `sync/atomic` (`atomic.AddInt64`) | seq_cst only; no relaxed/acquire/release exposed |
| Python | — | the GIL serializes bytecode, so simple ops look atomic, but `+=` is *not* (load-add-store across bytecodes); use `threading.Lock` |

The hardware operation is identical; what differs is the type system and the degree of ordering control the language exposes. Rust requires an explicit ordering at every call; Go hides ordering entirely; Python exposes no user-level atomics.

---

## 4. Locks

A lock (mutex) ensures only one thread executes a **critical section** at a time.

### Mutex

The basic exclusive lock.

```c
#include <pthread.h>

pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&m);
counter++;              // critical section
pthread_mutex_unlock(&m);
```

The kernel-level primitive backing most mutexes is a **futex** on Linux: fast userspace uncontended path, fall back to kernel only when contention happens. This is a concrete instance of the §0 layer map — the uncontended lock is a hardware atomic requiring no syscall; only a *contended* lock reaches the kernel to park the waiting thread.

### Language Note: Lock Release Across Languages

The C example above must call `pthread_mutex_unlock` explicitly. Any early `return`, `break`, or `goto` between lock and unlock leaks the lock — a common, hard-to-detect bug. Higher-level languages tie unlock to scope or to the data itself:

```
C:       lock; ... ; unlock;          // manual release required on every path
C++:     std::lock_guard g(m);        // RAII -- unlocks when g leaves scope, even on exception
Go:      mu.Lock(); defer mu.Unlock() // defer runs on function exit, all paths
Python:  with lock:                   // context manager releases on block exit
Rust:    let g = m.lock().unwrap();   // guard unlocks on drop -- AND m wraps the data:
                                       // the data is unreachable without locking
```

Rust is the strongest: `Mutex<T>` *owns* the protected data, so "accessing shared data without holding the lock" is a compile error, not a discipline. C gives the most control and the least protection.

### Read-Write Lock (RWLock)

Multiple readers, one writer.

```c
pthread_rwlock_t rw = PTHREAD_RWLOCK_INITIALIZER;

// Reader:
pthread_rwlock_rdlock(&rw);
read_data();
pthread_rwlock_unlock(&rw);

// Writer:
pthread_rwlock_wrlock(&rw);
write_data();
pthread_rwlock_unlock(&rw);
```

**When to use:** read-heavy workloads with rare writes. The bookkeeping overhead means an RWLock is **slower than a plain mutex when writes dominate or critical sections are very short.** Benchmark before assuming RWLock is faster.

### Reentrant (Recursive) Lock

Same thread can acquire the lock multiple times without deadlocking. Each acquire must have a matching release.

```c
pthread_mutex_t lock;
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
pthread_mutex_init(&lock, &attr);

void inner(void) {
    pthread_mutex_lock(&lock);   // safe to acquire again
    // ...
    pthread_mutex_unlock(&lock);
}

void outer(void) {
    pthread_mutex_lock(&lock);
    inner();                     // recursive acquire does not deadlock
    pthread_mutex_unlock(&lock);
}
```

**Generally avoid.** Reentrant locks complicate reasoning and indicate API design problems (the locking responsibility is unclear). Prefer non-reentrant locks; restructure code to acquire once.

### Spinlock

Busy-waits in user-space without going to the kernel. Useful only when:

- The critical section is extremely short (handful of instructions).
- The contention is low.
- You're on multiple cores (single-core spin = useless wait).

```c
#include <stdatomic.h>
#include <immintrin.h>   // _mm_pause on x86

atomic_flag locked = ATOMIC_FLAG_INIT;

// Acquire: spin until we flip the flag from clear to set.
while (atomic_flag_test_and_set_explicit(&locked, memory_order_acquire)) {
    _mm_pause();         // PAUSE: hint to the CPU that we're spin-waiting
}
// ... critical section ...
atomic_flag_clear_explicit(&locked, memory_order_release);
```

Kernel mutexes often start with a brief spin (adaptive spinning) before going to sleep — capturing both worlds.

### Lock Granularity

Coarse-grained locking (one lock for everything) is simple but limits parallelism. Fine-grained locking (one lock per resource) enables parallelism but is harder to reason about.

```
Coarse:  global cache_lock for the entire cache
         -> only one thread touches the cache at a time
         -> simple but slow under contention

Fine:    one lock per cache shard (e.g., 16 shards by key hash)
         -> 16x as many concurrent updates possible
         -> harder to do operations spanning shards
```

### Lock Hierarchy / Lock Ordering

Deadlock prevention: define a strict ordering of locks and always acquire in that order. When multiple locks are used together, every thread must acquire them in the same order.

```
Locks: A, B, C  (in alphabetical order)

OK:
  acquire A
  acquire B
  acquire C
  ...
  release C, B, A

NOT OK (different code path):
  acquire B
  acquire A          <-- could deadlock with the above
```

Static analysis tools (clang ThreadSanitizer, Java's lock detector) can catch some violations.

---

## 5. Higher-Level Synchronization

These primitives sit at the runtime/standard-library layer of §0: each is built from locks and atomics and encodes a common coordination pattern that is error-prone to assemble by hand from a bare mutex.

### Condition Variables

A condition variable lets a thread sleep until another thread signals that some shared predicate may have changed. The alternative — repeatedly testing the predicate in a loop while holding the lock — burns CPU and cannot release the lock while waiting, so no other thread could ever make the predicate true.

A condition variable is **always paired with a mutex** that protects the predicate. `pthread_cond_wait(&cv, &m)` performs three steps atomically: it releases `m`, blocks the thread, and re-acquires `m` before returning. That atomicity is the whole point — it closes the window between testing the predicate and going to sleep. Without it, a signal sent in that window would be missed (the **lost-wakeup** problem), and the thread would sleep forever.

```c
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t  cv = PTHREAD_COND_INITIALIZER;
queue_t q;   // some shared queue

// Producer:
pthread_mutex_lock(&m);
queue_push(&q, 42);
pthread_mutex_unlock(&m);
pthread_cond_signal(&cv);

// Consumer:
pthread_mutex_lock(&m);
while (queue_empty(&q))           // loop, not if -- see below
    pthread_cond_wait(&cv, &m);   // releases m while blocked, re-acquires on wake
int x = queue_pop(&q);
pthread_mutex_unlock(&m);
```

**Always wait in a loop** (re-check the predicate after waking), for two reasons: spurious wakeups are possible — `pthread_cond_wait` may return without a matching signal — and even after a genuine signal, another thread may have consumed the condition before the woken thread re-acquires the mutex.

`signal` wakes one waiter; `broadcast` wakes all of them. Use `broadcast` when a single change may satisfy several waiters, or when waiters on the same condition variable test different predicates.

### Semaphores

A counting semaphore holds an integer permit count. `sem_wait` (P) decrements it, blocking while the count is zero; `sem_post` (V) increments it and wakes a waiter. Initialized to N, it admits up to N concurrent holders — a binary semaphore (N = 1) degenerates to a lock.

```c
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 10);   // initial count 10; pshared=0 (threads, not processes)

sem_wait(&sem);   // decrements; blocks if zero
// critical section
sem_post(&sem);   // increments; wakes a waiter
```

Unlike a mutex, a semaphore has **no concept of ownership**: the thread that posts need not be the one that waited. This makes it suitable for *signaling* between threads (one produces a permit, another consumes it), not only for mutual exclusion. For plain mutual exclusion a mutex is preferable, because it tracks ownership and can support priority inheritance.

Used as: connection pool gates, bounded concurrency limiters, producer-consumer.

### Barriers

A barrier synchronizes a *phase*: N threads each block at the barrier until all N have arrived, then all proceed together. It is the core primitive of bulk-synchronous parallel algorithms, where no thread may begin phase k+1 until every thread has finished phase k.

```c
pthread_barrier_t sync_point;
pthread_barrier_init(&sync_point, NULL, 4);   // 4 participants

void *worker(void *arg) {
    do_phase_1();
    pthread_barrier_wait(&sync_point);   // wait for all 4 workers
    do_phase_2();
    return NULL;
}
```

Used in parallel algorithms (e.g., parallel sort with merge phases). `pthread_barrier_t` is **reusable**: once all participants pass, it resets automatically for the next round.

### Once / Lazy Initialization

Ensure some initialization happens exactly once, even when multiple threads reach it concurrently.

```c
pthread_once_t once = PTHREAD_ONCE_INIT;

void init_resource(void) {
    global_resource = expensive_init();
}

// Each thread calls this; the runtime guarantees init_resource runs exactly once.
pthread_once(&once, init_resource);
```

This replaces the error-prone hand-rolled double-checked-locking idiom (§13). The runtime guarantees both halves of correctness: the initializer runs exactly once, and a happens-before edge makes its result visible to every later caller — so no mutex is needed on subsequent accesses. Equivalents: C++ `std::call_once`, Go `sync.Once`, Rust `OnceCell`/`LazyLock`.

### Atomic Counters and Flags

For state that fits in a single machine word, an atomic provides coordination without a mutex: no syscall, no blocking, no risk of leaving a lock held. Suitable for counters, flags, and one-shot signals.

```c
atomic_bool shutdown = false;

// In threads:
while (!atomic_load_explicit(&shutdown, memory_order_acquire)) {
    do_work();
}

// In main:
atomic_store_explicit(&shutdown, true, memory_order_release);
```

The `release` store publishes all writes that preceded it; the matching `acquire` load guarantees the workers observe those writes once they see the flag set (§3). An atomic is not a substitute for a mutex when several values must change together — that requires either a lock or a careful lock-free protocol (§6).

---

## 6. Lock-Free and Wait-Free

A data structure is **lock-free** if at least one thread makes progress in finite time, regardless of other threads' states (no thread can be blocked indefinitely by another thread's blocking).

**Wait-free** is stronger: every thread completes in bounded steps regardless of others.

Lock-free structures are built from atomics and CAS, not locks. They avoid:

- Deadlock (no locks to acquire in wrong order).
- Priority inversion.
- Convoying (one slow thread blocking many).

They suffer from:

- Higher CPU cost per operation (CAS retries, memory barriers).
- Complexity (very easy to write buggy lock-free code).
- ABA problems (mentioned earlier).
- Memory reclamation hazards.

### When Lock-Free Pays Off

- High contention with short critical sections.
- Real-time systems where lock-induced jitter is unacceptable.
- Single-producer/single-consumer queues (often trivially lock-free).

### When Locks Win

- Most application code. The performance benefit of lock-free is overstated; the complexity cost is understated.
- Long critical sections (lock-free retries become expensive).
- Algorithms with multiple atomic variables that must change together (locks linearize naturally; lock-free requires careful protocol).

### Reading Material

Lock-free programming is its own field. Standard patterns:

- Lock-free stack (Treiber's stack).
- Lock-free queue (Michael-Scott).
- Hazard pointers for memory reclamation.
- Epoch-based reclamation (crossbeam in Rust).
- Read-Copy-Update (RCU, used heavily in the Linux kernel).

Prefer a battle-tested library to implementing these yourself — see the §13 anti-pattern "Reinventing concurrent data structures" for the relevant libraries and the reasoning.

---

## 7. The Shared-Memory Model

The default model in most languages: threads share an address space and communicate by reading and writing shared variables, with synchronization primitives controlling access.

```
Thread 1:                            Thread 2:
  lock(m);                             lock(m);
  shared_state.modify();               shared_state.modify();
  unlock(m);                           unlock(m);
```

**Pros:**

- Direct, low-overhead communication.
- Familiar mental model (same as single-threaded, plus locks).
- Maps well to hardware (caches, atomics).

**Cons:**

- Easy to introduce races.
- Hard to reason about as the lock count grows.
- Composition is hard (calling code with locks held into other locking code).

### Best Practices

- **Minimize shared state.** Each thread owns its state; communicate via messages where possible.
- **Encapsulate locks with the data they protect.** Don't expose the lock and require callers to use it.
- **Hold locks briefly.** Long critical sections kill throughput and increase deadlock risk.
- **Document lock ordering.** When multiple locks must be held, write down the order.
- **Avoid calling out while holding locks.** Calling external code with a lock held risks deadlock and reentrancy issues.

---

## 8. CSP (Communicating Sequential Processes)

Hoare's model (1978): threads don't share state; they communicate by sending messages through **channels**.

```
Thread A ----channel----> Thread B

A sends, B receives. No shared variables. No locks.
```

Go is the most prominent CSP-style language. Channels are first-class.

```go
ch := make(chan int)

go func() {
    ch <- 42        // send
}()

x := <-ch           // receive
```

### Channel Variants

- **Unbuffered:** send blocks until receive happens (rendezvous).
- **Buffered:** send succeeds while buffer has space; blocks when full.

```go
ch := make(chan int)      // unbuffered
ch := make(chan int, 100) // buffered, capacity 100
```

### Select

Wait on multiple channel operations.

```go
select {
case msg := <-incoming:
    process(msg)
case <-time.After(5 * time.Second):
    timeout()
case outgoing <- data:
    // sent
}
```

### Patterns

**Fan-out:** one producer to many consumers via one channel; receivers compete.

**Fan-in:** many producers to one consumer.

**Pipeline:** stages connected by channels, each stage transforming and forwarding.

```go
// Pipeline: gen -> square -> print
nums := gen(2, 3, 4)       // returns chan int
squares := square(nums)    // reads chan int, returns chan int
for s := range squares {
    fmt.Println(s)
}
```

### Pros and Cons

**Pros:** explicit communication, easier to reason about (no shared mutation), composes well.

**Cons:** every channel op has overhead; deeply pipelined systems have many context switches; deadlocks possible (all goroutines blocked, no progress).

CSP doesn't eliminate concurrency hazards; it shifts them. Channel deadlocks and starvation replace lock deadlocks.

---

## 10. Async / Event Loop

A single thread runs many concurrent tasks by giving each one a brief turn, yielding when it would block (typically on I/O). No true parallelism within the event loop; concurrency by interleaving.

```
+-------------------+
|    Event Loop     |
| (single thread)   |
+-------------------+
   |     |     |
  task1 task2 task3
   |     |     |
   (each runs until it awaits, then yields back to loop)

Behind the scenes: kernel I/O (epoll/kqueue/io_uring) tells the loop
which tasks are now ready to make progress.
```

**Languages:**

| Language | Primary mechanism |
|----------|------|
| JavaScript / Node.js | Single event loop, async/await over promises |
| Python | asyncio, async/await, gevent (greenlets) |
| Rust | tokio, async-std, Future trait + executor |
| C# / .NET | async/await over Task |
| Kotlin | coroutines |

### How Async Works

```python
async def fetch(url):
    response = await http.get(url)       # yields control here, returns when done
    return response.text()

results = await asyncio.gather(
    fetch('a'), fetch('b'), fetch('c')   # all three run concurrently
)
```

Under the hood, `await` is a yield point. The async runtime takes over, schedules other ready tasks, comes back when this one's I/O completes.

### Pros

- One thread can handle thousands of concurrent connections.
- No locking required between async tasks (within the same loop).
- Lower memory than thread-per-request.

### Cons

- **Function coloring:** async functions can only be called from async context. APIs split into sync/async worlds.
- **CPU work blocks the loop:** any task that doesn't yield blocks all others. Long CPU work in async code is fatal.
- **Awkward integration with blocking libraries.** Wrapping with thread pools is common.
- **Debugging is harder:** stack traces don't cross await boundaries cleanly; concurrency bugs differ from thread bugs.

### When Async Wins

- Many concurrent I/O-bound tasks (network servers, web scrapers, proxies).
- Low memory per task is important.
- Single-threaded determinism is helpful (browsers).

### When Threads Win

- CPU-bound parallelism (need multiple cores).
- Existing blocking libraries (sync code is simpler).
- Small concurrent count.

### Hybrid Models

Most async runtimes run on a thread pool. The loop itself is single-threaded, but blocking work can be offloaded to threads.

```
Async loop  --> CPU pool  (for synchronous CPU work)
            --> I/O pool  (for libraries that aren't async-aware)
```

Go takes this further with M:N — goroutines are async-ish but the runtime makes them feel synchronous, and the scheduler transparently moves them between OS threads.

---

## 11. Concurrency Hazards

### Data Race

Two threads access the same memory; at least one is a write; no synchronization between them. **Undefined behavior** in most languages — the compiler is permitted to assume races do not occur, so a data race can produce arbitrary output.

```
Thread 1: x = 1
Thread 2: read x
```

The read could see 0, 1, or some torn intermediate (for non-word-sized writes). On weak memory architectures, even more anomalies.

**Detection:** ThreadSanitizer (-fsanitize=thread in clang/gcc), Go's `-race` flag, Java's race detectors. Cheap to run in tests.

**Fix:** introduce synchronization (mutex, atomic, channel) so the operations are happens-before-related.

**Language note — degree of protection by language:**

- **C / C++:** no protection. A data race is undefined behavior; correctness is pure discipline. Run ThreadSanitizer.
- **Rust:** data races are *impossible in safe code*. The `Send`/`Sync` traits and the borrow checker reject, at compile time, any sharing of mutable state without synchronization. This is Rust's headline guarantee. (Logic-level race *conditions* are still possible — see below.)
- **Go:** no compile-time prevention, but a first-class race detector (`go run -race`) and a culture of "share by communicating" (channels) over shared memory.
- **Python / JS:** the GIL / single-threaded event loop prevents data races on the language's own objects, but check-then-act race *conditions* remain, and C-extension code or `multiprocessing` reintroduces real races.

The distinction matters: Rust eliminates *data races* (the memory-level bug), but no language eliminates *race conditions* (the logic-level bug) — that is the next subsection.

### Race Condition (vs Data Race)

Subtly different: a **race condition** is a higher-level bug where the program's behavior depends on the relative timing of events, even when individual memory accesses are properly synchronized.

```
Thread 1: if (balance >= 100)      // check  -- each access atomic/locked
              withdraw(100);       // act    -- but another thread withdrew in the gap

Both the read and the withdraw can be individually correct; the gap between
them is the bug. Both threads pass the check, both withdraw, and the balance goes negative.
```

The fix is to make check-and-act a single atomic step (hold a lock across both, or use a CAS/compare-and-set). Race conditions are application-level: no amount of per-access synchronization removes them — only redesigning the protocol does. **TOCTOU** (below) is the security-critical, file-system variant of this same bug.

### Deadlock

Two or more threads each hold a lock the other needs, neither can proceed.

```
Thread A:  lock(L1); lock(L2); ...
Thread B:  lock(L2); lock(L1); ...

If interleaved: A holds L1, waits for L2; B holds L2, waits for L1. Stuck.
```

**Conditions for deadlock (all four required — Coffman conditions):**

1. Mutual exclusion (locks are exclusive).
2. Hold and wait (hold one lock while waiting for another).
3. No preemption (locks can't be forcibly taken).
4. Circular wait (cycle in lock dependency graph).

**Prevention:** break one of the four conditions. Most common in practice is killing circular wait via a global lock order (the lock hierarchy detailed in §4) — always acquire locks in the same defined order.

**Detection:** Java has built-in deadlock detection. Many systems log "deadlock detected; aborting transaction X" — the DB picks a victim to abort.

### Livelock

Threads are running but make no progress because they keep responding to each other.

```
Thread A: detects conflict, backs off and retries.
Thread B: detects conflict, backs off and retries.
Both back off at the same time, retry at the same time, conflict again...
```

**Fix:** add randomized jitter to backoff. Or use a coordinator to break ties.

### Starvation

A thread waits indefinitely while others get serviced.

```
Reader-writer lock with many readers and one writer:
  Writer waits.
  Readers come and go continuously, always at least one reader holding the lock.
  Writer never gets a turn.
```

**Fix:** writer-preference RWLocks, fair scheduling, priority inheritance.

### Priority Inversion

A high-priority thread is blocked waiting for a lock held by a low-priority thread, while a medium-priority thread runs (preventing the low-priority thread from finishing).

Famous Mars Pathfinder bug. **Fix:** priority inheritance (the lock holder temporarily inherits the priority of the highest-priority waiter).

### TOCTOU (Time-of-Check, Time-of-Use)

The security-critical, file-system form of the race condition above: a check and the action based on it are separated, and an attacker changes the state in the gap.

```
if (file_owner == current_user) {   // check
    open(file);                      // use -- file might have been swapped (e.g. symlink)
}
```

Eliminate the gap by acting on a handle, not a path: `open()` first (with `O_NOFOLLOW` to refuse symlinks), then `fstat()` the returned fd to verify ownership — so that the entity checked is precisely the entity opened.

### False Sharing

Two variables in the same cache line, modified by different cores, cause cache-line thrashing even though the variables are logically independent. It's a *performance* hazard, not a correctness one. Mechanism, example, and the `_Alignas(64)` fix are in §2 (Cache Coherence) — listed here only so the hazard appears alongside its peers.

---

## 12. Patterns

### Immutable Data

The simplest concurrency primitive: data that doesn't change. Multiple threads can read freely without synchronization.

```c
// Read-only data that outlives the threads needs no synchronization:
static const int data[] = {1, 2, 3};

void *reader(void *arg) {
    for (size_t i = 0; i < 3; i++)
        printf("%d ", data[i]);   // many threads read freely, no lock
    return NULL;
}

pthread_t t1, t2;
pthread_create(&t1, NULL, reader, NULL);
pthread_create(&t2, NULL, reader, NULL);
```

C has no built-in reference counting, so immutable sharing means either static/long-lived data (as above) or a hand-rolled atomic refcount when lifetime is dynamic. Functional languages lean heavily on this; persistent data structures (Hash Array Mapped Tries, RRB-vectors) provide "modified" versions cheaply without copying.

### Read-Copy-Update (RCU)

Used heavily in the Linux kernel. Readers access data without locks; writers prepare a new version, atomically swap a pointer, and wait until existing readers finish before freeing the old version.

```
Readers:  follow pointer p, work with what they got.
Writer:   build new version p_new, atomically set p = p_new.
          Old readers still reading old version are fine -- old data not freed yet.
          After grace period (all old readers done), free old data.
```

Extremely fast for read-heavy data. Used for: route tables, file system metadata, mostly-read configuration.

### Single-Producer Single-Consumer (SPSC) Queue

The fastest concurrent queue. One thread writes, one reads. Lock-free with just an atomic head and tail.

```
Producer:                    Consumer:
  while (full) wait;            while (empty) wait;
  buffer[tail] = item;          item = buffer[head];
  tail.store(tail + 1);         head.store(head + 1);
```

When MPMC (multi-producer, multi-consumer) is not required, SPSC is dramatically simpler and faster.

### Worker Pool

A fixed pool of N worker threads pulls tasks from a queue. Caller submits work, gets a future for the result.

```
Submit -> Queue -> Worker 1, Worker 2, ..., Worker N
            ^
            single bottleneck (the queue) — make it lock-free for high throughput
```

Tune N to balance:

- CPU-bound work: N ≈ cores.
- I/O-bound work: N >> cores (or use async).

### Disruptor Pattern

LMAX's high-performance design for ring-buffer-based message passing. Pre-allocated ring buffer; producers and consumers track positions; mechanical sympathy (cache-line awareness, no allocation in hot path) yields millions of messages/second.

Used in low-latency trading, telemetry, log aggregation.

### Reactive Streams

A higher-level abstraction over async. Producer emits items; consumers express demand (backpressure). The system manages buffer sizes and propagates demand backward.

Implementations: RxJava, Project Reactor, Akka Streams, JavaScript RxJS.

```
Source --> map --> filter --> buffer --> consumer
                                 ^
                            buffer can apply backpressure if consumer is slow
```

### Read-Mostly Patterns

Many applications have one writer and many readers. Specialized patterns:

- **CopyOnWriteArrayList (Java):** write replaces the whole array; reads use the snapshot.
- **Persistent data structures:** new version shares structure with old.
- **Versioned reads:** readers grab a snapshot pointer atomically.

---

## 13. Anti-Patterns and Pitfalls

**Sharing mutable state casually.** Every shared mutable variable is a potential bug. Question every shared piece of state — does it really need to be shared?

**Holding locks across I/O.** If a thread holds a lock during a network request, every other thread waiting for that lock waits for the network. Latency multiplies. Restructure to release the lock before I/O.

**Double-checked locking, done wrong.** Without proper memory ordering (volatile, atomic), the classic "if (instance == null) { lock; if (instance == null) { instance = new(); } }" is broken on weak memory models.

**Manual sleep() for synchronization.** "Sleep 100 ms hoping the other thread finishes" is the worst race condition. Use condition variables or proper synchronization.

**Reinventing concurrent data structures.** Use `java.util.concurrent`, `crossbeam`, `tokio::sync`. They've been debugged. Yours hasn't.

**Async-over-blocking.** Wrapping a blocking library in async/await without offloading to a thread pool. The single event loop blocks; throughput collapses.

**Lock per-object pattern in low-level languages.** Allocating a mutex per object can be more expensive than the work it protects. Sharded locks or finer designs often win.

**Trusting `volatile` in C/C++ for synchronization.** `volatile` prevents some compiler optimizations but does not provide memory ordering across threads. Use atomics.

**Trusting that one variable's update is atomic.** On 32-bit architectures, a 64-bit value is two writes — racing reads can see partial updates. Use atomics explicitly.

**Channels as a workaround for missing locks.** Channels are great for communication, but spinning a goroutine to manage state via channels is sometimes overcomplicated compared to a mutex.

**Optimizing for contention that has not been measured.** Most code has ample headroom on plain mutexes. Lock-free structures, sharded locks, and similar techniques are last resorts, applied only after profiling demonstrates a benefit.

---

### Mental Model

Concurrency is a set of contracts:

```
1. What state is shared.                       (or: nothing shared, only messages)
2. Who reads it, who writes it.
3. How writers and readers don't conflict.     (locks, atomics, immutability)
4. What ordering is guaranteed across threads. (memory model, happens-before)
```

If all four can be answered for every piece of mutable state, the code is probably correct; if not, a latent bug exists.

**Most production systems do well with:**

- Coarse-grained locking on shared structures (mutex on a cache).
- Channels/queues for cross-thread communication.
- Immutable data wherever possible.
- A few well-known concurrent data structures from the standard library.
- Async I/O for network-heavy workloads.

The advanced techniques (lock-free, custom memory ordering, RCU) are a last resort for specific bottlenecks.
