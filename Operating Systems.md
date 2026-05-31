# Operating Systems

A quick-reference guide covering operating system fundamentals — the abstractions the kernel provides (processes, threads, virtual memory, files, syscalls), how programs become processes (compilation, linking, loading), and the mechanisms that shape program behavior in production (scheduling, IPC, signals, the page cache).

Linux is the reference, but concepts apply to Unix-like systems in general.

---

## Table of Contents

1. What the OS Provides
2. Processes
3. Threads
4. Scheduling
5. Virtual Memory
6. The Page Cache and File I/O
7. Filesystems
8. System Calls
9. IPC (Pipes, Sockets, Shared Memory)
10. Signals
11. Compilation, Linking, and Loading
12. ELF and Dynamic Linking
13. Boot and Init

---

## 1. What the OS Provides

An OS is a layer between hardware and user programs that provides four primary abstractions:

```
+-------------------------------------------------+
|                User programs                    |   userspace
|     +-----------+  +-----------+  +-----------+ |
|     |  process  |  |  process  |  |  process  | |
|     +-----------+  +-----------+  +-----------+ |
|           |              |              |       |
+-----------+--------------+--------------+-------+
            |              |              |
            |          syscalls           |
            v              v              v
+-------------------------------------------------+
|                                                 |
|                    Kernel                       |   kernel-space
|                                                 |
|  Processes | Memory | Files | Network | Devices |
|                                                 |
+-------------------------------------------------+
                       |
                  +----+----+
                  | hardware |
                  +----------+
```

**The four abstractions:**

| Abstraction | What it virtualizes | Why it matters |
|-------------|---------------------|----------------|
| Process | The CPU | Each process believes it has the CPU to itself |
| Virtual address space | RAM | Each process has its own private memory map |
| File | Disk + devices | Uniform I/O API for storage, sockets, pipes |
| Threads / scheduler | Multiple CPUs | Concurrent execution within a process |

Programs interact with the kernel through **system calls** — controlled entries into kernel-mode code, the only legitimate way to access hardware or shared resources.

### Privilege Levels

x86 (and most modern CPUs) have multiple protection rings. Linux uses two:

- **Ring 0 (kernel):** can execute privileged instructions, access any memory, talk to hardware.
- **Ring 3 (user):** restricted; must use syscalls to ask the kernel for anything.

A user process cannot disable interrupts, access physical memory directly, or talk to hardware. It must ask the kernel via syscall, which validates the request before performing it.

Memory is partitioned: each process's virtual address space has a userspace region (its code, data, heap, stack) and a kernel region (mapped into every process but only accessible while in kernel mode).

---

## 2. Processes

A process is the OS's representation of a running program. Each process has:

- A unique PID.
- Its own virtual address space.
- Open file descriptors.
- Credentials (UID, GID).
- Environment, working directory, resource limits.
- A parent process (the one that fork()ed it).
- Child processes.

### The Lifecycle

```
  fork()             execve()           exit()
parent ---> child ---> new program --> ... --> termination
                                                  |
                                                  v
                                              zombie (waiting for parent to wait())
                                                  |
                                                  v parent calls wait()
                                              reaped (gone)
```

### fork() and exec()

**The problem they solve.** A running program must start a *different* program (for example, a shell launching `grep`). Most systems offer a single "spawn this program" call. Unix instead splits the task into two composable primitives: `fork()` copies the current process, and `execve()` replaces a process's program with a new one. The split provides a window, *between* the fork and the exec, in which the child still runs the parent's code and can configure itself (redirect stdout, drop privileges, change directory) before the new program takes over.

**`fork()` — clone the current process.** It creates a near-identical copy of the calling process: same code, same memory contents, same open files. The one observable difference is the return value, and this is the key trick:

```c
pid_t pid = fork();
// Execution continues HERE in BOTH processes — parent and the new child.

if (pid == 0) {
    // We are the CHILD. fork() returned 0 to us.
    execve("/usr/bin/grep", argv, envp);   // replace ourselves with grep
    // execve only returns if it FAILED (e.g., file not found)
    perror("execve");
    _exit(127);
} else if (pid > 0) {
    // We are the PARENT. fork() returned the child's PID to us.
    int status;
    waitpid(pid, &status, 0);              // wait for the child to finish
} else {
    // fork() returned -1: the clone failed (out of memory / process limit)
    perror("fork");
}
```

After `fork()` there are two processes both executing the next line. They tell themselves apart by what `fork()` returned: `0` means "I'm the child," a positive number means "I'm the parent, and that's my child's PID." This is why the same code after `fork()` runs down two different branches.

**`execve()` — become a different program.** It throws away the current program's code, data, heap, and stack, loads the new executable from disk, and starts it at its entry point. What survives the transformation: the PID, the open file descriptors (unless marked close-on-exec), and the process's relationships (parent, process group). This survival is exactly why the child can, say, redirect a file descriptor before calling exec — the redirection persists into the new program. That's how shell pipelines and `>` redirection are built.

```
A shell running "ls > out.txt":

  fork()                 -> now two shells
  child: open("out.txt") -> get fd, dup2 it onto fd 1 (stdout)
  child: execve("ls")    -> ls runs; its stdout is already wired to out.txt
  parent: waitpid()      -> wait for ls to finish, then prompt again
```

**Why fork() isn't as expensive as "copy all the memory" sounds — copy-on-write (COW).** Naively, cloning a process means duplicating its entire address space, which could be gigabytes. That would also be wasteful, because the very next thing the child usually does is `execve()`, discarding all that copied memory.

Instead the kernel defers copying. After `fork()`, parent and child *share* the same physical memory pages, but every page is marked **read-only**. Nothing is copied yet. Both processes read freely from the shared pages. The moment *either* process attempts to **write** to a page, the CPU traps (write to read-only memory), the kernel intercepts it, makes a private copy of that single page for the writer, marks the copy writable, and allows the write to proceed. Hence "copy on write": pages are copied lazily, only when first modified, and only those actually modified.

```
After fork(): parent and child share pages, all marked read-only

  Parent ---\
             >--> [page A: ro] [page B: ro] [page C: ro]
  Child  ---/

Child writes to page B:
  -> CPU faults (write to read-only)
  -> kernel copies page B, gives child its own writable copy
  -> only page B is duplicated; A and C stay shared

  Parent --> [page A: ro] [page B: ro] [page C: ro]
  Child  --> [page A: ro] [page B': writable copy] [page C: ro]
```

So `fork()` followed immediately by `execve()` is cheap: almost nothing is copied before exec throws it all away. COW also explains a subtle production behavior — a process that forks while using 10 GB of RAM doesn't instantly double memory use, but if the child (or parent) then writes across all those pages, memory use climbs as copies accumulate.

### Process States

Linux process states (from `ps`):

| State | Meaning |
|-------|---------|
| R | Running or runnable (on a CPU or in the run queue) |
| S | Interruptible sleep (waiting for an event, can be woken by signal) |
| D | Uninterruptible sleep (typically waiting on disk I/O) — cannot be killed |
| T | Stopped (SIGSTOP, debugger) |
| Z | Zombie (terminated, awaiting parent's wait()) |
| X | Dead (about to be removed) |

**Zombies** exist because the OS retains a process's exit status until its parent retrieves it via `wait()`. A zombie consumes a process slot. Long-lived zombie processes indicate a bug in the parent (not calling wait()).

**Orphans** are processes whose parent died. They get adopted by PID 1 (init/systemd) which periodically reaps them.

### Process Groups, Sessions, Jobs

A **process group** is a set of related processes (e.g., a shell pipeline). A **session** is a collection of process groups, typically associated with a controlling terminal.

```
Shell session:
  Session leader: bash
  Process groups (jobs):
    - PG 1: vim
    - PG 2: cat | grep | sort   (3 processes in one group)
    - PG 3: my-daemon &          (background)

  Ctrl+C sends SIGINT to the foreground process group.
  Ctrl+Z sends SIGTSTP to the foreground process group.
```

### Resource Limits

`ulimit` / `setrlimit` cap resource usage per process:

```
RLIMIT_NOFILE     max open file descriptors (default often 1024)
RLIMIT_NPROC      max processes for this user
RLIMIT_AS         max virtual memory
RLIMIT_STACK      max stack size
RLIMIT_CORE       max core dump size
RLIMIT_CPU        max CPU seconds
```

`ulimit -n` (file descriptors) is the most common one engineers tune — web servers and databases hit it.

---

## 3. Threads

A thread is a unit of execution within a process. Multiple threads in one process share the same address space, file descriptors, and other process-level resources, but each has its own:

- Stack.
- Program counter (next instruction).
- Registers.
- Thread-local storage.

### Why Threads vs Processes

| Property | Process | Thread |
|----------|---------|--------|
| Memory | Isolated address space | Shared address space |
| Creation cost | Higher (clone full address space metadata) | Lower (share most state) |
| Communication | IPC (pipes, sockets, shm) | Shared memory directly |
| Failure isolation | Crash isolated | Crash kills all threads |
| Concurrent data access | Safe by default (separate memory) | Requires synchronization |

Threads are faster to create and communicate via shared memory; processes are isolated and safer.

### Thread Implementation Models

**1:1 (kernel threads):** every userspace thread is a kernel thread. The scheduler sees each thread. Linux (pthread → clone()), Windows. Default for most languages today.

**N:1 (green threads / user threads):** many userspace threads multiplexed onto one kernel thread. Cheaper but only uses one CPU.

**M:N (hybrid):** many userspace threads multiplexed onto a smaller pool of kernel threads. Go's goroutines (M:N over GOMAXPROCS kernel threads), Erlang's processes, Java's virtual threads.

```
Go runtime:
  Goroutines (millions possible)
       |
  Multiplexed onto P (logical processors, GOMAXPROCS)
       |
  Bound to M (kernel threads)
       |
  Run on CPUs
```

M:N requires a sophisticated runtime to handle blocking syscalls (must dispatch the kernel thread to other goroutines without losing the blocked one).

### Concurrency Primitives

Threads need ways to coordinate. The kernel provides primitives; languages wrap them.

| Primitive | What it does |
|-----------|--------------|
| Mutex | Mutual exclusion — only one thread holds the lock at a time |
| RWLock | Multiple readers OR one writer |
| Condition variable | Wait until a condition is signaled |
| Semaphore | Counter — N threads can acquire before blocking |
| Atomic operations | Lock-free read-modify-write on machine-word values |
| Barrier | N threads wait until all reach the barrier |
| Spinlock | Busy-wait (no syscall) — only for very short critical sections on multi-core |

Detailed treatment of memory models, lock-free algorithms, and the actual hazards (deadlock, race conditions, data races) is in `Concurrency.md`.

### Thread Pools

Creating a thread per task is wasteful (each thread costs ~8 MB stack by default). A thread pool maintains N worker threads that pull tasks from a queue.

```
Tasks ----> Queue ----> Worker 1
                  ----> Worker 2
                  ----> Worker N
```

Most language runtimes provide thread pools (Java ExecutorService, Python concurrent.futures, .NET ThreadPool). Sizing: typically `~cores` for CPU-bound work, `much larger than cores` for I/O-bound work — though async I/O is usually better than blocked threads.

---

## 4. Scheduling

**The problem.** A machine has 8 CPU cores but 500 runnable threads. Only 8 can execute at any instant. The **scheduler** is the kernel code that repeatedly determines which of the ready threads receive the cores, and for how long. It runs constantly — on every timer interrupt, every time a thread blocks or wakes — and its decisions determine whether the system is responsive, wastes CPU, or starves a workload.

It balances competing goals that cannot all be maximized simultaneously:

- **Fairness:** every thread gets a reasonable share of CPU.
- **Throughput:** get the most total work done (favors running things uninterrupted).
- **Latency/responsiveness:** wake up an interactive thread (a keystroke handler) fast (favors interrupting whatever's running).
- **CPU affinity:** keep a thread on the core where its data is already cached.

Throughput and latency pull in opposite directions: fewer switches = more throughput but worse responsiveness. The scheduler is a balancing act.

### Linux CFS (Completely Fair Scheduler)

This is the default scheduler for ordinary Linux processes. The goal in one sentence: **give every runnable thread an equal share of CPU over time**, as if they were all running simultaneously on an idealized machine.

**The core idea: virtual runtime (`vruntime`).** Each thread has a counter, `vruntime`, that measures *how much CPU time it has already received*. While a thread runs, its `vruntime` increases; while it waits, its `vruntime` stays still. The scheduler's rule is simple:

> Always run the thread with the **lowest** `vruntime` (the one that has had the *least* CPU so far). When another thread's `vruntime` falls below the running one's, switch to it.

This naturally equalizes CPU over time — whoever is "behind" gets to run until it catches up, then yields to whoever is now behind. "Fair" falls out of "always serve the most-starved."

**Worked example.** Three equal-priority threads A, B, C, all CPU-hungry:

```
Start:   A.vr=0   B.vr=0   C.vr=0      (tie -> pick A)
Run A for one slice (say it gains 6 ms of vruntime):
         A.vr=6   B.vr=0   C.vr=0      (lowest is B -> run B)
Run B:
         A.vr=6   B.vr=6   C.vr=0      (lowest is C -> run C)
Run C:
         A.vr=6   B.vr=6   C.vr=6      (tie -> back to A)
...
```

They round-robin, each accumulating CPU at the same rate. Over time all three get a third of the CPU.

**Priority (niceness).** A higher-priority (lower nice value) thread should receive *more* CPU. CFS implements this not with a separate queue but by making `vruntime` advance **more slowly** for high-priority threads. If A is high priority, 6 ms of real CPU might add only 3 ms to A's `vruntime`. A therefore stays "behind" longer and is picked more often, earning more real CPU to reach the same virtual time. Priority becomes a *rate multiplier on virtual time*, not a hard override.

**Why it doesn't starve a sleeping thread.** When a thread that was blocked (e.g., waiting on the network) wakes up, its `vruntime` is far behind the others (it didn't advance while sleeping). So it immediately becomes the lowest — and gets scheduled right away. This is why interactive tasks (mostly sleeping, occasionally woken by input) feel snappy: they're almost always the most-behind thread when they wake. CFS clamps how far behind a waking thread can be, so it can't hoard CPU after a long sleep.

**The data structure.** "Find the lowest `vruntime`, repeatedly, while inserting and removing threads" is exactly what a balanced binary search tree does well. CFS keeps runnable threads in a **red-black tree keyed by `vruntime`**; the leftmost node is always the next to run. Picking, inserting, and removing are all O(log N).

> Note: newer Linux kernels (6.6+) replace CFS with **EEVDF** (Earliest Eligible Virtual Deadline First), which adds an explicit latency dimension so latency-sensitive tasks can be favored more precisely. The `vruntime`/fairness intuition above still applies; EEVDF refines *when* a behind-but-not-yet-eligible task is allowed to run.

### Real-Time Scheduling

For tasks with hard deadlines (audio, robotics, networking). Two policies:

- **SCHED_FIFO:** runs until it yields or a higher-priority task preempts.
- **SCHED_RR:** like FIFO but with round-robin among same-priority tasks.

Real-time tasks preempt all SCHED_OTHER tasks. Misusing them can starve the rest of the system.

### Priorities (Nice / Niceness)

Niceness ranges from -20 (highest priority) to +19 (lowest). Default 0. A higher nice value means "nicer to others" (lower priority). `nice` and `renice` set it.

### CPU Affinity

Bind a thread to specific CPUs. Avoids cache-cold migrations between cores.

```bash
taskset -c 0-3 ./my-program   # restrict to CPUs 0-3
```

Most workloads don't need explicit affinity — CFS does a reasonable job of keeping tasks where their cache is warm. Real-time or HPC workloads pin manually.

### Context Switching

A **context switch** is the act of taking one thread off a CPU core and putting another on. The "context" is everything that makes the CPU *be* that thread: its register values, its program counter (the next instruction), its stack pointer. To switch, the kernel must save all of that for the outgoing thread and restore it for the incoming one — otherwise the incoming thread would resume with the wrong register contents and crash.

```
1. Save outgoing thread's CPU state (registers, PC, SP) into its kernel structure.
2. Update bookkeeping (vruntime, run queues, accounting).
3. If switching to a different PROCESS (not just thread), load the new page-table
   base register -> the CPU now sees a different virtual address space.
4. Restore the incoming thread's saved CPU state.
5. Resume — the CPU is now running the new thread as if it never stopped.
```

**Why it's more expensive than those five steps suggest.** The *direct* cost (saving/restoring registers) is small — a microsecond or so. The real cost is *indirect* and invisible in the step list:

- **Cache pollution.** The outgoing thread had its hot data in L1/L2 cache. The incoming thread evicts it. When the first thread runs again, its cache is cold and every access misses to slower memory. The "switch" was fast; the slow part is the hundreds of cache misses that follow.
- **TLB flush.** Switching to a *different process* means a different page table, so the cached virtual→physical translations (the TLB, see section 5) are now wrong and get flushed. The new process then takes a burst of TLB misses, each a full page-table walk. (Switching between threads of the *same* process avoids this — same address space.)
- **Branch predictor reset.** The CPU's learned branch history no longer applies.

A context switch can therefore cost far more than the microsecond of bookkeeping, once the cache and TLB rebuild are included. This is why **excessive context switching destroys throughput** even though each switch appears cheap, and why pinning a thread to one core (affinity) and switching between threads of the same process (cheaper than between processes) both help. The `cs` column in `vmstat` reflects this; a runaway count signals thrashing.

Context switches happen on:

- Timeslice expiry (the periodic timer interrupt fires and the scheduler reconsiders).
- A higher-priority / more-behind thread becoming runnable.
- The current thread blocking (a syscall that waits — I/O, a lock, `sleep`).
- A voluntary yield.

### NUMA (Non-Uniform Memory Access)

On multi-socket systems, each CPU socket has its own memory bank. Local access is fast; remote (cross-socket) access is slower.

```
+------+    +------+
| CPU0 |----| CPU1 |  <-- inter-socket link (QPI/UPI)
+------+    +------+
   |          |
+------+    +------+
| RAM0 |    | RAM1 |
+------+    +------+

CPU0 -> RAM0:  ~100 ns
CPU0 -> RAM1:  ~150 ns (50% slower)
```

NUMA-aware code:

- Allocates memory on the local node (`numa_alloc_local` or kernel default first-touch policy).
- Pins threads to NUMA nodes (`numactl --cpunodebind=0 --membind=0`).
- Avoids data sharing across nodes.

Most non-HPC workloads aren't deeply NUMA-tuned but suffer when a workload's data lives on the wrong node.

### Latency-Sensitive Workloads

For low-latency systems (high-frequency trading, real-time audio, kernel bypass networking):

- Pin threads to cores (`isolcpus` kernel boot parameter excludes a core from CFS).
- Disable CPU frequency scaling (constant clock).
- Disable hyperthreading on dedicated cores.
- Run with `SCHED_FIFO` or `SCHED_RR`.
- Avoid syscalls in the hot path (DPDK, io_uring).

---

## 5. Virtual Memory

**The problem.** If programs used physical memory addresses directly, two problems appear. First, two programs would fight over the same addresses — program A writes to address 0x1000, program B writes to address 0x1000, they corrupt each other. Second, a program could read or write any other program's (or the kernel's) memory, with no isolation. And a third, practical problem: a program would need to know, at compile time, exactly which physical addresses are free — impossible, since that depends on what else is running.

**The solution: a layer of indirection.** Every process receives its own private **virtual address space** — it sees memory as a clean, contiguous range of addresses starting from (near) zero, as if it owned the whole machine. These virtual addresses are *not* physical addresses. On every memory access, hardware called the **MMU (Memory Management Unit)** translates the virtual address into the physical address where the data resides in RAM.

```
Program uses:          MMU translates:        Physical RAM:
  virtual 0x4000   -->   (lookup)        -->   physical 0x7A000
  virtual 0x4001   -->   (lookup)        -->   physical 0x7A001
```

This indirection buys three things at once:

- **Isolation:** process A's virtual 0x4000 and process B's virtual 0x4000 map to *different* physical locations. They can't see each other. A bad pointer in A can't touch B.
- **Flexibility:** the program doesn't care where in physical RAM it ends up; the OS decides and can move it.
- **Illusion of more memory than exists:** a virtual page can be backed by disk instead of RAM (swap), or not exist at all until touched (demand paging).

### Pages and Page Tables

**Why pages.** Translating *every individual byte* address would need an impossibly large translation table. So memory is divided into fixed-size chunks called **pages** (4 KB on x86, with optional larger 2 MB / 1 GB "huge pages"). Translation happens per-page, not per-byte: figure out which physical **frame** (a page-sized slot in RAM) a virtual page maps to, then the low bits (the offset within the page) carry over unchanged.

```
A virtual address splits into:  [ page number | offset within page ]

  Translate the page number (table lookup) -> physical frame number
  Keep the offset unchanged

  virtual  0x4ABC  =  page 0x4, offset 0xABC
  if page 0x4 -> frame 0x7A, then physical = 0x7AABC  (offset preserved)
```

The **page table** is the lookup table doing this: virtual page number → physical frame number (or "not present").

```
Virtual page 0  -> Physical frame 17
Virtual page 1  -> Physical frame 42
Virtual page 2  -> (not present)         <-- accessing this triggers a page fault
Virtual page 3  -> Physical frame 9
```

Each process has its own page table, which is why each sees a different physical reality for the same virtual address. A context switch to a different process loads that process's page table (by pointing the CPU's page-table base register at it).

**Why the table is multi-level (hierarchical).** A 48-bit address space holds 2^36 pages. A flat table with one entry per page would be enormous — gigabytes of table per process — and almost entirely empty, because a process uses only a tiny scattered fraction of its address space. The fix: a **tree of tables**. Upper levels exist only for the regions actually in use; unused branches are simply absent. x86-64 uses 4 levels (named PML4 → PDPT → PD → PT). The 48-bit address is sliced into four 9-bit indices (one per level) plus a 12-bit offset:

```
Virtual address (48 bits), sliced into table indices:

  +-------+-------+-------+-------+----------------+
  | PML4  | PDPT  |  PD   |  PT   |  page offset   |
  |  9b   |  9b   |  9b   |  9b   |     12b        |
  +-------+-------+-------+-------+----------------+
  (each 9-bit field picks one of 512 entries in a table; 2^12 = 4 KB page)

Walking it (a "page-table walk") to translate virtual 0x0000_0000_0040_2ABC:

  1. CPU reads the page-table base register -> address of the PML4 table.
  2. PML4[ bits 47-39 ]  -> physical address of the next table (PDPT).
  3. PDPT[ bits 38-30 ]  -> physical address of the next table (PD).
  4. PD[   bits 29-21 ]  -> physical address of the next table (PT).
  5. PT[   bits 20-12 ]  -> the physical FRAME number for our page.
  6. final physical address = frame << 12 | offset(bits 11-0).
```

The key insight: one address translation requires **four memory reads to walk the tree**, before the requested data is accessed at all. This is expensive, which is why the TLB exists.

### TLB (Translation Lookaside Buffer)

Walking the page table on every memory access (four extra reads each) would make every load and store five times slower. The CPU avoids this with the **TLB** — a small, very fast hardware cache that remembers recent virtual-page → physical-frame translations.

```
Memory access:
  virtual address
       |
       v
  TLB hit?  --yes-->  use cached physical frame (≈ free, ~1 cycle)
       |
       no (TLB miss)
       v
  walk the page table (4 memory reads), then cache the result in the TLB
```

A TLB hit makes translation essentially free; a TLB miss costs a full page-table walk. Real programs hit the TLB the vast majority of the time *because* they access memory with locality (nearby addresses, same pages).

**Why this connects to context switches.** Each process has its own page table, so its translations are only valid for it. When the CPU switches to a different process, the cached translations in the TLB are now wrong and must be discarded (**TLB flush**). The new process then suffers a burst of TLB misses (full walks) until the TLB refills. This is a big chunk of the *indirect* cost of a context switch discussed in section 4. (Tagged TLBs / PCIDs let the hardware keep entries for multiple processes to soften this.)

**Huge pages** help here: a 2 MB page means one TLB entry covers 2 MB of memory instead of 4 KB, so a program touching large amounts of memory needs far fewer TLB entries to map it, producing fewer misses. Databases and JVMs with large heaps often enable huge pages for this reason.

### Page Faults

A **page fault** is what happens when the MMU tries to translate a virtual address and the page table says the page isn't currently mapped to a frame (or is mapped but with the wrong permissions). The CPU can't proceed, so it **traps into the kernel** to sort it out. Crucially, "page fault" is not always an error — it's often the normal mechanism by which memory gets loaded lazily.

**Minor (soft) fault — cheap, common, not an error.** The data is already in physical RAM, it just isn't mapped into *this* process's page table yet. The kernel only has to add the page-table entry. Examples: a shared library already in RAM for another process (just point this process's table at the same frames — COW), or the first touch of freshly-allocated memory the kernel can satisfy from a zeroed page. Microseconds.

**Major (hard) fault — expensive.** The data isn't in RAM at all; it must be fetched from disk. Examples: the page was swapped out, or it's part of a memory-mapped file not yet read in. The kernel issues disk I/O and blocks the thread until it completes. **Milliseconds** — thousands of times slower than a minor fault. A process taking many major faults per second is thrashing; that's the symptom of "working set doesn't fit in RAM."

**Segmentation fault — an actual error.** The address has no valid mapping *and* no legitimate reason to (a null-pointer dereference, a wild pointer, writing to read-only code). The kernel can't fix it, so it sends **SIGSEGV**, which by default kills the process. This is the page-fault mechanism reporting a genuine bug rather than doing lazy loading.

```
CPU accesses an address whose page isn't mapped
        |
        v
   trap to kernel (page fault handler)
        |
   why is it unmapped?
        |
        +-- valid, data in RAM elsewhere   -> minor fault: fix page table (µs)
        +-- valid, data on disk            -> major fault: disk I/O, then map (ms)
        +-- not a valid address            -> SIGSEGV (kill the process)
```

### Memory Layout of a Process

```
High addresses
  +-------------------+
  |  command line +   |
  |  environment vars |
  +-------------------+
  |                   |
  |       Stack       |   grows down
  |        |          |
  |        v          |
  +-------------------+
  |                   |
  |   (free region)   |
  |                   |
  +-------------------+
  |        ^          |
  |        |          |
  |       Heap        |   grows up (via brk(), mmap())
  |  (malloc'd mem)   |
  +-------------------+
  |   .bss (uninit)   |   zero-initialized
  +-------------------+
  |  .data (init)     |   read-write globals
  +-------------------+
  |  .rodata          |   read-only constants
  +-------------------+
  |  .text (code)     |   read-only executable
  +-------------------+
Low addresses
```

### mmap and Memory-Mapped Files

`mmap()` maps a file (or anonymous memory) into the process's address space. Reads from the mapped region are demand-paged — the kernel loads pages on first access.

```c
int fd = open("data.bin", O_RDONLY);
size_t size = file_size(fd);
char *data = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);

// Use data[i] like an array. Kernel pages in as needed.

munmap(data, size);
close(fd);
```

**Advantages:**

- No explicit read syscall in the hot path.
- Pages cached automatically (via the page cache).
- Multiple processes can map the same file and share physical pages.
- Suitable for large files where loading all at once is wasteful.

**Used by:** databases (LMDB, SQLite's WAL), search engines (Lucene), zero-copy I/O paths, dynamic linker (loads .so files via mmap).

**Anonymous mmap** (no backing file) is how large allocations bypass the heap. `malloc()` for large sizes uses mmap rather than `brk()` so memory can be returned to the OS on free.

### Swap

When physical RAM is exhausted, the kernel evicts cold pages to disk (swap space). The page table entry becomes "not present"; on access, a page fault loads it back.

Swapping is very slow — milliseconds vs nanoseconds for RAM. **Heavy swapping = service is dead.**

Modern systems often disable swap on production servers, preferring OOM kill (kernel kills the largest memory hog) over thrashing.

### OOM Killer

When the kernel runs out of memory and cannot reclaim, it selects a process to kill based on an OOM score (heuristic over memory usage, niceness, oom_score_adj). The OOM killer's choice is sometimes surprising — adjust `/proc/<pid>/oom_score_adj` to protect critical processes.

### Common Memory Issues

| Symptom | Likely cause |
|---------|--------------|
| Process grows over time | Memory leak |
| `RSS` much smaller than `VSZ` | Lots of mapped but untouched memory (normal) |
| Heavy swap I/O | Working set exceeds RAM |
| Sudden OOM kill | Allocation spike or another process consuming memory |
| High `Major page faults/sec` | Working set doesn't fit in RAM; paging from disk |

`ps -o pid,rss,vsz,cmd` shows resident vs virtual sizes. `/proc/<pid>/status` has more detail.

---

## 6. The Page Cache and File I/O

The OS aggressively caches file content in RAM (the page cache). Most reads are served from cache without hitting disk; writes go to cache and are flushed asynchronously.

### How It Works

```
Application calls read(fd, buf, size):
  1. Kernel checks: are these pages in the page cache?
     - Yes: copy from cache to buf. Return.
     - No:  issue disk I/O, populate cache, then copy to buf.

Application calls write(fd, buf, size):
  1. Kernel writes to page cache (dirty page).
  2. Marks pages dirty. Returns to application.
  3. Background flusher (pdflush / writeback) flushes dirty pages to disk
     periodically and under memory pressure.
```

The page cache uses all available free RAM. `free -h` reports it as "buff/cache" separately from "used."

### fsync()

If durability is required (the commit must survive a crash), `fsync(fd)` blocks until the file's dirty pages are physically on disk.

```c
write(fd, data, size);   // returns immediately (page cache)
fsync(fd);                // blocks until on disk (slow, milliseconds)
```

Skipping fsync trades durability for speed. Databases batch fsyncs (group commit) to amortize.

`fdatasync()` is similar but skips metadata not required for durability — slightly faster.

### O_DIRECT

Bypasses the page cache. Reads come directly from disk; writes go directly to disk. Used by databases that maintain their own buffer cache (Oracle, MySQL InnoDB optionally) to avoid double-caching.

**Pros:** predictable behavior, no cache pollution, no double-buffering overhead.
**Cons:** alignment requirements (typically 512-byte or page-aligned buffers), much slower if the same data is re-read, no readahead.

Postgres deliberately uses the page cache rather than O_DIRECT, trusting Linux to cache well.

### Synchronous vs Asynchronous I/O

**Blocking I/O:** thread waits in the kernel during read/write. One thread per concurrent operation.

**Non-blocking I/O (O_NONBLOCK):** read/write returns immediately with EAGAIN if data isn't ready. Combined with `poll`/`epoll`/`kqueue` to wait for many descriptors.

**Async I/O:**

- POSIX AIO: largely ignored; Linux implementation is poor.
- Linux `io_uring`: modern, high-performance, batched async I/O. Used by databases, network proxies, high-perf applications.

```
Old async I/O on Linux: epoll + non-blocking sockets (for network)
                         no good async for file I/O
                         
io_uring (5.1+): submission and completion queues shared with kernel.
                 Submit many ops without syscalls, reap completions.
```

### Readahead

Sequential reads are detected by the kernel, which prefetches subsequent blocks. Random reads have no readahead.

Sequential file I/O is 10-100x faster than random I/O on HDD, 2-10x faster on SSD. Database storage engines often arrange data for sequential access for this reason.

---

## 7. Filesystems

A filesystem is the on-disk layout of files and metadata, plus the kernel code that interprets it.

### Common Filesystems

| Filesystem | Notes |
|-----------|-------|
| ext4 | Default Linux for many years. Journaled, mature, reliable. |
| XFS | High-performance, scales to very large filesystems. Common in databases. |
| btrfs | Copy-on-write, snapshots, checksums. Mature for desktop, mixed for production. |
| ZFS | Copy-on-write, RAID-Z, snapshots, send/receive. Heavy memory user. Tier-1 on FreeBSD/Solaris; Linux via DKMS or experimental. |
| tmpfs | In-memory filesystem. `/dev/shm`, `/tmp` on many systems. |
| ntfs / exfat | Windows-compatible. |
| f2fs | Optimized for flash storage. Mobile devices. |

### Inodes and Files

A file consists of:

- An **inode** (metadata: size, owner, permissions, timestamps, block pointers).
- **Data blocks** (the file's content).
- One or more **directory entries** (names that point to the inode).

Multiple directory entries pointing to the same inode = **hard links**. The file's content exists as long as at least one link refers to its inode. `unlink()` removes a link; the file is freed only when the last link is gone AND no process has it open.

**Symlinks** are different: they're inodes that contain a path string interpreted at every access. Symlinks can dangle (point to nonexistent paths); hard links cannot.

### Journaling and Crash Consistency

Modifying a file involves updating multiple structures (inode, data blocks, directory). If the system crashes between updates, the filesystem can be inconsistent.

**Journaling** writes a log of pending changes; on recovery, replay or discard incomplete operations.

- **Metadata journaling (ext4 default):** journals metadata only. Data is written separately; partial writes possible but not in a way that corrupts the FS structure.
- **Data journaling:** journals both — slower but stronger consistency.

**Copy-on-write filesystems** (btrfs, ZFS) write new versions to new blocks and atomically swap pointers. No journal needed; crash consistency comes from the write-then-swap pattern.

### Permissions

Classic Unix permissions: owner, group, other × read, write, execute.

```
-rw-r--r--  1 alice  staff  1024  Jan 15  10:30  file.txt
^|||^|||^||
|   |   ||
|   |   +-- other:  r--   (4)
|   +------ group:  r--   (4)
+---------- owner:  rw-   (6)
              ^
              file type (- regular, d dir, l symlink, c char dev, b block dev, s socket, p pipe)
```

`chmod 644 file` = rw-r--r-- (6=rw, 4=r, 4=r).

**Special bits:**

- setuid (`s` on user x): execute with the file owner's UID.
- setgid (`s` on group x): execute with file group's GID; on dirs, new files inherit dir's group.
- sticky bit (`t` on other x): on dirs, only owner can delete files. Used on `/tmp`.

**ACLs (`getfacl`, `setfacl`)** extend basic permissions with per-user/per-group rules. Less commonly used but standard on modern filesystems.

**Capabilities** (Linux): split root's privileges into fine-grained capabilities (CAP_NET_BIND_SERVICE, CAP_SYS_ADMIN, etc.). A binary can be granted only the capabilities it needs (`setcap`) without setuid-root.

### Mounting

A filesystem becomes accessible by being **mounted** at a directory. The mount point's previous contents are hidden until unmounted.

```bash
mount /dev/sdb1 /mnt/data
umount /mnt/data
```

`/etc/fstab` configures persistent mounts. `findmnt` and `mount | column -t` show the current state.

**Bind mounts** mount an existing directory at another path. Used by containers and chroot environments.

---

## 8. System Calls

**The problem.** User programs run in the unprivileged CPU mode (ring 3, section 1). They physically *cannot* execute privileged instructions, touch hardware, or read another process's memory — the CPU forbids it. But programs constantly need operations only the kernel can perform: open a file, send a network packet, allocate memory, create a process. The challenge is allowing unprivileged code to request privileged operations safely, without permitting it to "call" into arbitrary kernel code.

**The solution: a controlled doorway — the system call.** There is exactly one sanctioned way to cross from user mode to kernel mode: a special CPU instruction (`syscall` on x86-64) that **traps** — it simultaneously switches the CPU to kernel mode *and* jumps to a single, fixed entry point that the kernel set up at boot. The program can't choose where in the kernel it lands; it always arrives at the kernel's syscall dispatcher, which then validates the request. This is the whole security model: user code can *request* privileged operations but can only enter the kernel through this one guarded door.

**How a call flows, step by step.** Take `read(fd, buf, count)`:

```
USER MODE                                    KERNEL MODE
---------                                    -----------
1. Put the syscall NUMBER in a register
   (e.g., rax = 0 for read on x86-64)
2. Put arguments in registers
   (rdi=fd, rsi=buf, rdx=count)
3. Execute the `syscall` instruction  ---->  4. CPU switches to ring 0, jumps to
                                                the kernel's fixed entry point.
                                             5. Dispatcher looks up syscall #0 -> sys_read
                                             6. Kernel VALIDATES: is fd valid? is buf a
                                                writable address this process owns? (this
                                                check is why a bad pointer gives EFAULT,
                                                not a kernel crash)
                                             7. Kernel does the work (maybe blocks on I/O)
                                             8. Puts the return value in rax
   4. Resume in user mode with        <----  9. Executes `sysret` -> switch back to ring 3
       the result in rax
```

The program never touches the disk or the file descriptor table directly — it asks, the kernel validates and acts. Note step 6: the kernel treats every user-supplied pointer and value as untrusted and checks it, which is what keeps a buggy or malicious program from corrupting the kernel.

In practice the register-loading is not written by hand — the C library (`libc`) wraps each syscall in an ordinary function (`read()`, `open()`), and *that* function contains the `syscall` instruction. Thus "calling `read()`" is in effect "calling a thin libc wrapper that performs the syscall."

### Categories

```
Process:      fork, execve, wait, exit, kill, getpid, setuid
Memory:       brk, mmap, munmap, mprotect
Files:        open, read, write, close, lseek, stat, dup, fcntl
Directories:  mkdir, rmdir, link, unlink, rename, getdents
FS:           mount, umount, sync, fsync
IPC:          pipe, socket, bind, accept, connect, send, recv, shmget
Signals:      signal, sigaction, kill, sigprocmask
Time:         time, gettimeofday, clock_gettime, nanosleep
Polling:      select, poll, epoll_*, io_uring_*
```

`man 2 <name>` documents each syscall.

### Cost of Syscalls

Crossing the user/kernel boundary isn't free. Each syscall pays for: the mode transition itself (saving user state, switching privilege level, and back), the kernel's argument validation, and — like a context switch — indirect costs from cache and branch-predictor disruption, plus on modern CPUs the overhead of Spectre/Meltdown mitigations that flush state on kernel entry. The total is on the order of **hundreds of nanoseconds to a few microseconds** per call — cheap individually, ruinous in a hot loop that does millions.

The optimization principle is therefore: **do more work per syscall, or avoid the boundary entirely.**

Examples of avoiding syscall overhead:

- Batch reads with larger buffers.
- Use `writev()` to write multiple buffers in one call.
- `io_uring` to submit many operations with few syscalls.
- `sendfile()` to copy file contents to a socket without userspace involvement.
- `mmap()` to read a file without read() calls.
- `vDSO`: the kernel exports some functions (like `gettimeofday`) as ordinary user calls, no syscall needed.

### strace and ltrace

`strace -p <pid>` traces syscalls. Invaluable for debugging "why is this program stuck?"

```
$ strace -c -p 12345    # summary by syscall
$ strace ./prog          # syscalls of new process
$ strace -e openat,read,write ./prog   # only specific syscalls
$ strace -f ./prog       # follow forks/threads
```

`ltrace` traces library calls (e.g., `malloc`, `printf`).

### Errors

Most syscalls return -1 on failure with `errno` set:

```c
fd = open("file", O_RDONLY);
if (fd < 0) {
    perror("open");        // prints "open: No such file or directory"
    fprintf(stderr, "errno: %d\n", errno);
}
```

Common errnos: ENOENT (not found), EACCES (permission denied), EAGAIN (would block), EINTR (interrupted by signal), ENOMEM, EBADF (bad fd), EPIPE (write to closed pipe).

---

## 9. IPC (Pipes, Sockets, Shared Memory)

Mechanisms for processes to communicate.

### Pipes

A unidirectional byte stream. One process writes, another reads.

```bash
ls | grep .txt | wc -l
```

The shell creates two pipes; each child's stdout/stdin is wired to a pipe. Data flows from `ls` to `grep` to `wc`.

`pipe()` syscall creates a pipe pair; `fork()` shares the file descriptors.

**Named pipes (FIFOs):** `mkfifo` creates a pipe that exists as a filesystem entry, usable by unrelated processes.

### Unix Domain Sockets

Like network sockets but local to one machine. Faster than TCP (no protocol overhead), supports streaming (SOCK_STREAM) and datagrams (SOCK_DGRAM), can pass file descriptors between processes.

Used for: docker.sock, X11, postgres unix socket, systemd socket activation.

### TCP/UDP Sockets

Network sockets — full treatment in `Networking Fundamentals.md`.

Syscalls: `socket`, `bind`, `listen`, `accept`, `connect`, `send`, `recv`, `close`.

### Shared Memory

Two processes map the same region of memory. Reads and writes by either are visible to the other immediately.

**POSIX shm:** `shm_open`, `mmap`. Persists in `/dev/shm` (tmpfs).
**SysV shm:** `shmget`, `shmat`. Older API, still used.

Fastest IPC (just memory). Requires synchronization (mutexes, semaphores) to coordinate access.

### Message Queues, Semaphores

POSIX and SysV provide named message queues and semaphores. Less common in modern code — most apps use sockets or shared memory + futexes.

### Choosing IPC

| Need | Use |
|------|-----|
| Stream of bytes, parent-child | Pipe |
| Unrelated processes, file-like API | Named pipe (FIFO) or Unix socket |
| Network-style API, same host | Unix socket |
| High-throughput shared state | Shared memory + synchronization |
| Network communication | TCP/UDP socket |
| Pass file descriptors | Unix socket with SCM_RIGHTS |

---

## 10. Signals

Asynchronous notifications delivered to a process. The kernel or another process sends a signal; the receiving process either handles it or executes a default action.

### Common Signals

| Signal | Default | Meaning |
|--------|---------|---------|
| SIGINT (2) | Terminate | Interrupt (Ctrl+C) |
| SIGQUIT (3) | Core dump | Quit + core (Ctrl+\ ) |
| SIGILL (4) | Core dump | Illegal instruction |
| SIGABRT (6) | Core dump | abort() |
| SIGFPE (8) | Core dump | Floating-point exception |
| SIGKILL (9) | Terminate | Cannot be caught or ignored |
| SIGSEGV (11) | Core dump | Segmentation fault |
| SIGPIPE (13) | Terminate | Write to closed pipe |
| SIGALRM (14) | Terminate | alarm() timer |
| SIGTERM (15) | Terminate | Polite termination request |
| SIGCHLD (17) | Ignore | Child stopped/terminated |
| SIGCONT (18) | Continue | Continue if stopped |
| SIGSTOP (19) | Stop | Cannot be caught or ignored |
| SIGTSTP (20) | Stop | Terminal stop (Ctrl+Z) |
| SIGUSR1, SIGUSR2 | Terminate | User-defined |
| SIGHUP (1) | Terminate | Hangup; conventionally "reload config" |

### Handling Signals

```c
#include <signal.h>

void handler(int sig) {
    // careful: only async-signal-safe functions allowed here
    write(STDERR_FILENO, "got signal\n", 12);
}

// `sigaction` is defined by the system headers (`<signal.h>`)
struct sigaction sa = { .sa_handler = handler }; 
sigemptyset(&sa.sa_mask); // Initializes the signal mask to empty
sigaction(SIGINT, &sa, NULL);
```

**Async-signal safety:** signals interrupt arbitrary code, including code holding locks. Only a specific list of functions is safe to call from a handler (`write`, `_exit`, `signal`, atomic operations). Calling `printf` or `malloc` from a signal handler can deadlock.

**Best practice:** signal handlers set a flag (or write to a self-pipe); the main loop checks the flag in a safe context.

### Signal Masks

A process can block (queue) certain signals temporarily with `sigprocmask`. Useful for critical sections that must not be interrupted.

`SIGKILL` and `SIGSTOP` cannot be blocked or caught — they always succeed.

### Common Signal Patterns

**Graceful shutdown:** SIGTERM handler sets a "shutting down" flag; main loop drains connections and exits.

**Config reload:** SIGHUP handler triggers re-reading config file.

**Crash dumps:** SIGSEGV / SIGABRT handlers print stack traces and re-raise the signal to get a core dump.

**Killing a stuck process:**

```bash
kill <pid>          # SIGTERM
kill -9 <pid>       # SIGKILL (can't be caught; use as last resort)
kill -HUP <pid>     # reload config (conventional)
```

---

## 11. Compilation, Linking, and Loading

How source code becomes a running program.

```
   source.c
      |
      v
   Preprocessor (cpp)     -- expand #include, #define
      |
      v
   Compiler (cc1)         -- C source -> assembly
      |
      v
   Assembler (as)         -- assembly -> object file (machine code + metadata)
      |
      v
   Linker (ld)            -- combine object files + libraries -> executable
      |
      v
   Loader (kernel + ld.so) -- map executable into memory, set up environment, jump to entry
      |
      v
   Running process
```

### Preprocessing

Textual transformation: macro expansion, header inclusion, conditional compilation.

```c
#include <stdio.h>      // pulls in stdio.h verbatim
#define MAX 100         // replace MAX with 100 in subsequent text

int main(void) {
    printf("max is %d\n", MAX);
}
```

After preprocessing: a large stream of declarations and code, no preprocessor directives.

### Compilation

The compiler turns one source file into assembly for the target CPU, type-checking, optimizing, and selecting machine instructions. It compiles **one file at a time, in isolation**. When `main.c` calls `printf` or a function defined in another file, the compiler does not know where those functions reside; it emits a call to a *name* (a **symbol**) and records that the real address must be supplied later. This is why compilation can proceed per-file in parallel, and why the linker is required afterward to connect the pieces.

### Assembly

The assembler turns the assembly text into actual machine-code bytes, producing an **object file** (`.o`). An object file is not yet runnable — it's machine code plus the bookkeeping needed to combine it with others:

- **Machine code** for this file's functions.
- A **symbol table:** which names this file *defines* (e.g., it contains `my_func`) and which it *references but doesn't define* (e.g., it uses `printf`).
- **Relocations:** a list of "holes" — places in the machine code where an address is currently blank because it depends on where other code/data is placed. Each relocation specifies: at byte offset X, patch in the address of symbol Y once it is known.

### Linking — the Core Idea

**Linking is resolving symbols and filling in relocations.** Each object file references names it does not define; the linker locates where each name is defined (in another object file or a library), then patches every relocation with the real address.

```
main.o:   defines: main
          uses:    helper (??), printf (??)        <- two unresolved symbols
helper.o: defines: helper
          uses:    printf (??)

Linker:
  1. Match every "uses" to a "defines" across all the pieces.
     helper -> found in helper.o.   printf -> found in the C library.
  2. Lay all the code out in one address space, assigning each function a location.
  3. Patch every relocation hole with the now-known address.
  -> a complete program with no blanks left.

If a "uses" has no matching "defines" anywhere:  "undefined reference to X" (link error).
```

This is *why* "undefined reference" errors occur: the linker fails at step 1.

### Static Linking

The linker copies the needed code from **static libraries** (`.a` files — archives of `.o` files) directly into the executable, resolving everything at build time.

```
main.o + helper.o + libc.a  -->  myprogram   (contains its own copy of the libc code it uses)
```

**Result:** one self-contained file. It runs anywhere with no external dependencies, starts fast (nothing to resolve at runtime), but is larger, and every statically-linked program on the machine carries its own copy of libc.

### Dynamic Linking — Why Linking Splits Into Two Phases

The dominant default. Instead of copying library code in, the linker just records *"this program needs `printf` from `libc.so`"* and leaves those symbols unresolved in the executable. Resolution is deferred to **program startup**, done by a special helper, the **dynamic linker** (`ld.so` / `ld-linux.so`).

So linking happens in two phases, at two different times:

```
BUILD time (static linker, ld):
  - resolve references among YOUR object files
  - for library symbols, just record "I need printf from libc.so" + leave a stub
  - record the path to the dynamic linker (PT_INTERP)

RUN time (dynamic linker, ld.so, runs before main()):
  1. Kernel loads myprogram, sees it has an "interpreter" (PT_INTERP) -> loads ld.so first
  2. ld.so maps the needed .so files (libc.so, ...) into the address space (via mmap)
  3. ld.so resolves the recorded symbols to real addresses in those libraries
  4. ld.so jumps to the program's entry point
```

**Why the runtime phase exists.** Three benefits:

- **Shared memory:** one copy of `libc`'s code in physical RAM is mapped into *every* process that uses it (read-only code pages are shared — see COW in section 2). Statically, every program would carry its own copy.
- **Smaller executables:** the binary does not contain libc.
- **Update once:** patching a security bug in `libc.so` benefits every dynamically-linked program on restart, with no recompilation.

**The costs:** slower startup (the dynamic linker has work to do), and "DLL hell" / version-compatibility issues — the library present at runtime must be compatible with the version built against (managed by symbol versioning, below).

```
ldd myprogram            # list the shared libraries it needs
LD_LIBRARY_PATH=...      # extra directories to search for .so files
LD_PRELOAD=lib.so        # force-load a library first, to override/intercept functions
```

---

## 12. ELF and Dynamic Linking

The Linux executable format is **ELF (Executable and Linkable Format)**. Both executables and shared libraries are ELF.

### ELF Layout

```
ELF Header                       <-- magic bytes, architecture, entry point
Program Headers                  <-- load directives for the kernel/loader
  PT_LOAD          .text segment (executable)
  PT_LOAD          .data segment (writable)
  PT_INTERP        path to dynamic linker (e.g., /lib64/ld-linux-x86-64.so.2)
  PT_DYNAMIC       dynamic linking info
Section Headers                  <-- finer-grained info, used by linker
  .text            code
  .rodata          read-only data (string literals, const)
  .data            initialized data
  .bss             uninitialized data (zero-filled at load)
  .symtab          symbol table
  .strtab          string table
  .rela.dyn        relocations
  .got, .plt       global offset table, procedure linkage table (for dynamic symbols)
  .interp          dynamic linker path
  .debug_*         debug info (DWARF)
```

### How a Dynamic Call Works (PLT and GOT)

The mechanism is built up in stages below.

**The problem.** A program calls `printf`, but `printf` resides in `libc.so`, which the dynamic linker maps into memory at startup — and, due to ASLR (below), at *a different address each run*. The compiler therefore cannot emit `call 0x7f...`, because that address is unknown at compile time and changes each run. Fixed machine code must call a function whose address is not known until the program is already running.

**A rejected approach.** The dynamic linker could scan all code at startup and patch every call instruction with the resolved address. But this would modify code pages (preventing read-only sharing between processes) and resolve *every* symbol up front, including those never called — too slow and too wasteful.

**The real answer: one level of indirection — a table of function pointers.** Instead of patching the call sites, the program calls *through a table slot*. The slot holds the function's address; resolving a symbol means writing its address into the slot — touching one data location, not rewriting code.

Two structures cooperate:

- **GOT (Global Offset Table):** an array of pointers in writable data. One slot per external symbol. This is where the *real, resolved address* gets written. Being data, it can be patched without modifying code pages.
- **PLT (Procedure Linkage Table):** a small block of code stubs, one per external function. Your code calls the stub; the stub jumps to whatever address is in the corresponding GOT slot.

**The first call vs later calls (lazy binding).** To avoid resolving unused symbols, the GOT slot initially points *back into the PLT stub*, at code that calls the dynamic linker to perform resolution on demand.

```
First call to printf:
  caller               call printf@plt           (call the stub)
  printf@plt stub      jump to *GOT[printf]      (GOT still points back here:
                                                   "not resolved yet" path)
                       -> push info, call the dynamic linker (ld.so)
  ld.so                find printf in libc.so -> WRITE its real address into GOT[printf]
                       -> jump to printf
  printf runs.

Every later call to printf:
  caller               call printf@plt
  printf@plt stub      jump to *GOT[printf]      (GOT now holds printf's real address)
                       -> jumps straight to printf.   No ld.so, no overhead.
```

So the *first* call pays a one-time resolution cost and patches the GOT; *every subsequent* call is just an indirect jump through a now-filled pointer — effectively free. The first time "teaches" the GOT slot; after that the PLT stub is a trivial forwarder.

**Lazy vs eager.** The above is **lazy binding** — each symbol is resolved the first time it is called, spreading the cost and skipping unused symbols. Setting `LD_BIND_NOW=1` (or building with full RELRO) forces **eager binding**: everything is resolved at startup. Startup is slower, but the GOT is then read-only, which is more secure (an attacker cannot overwrite a GOT slot to hijack a call).

### Position-Independent Code (PIC)

The PLT/GOT trick handles *calls into libraries*. PIC solves the related problem for the library's (and program's) **own** code and data: a shared library can be loaded at any base address (different per process, randomized per run), so its code can't contain hardcoded absolute addresses for its own functions and globals.

**The fix:** instead of "load from absolute address 0x404000", PIC emits "load from *here + a fixed offset*" (PC-relative addressing), and routes accesses to global data through the GOT. Because everything is expressed relative to the current location, the whole module works correctly no matter where it's mapped — nothing needs patching when the base address changes.

**Why everything is PIC now: ASLR.** Address Space Layout Randomization deliberately loads code, libraries, stack, and heap at random addresses every run, so an attacker exploiting a memory bug can't reliably "jump to a known function at a known address." PIC is what makes randomizing the main program's load address possible — a position-independent executable (**PIE**) can be placed anywhere. This is why PIE is the default for new compilations: it's a foundational exploit-mitigation, with a small performance cost from the extra indirection.

### Symbol Versioning

Libraries can export multiple versions of a symbol for compatibility:

```
libc.so has:
  printf@GLIBC_2.2.5     (old behavior)
  printf@GLIBC_2.17      (current)
```

Programs link against a specific version. Lets libc evolve without breaking old binaries.

### Static vs Dynamic in Practice

| Aspect | Static | Dynamic |
|--------|--------|---------|
| Binary size | Larger | Smaller |
| RAM (shared) | Each process has its own copy | One libc in RAM for all processes |
| Startup | Faster | Slower (dynamic linker work) |
| Updates | Recompile everything | Update one .so, restart |
| Deployment | Single file | Need to ship libraries |
| Go default | Yes (mostly) | No |
| Rust default | No (mostly) | Yes (relies on glibc/system libs) |
| C/C++ default | No | Yes |

Containers and immutable images sometimes prefer fully static binaries (musl-linked Rust/Go) for reproducibility.

---

## 13. Boot and Init

How a Linux system goes from power-on to a login prompt.

```
1. Firmware (BIOS/UEFI) initializes hardware, finds bootable disk
2. Bootloader (GRUB, systemd-boot) loaded from disk
3. Bootloader loads kernel + initramfs (initial root filesystem)
4. Kernel decompresses, initializes drivers, mounts initramfs
5. Kernel runs /sbin/init (PID 1)
6. init (typically systemd) starts services in parallel
7. systemd reaches "multi-user" or "graphical" target
8. Login prompt
```

### systemd

The init system on most modern Linux distros. Responsibilities:

- Start services (units) in parallel based on dependency declarations.
- Restart failed services.
- Manage cgroups, namespaces, resource limits per service.
- Provide socket activation (start a service on first connection).
- Handle logging (journald).
- Handle user sessions (logind).

```
systemctl status nginx       # service status
systemctl restart nginx
systemctl enable nginx       # start at boot
journalctl -u nginx          # logs

systemctl list-units --failed
systemctl list-dependencies multi-user.target
```

Service unit files (`/etc/systemd/system/*.service`):

```ini
[Unit]
Description=My App
After=network.target

[Service]
ExecStart=/usr/local/bin/myapp
Restart=on-failure
User=myapp
WorkingDirectory=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

### PID 1's Responsibilities

PID 1 is special:

- It cannot be killed (kernel refuses to deliver fatal signals).
- It reaps orphaned children (otherwise zombies accumulate).
- If PID 1 dies, the kernel panics.

In containers, the entrypoint is PID 1. Many container processes are bad PID 1s — they don't reap children, don't forward signals correctly. Solutions:

- Use `tini` or `dumb-init` as a minimal PID 1 wrapper.
- Use `docker run --init` which provides one automatically.
- Use a real init in the container (systemd, s6) if running multiple processes.

### cgroups, namespaces, capabilities

The kernel primitives behind containers. From an OS perspective: each is a way to restrict or virtualize what a process sees and can do.

- **cgroups:** limit and account for resources (CPU, memory, I/O).
- **namespaces:** isolate views of system resources (process tree, network, mounts, users, hostname, IPC).
- **capabilities:** finer-grained privileges than full root.

A container is a process running with custom cgroups + namespaces + capabilities. No special "container" kernel feature — just a combination of existing primitives.

---

### Tools Worth Knowing

```
ps, top, htop, atop          processes and resources
strace, ltrace               trace syscalls / lib calls
lsof                         list open files
fuser                        which processes have a file open
pmap                         process memory map
free, vmstat                 memory statistics
mpstat, iostat, sar          per-CPU, I/O, comprehensive stats
perf                         CPU profiling, hardware counters
bpftrace                     dynamic tracing via eBPF
ldd, nm, objdump, readelf    binary inspection
gdb, lldb                    debuggers
core(5)                      core dump analysis
dmesg                        kernel ring buffer
journalctl                   systemd logs
/proc/<pid>/*                everything about a process
/sys/                        kernel and device tree
```

### A Mental Model

```
Hardware
   |
Kernel (manages all hardware, schedules processes, enforces protection)
   |
   |   syscall boundary  <-- the only legal way for user code to ask for resources
   |
Userspace (processes, each in its own virtual address space)
   |
Libraries (libc, libstdc++, etc. — bridge between languages and syscalls)
   |
Applications (application code)
```

Every layer adds an abstraction, and every layer has costs. Understanding the layers enables reasoning about why an operation is slow, why a syscall is expensive, and why a context switch is costly. Most production performance work consists of removing or avoiding work at the higher-cost layers.
