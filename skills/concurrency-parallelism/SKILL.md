---
name: concurrency-parallelism
description: "Concurrency vs parallelism for backend services: I/O-bound vs CPU-bound work, threads, the event loop and why blocking it is fatal, race conditions and lost updates (including in single-threaded async code), and the fixes such as atomic updates, locks and optimistic concurrency. Use when choosing a concurrency model, debugging races or lost updates, or writing code that mutates shared state across concurrent requests."
---

# Concurrency and Parallelism in Backends

### The core asymmetry that motivates everything

A backend's job is to juggle many requests at once, and the reason it *can* is that a single request spends most of its life **waiting**, not computing. When you handle a request you might do a DB query, call another service, read a file, write a log — and each of those means your code sits idle while data travels over a wire or a disk head moves. The CPU, meanwhile, is absurdly fast.

One quick calibration: a modern core isn't 3 million instructions/second — it's closer to **3 *billion* cycles/second** (3 GHz), and with superscalar execution it often retires several instructions per cycle. So you're off by a factor of ~1000. That gap matters because it's the whole reason concurrency exists: the CPU is so fast that letting it sit blocked on a 5 ms DB call is like a Formula 1 car idling at a red light. The entire game is keeping it busy with *other* work during those waits.

This splits all backend work into two buckets:

- **I/O-bound** — the bottleneck is waiting on something external: DB calls, network calls, file I/O. The CPU is mostly idle. *Most* web backend work is here.
- **CPU-bound** — the bottleneck is computation itself: hashing passwords, image resizing, parsing huge payloads, running a model. The CPU is pinned.

The reason this distinction is the master key: **the right concurrency model depends entirely on which bucket you're in.** Get this wrong and you either waste cores or block your whole server.

### Concurrency vs. parallelism (the crisp distinction)

These get used interchangeably but they're different:

- **Concurrency** = *dealing with* many things at once — structuring your program so multiple tasks are *in progress*, interleaved. One cook making three dishes by switching between them whenever one is simmering.
- **Parallelism** = *doing* many things at the literal same instant — requires multiple cores. Three cooks, three dishes, simultaneously.

You can have concurrency on a single core (interleaving during waits) and you need multiple cores for true parallelism. A single-threaded Node process is *concurrent but not parallel*. A multi-threaded program on an 8-core box can be both.

Mapping to the buckets: **I/O-bound work wants concurrency** (interleave during the waits, one core is plenty), **CPU-bound work wants parallelism** (the only way to go faster is more cores actually computing).

### Threads — the OS's tool for parallelism

A **thread** is the unit the OS schedules onto a core. There are kernel/OS-level threads (the real ones the scheduler sees) and lighter program-level abstractions on top, but the model that matters is this: a process owns memory, and threads of the *same* process **share that memory** but keep a little private state each.

        `PROCESS  (one address space)
  ┌───────────────────────────────────────┐
  │  CODE   │  GLOBALS / STATIC  │  HEAP   │   ← SHARED by all threads
  └───────────────────────────────────────┘
        ▲                ▲              ▲
     Thread 1         Thread 2       Thread 3
   ┌─────────┐      ┌─────────┐    ┌─────────┐
   │ stack   │      │ stack   │    │ stack   │   ← PRIVATE per thread
   │ registers│     │ registers│   │ registers│
   │ PC       │     │ PC       │   │ PC       │
   └─────────┘      └─────────┘    └─────────┘`

Each thread has its own **stack** (its call frames, local variables), its own **registers**, and its own **program counter** — that's the per-thread execution state. Everything else (the heap, globals, open file descriptors) is **shared**. This is exactly why thread communication is cheap (just read/write shared memory — no copying) *and* exactly why threads are dangerous (two threads can touch the same memory at once).

Two costs to internalize:

- **Creation overhead** — spinning up an OS thread means allocating a stack, kernel bookkeeping, etc. Non-trivial, which is why real systems use **thread pools** instead of one-thread-per-request.
- **Context switching** — when the scheduler swaps thread A off a core for thread B, it must save A's registers/PC and restore B's, plus you blow away CPU caches and TLB entries. If you have thousands of threads each blocked on I/O, you pay this constantly for little gain — which is what pushed the industry toward event loops for I/O-heavy servers.

Threads shine for **CPU-bound parallelism**: 8 threads on 8 cores hashing in parallel = ~8× throughput.

### The event loop — one thread, never blocking

The event loop (Node.js, and how nginx/Redis handle connections) flips the model: **a single thread** that *never waits*. Instead of blocking on I/O, it kicks off the operation, hands it to the kernel (or a small internal pool), and immediately moves on to other work. When the I/O finishes, the result is queued back as a **callback** that the loop runs later.

   `┌──────────────────────────────────────────────┐
   │              EVENT LOOP (1 thread)             │
   │                                                │
   │   ┌─► pull next ready task ─► run it to        │
   │   │   completion (synchronously) ──┐           │
   │   │                                │           │
   │   └────────── loop ◄───────────────┘           │
   └───────┬────────────────────────▲───────────────┘
           │ start async I/O          │ "done" callback
           ▼                          │ enqueued
   ┌───────────────────┐     ┌────────┴─────────┐
   │ Kernel / libuv     │───►│  callback / task │
   │ pool (DB, net, fs) │    │     queue        │
   └───────────────────┘     └──────────────────┘`

The programming-model evolution you listed is just three syntaxes over this same machinery:

- **Callbacks** — "call this function when the I/O is done." Works, but nesting them gets ugly (callback hell).
- **Promises** — an object representing a future value; you chain `.then()`. Flattens the nesting.
- **async/await** — syntactic sugar over promises. `await someIO()` *looks* blocking but actually says "suspend this function here, let the loop do other work, resume me when the promise resolves." Reads like sequential code, runs non-blocking.

This is why one thread can serve tens of thousands of concurrent connections: while request A waits on its DB query, the loop is already handling B, C, D. The catch — and it's a big one — **a CPU-bound task blocks the entire loop.** One synchronous 200 ms computation freezes *every* connection, because there's only one thread. That's when you offload to `worker_threads` or a separate service.

### The failure mode: race conditions and lost updates

The moment two flows of execution touch shared state, ordering becomes nondeterministic, and you get **race conditions**. The classic one is the **lost update** — a read-modify-write that interleaves badly:

  `Shared:  highestBid = 100

  Request A                Request B
  ─────────────────────────────────────────
  read  highestBid (100)
                           read  highestBid (100)
  compute 100 + 10 = 110
                           compute 100 + 20 = 120
  write highestBid = 110
                           write highestBid = 120   ← A's update is lost
  ─────────────────────────────────────────
  Final: 120, but it should have reflected BOTH bids in sequence`

Both read the same starting value, both compute from it, and the last write wins — one update silently vanishes. Your auction platform is the textbook case for why this is a *correctness and fairness* bug, not just a curiosity: if two bids land at the same instant and one is lost, you've violated the fairness and auditability guarantees the whole system is supposed to enforce. The audit log would show an outcome that the inputs don't justify.

Here's the part most people get wrong, and you flagged it correctly by listing it for *both* threads and the event loop:

**Race conditions happen in single-threaded async code too.** People assume "Node is single-threaded, so no races" — false. Synchronous blocks *are* atomic (run-to-completion: the loop won't interrupt a function mid-statement). But the instant you write `await`, you create a **suspension point**. Between the read and the write, the loop can run *another* request that mutates the same state. So this is racy even in Node:

js

`const current = await db.getHighestBid(auctionId); // suspend → B runs here
if (newBid > current) {
  await db.setHighestBid(auctionId, newBid);       // stale 'current'
}`

No threads involved, still a lost update. The shared state lives in the DB; the interleaving happens across the `await`.

### The fixes

The fundamental idea is the **critical section**: the read-modify-write must be made *atomic* — indivisible — so no one else can interleave inside it.

- **Mutex** (mutual exclusion) — a lock with one owner at a time. A task acquires it before the critical section and releases after; everyone else waits. Binary, ownership-based.
- **Semaphore** — a counter allowing up to **N** concurrent holders. A *binary* semaphore (N=1) acts like a mutex; a *counting* semaphore is for limiting concurrency, e.g. "at most 10 simultaneous DB connections" or "at most 4 concurrent model inferences." Different job: a mutex protects *correctness*; a counting semaphore enforces a *capacity limit*.

In threaded programs these are real OS primitives guarding shared heap memory. But for a backend like yours, the practical versions live where the shared state actually lives — the database and the queue:

- **DB row locking** — `SELECT ... FOR UPDATE` locks the auction row so the read-check-write runs atomically; concurrent bids serialize behind it.
- **Optimistic concurrency** — add a `version` column, write only `WHERE version = $read_version`, and retry if zero rows updated. Great when conflicts are rare.
- **Atomic operations** — push the whole compare-and-set into one statement the DB executes atomically, so there's no application-side gap to interleave in.
- **Serialize via a queue** — funnel all bids for one auction through a single worker so they're processed one at a time by construction. You're already on **BullMQ** — partitioning by `auctionId` and processing serially per auction is a clean way to make lost updates structurally impossible while keeping a natural audit trail of ordered events.

(For true in-process shared-memory parallelism in Node — `worker_threads` + `SharedArrayBuffer` — the primitive is `Atomics`, which gives you lock-free compare-and-swap. Rarely needed for a web backend, but it's the same idea one level down.)

The one-line mental model: **threads = parallelism for CPU-bound work, paying memory-safety and context-switch costs; event loop = concurrency for I/O-bound work, one thread, blocked only by your own CPU work; and *any* shared mutable state across either model needs a critical section, whether that's a mutex in memory or a `FOR UPDATE` / queue in your stack.**
