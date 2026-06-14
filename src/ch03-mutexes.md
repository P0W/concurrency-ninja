# Chapter 3 — Mutexes & Locks

> *"The single-occupancy airplane bathroom of thread synchronization.  
> Someone is always inside. You will wait. The door will eventually open.  
> You will never know what happened in there."*

---

## Cold Open: The 3 AM ABBA Deadlock

Two threads. Two mutexes. Opposite acquisition order. One production incident.

```cpp
// BROKEN: Thread A does transfer(acc1, acc2), Thread B does transfer(acc2, acc1).
// They run simultaneously. Neither ever finishes. Congratulations.
void transfer_broken(Account& from, Account& to, long long cents)
{
    std::lock_guard lock_from(from.mtx);  // Thread A locks acc1, Thread B locks acc2
    std::lock_guard lock_to(to.mtx);      // Thread A waits for acc2, Thread B waits for acc1
    // Both threads wait forever. CPU is 0%. Service is frozen. Pager goes off.
    from.balance -= cents;
    to.balance   += cents;
}
```

One sentence: both threads now sit, arms folded, each holding what the other needs — a deadlock so textbook it has its own name.

The on-call engineer, staring at zero-CPU stack traces at 3:19 AM, writes:

> *two threads, two mutexes*  
> *each holds what the other needs*  
> *kubectl rollback*

The fix is one line:

```cpp
// FIXED: scoped_lock acquires both mutexes via a deadlock-avoidance algorithm.
// Argument order does not determine acquisition order. Never deadlocks.
void transfer(Account& from, Account& to, long long cents)
{
    if (&from == &to) return;             // self-transfer: same mutex, UB to double-lock
    std::scoped_lock lock(from.mtx, to.mtx);
    from.balance -= cents;
    to.balance   += cents;
}
```

The rest of this chapter is the explanation of why that fix works, and every other trap you will step on before you memorize it.

> **Intuition pump (real-world):** Classic ABBA deadlock. Thread A holds the key to the front door and needs the key to the back door. Thread B holds the key to the back door and needs the key to the front door. Both are standing there, arms folded, staring at each other with the dead-eyed conviction of people who have been waiting since 3:09 AM. `scoped_lock` is the manager who walks over and, using a documented internal protocol that may try, back off, and retry until both are acquired without circular wait, puts both keys on the table simultaneously. Both threads get both keys. Nobody deadlocks. The on-call engineer composes no haiku tonight.

---

## What a Mutex Actually Is

A mutex (*mutual exclusion*) ensures at most one thread is inside a critical section at a time. Not a barrier. Not a queue. Not magic. Under the hood it's a platform primitive: a Linux *futex*, a Windows `SRWLock`, a macOS `os_unfair_lock`. The standard wraps them uniformly.

Two guarantees matter:

1. **Mutual exclusion** — if A holds it, nobody else gets in.
2. **Memory ordering** — `lock()` is an *acquire*, `unlock()` is a *release*. Writes before A's `unlock()` are visible to B after B's `lock()`.

Without (2) you'd need separate atomics just to see each other's writes. The mutex does both.

> **Intuition pump (real-world):** `std::mutex` is the single-occupancy airplane bathroom. Doesn't matter if there are 20 people in the aisle — one in, one out, no exceptions, no negotiation. The door lock is a binary primitive. Occupied or not. The queue outside doesn't have priority, doesn't have fairness guarantees, doesn't have a ticket system. It's just 20 people sighing and shifting their weight while whoever's inside does whatever they're doing. You will never know what happened in there. The mutex makes the same guarantees: exclusion, memory visibility, and absolutely zero information about what went on between lock and unlock.

---

## The Mutex Family

| Type | Header | When to reach for it |
|------|--------|---------------------|
| `std::mutex` | `<mutex>` | Default. Fast. Non-recursive. Use this 90% of the time. |
| `std::recursive_mutex` | `<mutex>` | Same thread can lock N times. Usually a design smell. |
| `std::timed_mutex` | `<mutex>` | Adds `try_lock_for` / `try_lock_until`. Watchdogs and fallbacks. |
| `std::recursive_timed_mutex` | `<mutex>` | Combination of the above two. The code has made life choices. |
| `std::shared_mutex` | `<shared_mutex>` | C++17. Many readers OR one writer. Read-dominated workloads. |
| `std::shared_timed_mutex` | `<shared_mutex>` | C++14. `shared_mutex` with timed ops. Predates the former. |

The exoticism of your mutex is inversely proportional to the quality of your design. If you are reaching for `std::recursive_timed_mutex`, schedule a design review before the code review.

---

## RAII or Bust — Never Call `lock()` Directly

```cpp
// BROKEN: exception between lock() and unlock() = mutex locked forever
void do_work_broken()
{
    m.lock();
    might_throw();   // if this throws, unlock() never runs
    m.unlock();      // unreachable. congratulations on your hung service.
}
```

Exception thrown, early return added six months later, colleague "just refactoring" — any of these kills you. The mutex stays locked. Every waiter waits forever. No crash, no log — just silence, which is the worst possible diagnostic.

```cpp
// FIXED: destructor calls unlock() unconditionally, on every exit path
#include <mutex>

std::mutex m;

void do_work()
{
    std::lock_guard lock(m);  // lock() on construction
    might_throw();            // throws? destructor runs. mutex unlocked.
}                             // returns? destructor runs. mutex unlocked.
```

Commit to muscle memory: **if you write `m.lock()` without a RAII wrapper on the same line, the code review is "no."**

---

## The Lock Wrapper Family

| Type | Properties | When to use |
|------|-----------|-------------|
| `std::lock_guard<M>` | Cheapest. No unlock. No move. Scope-bound. | Default for a single mutex. |
| `std::unique_lock<M>` | Movable. Can defer, unlock, relock manually. | Condition variables. Any flexibility need. |
| `std::scoped_lock<M...>` | C++17. Variadic. Deadlock-free for multiple mutexes. | Whenever you need ≥2 mutexes simultaneously. |
| `std::shared_lock<M>` | Reader lock on `shared_mutex`. | Read-only critical section. |

Reach for `lock_guard` first. Upgrade to `unique_lock` only when condition variables or manual unlock are required. `unique_lock` is larger and slower — don't use it "for flexibility" on a code path that doesn't need it.

---

## Bank Transfer — Broken and Fixed, Side by Side

Money is in *cents* (integer). If you store money as `float` you will explain `0.10 + 0.20 == 0.30000000000000004` to a bank regulator. Use integers.

```cpp
#include <mutex>
#include <stdexcept>

struct Account
{
    explicit Account(int id, long long initial_cents) : id(id), balance(initial_cents) {}
    Account(const Account&)            = delete;
    Account& operator=(const Account&) = delete;

    const int id;
    mutable std::mutex mtx;
    long long balance;   // cents. always. no exceptions.
};
```

```cpp
// BROKEN: lock acquisition order depends on call-site argument order.
// transfer(a, b) in Thread 1 and transfer(b, a) in Thread 2 = ABBA deadlock.
void transfer_broken(Account& from, Account& to, long long cents)
{
    std::lock_guard lf(from.mtx);
    std::lock_guard lt(to.mtx);
    if (from.balance < cents) throw std::runtime_error("insufficient funds");
    from.balance -= cents;
    to.balance   += cents;
}
```

```cpp
// FIXED: scoped_lock uses an internal deadlock-avoidance algorithm.
// Argument order is irrelevant. Deadlock is impossible.
void transfer(Account& from, Account& to, long long cents)
{
    if (cents <= 0) throw std::invalid_argument("amount must be positive");
    if (&from == &to) return;   // same mutex twice on non-recursive = UB

    std::scoped_lock lock(from.mtx, to.mtx);

    if (from.balance < cents)
        throw std::runtime_error("insufficient funds");
    from.balance -= cents;
    to.balance   += cents;
}
```

---

## Pre-C++17 Multi-Lock Idiom (Know It, Don't Write It)

```cpp
#include <mutex>

// Pre-C++17: std::lock() + adopt_lock. Correct but verbose.
void transfer_pre17(Account& from, Account& to, long long cents)
{
    if (&from == &to) return;

    std::lock(from.mtx, to.mtx);  // deadlock-free acquisition of both

    // adopt_lock: mutex is already locked; just manage the unlock
    std::lock_guard<std::mutex> lf(from.mtx, std::adopt_lock);
    std::lock_guard<std::mutex> lt(to.mtx,   std::adopt_lock);

    from.balance -= cents;
    to.balance   += cents;
}
// You will see this in existing codebases. Understand it. Write scoped_lock instead.
```

> **Trap**: call `std::lock(a, b)` and forget to adopt-guard one of them — that mutex leaks its lock. This exact mistake has shipped in production code written by people with "Senior" in their title.

---

## `std::unique_lock` — All the Tags and Manual Controls

```cpp
#include <mutex>
#include <condition_variable>

std::mutex m;
std::condition_variable cv;
bool ready = false;

void unique_lock_demo()
{
    // 1. defer_lock: construct WITHOUT locking. Lock manually later.
    {
        std::unique_lock<std::mutex> lk(m, std::defer_lock);
        // ... do setup that doesn't need the lock ...
        lk.lock();   // acquire now
    }                // destructor unlocks

    // 2. Manual unlock and relock (impossible with lock_guard)
    {
        std::unique_lock<std::mutex> lk(m);
        lk.unlock();   // temporarily release
        // ... non-critical work ...
        lk.lock();     // re-acquire
    }

    // 3. adopt_lock: mutex already locked externally; transfer ownership to unique_lock.
    //    Exactly one RAII owner at a time — raw lock first, then adopt. Never overlap.
    {
        m.lock();                                              // lock manually
        std::unique_lock<std::mutex> lk(m, std::adopt_lock);  // adopt: m is now owned by lk
    }  // lk destructor unlocks

    // 4. try_to_lock: non-blocking attempt. m must NOT be held by this thread.
    {
        std::unique_lock<std::mutex> lk(m, std::try_to_lock);
        if (lk.owns_lock())
        {
            // 5. release(): surrender ownership WITHOUT unlocking (C API hand-off).
            //    Only call release() when you know you have the lock — check owns_lock() first.
            std::mutex* raw = lk.release();   // lk no longer tracks it; you are responsible
            raw->unlock();
        }
        // if !owns_lock(): lk destructor does nothing — nothing to release
    }
}

// Why condition_variable requires unique_lock and NOT lock_guard:
// cv.wait() must atomically unlock + sleep + relock. lock_guard has no unlock().
void consumer()
{
    std::unique_lock<std::mutex> lk(m);      // lock_guard won't compile here
    cv.wait(lk, []{ return ready; });        // unlock, sleep, relock on notify
    // process data with lock held
}

void producer()
{
    { std::lock_guard<std::mutex> lk(m); ready = true; }
    cv.notify_one();
}
```

---

## `std::shared_mutex` — The Read-Mostly Hammer

Writers shut down all lanes. Readers cruise in parallel. You can have many readers simultaneously or one writer — not both.

```cpp
#include <shared_mutex>
#include <mutex>
#include <unordered_map>
#include <string>

class ConfigStore
{
public:
    // Many threads can call get() simultaneously — shared_lock allows it
    std::string get(const std::string& key) const
    {
        std::shared_lock lock(m_mutex);   // reader: shared ownership
        auto it = m_data.find(key);
        return it != m_data.end() ? it->second : std::string{};
    }

    // Only one thread at a time — unique_lock excludes all readers and writers
    void set(const std::string& key, std::string value)
    {
        std::unique_lock lock(m_mutex);   // writer: exclusive ownership
        m_data.insert_or_assign(key, std::move(value));
    }

private:
    mutable std::shared_mutex m_mutex;
    std::unordered_map<std::string, std::string> m_data;
};
```

`shared_mutex` wins when reads genuinely dominate writes by 10x or more and the critical section is non-trivial enough to amortize the reader-tracking overhead. On an uncontended path, `std::mutex` is often faster — `shared_mutex` reader acquisition still touches a shared cache line that every reader writes to.

The standard does not guarantee writer fairness. High-throughput readers can starve writers indefinitely. If writes start taking unexpectedly long under read load, that's your diagnosis.

> **Intuition pump (real-world):** `shared_mutex` is the library reading room. Dozens of people can be in there simultaneously, all reading different books — perfect harmony. But the moment someone shows up with a ladder and paint samples to repaint the walls (that's a writer), the implementation *may or may not* stop new readers from entering — the C++ standard makes no fairness guarantee. In practice, a relentless stream of readers can keep the painter waiting outside indefinitely: the room never fully empties, the walls never get painted. It's a good system when reads genuinely dominate. Just don't repaint the walls 40 times a second and expect it to be fast, and don't expect the standard to be fair to your painter.

> **No upgrade path exists.** You cannot atomically upgrade a `shared_lock` to a `unique_lock`. Release and re-acquire creates a race window — always re-validate after upgrading.

---

## `std::recursive_mutex` — Almost Always Wrong

```cpp
// SMELL: foo() holds the lock and calls bar(), which also tries to lock.
// On std::mutex this deadlocks. So you reach for recursive_mutex. Stop.
class Cache
{
    std::recursive_mutex m_mutex;   // code smell
public:
    void insert(const std::string& k, int v)
    {
        std::lock_guard lock(m_mutex);
        m_data[k] = v;
        invalidate_stats();   // this locks m_mutex again. "works" with recursive_mutex.
    }
    void invalidate_stats()
    {
        std::lock_guard lock(m_mutex);   // locks again on same thread
        m_stats_dirty = true;
    }
    // ...
};
```

```cpp
// FIXED: split public surface (acquires lock) from _locked impl (assumes lock held).
// Suffix _locked = "caller must hold m_mutex". Enforce in code review.
class Cache
{
    std::mutex m_mutex;   // plain mutex. no recursion needed.
public:
    void insert(const std::string& k, int v)
    {
        std::lock_guard lock(m_mutex);
        insert_locked(k, v);           // no re-lock; invariant: lock is held
        invalidate_stats_locked();     // same
    }
    void invalidate_stats()
    {
        std::lock_guard lock(m_mutex);
        invalidate_stats_locked();
    }
private:
    void insert_locked(const std::string& k, int v) { m_data[k] = v; }
    void invalidate_stats_locked()                  { m_stats_dirty = true; }

    std::unordered_map<std::string, int> m_data;
    bool m_stats_dirty{false};
};
```

Needing `recursive_mutex` means your invariants don't hold at function call boundaries. Fix the design. Legitimate exception: recursive user-provided callback graphs you can't restructure — but even then, a deferred-dispatch queue usually beats recursion.

> **Intuition pump (real-world):** `recursive_mutex` lets the same person walk into the airplane bathroom five times in a row. Technically legal. But if you've designed a system where the same thread *needs* to re-enter a lock it already holds, your code is standing in a bathroom doorway, stepping in, stepping back out, stepping in again, and asking "wait, why am I in here?" That question is worth answering before you reach for the recursive variant. The correct answer is almost always: split your public API from your internal `_locked` implementation and stop trying to hold the lock from both sides of the same call stack.

---

## Production Snippet — Thread-Safe Metrics Registry

Reads dominate (every hot-path call increments a counter). Writes are rare (new counter names). Classic `shared_mutex` workload.

```cpp
#include <mutex>
#include <shared_mutex>
#include <unordered_map>
#include <string>
#include <atomic>
#include <memory>
#include <stdexcept>

class MetricsRegistry
{
public:
    void increment(const std::string& name, long long delta = 1)
    {
        // Fast path: counter exists — shared lock, lock-free increment
        {
            std::shared_lock lock(m_mutex);
            auto it = m_counters.find(name);
            if (it != m_counters.end())
            {
                it->second->fetch_add(delta, std::memory_order_relaxed);
                return;
            }
        }
        // Slow path: new counter — exclusive lock, then re-check existence before
        // allocating. Without the re-check, contending threads all call make_shared
        // under the exclusive lock and discard the losing allocations.
        std::unique_lock lock(m_mutex);
        auto it2 = m_counters.find(name);
        if (it2 == m_counters.end())
            it2 = m_counters.emplace(
                name, std::make_shared<std::atomic<long long>>(0LL)).first;
        it2->second->fetch_add(delta, std::memory_order_relaxed);
    }

    std::unordered_map<std::string, long long> snapshot() const
    {
        std::shared_lock lock(m_mutex);
        std::unordered_map<std::string, long long> result;
        result.reserve(m_counters.size());
        for (const auto& [name, ctr] : m_counters)
            result[name] = ctr->load(std::memory_order_relaxed);
        return result;
    }

private:
    mutable std::shared_mutex m_mutex;
    // shared_mutex guards map structure only. counter values are atomics — lock-free.
    std::unordered_map<std::string, std::shared_ptr<std::atomic<long long>>> m_counters;
};
```

---

## Production Snippet — Copy-on-Write Config Holder

Read on every request. Updated infrequently. Lock held for nanoseconds (pointer swap), never for the duration of actual config access.

```cpp
#include <mutex>
#include <shared_mutex>
#include <memory>
#include <string>
#include <vector>

struct AppConfig
{
    int    pool_size{10};
    int    timeout_ms{5000};
    bool   debug{false};
    std::vector<std::string> allowed_origins;
};

class ConfigHolder
{
public:
    static ConfigHolder& instance()
    {
        static ConfigHolder h;
        return h;
    }

    // Hot path: shared lock + one refcount bump. Lock held ~nanoseconds.
    // Caller gets a stable snapshot. No lock held during actual use.
    std::shared_ptr<const AppConfig> get() const
    {
        std::shared_lock lock(m_mutex);
        return m_config;
    }

    // Called by config-reload thread. Build new config entirely outside the lock.
    // Lock held only for the pointer swap. Old config destructs *after* unlock.
    void update(std::shared_ptr<const AppConfig> new_cfg)
    {
        std::unique_lock lock(m_mutex);
        m_config.swap(new_cfg);
        // lock releases here; new_cfg (now holding the old config) destructs after.
    }

private:
    ConfigHolder() : m_config(std::make_shared<AppConfig>()) {}
    mutable std::shared_mutex m_mutex;
    std::shared_ptr<const AppConfig> m_config;
};

// Usage:
void handle_request()
{
    auto cfg = ConfigHolder::instance().get();  // nanoseconds, shared lock
    // No lock held here. cfg is an immutable snapshot.
    if (cfg->debug) { /* ... */ }
}
```

---

## Lock Granularity — Lock as Little as Possible

```cpp
#include <fstream>
#include <mutex>
#include <string>
#include <vector>

// BROKEN: lock held across file I/O.
// File I/O takes milliseconds. Every other thread blocks for milliseconds.
std::mutex m_records_mutex;
std::vector<std::string> m_records;

void flush_broken(const std::string& path)
{
    std::lock_guard lock(m_records_mutex);

    std::ofstream f(path, std::ios::app);
    for (const auto& r : m_records)
        f << r << '\n';   // slow I/O under lock. everyone waits.
    m_records.clear();
}
```

```cpp
// FIXED: copy out under lock (microseconds), then do I/O lock-free (milliseconds)
void flush(const std::string& path)
{
    std::vector<std::string> to_flush;
    {
        std::lock_guard lock(m_records_mutex);
        to_flush = std::move(m_records);   // O(1) move
    }                                       // lock released here

    // I/O happens with no mutex held. Other threads write records freely.
    std::ofstream f(path, std::ios::app);
    for (const auto& r : to_flush)
        f << r << '\n';
}
```

Rule: compute and prepare outside the lock, swap or copy under the lock, finish outside. Things you must not do while holding a lock: file I/O, network I/O, `sleep`/`yield`, logging (unless lock-free), user callbacks, heap allocation in hot paths.

> **Intuition pump (real-world):** Lock granularity is about how wide you cast the exclusion net. You don't lock the entire library to read one book — you grab the book, step outside, read at your leisure. You don't hold the kitchen mutex while you're on the phone ordering DoorDash. The delivery ETA is not your critical section. The moment you touch shared state: lock. The moment you're done with it: unlock. Everything in between — the I/O, the computation, the three-minute wait for the network call — that happens in the open air, with no mutex in sight, while other threads proceed with their lives unimpeded.

---

## `try_lock` — Watchdog in 10 Lines

```cpp
#include <mutex>
#include <chrono>
#include <thread>
#include <atomic>

std::timed_mutex  g_work_mutex;
std::atomic<bool> g_shutdown{false};

// Watchdog: if worker holds the lock >500ms, it has probably hung.
void watchdog()
{
    using namespace std::chrono_literals;
    while (!g_shutdown.load(std::memory_order_relaxed))
    {
        {
            std::unique_lock<std::timed_mutex> lock(g_work_mutex, std::defer_lock);
            if (!lock.try_lock_for(500ms))
            {
                // log_hang_detected();
            }
        }   // lock released here — worker can proceed whether or not it was acquired
        std::this_thread::sleep_for(1s);
    }
}
```

Don't use `try_lock` as a routine performance trick. `if (!m.try_lock()) { skip(); }` in a hot path means you have a contention problem. `try_lock` masks it; profiling solves it.

> **Intuition pump (real-world):** `try_lock` is checking if the airplane bathroom is free without committing to the queue. You walk up, jiggle the handle, occupied — fine, you go sit back down and find something else to do. You try again in a minute. This is *correct and useful* for watchdog patterns and backoff strategies. What it is NOT: a performance hack. If your hot path is `try_lock`-ing and skipping on failure, you have just invented a probabilistic skip-work-under-contention mechanism and called it a feature. The actual feature is: fix the contention. Profile. Narrow the critical section. `try_lock` is a handle-jiggle, not an architecture.

---

## Lock Ordering Hygiene

`scoped_lock` covers the common case — two or three known mutexes in the same scope. The case it *cannot* cover is a **dynamic collection**: a `std::vector<Node*>` where each node carries its own mutex and you need to lock an arbitrary subset at runtime. You cannot feed a `vector` to `scoped_lock`. This is the one scenario where manual address ordering is not just idiomatic but necessary.

```cpp
#include <algorithm>
#include <functional>
#include <mutex>
#include <vector>

// Lock N mutexes from a dynamic collection, deadlock-free.
// Strategy: every call site on every thread sorts by the same key (pointer address).
// Global consistent order → no ABBA, no matter how many threads or how many nodes.
//
// RAII gap: callers own the unlock. If the locked state must escape this call site,
// return std::vector<std::unique_lock<std::mutex>> with std::adopt_lock instead.
void lock_all(std::vector<std::mutex*> mutexes)
{
    // 1. Consistent global acquisition order — std::less<std::mutex*> gives a
    //    portable strict total order over pointers to unrelated objects.
    //    Production code often assigns stable numeric IDs to lock-bearing objects
    //    so ordering is independent of allocation address.
    std::sort(mutexes.begin(), mutexes.end(), std::less<std::mutex*>{});

    // 2. Remove duplicates — locking the same non-recursive mutex twice is UB.
    mutexes.erase(std::unique(mutexes.begin(), mutexes.end()), mutexes.end());

    // 3. Sequential acquisition. Unwind cleanly if any lock() throws std::system_error.
    std::size_t locked = 0;
    try
    {
        for (auto* m : mutexes) { m->lock(); ++locked; }
    }
    catch (...)
    {
        for (std::size_t i = locked; i-- > 0;) mutexes[i]->unlock();
        throw;
    }
}
```

If you are about to write this for the two-mutex case: stop. Use `scoped_lock`.

| You have | Correct tool |
|----------|-------------|
| 2–N *known* mutexes in the same scope | `std::scoped_lock lock(a, b, c);` — one line, done |
| N mutexes in a `vector` you cannot enumerate at compile time | `lock_all(vec)` — sort, deduplicate, sequential acquire |

Document your lock ordering at the top of every file that acquires multiple mutexes. Lock ordering that lives in someone's head leaves with them.

---

## Pitfall Table — The Museum of Regrettable Decisions

| Pitfall | What Happens | Fix |
|---------|-------------|-----|
| Releasing a lock you don't hold | UB. Silent corruption or debug assertion. | Use RAII. Never call raw `unlock()`. |
| Locking the same `std::mutex` twice on one thread | UB. Manifests as deadlock. | `_locked` pattern. Or `recursive_mutex` if you must. |
| Holding a lock across a callback | Callback calls back into your code. Deadlock. | Copy callback list under lock, release, invoke outside. |
| Locking inside a destructor | May be called with unrelated locks held. Ordering inversion. | Explicit `shutdown()` method. Destructor asserts it was called. |
| `shared_lock` upgrade attempt | Release shared + re-acquire exclusive = race window. | Re-validate after upgrading. Or acquire exclusive from the start. |
| `adopt_lock` when mutex is not locked | Destructor unlocks something you don't own. UB. | Only use `adopt_lock` after `std::lock()` or a verified `m.lock()`. |
| Exposing the mutex as a public member | Any thread locks it for any duration. You've surrendered all invariants. | Keep the mutex private. Provide atomic multi-step operations instead. |
| Copy or move a mutex | Doesn't compile. | Good. That's the compiler doing you a favor. |

---

## Callouts

> **`mutable` on mutex members** — A `const` member function that logically reads-only still needs to lock. Declare the mutex `mutable std::mutex m_mutex` so `const` methods can acquire it without `const_cast`. This is the one legitimate use of `mutable` in a const-correct class.

> **`std::mutex` is neither copyable nor movable** — Any class containing a mutex by value is also non-copyable. A movable class containing a mutex needs a custom move constructor that locks the source, moves the data, then unlocks. This is a common "why won't this compile" moment at 2 PM on a Friday.

> **Don't confuse the critical section with the lock scope** — The critical section is the *minimum* set of operations that must run atomically for your invariants to hold. Holding the lock beyond that is a performance bug. Releasing it before that is a correctness bug. These are symmetrically painful.

---

## Muscle-Memory Cheat Sheet

```cpp
#include <mutex>
#include <shared_mutex>

std::mutex              m;
std::shared_mutex       ms;
std::timed_mutex        mt;

// One mutex, scope-bound
std::lock_guard lock(m);

// One mutex, condition variable
std::unique_lock lock(m);
cv.wait(lock, pred);

// Two or more mutexes — never deadlocks
std::scoped_lock lock(m1, m2, m3);

// Reader lock
std::shared_lock lock(ms);

// Try without blocking
if (auto lk = std::unique_lock(m, std::try_to_lock); lk.owns_lock()) { /*...*/ }

// Try with timeout
std::unique_lock lk(mt, std::defer_lock);
if (lk.try_lock_for(std::chrono::milliseconds(100))) { /*...*/ }

// Snapshot (copy out under lock, use outside)
T snapshot;
{ std::lock_guard lk(m); snapshot = shared_data; }
use(snapshot);  // no lock held

// Copy-on-write update (build outside, swap under lock)
auto fresh = build_new_value();
{ std::unique_lock lk(m); shared_ptr_value = std::move(fresh); }
```

---

## What We Learned

The mutex is the oldest synchronization primitive in the book and the most misused. The problems are not subtle:

1. Forgetting to unlock — RAII has been the answer since 1994.
2. Wrong lock order — `scoped_lock` has been the answer since C++17.
3. Holding the lock too long — compute outside, copy out, release early.
4. Locking too little — leaving a window between related operations.
5. Wrong mutex type — start with `std::mutex`; reach for `shared_mutex` only with measurements in hand.

Get these right and the 3 AM pager stops going off. The on-call engineer does not compose haiku. The payment pipeline processes payments. Everyone sleeps.

---

## Closing Mantra

**`lock_guard` for one, `scoped_lock` for many, and never roll your own.**
