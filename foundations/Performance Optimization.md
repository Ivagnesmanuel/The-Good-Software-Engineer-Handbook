# Performance Optimization

A practical guide to making software fast: how to think about performance, how to measure it, and the concrete techniques that matter — from the memory hierarchy and data layout up through I/O, concurrency, and the compiler.

---

## Table of Contents

1. The Optimization Mindset
2. Methodology: Measure, Don't Guess
3. Profiling and Measurement
4. The Memory Hierarchy
5. Data-Oriented Design
6. CPU-Level Performance
7. Memory Management
8. Algorithmic Optimization
9. I/O Optimization
10. Concurrency for Throughput
11. Compiler Optimizations
12. System-Level Tuning
13. Latency and Tail Latency
14. Anti-Patterns and Checklist

---

## 1. The Optimization Mindset

The first rule is that most code does not need optimizing. The second rule is that when it does, intuition is almost always wrong about *where*. Performance work is an empirical discipline: measure, find the bottleneck, fix the bottleneck, measure again.

### Amdahl's Law

The speedup from optimizing one part is capped by how much of the total time that part consumes.

```
If a section is fraction P of runtime, sped up by factor S:

                    1
  speedup = -----------------
             (1 - P) + P / S

Example: a function is 20% of runtime (P = 0.2). Make it infinitely fast (S = inf):
  speedup = 1 / (1 - 0.2) = 1.25x   -- a 25% win, no matter how aggressively it is optimized.

Optimizing the 80% is where the leverage is.
```

The lesson: find the part that dominates runtime before touching anything. A 2x improvement on the hot 90% beats a 100x improvement on the cold 1%.

---

## 2. Methodology: Measure, Don't Guess

### Latency vs Throughput

Two different things, often in tension.

```
Latency:     how long one operation takes.        (seconds per request)
Throughput:  how many operations per unit time.   (requests per second)
```

They are not reciprocals once concurrency enters. Batching raises throughput while *increasing* per-item latency. Pipelining raises throughput without lowering single-item latency. Know which one is being optimized — they often require opposite choices.

### Use Percentiles, Not Averages

Averages conceal the tail. A service with a 10 ms average might have a 2-second p99, and that p99 is what users notice.

```
p50  (median):   half of requests are faster than this
p99:             1 in 100 requests is slower than this
p99.9:           1 in 1000 -- the tail that dominates user-visible pain at scale

At 100 requests/page, a 1% slow rate (p99) means ~63% of pages hit the tail (1 - 0.99^100); even modest fan-out makes the tail the common case.
```

Always report distributions. A single number misrepresents a distribution.

### The USE Method (for resources)

For every resource (CPU, memory, disk, network), check three things:

```
Utilization:  percent of time the resource was busy
Saturation:   how much queued work is waiting for it
Errors:       error count
```

High utilization with high saturation is a bottleneck. High utilization with no saturation is fine — the resource is just being used.

### Establish a Baseline and a Target

Measure current performance. Define what "fast enough" means *numerically* (p99 under 50 ms; 10k req/s on this hardware). Without a target there is no way to know when to stop, and optimization has no natural end.

---

## 3. Profiling and Measurement

### Sampling vs Instrumentation

```
Sampling profiler:        interrupts the program N times/sec, records the stack.
                          Low overhead, statistical. Shows where time is spent.
                          Tools: perf, async-profiler, pprof.

Instrumenting profiler:   inserts counters/timers into the code.
                          Exact counts, higher overhead, can distort timing.
                          Tools: callgrind, gprof, manual timers.
```

Start with sampling — it is cheap and shows the hot path without distortion. Use instrumentation when exact call counts are needed.

### Flame Graphs

The single most useful visualization. Each box is a stack frame; width is the fraction of samples it appears in (≈ inclusive time — the x-axis is not wall-clock order). The root is always full width, so the signal is a wide *leaf* or a wide plateau (a frame much wider than its widest child): that is where the CPU actually sits.

```
            +--------------------------------------------------+
            |                  main                            |
            +-----------------------+--------------------------+
            |    parse_request      |     handle_query         |
            +-----------+-----------+-------+------------------+
            | tokenize  |  validate |  ...  |  db_lookup       |  <- widest leaf
            +-----------+-----------+-------+------------------+
                                            ^
                                  focus effort here
```

### perf — the Linux Swiss Army Knife

```bash
perf stat ./program            # cycles, instructions, cache misses, branch misses
perf record -g ./program       # sample with call graphs
perf report                    # interactive breakdown
perf stat -e cache-misses,L1-dcache-load-misses ./program
```

Key derived metrics:

```
IPC (instructions per cycle):  > 2 is good; < 1 suggests stalls (memory, branches)
cache-miss rate:               high rate -> memory-bound, look at data layout
branch-miss rate:              high rate -> unpredictable branches, consider branchless
```

### Microbenchmark Pitfalls

Microbenchmarks lie more often than they tell the truth:

- **Dead-code elimination:** the compiler deletes work whose result is unused. Consume the result (e.g., assign it to a `volatile` sink, as below).
- **Constant folding:** if inputs are compile-time constants, the compiler precomputes the answer. Use runtime inputs.
- **Cold caches / warmup:** the first iterations pay for cache and branch-predictor warmup. Discard them.
- **Unrealistic data:** sorted input, all-same values, or tiny working sets that fit in L1 give numbers never seen in production.
- **No statistical rigor:** run many times, report the distribution, watch for variance.

```c
// Defeat dead-code elimination: force the compiler to keep the result.
static volatile uint64_t sink;

uint64_t total = 0;
for (size_t i = 0; i < n; i++)
    total += work(input[i]);   // runtime input, not constants
sink = total;                  // observable side effect -> work can't be deleted
```

This section is about profiling to *optimize* — short, controlled runs driven directly. For profiling a *live production system* (attaching to a running process with no redeploy, fleet-wide continuous profiling, reading a profile during an incident), see `Observability.md`.

---

## 4. The Memory Hierarchy

The defining fact of modern performance: **the CPU is fast and memory is slow.** A core executes several instructions per nanosecond; a main-memory access costs ~100 ns. That is hundreds of wasted instruction slots per cache miss. Most "slow" code is not compute-bound — it is waiting on memory.

### The Latency Numbers Every Engineer Should Know

```
L1 cache reference                 ~1 ns      (~4 cycles)
L2 cache reference                 ~4 ns
L3 cache reference                 ~10-20 ns
Main memory reference              ~100 ns    (~300 cycles)
SSD random read                    ~16 us
Rotational disk seek               ~2-10 ms
Network round trip (same DC)       ~0.5 ms
Network round trip (cross-region)  ~50-150 ms
```

These span eight orders of magnitude. The objective is to keep the working set in the fast levels.

### Cache Lines and Locality

Memory moves in **cache lines** (typically 64 bytes), not bytes. Touching one byte pulls in its whole line. This makes two kinds of locality decisive:

```
Spatial locality:   data used together should live together in memory.
                    Accessing a[i] makes a[i+1] cheap (same line, prefetched).

Temporal locality:  data used recently will be used again -> keep it hot in cache.
```

The classic demonstration is array traversal order:

```c
// Row-major array. C stores rows contiguously.
int a[N][N];

// FAST: sequential in memory, one cache line serves ~16 ints, prefetcher kicks in.
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += a[i][j];

// SLOW: strides by a full row each step -> a cache miss almost every access.
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        sum += a[i][j];
```

Same operations, same result. The cache-friendly version can be 5-10x faster purely from access order.

### Prefetching

Hardware prefetchers detect sequential and strided access and fetch ahead of the CPU, hiding memory latency. Linear scans are nearly free because of this. Pointer-chasing (linked lists, trees of small nodes scattered in the heap) defeats the prefetcher — each `next` pointer is an unpredictable address and a likely cache miss. This is why a contiguous array often beats a linked list even for "list-like" workloads.

```c
__builtin_prefetch(&data[i + 8], 0, 3);  // hint: read data[i+8] soon, keep it hot
```

Manual prefetch helps only when the access pattern is known and the hardware prefetcher cannot detect it. Measure: it often does nothing or hurts.

---

## 5. Data-Oriented Design

The idea: design data layout around how it is *accessed*, not around conceptual objects. The CPU spends most of its time moving data; arrange data so the moves are cheap.

### Array of Structs vs Struct of Arrays

```c
// AoS: array of structs. Good when you touch most fields of one element together.
struct Particle { float x, y, z; float vx, vy, vz; int id; char name[32]; };
struct Particle particles[N];

// SoA: struct of arrays. Good when you sweep one field across all elements.
struct Particles {
    float x[N], y[N], z[N];
    float vx[N], vy[N], vz[N];
    int   id[N];
};
```

```
Updating just positions (x += vx) across all particles:

AoS: each element is ~60 bytes. Touching x and vx loads whole cache lines
     full of z, name, id that are not needed. Wasted bandwidth, more misses.

SoA: x[] and vx[] are dense contiguous arrays. Every loaded byte is useful.
     Also vectorizes cleanly (Section 6).
```

There is no universal winner. AoS wins when operations touch whole objects; SoA wins when operations sweep one or two fields across many objects. Choose based on the hot loop.

### Pack and Order Struct Fields

The compiler inserts padding to satisfy alignment. Field order changes struct size.

```c
struct Bad  { char a; long b; char c; };   // 24 bytes: pad after a, pad after c
struct Good { long b; char a; char c; };   // 16 bytes: fields ordered large -> small
```

Smaller structs mean more fit per cache line and per page. Order fields largest-to-smallest to minimize padding. Put fields touched together adjacent so they share a line.

### Avoid Pointer Chasing; Prefer Contiguous Storage

A tree of heap-allocated nodes scatters data across memory. The same logical structure stored in a flat array (e.g., a heap-ordered array, or indices instead of pointers) keeps it contiguous and prefetch-friendly. Replacing pointers with 32-bit indices also halves pointer size on 64-bit systems, packing more into cache.

### Language Note: Degree of Control Over Layout

Data-oriented design assumes control over memory layout. Languages differ enormously:

```
C:       full control. structs are exactly their fields; arrays are contiguous bytes.
C++:     same control as C, plus std::vector (contiguous) vs std::list (scattered) --
         the chosen type dictates the layout.
Rust:    same control as C; Vec<T> is contiguous, T stored inline (no boxing by default).
Go:      value types and contiguous slices ([]T stores T inline); good control,
         though interface values and maps add indirection.
Python:  almost no control. a list of objects is an array of *pointers* to boxed
         objects scattered on the heap -- every element is a cache miss. NumPy exists
         precisely to escape this: ndarray stores raw packed values, not Python objects.
Java:    objects are boxed and reference-typed; an array of objects is an array of
         references. Project Valhalla (value classes) aims to allow inline layout.
```

The practical consequence: AoS-vs-SoA and struct packing are everyday tools in C/C++/Rust/Go, largely unavailable in idiomatic Python/Java without dropping to packed arrays (NumPy, `ByteBuffer`) or primitive-typed arrays.

---

## 6. CPU-Level Performance

When a workload is not memory-bound, the CPU's own machinery — pipelines, branch predictors, vector units — sets the ceiling.

### Branch Prediction

The pipeline speculatively executes past branches by guessing the outcome. A correct guess is free; a misprediction flushes the pipeline (~15-20 cycle penalty). Predictable branches (almost always taken, or following a pattern) are cheap; random branches are expensive.

```c
// Sorting the data first makes this branch predictable -> dramatically faster,
// even counting the sort, for large repeated passes. (Classic benchmark.)
for (size_t i = 0; i < n; i++)
    if (data[i] >= threshold)   // unpredictable if data is random
        sum += data[i];
```

Branchless reformulation removes the branch entirely:

```c
// Branchless: compute a mask, no misprediction possible.
for (size_t i = 0; i < n; i++) {
    int take = (data[i] >= threshold);   // 0 or 1
    sum += data[i] * take;               // adds 0 when not taken
}
```

Branchless is not always faster — it does more work unconditionally. It wins when the branch is genuinely unpredictable. Predictable branches should stay as branches.

### SIMD / Vectorization

Modern CPUs include **wide registers** — 128 bits in SSE (NEON on ARM), 256 in AVX2, 512 in AVX-512 — that hold multiple scalar values packed side by side. Each slot is a **lane**. A SIMD (Single Instruction, Multiple Data) instruction operates on all lanes in parallel: one `vaddps` on an AVX2 register performs eight 32-bit float additions in the time a scalar `addss` performs one. The lane count is simply register width divided by element size:

```
128-bit SSE / NEON :  4 x float32,  2 x float64,  16 x int8
256-bit AVX2       :  8 x float32,  4 x float64,  32 x int8
512-bit AVX-512    : 16 x float32,  8 x float64,  64 x int8
```

The speedup is bounded by that lane count (4×–16× for floats), minus the overhead of loading, storing, and any per-lane masking the algorithm needs. On some CPUs the widest instructions (notably AVX-512) also lower the core clock while active, so wider is not always faster — measure rather than assume.

```c
// Auto-vectorizable: no loop-carried dependencies, contiguous access,
// known trip count. With -O3 -march=native the compiler emits SIMD,
// processing 4-16 elements per iteration instead of one.
void add_arrays(float *restrict a, const float *restrict b, size_t n) {
    for (size_t i = 0; i < n; i++)
        a[i] += b[i];
}
```

**Auto-vectorization** is the compiler recognizing such loops and emitting SIMD instructions on its own. It will only do so when it can *prove* the transformation preserves semantics. The common reasons it falls back to scalar code, each with the underlying obstacle:

- **Loop-carried dependencies.** `a[i] = a[i-1] + x` requires iteration i to finish before i+1 can start; lanes cannot execute in parallel when each depends on the previous.
- **Aliasing.** If `a` and `b` might point to overlapping memory, the compiler cannot reorder loads and stores safely. `restrict` (C) or `__restrict__` (C++) asserts non-overlap and unblocks vectorization. In Rust, the borrow checker rules out aliasing of `&mut` references, so the equivalent guarantee is implicit.
- **Function calls in the loop body.** The compiler cannot see through an opaque call to vectorize across it. Mark small helpers `static inline` so they are visible at the call site.
- **Complex control flow.** Divergent per-iteration branches require *masked* SIMD instructions (each lane conditionally enabled); compilers handle this inconsistently and often give up.
- **Non-contiguous access.** Strided or gathered loads (`a[idx[i]]`) break the contiguous-element assumption that makes vector loads a single wide memory transaction.

Confirm what the compiler actually did with `-fopt-info-vec` (GCC) or `-Rpass=loop-vectorize` (Clang) before reaching for intrinsics. When auto-vectorization fails — or when the algorithm needs explicit lane manipulation such as shuffles, masking, or horizontal reductions — drop to **intrinsics**: per-instruction wrappers (`_mm256_add_ps`, `vaddq_f32`, etc.) that give the same control as assembly while keeping C-level types and register allocation.

**Across languages:** C/C++ expose hardware intrinsics directly (`<immintrin.h>`, AVX/NEON) for hand-written SIMD. Rust has stable `std::arch` intrinsics plus the (still-stabilizing) portable `std::simd`. Go has no portable SIMD, requiring assembly or reliance on limited compiler autovectorization. Python does not vectorize at the interpreter level at all; SIMD is available only by calling into NumPy/native code, which is why numeric Python pushes loops down into C.

### Instruction-Level Parallelism

A core executes multiple independent instructions per cycle out of order. Long dependency chains starve it; independent work fills the slots.

```c
// Dependency chain: each add waits for the previous. Limited ILP.
for (i = 0; i < n; i++) sum += a[i];

// Multiple accumulators: four independent chains -> the core overlaps them.
double s0=0, s1=0, s2=0, s3=0;
for (i = 0; i + 3 < n; i += 4) {
    s0 += a[i];   s1 += a[i+1];
    s2 += a[i+2]; s3 += a[i+3];
}
double sum = s0 + s1 + s2 + s3;
```

(Note: with `-ffast-math` the compiler may do this reassociation itself; without it, floating-point associativity rules forbid it, so it must be done manually.)

---

## 7. Memory Management

### Allocation Is Not Free

`malloc`/`free` involve bookkeeping, possible locking, and can fragment the heap. In a hot loop, allocation is often the bottleneck — and a cache miss generator, since fresh allocations are cold.

Strategies, in rough order of preference:

```
1. Don't allocate in the hot path.       Allocate up front; reuse buffers.
2. Stack allocation.                     Automatic, cache-hot, freed for free.
3. Arena / bump allocator.               Allocate from a block, free all at once.
4. Object pool / free list.              Recycle fixed-size objects.
5. General malloc.                       The fallback, not the default in hot code.
```

### Arena (Bump) Allocator

When many objects share a lifetime, allocate them from one arena and free the whole arena in one operation. No per-object free, no fragmentation, allocation is a pointer bump.

```c
typedef struct {
    char  *base;     // start of the backing block
    size_t size;     // total bytes in the block
    size_t offset;   // bytes already handed out
} Arena;

// The one and only real allocation: obtain a block from the OS once.
void arena_init(Arena *a, size_t size) {
    a->base   = malloc(size);   // <-- the actual allocation lives HERE
    a->size   = size;
    a->offset = 0;
}

// Hand out a slice of the existing block. No malloc, no syscall.
void *arena_alloc(Arena *a, size_t n) {
    n = (n + 15) & ~(size_t)15;           // round up to 16-byte alignment
    if (a->offset + n > a->size) return NULL;
    void *p = a->base + a->offset;
    a->offset += n;                        // the "allocation" is a pointer bump
    return p;
}

void arena_reset(Arena *a) { a->offset = 0; }   // "free" every allocation at once
void arena_destroy(Arena *a) { free(a->base); } // return the block to the OS

// Usage:
Arena a;
arena_init(&a, 1 << 20);          // one 1 MB malloc, up front
for (int i = 0; i < 10000; i++) {
    Node *n = arena_alloc(&a, sizeof *n);   // zero malloc cost per iteration
    /* ... */
}
arena_reset(&a);                  // reuse the same 1 MB for the next batch
arena_destroy(&a);                // when the arena itself is done
```

**Walking through it.** The only real allocation is the single `malloc` in `arena_init`; `arena_alloc` carves slices out of that block, so the cost of `malloc` is paid once and amortized across every allocation. The one non-obvious line is the rounding: `(n + 15) & ~(size_t)15` clears the low 4 bits of `n + 15`, rounding the request *up* to the next multiple of 16 (adding 15 *first* is what makes it round up rather than down — `n & ~15` alone rounds down). That keeps every allocation, and the cursor after it, 16-byte aligned, which SIMD loads require and ordinary access prefers; it works because `malloc` already returns memory aligned for any type (`max_align_t`, typically 16 bytes). `arena_reset` then marks the whole block reusable in a single store, with the sole contract that no pointer into the arena is dereferenced afterward.

Ideal for per-request, per-frame, or per-parse lifetimes: reset the arena at the end and all those allocations vanish for free.

### Alignment

Misaligned data can cost extra loads or, on some architectures, fault. Align hot data to cache-line boundaries to avoid straddling two lines, and to prevent false sharing between cores (see `Concurrency.md`).

```c
struct HotCounters {                  // aligned to its own cache line
    _Alignas(64) atomic_long requests;
    atomic_long errors;
};
```

### Memory Bandwidth Is a Shared Resource

On a multicore machine, all cores share memory bandwidth. A workload that is fine single-threaded can saturate the memory bus when run on every core, so it stops scaling. If adding cores stops helping, suspect bandwidth, not locks.

### Roofline: Compute-Bound or Memory-Bound

Attainable performance is bounded by the lesser of two ceilings: peak compute (FLOP/s — floating-point operations per second) and peak memory bandwidth multiplied by arithmetic intensity, where arithmetic intensity is the number of operations performed per byte moved from memory.

```
attainable FLOP/s = min( peak_compute,  bandwidth * arithmetic_intensity )

 FLOP/s |          ____________  peak compute (flat roof)
        |         /
        |        /  <- bandwidth-bound      compute-bound ->
        |       /     (sloped roof)
        +------/--------------------------- arithmetic intensity (FLOP/byte)
```

Low-intensity kernels (a single operation per loaded element — vector add, SAXPY) are bandwidth-bound, and faster arithmetic does not help. High-intensity kernels (dense matrix multiply, which reuses each loaded byte across many operations) are compute-bound. The lever differs by regime: bandwidth-bound code is fixed by moving fewer bytes (better layout, cache blocking, smaller types); compute-bound code by SIMD and instruction-level parallelism.

### Language Note: Manual vs GC vs Ownership

C's `malloc`/`free` is *manual* — full control, full responsibility (leaks, use-after-free, double-free). The other languages move the cost somewhere else:

```
C:       manual malloc/free. Control + bugs are yours.
C++:     RAII (Resource Acquisition Is Initialization) + smart pointers (unique_ptr,
         shared_ptr). Deterministic destruction;
         shared_ptr adds atomic refcount traffic. Still no GC pauses.
Rust:    ownership + borrow checker. Deterministic free at scope end, no GC, and the
         compiler proves there's no use-after-free -- the safety of GC without the pauses.
Go:      garbage collected. Low-pause concurrent collector; allocation rate drives GC
         cost. Escape analysis keeps some allocations on the stack.
Java:    garbage collected. Mature, tunable collectors (G1, ZGC, Shenandoah) trade
         throughput vs pause time. Everything non-primitive is heap-allocated by default.
Python:  reference counting + a cycle collector. Per-object overhead is large; the
         allocator and refcounting dominate many "pure Python" hot loops.
```

The performance implications: in GC languages the lever is *allocation rate* — fewer allocations means less GC pressure and fewer pauses (this is why low-latency Java preallocates everything during warmup). In C/C++/Rust the lever is *which allocator and what lifetime* — arenas, pools, and stack allocation. Rust is notable for giving C-level allocation control with compile-time safety and no collector. GC pauses are also a primary tail-latency source.

---

## 8. Algorithmic Optimization

No amount of micro-optimization saves a quadratic algorithm on large input. Algorithm choice dominates everything else as N grows.

### Complexity First

```
n = 1,000,000:
  O(n)        ->  1,000,000 ops          (instant)
  O(n log n)  ->  ~20,000,000 ops        (fine)
  O(n^2)      ->  1,000,000,000,000 ops  (many minutes to hours)
```

Before optimizing constants, confirm the complexity class is acceptable for the real input size. Swapping an O(n^2) scan for a hash lookup (O(n)) is worth more than any cache trick.

### But Constant Factors Matter at Scale

Once complexity is right, constants decide real performance. A cache-friendly O(n log n) can far outperform a pointer-chasing O(n log n). Big-O hides the constant; profiling reveals it.

### Know When the "Worse" Algorithm Wins

```
Small n:        linear scan beats a hash table (no hashing, cache-friendly).
Nearly sorted:  insertion sort beats quicksort.
Tiny arrays:    don't allocate a tree -- a flat array searched linearly wins.
```

Theoretical complexity assumes large N. For small N, simple and contiguous usually wins because of constant factors and cache behavior. Many standard sorts switch to insertion sort below ~16 elements for exactly this reason.

### Precompute, Cache, Memoize

If a result is reused and inputs don't change, compute it once. Lookup tables, memoization, and caching trade memory for time. (Caching introduces invalidation problems — see distributed/data systems material — but in a single process it is often the biggest single win.)

---

## 9. I/O Optimization

I/O is orders of magnitude slower than compute. The strategies all reduce the number of slow operations or hide their latency.

### Batch and Buffer

Each syscall crosses the user/kernel boundary (~hundreds of ns to microseconds — see `Operating Systems.md`). One read of 64 KB beats 64,000 reads of one byte.

```c
// SLOW: a syscall per byte.
char c;
while (read(fd, &c, 1) == 1) process(c);

// FAST: read in large chunks, process from a user-space buffer.
char buf[65536];
ssize_t n;
while ((n = read(fd, buf, sizeof buf)) > 0)
    for (ssize_t i = 0; i < n; i++) process(buf[i]);
```

Buffered I/O (`stdio`, or a custom buffer) amortizes the syscall cost over many logical operations. Vectored I/O (`readv`/`writev`) submits multiple buffers in one call.

### Sequential Beats Random

On every storage medium, sequential access vastly outperforms random access — disks for seek reasons, SSDs for read-amplification and prefetch reasons. Lay data out to be read in the order it is needed. This is why log-structured designs and column stores exist.

### Zero-Copy

Every copy between buffers costs bandwidth and cache. Kernel facilities move data without bouncing it through user space:

```
Traditional: disk -> kernel buffer -> user buffer -> kernel socket buffer -> NIC (network interface card)
             (two extra copies, two extra crossings)

sendfile():  disk -> kernel buffer -> NIC
             (kernel moves it directly; no user-space copy)

mmap():      map a file into the address space; access it as memory, the kernel
             pages it in on demand. No explicit read() copies.
```

`sendfile`, `splice`, and `mmap` eliminate copies for file-to-socket and file-scanning workloads.

### Asynchronous and Batched Submission

Blocking one thread per I/O wastes a thread per in-flight operation. Async models (`epoll`, `io_uring`) let one thread keep thousands of operations in flight. `io_uring` additionally batches submissions and completions through shared ring buffers, cutting syscalls dramatically. See the async section of `Concurrency.md` for the programming model.

---

## 10. Concurrency for Throughput

Parallelism is a performance tool, but it has its own cost model. Correctness aside (covered in `Concurrency.md`), the performance traps are specific.

### Scaling Is Not Free — Revisit Amdahl

The serial fraction caps parallel speedup. If 5% of the work is inherently serial, the maximum speedup is 20x no matter how many cores are added. Worse, coordination overhead can make additional cores *slower* past a point.

### Contention Kills Scaling

A lock that every thread needs serializes them — N cores are present but used one at a time. Symptoms: throughput flat or falling as cores are added; high time in lock/futex (fast userspace mutex).

```
Fixes, in order of preference:
  1. Don't share.            Per-thread state, combine at the end.
  2. Shard the lock.         N locks by key hash -> N-way parallelism.
  3. Shorten critical sections. Do work outside the lock; hold it only to publish.
  4. Lock-free / atomics.    Last resort; high complexity (see Concurrency.md).
```

### False Sharing

Two threads writing different variables that share a cache line ping-pong the line between cores, destroying performance even with zero logical contention. Pad hot per-thread data to separate cache lines. (Full treatment in `Concurrency.md`; it shows up here because it is a silent throughput killer.)

```c
// Per-thread counters, each on its own cache line.
struct ThreadCounter { _Alignas(64) atomic_long value; };  // alignof == sizeof == 64
struct ThreadCounter counters[NUM_THREADS];
```

### Thread Count

```
CPU-bound work:   threads ~= physical cores. More just adds context-switch overhead.
I/O-bound work:   threads >> cores, or use async to avoid threads entirely.
Mixed:            separate pools, or async with a CPU offload pool.
```

Oversubscribing CPU-bound work hurts: context switches pollute caches and the TLB (Translation Lookaside Buffer; see `Operating Systems.md`).

---

## 11. Compiler Optimizations

The compiler is the cheapest optimizer available. Let it work before hand-optimizing.

### Optimization Levels

```
-O0   no optimization; fast compiles, for debugging
-O1   basic optimizations
-O2   the production default: inlining, most optimizations, safe
-O3   aggressive: more inlining/vectorization; can bloat code, occasionally slower
-Os   optimize for size (smaller code can mean fewer instruction-cache misses)
-Ofast  -O3 plus -ffast-math: breaks strict IEEE float; use only if that is acceptable
```

Default to `-O2`. Measure `-O3` against it — it is not always faster, because code bloat can hurt the instruction cache.

### Communicate Invariants to the Compiler

```c
// restrict: promises pointers don't alias -> enables vectorization and reordering.
void scale(float *restrict out, const float *restrict in, size_t n, float k);

// inline / static: lets the compiler eliminate call overhead for small hot functions.
static inline int clamp(int x, int lo, int hi);

// likely/unlikely: bias branch layout for the common case.
if (__builtin_expect(error, 0)) handle_error();   // error path is cold
```

### Link-Time Optimization (LTO)

Without LTO, the compiler optimizes each translation unit in isolation — it cannot inline across files. LTO (`-flto`) defers optimization to link time with whole-program visibility, enabling cross-module inlining and dead-code elimination. Often a few percent for free; sometimes much more.

### Profile-Guided Optimization (PGO)

Compile with instrumentation, run on representative workloads to collect a profile, then recompile using that profile. The compiler then knows which branches are hot, which are cold, and which functions to inline — informed by real behavior rather than heuristics.

```
1. gcc -fprofile-generate ...        # build instrumented binary
2. ./program <representative input>  # collect runtime profile
3. gcc -fprofile-use ...             # rebuild using the profile
```

PGO commonly yields 5-20% on branch-heavy code (parsers, interpreters, databases) for no source changes. `-march=native` additionally lets the compiler use every instruction the CPU supports (AVX, etc.) at the cost of portability.

### Language Note: AOT vs JIT vs Interpreted

This entire section assumes an *ahead-of-time* compiler (the C/C++/Rust/Go model): code is compiled once into a native binary, and the optimizer has the whole program (with LTO) but no runtime information. Other execution models change the picture:

```
AOT compiled (C, C++, Rust, Go):
    Native code before running. No warmup. Optimizer sees code, not behavior
    (PGO feeds behavior back in). Predictable steady-state performance.

JIT (just-in-time) compiled (Java/JVM, C#/.NET, JavaScript/V8, PyPy):
    Starts interpreting, then compiles hot methods at runtime using *observed* behavior
    (actual branch frequencies, actual types, inlining across virtual calls).
    Can in principle beat AOT on hot loops -- but pays a WARMUP cost: early requests are
    slow, and the first measurements after start are unrepresentative. Deopt/recompile
    can also cause jitter. Benchmarks must warm up before measuring.

Interpreted (CPython, Ruby MRI):
    Bytecode executed by a loop; no native compilation of the code. 10-100x slower than
    native for compute. The escape hatch is native extensions (NumPy, Cython, C modules)
    that run AOT-compiled code underneath -- which is why "optimizing Python" so often
    means "do less in Python and more in C."
```

Practical consequences for measurement: a JIT must warm up before a benchmark can be trusted, and its tail latency includes compilation pauses. The LTO and PGO discussed above are AOT concepts; a JIT does the equivalent online and automatically.

---

## 12. System-Level Tuning

Beyond the program, the OS and hardware configuration affect performance.

### NUMA

On multi-socket systems, memory is attached to specific sockets (NUMA — Non-Uniform Memory Access). Accessing a remote socket's memory is slower than local. The fix is locality: keep a thread's data on the node where the thread runs.

```
Socket 0           Socket 1
[cores] -- fast -- [RAM 0]      [cores] -- fast -- [RAM 1]
   \                                /
    \------- slow (interconnect) --/

numactl --cpunodebind=0 --membind=0 ./program   # pin to one node
```

### Huge Pages

The default 4 KB page means a large working set needs many TLB entries; TLB misses trigger page-table walks (see `Operating Systems.md`). Huge pages (2 MB / 1 GB) cover more memory per TLB entry, cutting TLB misses for large, dense data sets (databases, in-memory caches).

### CPU Affinity and Isolation

Pinning threads to cores (`taskset`, `pthread_setaffinity_np`) keeps caches warm and avoids migration. For latency-critical work, isolating cores (`isolcpus`) keeps the scheduler and other tasks off them — the same toolkit hard-real-time systems use to bound scheduling jitter.

### Frequency Scaling

CPUs throttle frequency to save power. For benchmarking and latency-sensitive services, the `powersave` governor's ramp-up adds jitter; set the `performance` governor for consistent, repeatable numbers.

---

## 13. Latency and Tail Latency

Throughput optimization and latency optimization diverge. At scale, the *tail* (p99, p99.9) is what users feel, and the tail has its own causes.

### Sources of Tail Latency

```
- GC / allocator pauses            stop-the-world or slow-path allocation
- Lock contention spikes           occasional convoying under load
- Context switches / scheduling    thread waits behind others
- Cache and TLB misses             cold data on the slow path
- Background work                  compaction, flushing, log rotation
- Queueing                         work waiting behind other work (the big one)
```

### Queueing Is the Dominant Effect

Little's Law: `L = lambda * W` (items in system = arrival rate x time in system). As utilization approaches 100%, queue length — and thus latency — explodes nonlinearly.

```
Mean queueing delay grows as  rho / (1 - rho)   (rho = utilization, M/M/1 model):

  utilization 50%  ->  baseline (~1x)
  utilization 80%  ->  ~4x
  utilization 95%  ->  ~20x
  utilization 99%  ->  ~100x
```

This is why high-throughput systems deliberately run below saturation (the M/M/1 curve is idealized, but the cliff near 100% utilization is universal). The last 10% of capacity costs enormous latency. Headroom is not waste — it is the tail-latency budget.

### Techniques for the Tail

- **Run with headroom.** Don't push utilization to the cliff.
- **Bound queues.** Unbounded queues hide overload and inflate latency; shed load instead.
- **Hedged requests.** Send a duplicate request if the first is slow; take the first reply. Trades a little extra load for a much better p99.
- **Separate latency-critical from batch work.** Don't let background compaction share the core with request handling.
- **Avoid stop-the-world pauses.** In managed runtimes, tune or choose a low-pause GC; in C, control allocation so it never stalls (arenas, pools).

For *measuring* the tail in production (percentile/histogram metrics, latency heatmaps) and validating capacity under load, see `Observability.md`. Fan-out amplification and hedging as a design pattern are treated at the system-design altitude in `System Design.md`.

---

## 14. Anti-Patterns and Checklist

### Anti-Patterns

**Optimizing without measuring.** The single most common mistake; it optimizes the wrong thing.

**Trusting microbenchmarks.** Dead-code elimination, constant folding, and unrealistic data make them misleading. Validate against the real workload.

**Optimizing the cold path.** Effort off the critical path is wasted (Amdahl). Profile to find the hot path first.

**Premature lock-free / SIMD / assembly.** High-complexity techniques applied before establishing they are the bottleneck. Exhaust the cheap wins (algorithm, layout, compiler flags) first.

**Ignoring the memory hierarchy.** Treating memory as uniform and free. Most "compute" bottlenecks are actually memory stalls.

**Allocating in the hot loop.** A `malloc` per iteration is both slow and a cache-miss generator. Hoist allocations out.

**Scaling a contended design.** Adding cores to a workload bottlenecked on one lock or on memory bandwidth. Fix the bottleneck, not the core count.

**Running at 100% utilization.** Maximizes throughput while destroying tail latency. Leave headroom.

**`-Ofast` / `-ffast-math` without understanding it.** Silently changes floating-point results; can break numerical code.

### Checklist

```
[ ] Does it work correctly? (optimize nothing before this)
[ ] Is there a measured, numerical performance target?
[ ] Has profiling identified the actual bottleneck?
[ ] Is the algorithmic complexity right for the real input size?
[ ] Is it memory-bound? (check cache-miss rate, IPC)
[ ] Is data laid out for the access pattern? (locality, AoS vs SoA)
[ ] Is it allocating in the hot path?
[ ] Is I/O batched, buffered, and sequential where possible?
[ ] Are compiler optimizations enabled? (-O2/-O3, LTO, PGO, restrict, -march=native)
[ ] If parallel: is it contention-free and bandwidth-aware?
[ ] If latency matters: is it running with headroom, with bounded queues?
[ ] Has it been measured again, against the target, to confirm the win?
```

### Mental Model

Performance optimization is a loop, not a phase:

```
   measure  ->  find the bottleneck  ->  fix the bottleneck  ->  measure again
      ^                                                              |
      +--------------------------------------------------------------+
                        stop on reaching the target
```

The bottleneck moves every time one is fixed. Without measuring after each change, the process is guesswork — and the machine's behavior is too counterintuitive (caches, branches, queueing) to guess correctly. Let data, not intuition, decide where the next increment of effort goes.
