# Chapter 1 — Foundations: The Machine Beneath

The hardware stopped getting faster around 2005. Clock speeds hit the thermal wall. The chip designers shrugged and added more cores. All the remaining performance is in parallelism — which means it's in your hands, and your hands need to understand what the machine is actually doing before they touch a mutex.

This chapter is the substrate. No primitives yet. Just hardware, memory ordering, and a list of the ways you will wreck yourself.

---

## The Cold Open: A Race You Cannot See

Two threads. Completely separate variables. Nothing shared. The benchmark is slower with two threads than with one.

```cpp
#include <atomic>
#include <cstdint>
#include <thread>

// "Separate variables, no sharing." You'd think.
struct Counters {
    std::atomic<std::int64_t> a;  // Thread A writes only this
    std::atomic<std::int64_t> b;  // Thread B writes only this
};

Counters g{};

void count_a(std::int64_t n) { for (std::int64_t i = 0; i < n; ++i) g.a.fetch_add(1, std::memory_order_relaxed); }
void count_b(std::int64_t n) { for (std::int64_t i = 0; i < n; ++i) g.b.fetch_add(1, std::memory_order_relaxed); }

int main() {
    constexpr std::int64_t N = 100'000'000;
    std::thread ta(count_a, N);
    std::thread tb(count_b, N);
    ta.join(); tb.join();
    // Typical result: ~2.5x SLOWER than running count_a and count_b serially.
}
```

`a` is at offset 0. `b` is at offset 8. A cache line is 64 bytes. They share one. Every write to `a` by Core 0 invalidates the line on Core 1 — which holds `b`. Core 1 must re-fetch before it can increment. They bat the same 64 bytes back and forth across the L3 interconnect all day. This is **false sharing**. Fix:

```cpp
struct Counters {
    alignas(64) std::atomic<std::int64_t> a;  // Own cache line
    alignas(64) std::atomic<std::int64_t> b;  // Own cache line
    // Portable: alignas(std::hardware_destructive_interference_size) — see § Cache Lines.
};
// Now: near-linear speedup.
```

This is the problem in miniature. The machine does not care about your variable names. It cares about bytes, cache lines, and coherence state. Everything in this chapter exists to make that fact legible.

---

## The Memory Hierarchy

Numbers matter. Here they are, with a human-scale column (1 ns = 1 second) to make the gaps visceral.

| Level | Latency | Human-Scale | Size |
|---|---|---|---|
| CPU register | ~0.3 ns | 0.3 s — blink | ~1 KB |
| L1 cache (per-core) | 1–4 ns | 1–4 s — reach across desk | 32–64 KB |
| L2 cache (per-core) | 5–12 ns | 5–12 s — walk to whiteboard | 256 KB–1 MB |
| L3 cache (shared) | 20–50 ns | 20–50 s — walk down the hall | 4–64 MB |
| DRAM | 60–100 ns | 1–2 min — coffee run | GBs |
| NVMe SSD | 100–200 μs | ~2 days | TBs |
| HDD | 5–10 ms | ~3 months | TBs |
| Cross-datacenter | 50–150 ms | ~5 years | — |

A coherence miss — two cores fighting over the same cache line — lands somewhere between L3 and DRAM. At 3 GHz, 60 ns is 180 clock cycles. If your loop body is three instructions, you spend 60× more time on cache overhead than on work. This is the false-sharing benchmark result, explained.

> **Intuition pump (real-world):** Your register is the pen in your hand. L1 cache is the sticky note on your monitor — you can read it without moving (1 second, human-scale). L2 is your desk drawer: 5–12 seconds of rummaging. L3 is the filing cabinet across the room: a 20–50 second walk. DRAM is the coffee shop two blocks down: a 2-minute round trip. NVMe SSD is a cross-country mail order: about *two days*. HDD is a warehouse three months away, accessible only by a courier on a very old bicycle. Cross-datacenter is a letter to the London office — allow five years, per the latency table above, if you are feeling dramatic. Your job as a concurrency engineer is to keep the data your threads fight over on the sticky note. The moment you send the CPU to the filing cabinet 180 times per loop iteration, you have stopped writing parallel code and started writing very expensive single-threaded code with extra suffering attached.

> **Pro tip:** Memory-bound code does not scale with cores. If the working set doesn't fit in the caches available to your threads, more threads just fight over the same bus. Profile cache misses with `perf stat -e cache-misses` before blaming your synchronization design.

---

## Cache Lines: 64 Bytes That Rule Your Life

64 bytes. Every read, write, and invalidation operates at that granularity. The hardware does not move individual bytes.

```cpp
#include <atomic>
#include <cstdint>
#include <new>      // std::hardware_destructive_interference_size

// How much fits in one cache line (typically 64 bytes on x86/ARM64):
//   8 × int64_t,  16 × int32_t,  8 × std::atomic<uint64_t>  ← last row: the trap

// BROKEN: eight "independent" per-thread counters, all on one cache line.
struct Bad {
    std::atomic<std::uint64_t> per_thread[8];  // 64 bytes total = one line.
};                                              // Every write by any thread invalidates
                                               // every other thread's copy. Brutal.

// FIXED: each counter on its own cache line.
// hardware_destructive_interference_size is the portable constant — 64 on x86,
// 128 on Apple M-series. Hardcoding 64 breaks ARM desktops.
struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<std::uint64_t> val{0};
};
Counter per_thread[8];  // Each core touches its own line. No coherence ping-pong.
```

Two unrelated atomics sharing a line produce coherence traffic even though neither operation is wrong. The atomics give you correctness. The shared line costs you throughput. Chapter 10 covers the full pattern.

> **Intuition pump (real-world):** You and a coworker share a single notebook. Your job: update Column A. Their job: update Column B. Two people, two columns, zero overlap in the actual *work* — but management printed both columns on the *same page*, and only one person can hold the notebook at a time. You spend 90% of the workday wrestling it back and forth across the table. That is false sharing. The CPU does not care about your variable names. It cares about real estate: 64 contiguous bytes, the cache line it moves as one atomic chunk. `alignas(64)` gives each variable its own page — its own notebook, permanently on its own desk. Nobody travels. Nobody waits. The CPU stops moonlighting as your coworker's administrative assistant and starts doing actual work.

---

## MESI: Why Cache Coherence Hurts

Every cache line has a state. These four letters explain every false-sharing profile you will ever read.[^mesi]

[^mesi]: Intel uses MESIF (a Forward state for clean-shared lines optimized for cache-to-cache transfer); AMD uses MOESI (an Owned state that allows dirty-shared, avoiding a writeback on every read miss). MESI is the pedagogical baseline. The cost story — coherence traffic dominates the cycle budget under contention — is identical across all variants.

| State | Meaning | Read | Write |
|---|---|---|---|
| **M**odified | Only valid copy; dirty (differs from DRAM) | Yes | Yes |
| **E**xclusive | Only valid copy; clean | Yes | Yes (→ M) |
| **S**hared | Multiple cores have valid read copies | Yes | No — must invalidate first |
| **I**nvalid | Stale; do not use | No | No |

When Thread B on Core 1 wants to write `b` (which shares a line with `a` last written by Core 0):

```
1. Core 1 broadcasts "Read-For-Ownership" on the coherence bus.
2. Core 0 flushes its dirty line to L3 (or directly to Core 1). Latency: ~40 ns.
3. Core 0 transitions to I. Core 1 transitions to M.
4. Core 1 writes b.
5. Core 0 wants to write a: same story in reverse.
Round-trip cost per iteration: ~40–60 ns. Actual work per iteration: ~0.3 ns.
```

The protocol is correct. It is protecting you from stale reads. The protection just costs ~200× the work it protects.

> **Pitfall:** False sharing requires at least one *writer*. Many readers with no writers is fine — readers hold the line in S state, pay L3 latency once, and coast. Writes invalidate other cores' cached copies and create the ping-pong. Diagnose with `perf c2c`; look for high "HITM" counts (hits in modified state).

---

## Reordering: The Compiler and CPU Conspiracy

Both the compiler and the CPU reorder your memory accesses. This is not a bug. It is the performance you are getting. The C++ memory model's deal: if you don't synchronize, the implementation optimizes freely.

### The Compiler's Half

```cpp
// You write:
x = 42;
ready = true;
```

```
; Compiler may emit — stores to unrelated locations, no dependency between them:
mov [ready], 1    ; ready first
mov [x], 42       ; x second
; Valid: single-threaded observable behavior is identical.
; A second thread watching x has no say in this contract.
```

### The CPU's Half

Even with correct emit order, the store buffer can drain out-of-order. `ready = true` may become visible to another core before `x = 42` drains from the store buffer.

### The Classic Race — Broken and Fixed

```cpp
// BROKEN: plain variables, no synchronization.
int      x     = 0;
bool     ready = false;

void producer() {           // Thread 1
    x     = 42;             // (A)
    ready = true;           // (B) may execute before (A) is visible elsewhere
}

void consumer() {           // Thread 2
    while (!ready) {}       // (C) may never see ready=true (cached in register)
    int v = x;              // (D) data race on x: UB. v may be 0.
}
// TSan fires. Optimizer may hoist (C)'s load. Ship this and enjoy your incident.
```

```cpp
// FIXED: atomic flag with release/acquire.
#include <atomic>

int                  x     = 0;
std::atomic<bool>    ready{false};

void producer() {           // Thread 1
    x = 42;                                       // (A)
    ready.store(true, std::memory_order_release); // (B) — no op before (B) may move after it
}

void consumer() {           // Thread 2
    while (!ready.load(std::memory_order_acquire)) {} // (C) — no op after (C) may move before it
    int v = x;              // (D) guaranteed: v == 42.
    // (B) release synchronizes-with (C) acquire → (A) happens-before (D).
}
```

Release: no read **or** write sequenced before this store may be reordered after it. Acquire: no read **or** write sequenced after this load may be reordered before it. Together they form the synchronizes-with edge that makes (A) visible at (D).

> **Intuition pump (real-world):** You hand an intern your to-do list: *(1) put the documents in the safe, (2) lock the safe.* The compiler is a snarky intern who reads it, decides steps 1 and 2 look unrelated in a single-threaded world, and quietly swaps them for "efficiency." The CPU — an even snarkier intern who never read your original list — looks at the reordered list and, because its store buffer is still draining, delivers step 2 to the observable world before step 1 has actually landed. The safe is now locked. The documents are still on the desk. Both interns were technically "correct" within their own context. Your data is on the wrong side of a welded door. `release` and `acquire` are the two items on the to-do list that are stamped **"neither intern may move this."** Everything above the `release` stays above it. Everything below the `acquire` stays below it. The rest of the list the interns can shuffle to their hearts' content, and honestly, that's fine — that shuffling is your performance budget.

### x86 vs ARM: Why "Works on My Machine" Lies

x86 uses **Total Store Order (TSO)** — stores from a single core become visible in issue order. Many races that crash on ARM are invisible on x86 because TSO happens to do the right thing for free.

| Reordering | x86 | ARM / POWER |
|---|---|---|
| Store → Store | No | Yes (possible) |
| Load → Load | No | Yes (possible) |
| Load → Store | No | Yes (possible) |
| Store → Load | **Yes** | Yes (possible) |

ARM allows almost everything to be reordered unless you emit explicit barrier instructions — which `std::atomic` with acquire/release does for you. Code that races silently on x86 for six months will fail immediately on AWS Graviton. TSan models the abstract C++ memory model, not x86's, which is why it catches ARM-only races on your Intel dev box.

---

## `volatile` Is Not for Threading

```cpp
#include <atomic>

void do_work() { /* ... */ }

// BROKEN — volatile for inter-thread flag.
volatile bool g_stop_v = false;

void worker_v() {
    while (!g_stop_v) { do_work(); }  // Compiler won't cache g_stop_v in a register.
}                                     // CPU store buffer still reorders. No fence.
                                      // `volatile` does not make the access atomic.
                                      // Per [intro.races], two unsynchronized non-atomic
                                      // accesses with at least one write is UB. `volatile`
                                      // is irrelevant to the race; it just stops the compiler
                                      // from caching the load in a register.

// FIXED — atomic.
std::atomic<bool> g_stop{false};

void worker() {
    while (!g_stop.load(std::memory_order_acquire)) { do_work(); }
}
```

`volatile` is for memory-mapped I/O and `setjmp`/signal interactions. It prevents the compiler from eliding accesses. It does nothing about CPU reordering, store buffers, or happens-before. It is not synchronization. There is no scenario where `volatile` is the right answer for inter-thread communication. Use `std::atomic<T>`. Period.

> **Intuition pump (real-world):** `volatile` is the sticky note you put on the whiteboard that says **"do not autocomplete this — the value really does change between reads, hardware does it."** Perfectly legitimate for one thing: memory-mapped I/O registers wired directly to external hardware that updates them behind the program's back. That is its entire résumé. It does absolutely nothing about the *other* problem: two people in the same noisy open-plan office both reaching for the same dry-erase marker at the same time. `volatile` makes the compiler watch the whiteboard more carefully. `std::atomic` hires a bouncer who ensures exactly one person writes at a time *and* guarantees that everyone else sees the updated text before they act on it. You can stare at that whiteboard with total, undivided, undisturbed focus and still get hit in the head by someone else's marker. That's `volatile` in a multithreaded context. Hire the bouncer.

---

## The Data Race: UB with a Friendly Name

The standard's essence:

> Two conflicting evaluations in different threads, **at least one of which is non-atomic**, with neither happens-before the other → **Undefined behavior.**

Two `memory_order_relaxed` operations on the same `std::atomic<int>` with no happens-before edge are conflicting and concurrent, yet are explicitly *not* a data race — they just have no ordering guarantee about what each one observes.

UB does not mean "wrong answer." It means the optimizer is free to assume the race cannot happen — and will rewrite your code accordingly.

```cpp
#include <atomic>

void process(int) { /* ... */ }

// BROKEN — data race on `value_bad`.
int value_bad = 0;

void writer_bad() { value_bad = 42; }  // Thread 1: unsynchronized write
void reader_bad() {                     // Thread 2: unsynchronized read
    while (value_bad == 0) {}           // Races are UB → optimizer assumes no concurrent writes
    process(value_bad);                 // exist. It may hoist the load out of the loop, cache the
}                                       // initial 0 forever, and spin you into an infinite loop.
```

```cpp
// FIXED — atomic eliminates the race; optimizer has nothing to exploit.
std::atomic<int> value{0};

void writer() { value.store(42, std::memory_order_relaxed); } // Thread 1
void reader() {                                                // Thread 2
    int v = value.load(std::memory_order_relaxed);
    if (v > 0) { process(v); }  // No race. No UB.
}
// relaxed is fine: value guards nothing else here.
// If it guarded other data, use release/acquire on write and read respectively.
```

> **Pitfall:** "It works in debug mode" means the race exists but `-O0` prevents the optimizer from exercising its UB license. Ship the optimized binary and that license gets used. TSan finds races regardless of optimization level — run it.

> **Intuition pump (real-world):** Two interns are editing the same Google Doc simultaneously, no coordination, no rules, just two people typing into the same file because "it'll probably be fine." You open the doc afterward and check the final number. "Wrong" would be too charitable — "wrong" implies there is a correct answer you can compare it against. What you actually have is a document in quantum superposition: its contents are whatever the optimizer froze into a register two function calls ago, based on its perfectly standard-compliant assumption that *this situation cannot happen*. It might look right. It might be zero. It might be stuck in an infinite loop because the compiler proved — correctly, given the UB license the standard handed it — that the value never changes and hoisted the load out of your spin loop permanently. You are not getting "sometimes wrong." You are getting "the toolchain has formally washed its hands of this region of code, and whatever comes out is your problem." TSan is the manager who watches both interns' screens in real-time and sends the Slack message *before* the doc goes to print.

---

## Happens-Before in One Page

Forget the full memory model for now. One relationship matters: **happens-before**.

Action A happens-before B if:
1. A and B are in the same thread and A is sequenced before B, **or**
2. A **synchronizes-with** B (release/acquire pair, mutex unlock/lock, etc.), **or**
3. A happens-before C and C happens-before B (transitivity).

*Synchronizes-with* is the cross-thread building block; combined with sequenced-before within each thread it forms *happens-before*. Consume ordering and dependency-ordered-before are weaker cousins covered in Chapter 6.

If A happens-before B, every write performed by A (and everything sequenced before A) is visible at B.

```cpp
#include <atomic>

int               payload = 0;
std::atomic<bool> flag{false};

void producer() {   // Thread 1
    payload = 99;                                        // (1) sequenced-before (2)
    flag.store(true, std::memory_order_release);         // (2) release store
}

void consumer() {   // Thread 2
    while (!flag.load(std::memory_order_acquire)) {}    // (3) acquire load reads value from (2)
                                                         //     → (2) synchronizes-with (3)
    int v = payload;                                     // (4) sequenced-after (3)
    // Chain: (1) hb (2) sw (3) hb (4) → (1) hb (4).
    // payload == 99 guaranteed. Not "probably." Guaranteed.
}
```

Without the release/acquire pair, you have no happens-before across threads, no visibility guarantee, and a data race on `payload`.

> **Intuition pump (real-world):** You email your colleague the final PDF. They open *the* PDF. The sequence makes complete sense because there is a provable causal chain: you *sent* it, they *received* it, *then* they *opened* it. That email is the synchronizes-with edge. Without it — if you just silently uploaded it to a shared folder with no notification — your colleague opening "the PDF" might be opening last Tuesday's draft. Or the one from six months ago with the same filename. Or a blank document, because your upload is still buffering on a bad connection. "Happens-before" does not mean "probably happened before" or "happened before in every test I ran." It means **there is a provable causal chain the hardware cannot violate**. The `release` store *is* sending the email. The `acquire` load *is* confirming receipt. Everything you wrote before hitting send is guaranteed visible to anyone who successfully receives it. Skip the email, and "visible" becomes "whatever happened to be sitting on their desktop that morning."

Synchronizes-with sources you will use constantly:

| Mechanism | Release side | Acquire side |
|---|---|---|
| `std::atomic` | `store(release)` | `load(acquire)` |
| `std::mutex` | `unlock()` | `lock()` |
| `std::promise` / `std::future` | `set_value()` | `get()` / `wait()` (successful return only[^fut]) |
| `std::latch` | `count_down()` | `wait()` |
| Thread creation | (spawning thread) | (thread body entry) |
| Thread join | (thread body exit) | `join()` return |

[^fut]: A timed-out `wait_for` does not establish the synchronizes-with edge. The guarantee applies only when the shared state is made ready and the wait returns successfully.

---

## Taxonomy of Failure Modes

| Failure | Cause | Production tell |
|---|---|---|
| **Data Race** | Shared write without happens-before | Nondeterministic crash; wrong answers; load-hoisting / infinite loop by optimizer |
| **Deadlock** | Circular lock wait | All threads blocked; watchdog kills process; stack trace shows everyone in lock/wait |
| **Livelock** | Threads spinning in reaction to each other, no progress | 100% CPU, zero throughput; profiler shows tight retry loop |
| **Starvation** | Scheduler or priority scheme perpetually skips a thread | Tail latency = ∞; queue never drains |
| **Priority Inversion** | High-prio blocked on lock held by preempted low-prio | RT deadline missed; high-prio task looks "stuck" |
| **ABA Problem** | CAS reads A; another thread writes B then A; CAS succeeds incorrectly | Lock-free structure silently corrupts; freed node reused at same address |
| **False Sharing** | Unrelated writers on same cache line | Two-thread benchmark slower than one; `perf c2c` shows hot line |
| **Lost Wakeup** | Condition becomes true *before* `wait()` enters | Thread parked forever; producer done, consumer never wakes |

Each of these has dedicated coverage ahead. Right now: know the names. When you are on-call and threads are blocked, you need to know which question to ask before you know where to look.

---

## Spurious Wakeups and Predicate Waits

The OS can return from `cv.wait(lock)` without any `notify` being called — this is a **spurious wakeup**, and it is real, documented, and bites everyone who skips the predicate form. (The "lost wakeup" — where the producer notifies before the consumer waits — is a related but distinct bug; it gets dedicated treatment in Chapter 4.)

```cpp
#include <mutex>
#include <condition_variable>

std::mutex              mu;
std::condition_variable cv;
bool                    data_ready = false;

void use_data() { /* ... */ }
```

```cpp
// BROKEN — `if` instead of a predicate loop.
// The OS can deliver a spurious wakeup (no notify was called).
// cv.wait(lock) returns, data_ready is still false, consumer plows ahead.
void consumer_broken() {
    std::unique_lock<std::mutex> lock(mu);
    if (!data_ready) {
        cv.wait(lock);   // Spurious wakeup: returns with data_ready == false.
    }
    use_data();          // Proceeds on garbage.
}
```

```cpp
// FIXED — predicate overload re-evaluates on every wakeup.
// Internally: while (!pred()) cv.wait(lock);
// Handles spurious wakeups and the already-true case (producer ran first).
void consumer_fixed() {
    std::unique_lock<std::mutex> lock(mu);
    cv.wait(lock, [] { return data_ready; });
    use_data();  // data_ready is guaranteed true here.
}
```

Always use the predicate overload of `condition_variable::wait`. Always.

---

## Tools

Four tools. Know they exist before you need them.

**ThreadSanitizer (TSan)** — Clang/GCC. Instruments binaries to catch data races at runtime. False-positive rate is near zero. If TSan fires, you have a real bug. 5–20× runtime overhead; run it in CI, not production.

**MSVC Concurrency Visualizer** — Visual Studio extension. Thread timelines, lock-contention graphs, context-switch visualization. First stop on Windows for diagnosing hot locks and deadlock candidates.

**`perf c2c`** — Linux. Profiles cache-line bouncing between cores. Reports "HITM" counts (hits-in-modified-state) per cache line and the functions responsible. False sharing is invisible to TSan (it's a performance problem, not a race); `perf c2c` is how you find it.

**`rr`** — Mozilla record-and-replay for Linux. Records the exact thread interleaving of a run; replays it deterministically in gdb, including backward stepping. A one-in-ten-thousand heisenbug becomes debuggable. Install it before you need it.

---

## The Four-Rule Contract

Every subsequent chapter builds on these. Violate one and the techniques become cargo cult.

### Rule 1 — Shared Mutable State Requires Synchronization

```cpp
// BROKEN
#include <cstdint>
int g_count_bad = 0;
void increment_bad() { ++g_count_bad; }  // Two threads calling this: data race.
```

```cpp
// FIXED (mutex)
#include <mutex>
std::mutex g_mu;
int g_count_m = 0;
void increment_mutex() {
    std::lock_guard<std::mutex> lock(g_mu);
    ++g_count_m;
}
```

```cpp
// FIXED (atomic — for simple counters)
#include <atomic>
std::atomic<int> g_count_a{0};
void increment_atomic() { g_count_a.fetch_add(1, std::memory_order_relaxed); }
```

Atomic for simple read-modify-write. Mutex for multi-step invariants or when you need to guard several variables together.

### Rule 2 — Atomics Give Indivisibility AND Ordering. Pick Both.

```cpp
#include <atomic>

int  compute() { return 42; }  // stand-in
void use_result(int) { /* ... */ }

std::atomic<bool> ready{false};
int               result = 0;

// BROKEN — relaxed flag does NOT synchronize result.
void producer_bad() {
    result = compute();
    ready.store(true, std::memory_order_relaxed);  // No fence: result may not be visible.
}
void consumer_bad() {
    while (!ready.load(std::memory_order_relaxed)) {}
    use_result(result);  // Data race on `result`. relaxed does NOT synchronize.
}

// FIXED — release/acquire establishes happens-before across result.
void producer_good() {
    result = compute();
    ready.store(true, std::memory_order_release);  // Fence: result visible before flag.
}
void consumer_good() {
    while (!ready.load(std::memory_order_acquire)) {}
    use_result(result);  // Guaranteed: sees compute()'s return value.
}
```

When unsure, start with `memory_order_seq_cst`. Optimize to weaker orderings only with a proof.

### Rule 3 — `volatile` Is Not Synchronization

Already shown. `std::atomic<T>` replaces every threading use of `volatile`. No exceptions.

### Rule 4 — TSan Before You Ship

Not once. Every CI run that touches concurrent code. No suppression without a written explanation. TSan catches races on x86 that only manifest in production on ARM. It is the cheapest bug-finding tool you have.

---

## Four Questions for Every Piece of Shared State

Before writing synchronization code, answer these:

1. Which threads **read** this?
2. Which threads **write** this?
3. What must be true when a reader sees a particular value?
4. What establishes the happens-before edge that guarantees it?

If question 4 has no answer, you have a race. If question 3's answer is "nothing in particular," you can use relaxed ordering. If question 3's answer is "this value guards other data that must also be visible," you need release/acquire.

```cpp
// Example: applying the four questions.
//
// Shared state: a Config, updated by one thread, read by many.
// Q1: all reader threads.   Q2: one writer (config reload).
// Q3: when a reader sees the new config, it must see the new field values.
// Q4: release/acquire on the atomic<shared_ptr> swap.
//
// std::atomic<std::shared_ptr<T>> (C++20) handles lifetime correctly.
// Raw atomic<T*> leaks old configs or races on reclamation. Don't go there.

#include <atomic>
#include <memory>

struct Config { int timeout; int retries; };

void use_config(int, int) { /* ... */ }

std::atomic<std::shared_ptr<Config>> g_config{
    std::make_shared<Config>(Config{5, 3})
};

void reload_config() {
    auto fresh = std::make_shared<Config>(Config{10, 5});
    // Release: all writes to *fresh happen-before any reader's acquire load.
    // Old Config is ref-counted: released when the last shared_ptr drops — no leak, no UAF.
    g_config.store(fresh, std::memory_order_release);
}

void handle_request() {
    // Acquire: sees all writes that happened-before the store in reload_config().
    auto cfg = g_config.load(std::memory_order_acquire);
    use_config(cfg->timeout, cfg->retries);  // Correct values guaranteed.
}
```

---

## Closing Mantra

**The machine is not wrong; your mental model is.**

---

*Next: Chapter 2 — `std::thread`, `std::jthread`, and the Art of Not Shooting Yourself in Both Feet Simultaneously.*
