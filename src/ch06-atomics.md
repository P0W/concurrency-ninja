# Chapter 6 — std::atomic & the Memory Model

## Where the Dragons Live and Your Unit Tests Cannot Follow

This bug does not reproduce on your laptop. It does not reproduce on CI.
It detonates on an ARM server in Singapore at 3 AM, once every eleven days.
The stack trace points at code that is obviously correct. The rubber duck is also confused.

Welcome to the memory model.

---

## 6.1 Cold Open: Peterson's Algorithm and the Hardware That Laughed at It

The textbook version. Works fine under sequential consistency:

```cpp
#include <atomic>

// Peterson's mutual exclusion — correct under seq_cst (the default).
std::atomic<bool> flag[2]{ false, false };
std::atomic<int>  turn{ 0 };

void enter(int me, int other)
{
    flag[me].store(true);          // seq_cst by default
    turn.store(other);
    while (flag[other].load() && turn.load() == other) {}  // spin
}

void leave(int me)
{
    flag[me].store(false);
}
```

Runs fine. Now someone reads that `relaxed` is cheaper and makes a "harmless optimization":

```cpp
// "Optimized." Broken everywhere — including x86. Do not do this.
void enter_broken(int me, int other)
{
    flag[me].store(true,  std::memory_order::relaxed);  // no ordering
    turn.store(other,     std::memory_order::relaxed);  // no ordering
    while (flag[other].load(std::memory_order::relaxed) &&
           turn.load(std::memory_order::relaxed) == other) {}
}
```

The failure mode is **Store-Load reordering**: the load of `flag[other]` is allowed to
execute before the store to `flag[me]` is visible to the other CPU. x86's TSO model
explicitly permits this — stores drain from the store buffer before other CPUs see them,
but loads can bypass the buffer. Thread 0 loads `flag[1]` as `false` before its own
`flag[0] = true` has drained to cache. Thread 1 does the same. Both see clear flags.
Both enter the critical section. This is not an ARM-only bug.
The model is the contract. Ignore it and your invariants become suggestions.

---

## 6.2 What `std::atomic<T>` Actually Gives You — Two Things

- **Indivisibility** — reads/writes are atomic; no torn values. True even for `relaxed`.
- **Ordering** — the `memory_order` parameter fences surrounding non-atomic operations
  relative to this op. This is independent of indivisibility.

```cpp
#include <atomic>
#include <cstdint>

// --- Indivisibility (no torn 64-bit read, even relaxed) ---
std::atomic<uint64_t> counter{ 0 };
uint64_t v = counter.load(std::memory_order::relaxed); // complete value; never half-old/half-new

// --- Ordering (acquire/release publish a Config, not just the pointer) ---
struct Config { int timeout; double threshold; };
std::atomic<const Config*> g_cfg{ nullptr };

// Producer:
auto* cfg = new Config{ 5000, 0.95 };
g_cfg.store(cfg, std::memory_order::release);      // publishes *cfg too

// Consumer:
const Config* p = g_cfg.load(std::memory_order::acquire);
if (p) { /* p->timeout and p->threshold are fully visible */ }
```

You can have indivisibility without ordering (`relaxed`).
You cannot have ordering without indivisibility — ordering is expressed *through* the atomic op.

---

## 6.3 What Is Lock-Free For Free?

```cpp
#include <atomic>
#include <cstdint>

// Compile-time: is this type ALWAYS lock-free on this target?
static_assert(std::atomic<long>::is_always_lock_free);
static_assert(std::atomic<void*>::is_always_lock_free);

struct Big { uint64_t a; uint64_t b; };   // 128 bits
// NOT static_assert: may or may not be lock-free depending on CPU + alignment.
std::atomic<Big> big{};
bool lf = big.is_lock_free();              // ask at runtime
```

On x86-64: integers up to 64-bit and pointers are lock-free. 128-bit may use `CMPXCHG16B`
(hardware-dependent) or an internal lock. On ARM64: 128-bit uses `LDXP`/`STXP` (or `CASP` on ARMv8.1+ LSE).
For types you cannot verify: store a pointer, not the object.

---

## 6.4 The Memory Orderings

| Order | What it means | x86-64 cost | ARM64 cost |
|---|---|---|---|
| `relaxed` | Atomic op, no ordering constraint | No fence cost† | No fence cost† |
| `consume` | Dependency-ordered load | *Standard temporarily discourages it; every compiler promotes to acquire. Skip.* | |
| `acquire` | Load: nothing after reorders before this | Free (TSO) | `LDAR` (or `LDAPR` on v8.3+) |
| `release` | Store: nothing before reorders after this | Free (TSO) | `STLR` |
| `acq_rel` | Both; for RMW ops | `LOCK` prefix (Expensive) | `LDAXR`/`STLXR` |
| `seq_cst` | Total order across all seq_cst ops | `MFENCE` / `LOCK XCHG` on store | `LDAR`/`STLR` (ARMv7: `DMB ISH`) |

**Practical default:** use `seq_cst` until a profiler gives you a specific reason not to.
On x86, `seq_cst` *loads* are free (plain `MOV`), but `seq_cst` *stores* require an `MFENCE` or `LOCK XCHG` to flush the store buffer — roughly 15–40 cycles versus ~1 cycle for a `release` store. On ARM64, `STLR`/`LDAR` already enforce sequential consistency (preventing `STLR` → `LDAR` reordering), whereas ARMv7 requires full `DMB` barriers. Only downgrade with measurement and explicit reasoning.

> †`relaxed` removes ordering constraints, but the atomic operation itself is not free. Relaxed loads and stores carry no fence overhead. Relaxed RMW operations (e.g. `fetch_add(relaxed)`) still require a locked instruction on x86 or an exclusive-access pair (LDXR/STXR) on ARM, plus cache-line ownership traffic. Distinguish "no ordering cost" from "no cost."

> **Intuition pump (real-world):** `seq_cst` is the courtroom stenographer. Every utterance from every thread goes into a single, agreed-upon transcript in the exact order spoken. No judge from any jurisdiction can dispute the sequence. This is expensive — on ARM, *actually* expensive, not "expensive" as in the benchmarks-look-slightly-worse kind. But you can reason about the transcript, which is more than can be said for the orders below it. Start here. Downgrade only when a profiler is pointing a gun at you and you can explain in a comment precisely why the downgrade is safe.

---

## 6.5 Release/Acquire: The Handoff

Acquire/release is a handoff between threads. The releaser finishes work and passes the baton.
The acquirer grabs the baton and is guaranteed to see everything the releaser wrote before passing it.

> **Intuition pump (real-world):** You pack a box — shove all the contents in — then slap a "READY" sticker on it and hand it to FedEx (release-store). FedEx shows up at the other site, sees the sticker (acquire-load), and *only then* opens the box. The sticker is the synchronisation point. Everything that went in before the sticker is guaranteed packed and visible. Anything added after the sticker is outside the contract — it may or may not be seen independently, but it is not part of the handoff. Tear off the sticker — use `relaxed` instead of `release` — and you're handing someone a half-packed box in a dark room and hoping for the best. The sticker costs almost nothing on x86. On ARM it's a real instruction. Worth every nanosecond.

```cpp
#include <atomic>

struct Config { int timeout_ms; double threshold; };

void process(int, double);   // forward declaration — defined elsewhere

std::atomic<const Config*> g_config{ nullptr };

// ---- Producer thread ----
void publish(const Config* cfg)
{
    // Release store: everything written before this is visible to any thread
    // that acquire-loads the new pointer value.
    g_config.store(cfg, std::memory_order::release);
}

// ---- Consumer thread ----
void consume()
{
    // Acquire load: synchronises-with the release store above.
    const Config* p = g_config.load(std::memory_order::acquire);
    if (!p) return;

    // Guaranteed: p->timeout_ms and p->threshold are fully initialised.
    // Without acquire/release, they could be garbage on ARM.
    process(p->timeout_ms, p->threshold);
}
```

If the acquire load reads the pointer value stored by the release store, the two
operations *synchronise-with* each other. That relationship is what makes the
entire pointed-to object visible — not just the pointer. An acquire load that
reads `nullptr` or an older pointer value does not pair with the later release;
there is no synchronisation and no visibility guarantee for `*cfg`. Nothing else
provides this guarantee when the pairing does hold.

---

## 6.6 `relaxed` Counter for Telemetry

When the counter is the entire message — not a gate for other memory — `relaxed` is correct
and `seq_cst` is just a tax.

> **Intuition pump (real-world):** `relaxed` is the Post-It note thrown over your shoulder. It will land on someone's desk. Eventually. You don't know when. You don't know whose desk. You don't care — because all you need is for the *number* to add up correctly at the end of the day. Perfectly appropriate for hit/miss counters that are read in aggregate and gate nothing. Spectacularly wrong as a flag that guards other data. If someone `relaxed`-stores "data is ready" and you `relaxed`-load it, the data behind it might still be garbage. Know what you're counting, and whether the count *means* anything beyond itself.

```cpp
#include <atomic>
#include <cstdint>

std::atomic<uint64_t> g_hits{ 0 };
std::atomic<uint64_t> g_misses{ 0 };

void record_hit()   { g_hits.fetch_add(1,   std::memory_order::relaxed); }
void record_miss()  { g_misses.fetch_add(1, std::memory_order::relaxed); }

uint64_t read_hits()   { return g_hits.load(std::memory_order::relaxed);   }
uint64_t read_misses() { return g_misses.load(std::memory_order::relaxed); }
```

No other memory is coupled to these counters. `relaxed` gives you indivisibility (no torn
reads) with zero fence overhead. On ARM, that is a real difference on hot paths.

---

## 6.7 `compare_exchange_weak` vs `compare_exchange_strong`

| | `weak` | `strong` |
|---|---|---|
| Spurious failure | Yes (LL/SC architectures) | No |
| Cost on ARM/POWER | Cheaper (no internal retry) | More expensive |
| Cost on x86 | Identical (both use `CMPXCHG`) | Identical |
| Use in a retry loop | Correct — loop handles it | Works but over-pays |
| One-shot CAS | Wrong — can't distinguish spurious from real conflict | Correct |

Both take **two** memory orders: success and failure.
Failure order cannot be `release` or `acq_rel` — no store happens on failure.

> **Intuition pump (real-world):** `weak` is a doorbell that sometimes lies. The door is unlocked, you ring, nobody answers — not because they're gone, just because the doorbell occasionally doesn't feel like it (LL/SC spurious failure on ARM/POWER). Fine in a retry loop: you'll ring again, it costs nothing, the loop was going to iterate anyway. `strong` is an honest doorbell: if it says nobody's home, nobody is home — no spurious failures, no internal retry trickery. Use `weak` in loops where retrying is the default. Use `strong` when you only get one shot and a false "failure" would send you down the wrong branch permanently.

**CAS loop (use `weak`):**

```cpp
#include <atomic>
#include <limits>

std::atomic<int> g_value{ 0 };

void increment_if_positive()
{
    int current = g_value.load(std::memory_order::acquire);
    while (current > 0 && current < std::numeric_limits<int>::max())
    {
        // weak: spurious failure just means we retry, which we would anyway.
        // On failure, current is refreshed with the actual value automatically.
        // (Guard < max(): unlike atomic::fetch_add, plain 'current + 1' is signed UB at INT_MAX.)
        if (g_value.compare_exchange_weak(current, current + 1,
                std::memory_order::acq_rel,   // success: we stored
                std::memory_order::acquire))  // failure: we only loaded
            break;
    }
}
```

**One-shot CAS (use `strong`):**

```cpp
// Lazy-init a singleton: first writer wins, rest discard their allocation.
std::atomic<Config*> g_singleton{ nullptr };

void init_singleton()
{
    Config* expected = nullptr;
    Config* fresh    = new Config{};
    // success: release — publishes the fully-constructed Config to any acquire load.
    // failure: relaxed — no store occurs on failure, and we don't need to observe
    //   anything the winner wrote. The only constraint on the failure order is that
    //   it not be release or acq_rel (since C++17; the older "no stronger than success"
    //   rule was removed by P0418R2).
    if (!g_singleton.compare_exchange_strong(expected, fresh,
            std::memory_order::release,
            std::memory_order::relaxed))
        delete fresh;  // someone beat us; discard ours
}
```

---

## 6.8 CAS-Based Maximum Tracker

```cpp
#include <atomic>
#include <cstdint>

class AtomicMax
{
public:
    void update(int64_t candidate)
    {
        int64_t cur = m_max.load(std::memory_order::relaxed);
        while (candidate > cur)
        {
            // Pure scalar tracker; no other memory is coupled — same reasoning as §6.6.
            // On failure cur is refreshed; loop condition re-evaluates.
            if (m_max.compare_exchange_weak(cur, candidate,
                    std::memory_order::relaxed,
                    std::memory_order::relaxed))
                return;
        }
    }

    int64_t get() const { return m_max.load(std::memory_order::relaxed); }

private:
    std::atomic<int64_t> m_max{ INT64_MIN };
};
```

The code is the lesson. `AtomicMax` tracks a single scalar — no other memory is coupled to
the result, so there is nothing to publish and nothing to observe. `relaxed` gives
indivisibility with zero fence overhead, exactly as §6.6's telemetry counter. If another
thread concurrently writes a larger value, `cur` is refreshed and `candidate > cur` becomes
false — we correctly yield to the larger value.

---

## 6.9 `fetch_add` and Friends — They Return the OLD Value

This surprises everyone exactly once.

```cpp
#include <atomic>

std::atomic<int> x{ 10 };

int old_val = x.fetch_add(5);
// old_val == 10.  x == 15.
// NOT 15. Not "new value". OLD value. Always.

// Same for: fetch_sub, fetch_and, fetch_or, fetch_xor
// operator++ returns NEW value (because C++ consistency is a myth):
int new_val = ++x;   // new_val == 16.  x == 16.
```

`fetch_add` returning old is a feature: in lock-free algorithms you need to know what you
atomically displaced. If you want the new value, compute it: `old_val + 5`.

---

## 6.10 `std::atomic_flag` — The Guaranteed Lock-Free Primitive

The only type the standard guarantees lock-free everywhere. C++20 adds `test()`, `wait()`,
`notify_one()`, `notify_all()`.

> **Intuition pump (real-world):** `std::atomic_flag` is the occupied/vacant slider on the back of a bathroom stall door. Two states: *Vacant* or *Occupied*. No "partially occupied," no "occupied but negotiable," no internal lock to implement the lock. It is the cheapest synchronisation primitive the standard offers — guaranteed lock-free on every platform, no exceptions. If your entire problem fits in two states and one atomic test-and-set, this is your tool. Everything else is paying for a hotel lobby when all you needed was a toilet.

```cpp
#include <atomic>
#include <thread>

class SpinLock
{
public:
    void lock()
    {
        while (m_flag.test_and_set(std::memory_order::acquire))
        {
            // C++20 TTAS: block until the flag looks free before retrying.
            // wait() uses OS primitives (futex / WaitOnAddress) instead of
            // hammering the cache line with RMW traffic while the lock is held.
            m_flag.wait(true, std::memory_order::relaxed);
        }
    }

    void unlock()
    {
        m_flag.clear(std::memory_order::release);
        m_flag.notify_one();   // wake one waiter
    }

private:
    std::atomic_flag m_flag;  // default-initialized to clear since C++20; ATOMIC_FLAG_INIT is redundant noise
};
```

`test_and_set` returns the *previous* value. If it was already set, we lost the race.
The C++20 `wait(true, relaxed)` blocks until the flag is cleared (i.e., until `unlock()`
calls `clear()` + `notify_one()`), then we retry. This replaces a hot RMW spin loop with
an OS-level sleep, eliminating cache line thrash while the lock is held.

**Do not use spinlocks in user-space unless you have measured, profiled, and accepted that
a preempted lock holder will starve every spinning thread.** A pure busy-wait spinlock never
sleeps; even the `wait(true)`-assisted version above lacks `std::mutex`'s OS priority-inheritance
and adaptive spinning when the lock holder gets preempted.

---

## 6.11 `std::atomic_ref` (C++20) — Atomics on Demand

Wrap a non-atomic object as atomic for one critical window without permanently paying the cost.

```cpp
#include <atomic>
#include <thread>

struct WorkItem { int priority; int result{ 0 }; alignas(std::atomic_ref<bool>::required_alignment) bool done{ false }; };

void produce(WorkItem& item)
{
    item.result = item.priority * 2;                          // single writer
    std::atomic_ref<bool>{ item.done }                        // temporary atomic view
        .store(true, std::memory_order::release);             // publish result
}

void consume(WorkItem& item)
{
    std::atomic_ref<bool> ref{ item.done };
    while (!ref.load(std::memory_order::acquire))
        std::this_thread::yield();
    // item.result is visible here — synchronises-with the release store above.
}
```

Alignment requirement: the object must satisfy `std::atomic_ref<T>::required_alignment`.
While any `atomic_ref` to the object exists, **all** accesses to that object — not just
concurrent ones — must go through `atomic_ref`. Do not use this on packed fields or
under-aligned storage; the constructor is UB if alignment is not met.
Mixing atomic and non-atomic access to the same memory is undefined behaviour.

---

## 6.12 `std::atomic<std::shared_ptr<T>>` (C++20)

Lock-free (or lock-based — implementation-defined; call `is_lock_free()` to find out) atomic
shared pointer. Replaces the deprecated `std::atomic_load`/`std::atomic_store` free functions.

```cpp
#include <atomic>
#include <memory>

struct Config { int value; };

std::atomic<std::shared_ptr<Config>> g_config;

// Producer — publish a new config snapshot
void update_config(int v)
{
    g_config.store(std::make_shared<Config>(Config{v}), std::memory_order::release);
}

// Consumer — get a consistent snapshot; lifetime managed by shared_ptr
std::shared_ptr<Config> get_config()
{
    return g_config.load(std::memory_order::acquire);
}
```

This is the "atomic publish" pattern for read-heavy, write-rare shared state. Config objects
are immutable once published; updates swap the pointer. Chapter 11 covers this as a
full concurrent data structure pattern.

**Caveat:** shipping standard-library implementations today (MSVC STL, libstdc++; libc++
does not yet implement it) back `atomic<shared_ptr<T>>` with an internal lock — typically a
lock bit in the control-block pointer or a spinlock table keyed by address. `is_lock_free()`
returns false everywhere. If lock-freedom is a requirement, use `atomic<T*>` plus a manual
reclamation scheme (hazard pointers, RCU, epoch). See Arthur O'Dwyer's critique for why the
standard's design forces this.

---

## 6.13 Standalone Fences — They Exist; You Rarely Need Them

```cpp
#include <atomic>

// Release fence: all prior loads AND stores cannot be reordered past
// any STORE that is sequenced after this fence. (Not a total barrier —
// prior ops can still move past subsequent loads.)
std::atomic_thread_fence(std::memory_order::release);

// Acquire fence: all subsequent loads AND stores cannot be reordered before
// any LOAD that is sequenced before this fence. (Not a total barrier —
// subsequent ops can still move before prior stores.)
std::atomic_thread_fence(std::memory_order::acquire);

// Full barrier (seq_cst): orders everything in both directions.
std::atomic_thread_fence(std::memory_order::seq_cst);
```

A standalone fence is more powerful than the same ordering on a single atomic — it applies
to all surrounding memory operations, not just one. That power comes at the cost of
readability: the fence is detached from any variable and gives no hint at the call site
about what it is ordering.

Prefer expressing ordering through atomic operations with explicit `memory_order` parameters.
Use fences when writing allocators, OS primitives, or integrating with assembly. Not for
application-level code. Note that a release fence creates a synchronizes-with edge only when
paired with a subsequent atomic store whose value is observed by an acquire load or acquire
fence on another thread — the fence has no unilateral effect ([atomics.fences]).

> **Intuition pump (real-world):** `std::atomic_thread_fence` is the velvet rope at a nightclub. The bouncer (CPU) is told: "nothing behind this rope gets moved in front of it." It applies to *all* surrounding memory operations, not just one variable — which sounds powerful until you notice that the rope is also completely detached from any specific variable, leaving your colleagues staring at it wondering what exactly is being ordered and why that comment says "DO NOT REMOVE." In 99% of application code, release/acquire on the actual atomics is clearer, cheaper to audit, and less likely to become haunted. Fences are for allocator authors and the kind of people who read the ARMv8 memory model spec for fun. You know who you are.

---

## 6.14 C++20 Atomic Wait / Notify — Lightweight Signalling

Like a futex, but standard. Blocks without busy-spinning. Replaces the
`while (!flag) { yield(); }` loop for situations where wakeups are infrequent.

```cpp
#include <atomic>
#include <thread>
#include <cstdio>

std::atomic<int> g_phase{ 0 };

void worker()
{
    // Block efficiently until g_phase is no longer 0.
    // wait(old) returns only after the value differs from old — it does not
    // return while the value is still equal. For compound conditions or state
    // machines, loop around load/check/wait: a changed value may still not
    // satisfy your full predicate.
    g_phase.wait(0, std::memory_order::acquire);
    std::printf("Worker: phase is now %d\n", g_phase.load());
}

void controller()
{
    g_phase.store(1, std::memory_order::release);
    g_phase.notify_all();   // wake every thread blocked in wait()
}

int main()
{
    std::thread t(worker);
    controller();
    t.join();
}
```

`wait(old)` returns immediately if the current value is already different from `old` — it
does not miss a notification that happened before the call.
Use `notify_one` when exactly one waiter should proceed. Use `notify_all` when all should.

---

## 6.15 Anti-Patterns: The Wall of Shame

### `volatile` for synchronisation

```cpp
// BROKEN. volatile prevents register caching; it does NOT produce memory barriers.
volatile bool g_ready = false;
int g_data = 0;

// Thread A:
g_data  = 42;
g_ready = true;           // not a release; no ordering

// Thread B:
while (!g_ready) {}       // not an acquire; no ordering
int x = g_data;           // may read 0 on ARM. UB.
```

`volatile` is for memory-mapped hardware registers, not threads. Use `std::atomic`.

---

### Load + Store instead of RMW

```cpp
// BROKEN: race window between load and store.
void increment_wrong(std::atomic<int>& x)
{
    x.store(x.load() + 1);   // two ops; another thread can modify x in between
}

// FIXED: single atomic RMW.
void increment_right(std::atomic<int>& x)
{
    x.fetch_add(1, std::memory_order::relaxed);
}
```

---

### Busy-wait without backoff

```cpp
#include <atomic>
#include <thread>

std::atomic<bool> g_done{ false };

// BAD: burns 100% of a core; may starve the thread it is waiting for.
while (!g_done.load(std::memory_order::acquire)) {}

// BETTER: at least yield.
while (!g_done.load(std::memory_order::acquire))
    std::this_thread::yield();

// BEST: use C++20 wait — OS-level block, zero spin.
g_done.wait(false, std::memory_order::acquire);
```

---

### Mixing `seq_cst` and `relaxed` expecting seq_cst guarantees

The classic trap: you added `seq_cst` to the stores, slapped `relaxed` on the loads to
"save performance", and then wondered why the total order you were relying on just
evaporated. Congratulations — you paid the `seq_cst` store tax and got none of the benefits.

The failure mode is the **store-load** (Dekker) pattern:

```cpp
// Both threads do: seq_cst store to own flag, then relaxed load of other's flag.
std::atomic<int> x{ 0 }, y{ 0 };
int r1, r2;

// Thread 1:
x.store(1, std::memory_order::seq_cst);    // [A] full fence + store
r1 = y.load(std::memory_order::relaxed);  // [B] NOT in the total order

// Thread 2:
y.store(1, std::memory_order::seq_cst);    // [C] full fence + store
r2 = x.load(std::memory_order::relaxed);  // [D] NOT in the total order

// After both threads finish: r1 == 0 && r2 == 0 IS possible.
// If [B] and [D] were seq_cst, this outcome would be forbidden.
```

Here is why this hurts. If both loads were `seq_cst`, the total order would
force either `[A]` before `[C]` or vice versa — guaranteeing at least one of
`r1` or `r2` sees 1. You cannot have both threads sneak in before the other's
store in a single total order.

But `relaxed` loads are **outside** that total order. While `relaxed` operations still obey
single-thread program order (`sequenced-before`) and per-variable coherence, they do not
participate in the cross-variable `seq_cst` total order — nor does a preceding `seq_cst` store
act as a downward fence for a later `relaxed` load on a *different* variable. Both ARM and x86
are completely free to let `[B]` execute before `[A]` becomes globally visible and `[D]` before
`[C]`, giving both threads a stale zero.

`seq_cst`'s total order applies only to operations that are themselves `seq_cst`.
One `relaxed` op drops you out of that order for that specific access — and,
crucially, the expensive `seq_cst` stores you already paid for cannot save you.

---

### "It works on x86, therefore it is correct"

```cpp
#include <atomic>

void use(int);   // defined elsewhere

std::atomic<int*> g_ptr{ nullptr };
int g_data = 0;

// C++ UNDEFINED BEHAVIOUR — not just an ARM hardware problem.
// Without a synchronises-with edge between the write to g_data and the
// read of g_data, the C++ abstract machine offers no defined publication
// of the non-atomic object — regardless of hardware. x86's TSO may hide
// the symptom; ARM exposes it; neither platform makes the program correct.
// Thread A:
g_data = 42;
g_ptr.store(new int(99), std::memory_order::relaxed);  // no release

// Thread B:
int* p = g_ptr.load(std::memory_order::relaxed);       // no acquire
if (p) use(g_data);  // data race on g_data → UB. x86 may return 42 by accident.
```

Write to the C++ memory model, not to the x86 memory model. If you do not have ARM CI, get some before ARM CI gets you.

---

## 6.16 Putting It Together: SPSC Queue

Theory in a working snippet. Single-producer / single-consumer bounded queue.
Every memory order choice is the minimum correct — no over-payment.

```cpp
#include <atomic>
#include <array>
#include <optional>
#include <utility>
#include <cstddef>

template<typename T, size_t N>
class SpscQueue
{
    static_assert(N > 1 && (N & (N-1)) == 0, "N must be power of two");
public:
    bool push(T v)
    {
        const size_t h    = m_head.load(std::memory_order::relaxed);
        const size_t next = (h + 1) & (N - 1);

        // Acquire: see slot reclamations from pop()'s release store on m_tail.
        if (next == m_tail.load(std::memory_order::acquire))
            return false;  // full

        m_buf[h] = std::move(v);

        // Release: publish element and head to consumer's acquire load.
        m_head.store(next, std::memory_order::release);
        return true;
    }

    std::optional<T> pop()
    {
        const size_t t = m_tail.load(std::memory_order::relaxed);

        // Acquire: see element published by producer's release store on m_head.
        if (t == m_head.load(std::memory_order::acquire))
            return std::nullopt;  // empty

        T v = std::move(m_buf[t]);

        // Release: tell producer this slot is free.
        m_tail.store((t + 1) & (N - 1), std::memory_order::release);
        return v;
    }

private:
    alignas(64) std::atomic<size_t> m_head{ 0 };  // written by producer only
    alignas(64) std::atomic<size_t> m_tail{ 0 };  // written by consumer only
    alignas(64) std::array<T, N>    m_buf{};       // own cache line: no false sharing with m_tail
};
```

`alignas(64)` prevents false sharing — head, tail, and the buffer each occupy their own cache line. Without it, `m_buf` would share a line with `m_tail`: every producer write into `m_buf[0]` would invalidate the consumer's cached copy of `m_tail`, and every consumer update to `m_tail` would bounce the line back to the producer.

---

## 6.17 The Five-Question Audit

Before writing any atomic, answer these. All of them. In order.

1. **What invariant am I protecting?** Not "thread safety" — be specific. "After `flag` is
   `true`, `buffer` holds a complete, valid object."

2. **Who reads, who writes?** SPSC allows simpler design than MPMC. Do not write MPMC for
   a SPSC problem.

3. **What other memory must be visible together?** If the atomic is a gate for other data,
   that data must be ordered with the atomic. If it is a standalone counter, nothing else is.

4. **What is the cheapest order that provides that visibility?** Start at `seq_cst`. Downgrade
   to `acq_rel`, then `acquire`/`release`, then `relaxed` — only when you can prove it is
   sufficient. Document the proof in a comment.

5. **Does TSan agree?** `-fsanitize=thread` (Clang/GCC on Linux/macOS; MSVC does not yet ship TSan).
   If TSan fires, you have a race. If TSan is silent, it is necessary but not sufficient —
   write tests that actually exercise the concurrent paths.

---

## Quick-Reference: The Whole Toolbox

```cpp
#include <atomic>
#include <memory>

std::atomic<int>    a{ 0 };
std::atomic_flag    f;  // default-initialized to clear since C++20; ATOMIC_FLAG_INIT deprecated

// Load / store
int  v = a.load(std::memory_order::acquire);
a.store(42, std::memory_order::release);

// RMW — all return OLD value
int prev = a.fetch_add(1,    std::memory_order::acq_rel);
prev      = a.fetch_sub(1,   std::memory_order::acq_rel);
prev      = a.fetch_and(0xFF, std::memory_order::relaxed);
prev      = a.fetch_or(0x01,  std::memory_order::relaxed);
prev      = a.fetch_xor(0x01, std::memory_order::relaxed);

// CAS
int expected = 0;
a.compare_exchange_weak(expected,   1, std::memory_order::acq_rel, std::memory_order::acquire);
a.compare_exchange_strong(expected, 1, std::memory_order::seq_cst, std::memory_order::acquire);

// atomic_flag
bool was_set = f.test_and_set(std::memory_order::acquire);
f.clear(std::memory_order::release);
bool cur = f.test(std::memory_order::acquire);  // C++20

// C++20 wait / notify
a.wait(0, std::memory_order::acquire);   // block until a != 0
a.notify_one();
a.notify_all();

// Lock-free check
static_assert(std::atomic<long>::is_always_lock_free);
bool lf = a.is_lock_free();

// atomic_ref (C++20)
int raw = 0;
std::atomic_ref<int>{ raw }.store(42, std::memory_order::release);

// Fence (rare)
std::atomic_thread_fence(std::memory_order::seq_cst);
```

---

## Closing Mantra

If you do not know why your memory order is correct, it is wrong.

---

*Chapter 7 — Futures: `async`, `future`, `promise`, `packaged_task`*
