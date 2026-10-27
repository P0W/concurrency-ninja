# Chapter 8 — C++20 Synchronization: latch, barrier, semaphore

> *The standard library finally noticed that you've been writing the same broken barrier since 2003.*

---

## 8.1 Cold Open: Before and After

You need N threads to synchronize at a phase boundary. Simple, you say. You reach for a mutex, a condition variable, and an int. One hour later:

```cpp
// The hand-rolled barrier every senior engineer has written at least once.
// Count the bugs. I'll wait.
class ManualBarrier {
    std::mutex              m_mtx;
    std::condition_variable m_cv;
    int                     m_count;
    int                     m_total;
    int                     m_generation{0};   // BUG #1 candidate: missed increment → stale wake
public:
    explicit ManualBarrier(int n) : m_count(n), m_total(n) {}

    void arrive_and_wait() {
        std::unique_lock lock(m_mtx);
        int gen = m_generation;
        if (--m_count == 0) {
            ++m_generation;
            m_count = m_total;                 // Reset and notify both happen under the lock,
            m_cv.notify_all();                 //   so late arrivals see the new generation and
                                               //   wait correctly for the next phase.
                                               // Real hazard: exception safety — a thread that
                                               //   throws before reaching here skips its
                                               //   count-down and everyone waits forever.
        } else {
            m_cv.wait(lock, [&] {              // BUG #3 candidate: signed int m_generation
                return m_generation != gen;    //   overflows after 2^31 phases — undefined
            });                                //   behaviour, not a clean wrap. Use unsigned.
        }
        // And you still haven't added a completion function.
        // That's another 30 lines, a second cv, and a bool flag.
        // Your reviewer just opened a comment thread.
    }
};
```

Six weeks, two reviewer threads, one production incident (physics sim, occasionally skipped a frame, root cause: generation integer overflow on a 96-core server), one 2 AM postmortem.

The C++20 version:

```cpp
#include <barrier>
#include <utility>   // std::swap

// Schematic — N, front, and back are your application's state:
std::barrier sync(N, [&]() noexcept { std::swap(front, back); }); // completion fn included

// In each thread, every phase:
sync.arrive_and_wait();
```

That's the thesis of this chapter. Everything else is just API surface.

---

## 8.2 The Three Primitives at a Glance

| Primitive | Header | Reusable? | Mental model | Canonical use |
|---|---|---|---|---|
| `std::latch` | `<latch>` | **No** — one-shot | Starting pistol, fires once | Wait until N things finish, exactly once |
| `std::barrier<Fn>` | `<barrier>` | **Yes** — auto-resets per phase | Dance studio: nobody moves until everyone's in formation | N threads sync every iteration; optional completion fn between phases |
| `std::counting_semaphore<N>` | `<semaphore>` | **Yes** | Parking-lot attendant with N permits | Throttle concurrency, signal, resource pool |
| `std::binary_semaphore` | `<semaphore>` | **Yes** | Single-slot signal | One thread kicks another; alias for `counting_semaphore<1>` |

---

## 8.3 `std::latch` — The One-Shot Starting Pistol

**API in one table:**

| Method | Blocking? | What it does |
|---|---|---|
| `count_down(n=1)` | No | Subtract n from counter; unblocks waiters if counter reaches 0 |
| `wait()` | Yes | Block until counter == 0 |
| `try_wait()` | No | Return true if counter already == 0; poll, not wait. May return false spuriously ([thread.latch.class]/12). |
| `arrive_and_wait(n=1)` | Yes | `count_down(n)` + `wait()` in one call |

Three rules burned into your brain:

1. **One-shot.** Counter reaches zero; the latch stays done. There is no reset. `count_down` past zero is UB; further `wait`/`try_wait` calls are well-defined but return immediately.
2. **`count_down` past zero → UB.** Not an exception. Memory corruption.
3. **Counter is `ptrdiff_t`.** Signed. Don't pass negative values.

```cpp
// Quick sanity check on latch mechanics:
std::latch fence(3);
fence.count_down();       // counter = 2
fence.count_down();       // counter = 1
fence.count_down();       // counter = 0 → all wait() calls unblock
// fence.count_down();    // DON'T. UB. The pistol already fired.
```

> **Intuition pump (real-world):** `std::latch` is the one-shot starting pistol at a track meet. You pull it ONCE. The runners go. You cannot un-fire it. Try to `count_down()` past zero and you get UB — the official confiscates your starter's pistol, your license, and probably your track-and-field career. The pistol has exactly one job. It does it exactly once. Congratulations on understanding a piece of state machinery, apparently the first time.

### 8.3.1 Production Snippet — Parallel Initialization Fence

Three subsystems init in parallel; main thread waits on a latch before opening the service port. The `count_down` must fire on the error path too, or the main thread parks forever.

```cpp
#include <latch>
#include <thread>
#include <exception>

void warm_cache();
void open_db_pool();
void load_ml_model();
void open_service_port();

void start_server() {
    constexpr std::ptrdiff_t k_n = 3;
    std::latch init_fence(k_n);

    std::exception_ptr err_cache, err_db, err_model;

    std::jthread t1([&] {
        try   { warm_cache(); }
        catch (...) { err_cache = std::current_exception(); }
        init_fence.count_down();   // fires unconditionally — error or not
    });
    std::jthread t2([&] {
        try   { open_db_pool(); }
        catch (...) { err_db = std::current_exception(); }
        init_fence.count_down();
    });
    std::jthread t3([&] {
        try   { load_ml_model(); }
        catch (...) { err_model = std::current_exception(); }
        init_fence.count_down();
    });

    init_fence.wait();   // blocks until all three count_down() calls complete

    t1.join(); t2.join(); t3.join();  // join before inspecting exception_ptrs

    if (err_cache) std::rethrow_exception(err_cache);
    if (err_db)    std::rethrow_exception(err_db);
    if (err_model) std::rethrow_exception(err_model);

    open_service_port();
}
```

No mutex. No condition variable. No atomic counter. No predicate lambda. The latch cannot be misused to fire twice because there is no increment path.

---

## 8.4 `std::barrier<Fn>` — The Dance Studio

**API in one table:**

| Method | Blocking? | What it does |
|---|---|---|
| `arrive_and_wait()` | Yes | Arrive; block until all N arrive; completion fn runs once; everyone unblocks; resets for next phase |
| `arrive(n=1)` | No | Arrive without blocking; returns `arrival_token` |
| `wait(token&&)` | Yes | Block using a token from a prior `arrive()` call |
| `arrive_and_drop()` | No | Last arrival from this thread ever; decrements expected count permanently |

Two rules you will violate exactly once before you remember them:

1. **Completion function must be nothrow-invocable; mark it `noexcept`.** This is a C++20 type requirement, not just a runtime guard — a conforming implementation may reject a potentially-throwing lambda at compile time. If a `noexcept`-marked completion somehow lets an exception escape at runtime, `std::terminate` is called. Catch inside it if needed.
2. **`arrive_and_drop()` then `arrive_and_wait()` in the same phase = double arrival = permanent deadlock.** Drop means you're gone. Act like it.

```cpp
// noexcept or die:
std::barrier sync(N, []() noexcept { /* swap buffers, rotate logs, whatever */ });

// arrive_and_drop: worker leaves the party permanently
if (this_thread_is_done_forever) {
    sync.arrive_and_drop();   // participates in this phase's sync, then exits
    return;
}
sync.arrive_and_wait();       // still in the rotation
```

> **Intuition pump (real-world):** `std::barrier` is the synchronized swim coach. Every swimmer arrives at the wall (`arrive_and_wait()`), touches the wall, and waits. The moment the *last* swimmer touches — the completion function fires: the coach yells "GO!" exactly once, while everyone is still paused. Then the whole team pushes off together for the next lap. Reusable across every lap, automatically. `arrive_and_drop()` is the swimmer who quietly climbs out of the pool after this lap — they still count for the current sync, but nobody waits for them next lap. The coach does not re-hire them. Pretending otherwise (calling `arrive_and_wait()` after dropping) leaves every remaining swimmer gripping the wall for eternity.

### 8.4.1 Production Snippet — Parallel Simulation with Phase Boundaries

N workers each own a partition of a grid. Each frame: update partition, sync, completion fn swaps double-buffers, repeat. This is the exact pattern physics sims, ray tracers, and ML training loops use.

```cpp
#include <barrier>
#include <thread>
#include <vector>
#include <stdexcept>
#include <cstddef>

struct Grid { std::vector<float> front, back; std::size_t W, H; };

void simulate(Grid& g, int frames, int nthreads) {
    if (nthreads <= 0)
        throw std::invalid_argument("simulate: nthreads must be > 0");
    const std::ptrdiff_t N = nthreads;

    // Completion fn: runs once per phase on one unspecified thread,
    // while all workers are blocked — zero concurrency, safe to mutate shared state.
    auto swap_fn = [&g]() noexcept { std::swap(g.front, g.back); };
    std::barrier sync(N, swap_fn);   // CTAD deduces CompletionFn; no std::move needed

    auto worker = [&](std::size_t tid) {
        const std::size_t row0 = tid * (g.H / N);
        const std::size_t row1 = (tid + 1 == static_cast<std::size_t>(N)) ? g.H
                                                                           : row0 + g.H / N;
        for (int f = 0; f < frames; ++f) {
            // Each thread writes to g.back[row0..row1); reads g.front. No overlap → no locks.
            for (std::size_t r = row0; r < row1; ++r)
                for (std::size_t c = 0; c < g.W; ++c)
                    g.back[r * g.W + c] = g.front[r * g.W + c]; // stencil goes here

            // All threads rendezvous here. swap_fn fires. Then everyone continues to frame f+1.
            sync.arrive_and_wait();
        }
    };

    std::vector<std::jthread> threads;
    threads.reserve(N);
    int spawned = 0;
    try {
        for (; spawned < nthreads; ++spawned)
            threads.emplace_back(worker, static_cast<std::size_t>(spawned));
    } catch (...) {
        // If thread creation throws halfway through, drop the unspawned participants from
        // the barrier so already-running workers don't wait forever during stack unwinding.
        for (int j = spawned; j < nthreads; ++j)
            sync.arrive_and_drop();
        throw;
    }
    // jthread joins on destruction
}
```

The barrier auto-resets. There is no generation counter. The completion function swap is serialized with zero extra locks. If you used a latch here you'd be constructing a new one every frame — please don't.

---

## 8.5 `std::counting_semaphore<N>` — The Parking-Lot Attendant

**API in one table:**

| Method | Blocking? | What it does |
|---|---|---|
| `acquire()` | Yes | Block until count > 0, then decrement |
| `try_acquire()` | No | Decrement if count > 0; else return false. May return false spuriously ([thread.sema.cnt]/13). |
| `try_acquire_for(dur)` | Bounded | Decrement if count > 0 within timeout (`false` on timeout); **check the bool**. |
| `try_acquire_until(tp)` | Bounded | Same, deadline-based (`false` on timeout). |
| `release(n=1)` | No | Add n to count; wake up to n waiters |

**No ownership.** Any thread can call `release()` — including one that never called `acquire()`. This is intentional for signaling and throttling. It is catastrophic if you treat it like a mutex. The distinction: mutex = ownership, semaphore = counting.

> **Intuition pump (real-world):** `counting_semaphore` is the parking-lot attendant with N permits clipped to their vest. You want to park — you `acquire()`, which blocks until a permit is available. You leave the lot — you `release()`, handing back a permit. Here's the part that bites people: *any* driver can hand back a permit. The person walking up from the street who never parked can `release()` and inflate the count. There is no "you must have parked here to leave" check. That is by design for signaling. It is catastrophic if you wanted a mutex. The distinction: mutex says "only the thread that locked it can unlock it." Semaphore says "I don't care who you are, put the permit in the box."

```cpp
// LeastMaxValue is a compile-time hint, not a runtime cap.
// Use a value >= your actual maximum. When in doubt:
std::counting_semaphore<> sem(16);          // default LeastMaxValue is impl-defined (>= PTRDIFF_MAX in practice), runtime count = 16
std::counting_semaphore<16> sem_16(16);     // implementation may use smaller storage; max() is still only >= 16

// try_acquire_for: CHECK THE RETURN VALUE or suffer silently
if (!sem.try_acquire_for(std::chrono::seconds(1))) {
    return handle_timeout();   // did NOT acquire; do NOT call release()
}
// here: you own a permit. release() when done.
```

> **Intuition pump (real-world):** You stood in the parking-lot queue for 5 minutes. No permit materialized, so you drove off. Driving off does NOT mean you took a permit. The `bool` return value is the attendant shaking their head at your departing bumper. `false` = no acquire. If you ignore that return value and `release()` anyway, you just handed back a permit you never had. The count inflates. The next 17 cars park free. Nobody notices until audit day when the lot shows 23 permits outstanding on a 16-space lot. Check. The. Bool.

### 8.5.1 Production Snippet — Throttled Fan-Out (max N in-flight)

Cap async concurrency at 16: acquire before launch, RAII-release in the task. Classic for connection pools, rate-limited HTTP, DB query queues.

```cpp
#include <atomic>
#include <semaphore>
#include <thread>
#include <vector>
#include <mutex>
#include <stdexcept>
#include <utility>
#include <cstddef>

template<std::ptrdiff_t MaxConcurrent>
class ThrottledExecutor {
public:
    ThrottledExecutor() : m_sem(MaxConcurrent), m_closing(false) {}

    // Safe even if the caller forgets drain(): destructor joins all in-flight tasks.
    ~ThrottledExecutor() { drain(); }

    template<typename Fn>
    void submit(Fn&& fn) {
        if (m_closing.load(std::memory_order_acquire))
            throw std::logic_error("ThrottledExecutor: submit() called after drain()");

        m_sem.acquire();  // blocks if MaxConcurrent tasks are already live

        // RAII permit guard: if we throw below or emplace_back throws (e.g. OOM),
        // the permit is returned. disarm() after emplace_back transfers ownership.
        struct PermitGuard {
            std::counting_semaphore<MaxConcurrent>& s;
            bool owns{true};
            ~PermitGuard() noexcept { if (owns) s.release(); }
            void disarm() noexcept { owns = false; }
        } permit{m_sem};

        std::lock_guard lk(m_mtx);
        // Re-check under m_mtx after unblocking: drain() may have started while we were parked.
        if (m_closing.load(std::memory_order_relaxed))
            throw std::logic_error("ThrottledExecutor: submit() raced with drain()");

        m_threads.emplace_back([this, task = std::forward<Fn>(fn)]() mutable {
            struct Guard {
                std::counting_semaphore<MaxConcurrent>& s;
                ~Guard() { s.release(); }   // fires on return AND on exception
            } g{m_sem};
            try { task(); } catch (...) {}  // prevent unhandled task exception from std::terminate-ing jthread
        });
        permit.disarm();  // emplace_back succeeded; task's Guard now owns the permit
    }

    // Block until all submitted tasks have finished.
    // Precondition: no concurrent calls to submit() while drain() is running.
    // m_closing enforces this: any submit() racing with drain() throws rather than
    // silently creating a lifetime hazard.
    void drain() {
        std::vector<std::jthread> done;
        {
            std::lock_guard lk(m_mtx);
            m_closing.store(true, std::memory_order_release);
            done = std::move(m_threads);
        }
        // jthread destructors join here — all tasks complete before drain() returns.
        // m_sem is never accessed by a task after its jthread has been joined.
    }

private:
    std::counting_semaphore<MaxConcurrent> m_sem;
    std::atomic<bool>                      m_closing;
    std::mutex                             m_mtx;
    std::vector<std::jthread>              m_threads;
};

struct WorkItem { /* ... */ };
void process_one(WorkItem const&);

void process_all(std::vector<WorkItem> const& items) {
    ThrottledExecutor<16> ex;
    for (auto const& item : items)
        ex.submit([item] { process_one(item); });  // capture by value: item is a loop variable
    ex.drain();
}
```

The `Guard` RAII wrapper is not optional. If `task()` throws and you skip `release()`, that permit is gone forever. Your application silently throttles toward zero concurrency. You will not notice until load testing.

> **Note:** `m_threads` grows by one `jthread` per `submit()` call and is only reaped in `drain()`. While the semaphore caps *active* concurrency at `MaxInFlight`, completed-but-unjoined `jthread`s in `m_threads` still retain OS thread handles (and potentially kernel/stack resources) until `drain()` joins them. For large or unbounded batches, prefer a fixed-size worker pool over this fan-out helper. This design is intentional for bounded-lifetime batches (submit a handful of tasks, drain, done); it is not a long-running worker pool.

The previous edition of this snippet used `detach()` and a semaphore-acquire-loop drain. Reviewers correctly flagged the lifetime race: `drain()` could complete and destroy `m_sem` while a detached thread was still executing inside `s.release()` — deep inside futex wake logic. Tracking `jthread`s and joining them in `drain()` closes that window. Two additional hazards close the remaining race: the post-`acquire()` re-check catches a `drain()` that completed while `submit()` was parked, and the `PermitGuard` returns the slot if `emplace_back` throws so the pool never silently loses capacity. The semaphore does the backpressure; the join does the lifetime guarantee; both checks do the correctness guarantee. All are required.

### 8.5.2 Production Snippet — One-Shot Kick (binary_semaphore)

Worker blocks until controller signals it. No mutex, no condition variable, no `bool has_work`, no spurious-wake predicate. The semaphore count *is* the signal.

> **Intuition pump (real-world):** `binary_semaphore` is the single-permit version of the parking lot — one light switch, one bulb. Either the slot is available or it isn't. Its primary job is as a "kick the worker" signal: controller calls `release()`, worker's `acquire()` unblocks, worker does the thing, worker goes back to sleep. No predicate, no spurious wakes, no mutex handshake required. It is also the dumbest correct solution to the "I need one thread to poke another thread" problem, which is why you'll reach for it constantly once you know it exists.

```cpp
#include <semaphore>
#include <atomic>
#include <thread>

class SignaledWorker {
public:
    SignaledWorker() : m_kick(0), m_stop(false), m_pending(false) {
        m_thread = std::jthread([this] { run(); });
    }

    // Destructor guarantees the thread exits cleanly even if the caller never
    // calls shutdown(). Without this, jthread's destructor calls request_stop()
    // and join() — but m_kick.acquire() is not stop-token-aware, so the thread
    // sleeps forever and join() deadlocks.
    ~SignaledWorker() { shutdown(); }

    // Guards release() with an atomic pending flag: only one kick outstanding at
    // a time. If a kick is already queued and not yet consumed, this is a no-op.
    void kick() {
        if (!m_pending.exchange(true, std::memory_order_seq_cst))
            m_kick.release();
    }

    // Coalesces with kick() through the same m_pending gate so kick() followed by
    // shutdown() before the worker wakes never calls release() twice on a
    // binary_semaphore (which would exceed max() == 1 and invoke UB).
    void shutdown() {
        m_stop.store(true, std::memory_order_seq_cst);
        if (!m_pending.exchange(true, std::memory_order_seq_cst))
            m_kick.release();
    }

private:
    void run() {
        while (true) {
            m_kick.acquire();                                   // sleeps until kicked
            m_pending.store(false, std::memory_order_seq_cst);  // slot free for next signal
            if (m_stop.load(std::memory_order_seq_cst)) break;
            do_work();
        }
    }

    void do_work();   // defined elsewhere

    std::binary_semaphore m_kick;    // counting_semaphore<1>; starts at 0
    std::atomic<bool>     m_stop;
    std::atomic<bool>     m_pending; // prevents double-release from back-to-back kick()s
    std::jthread          m_thread;
};
```

Two cautions embedded in the design above. First: both `kick()` and `shutdown()` gate
`m_kick.release()` through the same `m_pending` flag (`m_stop` is set before testing
`m_pending`). Without sharing that gate, a `kick()` followed by `shutdown()` before the
worker wakes would issue two `release()` calls on a `binary_semaphore` whose count is
already at 1, which is UB whenever `max() == 1` ([thread.sema.cnt]/8 guarantees `max() >= 1`,
and libstdc++, libc++, and MSVC STL all set `max() == 1` for `counting_semaphore<1>`). Second: the worker
clears `m_pending` (with `seq_cst`) immediately after `acquire()` and *before* checking
`m_stop` or running `do_work()`, so any `kick()` or `shutdown()` that arrives afterward
either observes `m_stop` already or posts a fresh permit. For work queues where each kick
represents a distinct countable item, use `counting_semaphore<N>` with N sized to absorb
the maximum outstanding burst.

---

## 8.6 When to Use Which — Decision Table

| Situation | Use |
|---|---|
| Wait for N things once, then move on forever | `std::latch` |
| N threads sync at every iteration / phase | `std::barrier` |
| Need to run something between phases with no concurrent threads | `std::barrier` with completion fn |
| Cap how many threads/tasks run simultaneously | `std::counting_semaphore<N>` |
| One thread signals another to wake up | `std::binary_semaphore` |
| One thread must exclusively own a resource; same thread must release | `std::mutex` (wrong chapter) |

---

## 8.7 Pitfalls

**Latch: decrement past zero.**
```cpp
std::latch l(2);
l.count_down(); l.count_down();  // counter = 0, waiters unblock
l.count_down();                  // UB. Memory corruption. Debug builds may assert; release builds lie.
```

**Barrier: throwing completion function.**
```cpp
void might_throw();   // declared elsewhere

// WRONG — the completion fn is not noexcept; C++20 requires nothrow-invocable,
//         so this may be ill-formed at compile time. If it runs and throws: std::terminate.
std::barrier b(N, []() { might_throw(); });

// CORRECT — wrap the fallible call; noexcept is not optional
std::barrier b(N, []() noexcept {
    try { might_throw(); } catch (...) { /* log and eat it; you have no choice */ }
});
```

**Barrier: arrive_and_drop then arrive_and_wait in same phase.**
```cpp
sync.arrive_and_drop();   // you just permanently left the party
sync.arrive_and_wait();   // you're arriving twice. everyone else waits forever. oops.
```

**Semaphore: ignoring try_acquire_for return value.**
```cpp
// using namespace std::chrono_literals;  -- or just use std::chrono::seconds(1) below
sem.try_acquire_for(std::chrono::seconds(1));  // forgot to check bool — proceeds as if permit acquired
use_resource();            // using resource you don't hold — count drifts permanently
sem.release();             // releasing something you never acquired — count inflates
```

**Semaphore: treating it like a mutex.**
```cpp
// Mutex: same thread locks and unlocks. Ownership.
// Semaphore: any thread can release. No ownership. No recursion. No priority inversion tracking.
// If you want "thread A holds resource, only thread A can release," use std::mutex.
```

**Semaphore: LeastMaxValue too small.**
```cpp
std::counting_semaphore<4> sem(4);
sem.release(5);  // UB if counter + update > max(); portably assume max() == LeastMaxValue,
                 // so with counter==4 this is UB on a conforming implementation
// Fix: LeastMaxValue >= your maximum runtime count
```

---

## 8.8 Performance: Not Magic, Just Correct

These primitives sit on the same OS layer as `mutex`/`cv`: futexes on Linux, `WaitOnAddress`/SRWLOCK on Windows, Mach semaphores on macOS. They are not faster. They are *correct by construction* and harder to misuse than hand-rolled equivalents.

Your hand-rolled barrier almost certainly has a generation-counter bug that only fires on 64+ cores. These don't.

Platform requirements: MSVC 19.29+ (VS 2019 16.10+), GCC 11+, Clang with libc++ 14+. If your toolchain predates these, you are not shipping C++20 and this chapter is aspirational reading.

---

## 8.9 Headers

```cpp
#include <latch>      // std::latch
#include <barrier>    // std::barrier<Fn>
#include <semaphore>  // std::counting_semaphore<N>, std::binary_semaphore
#include <thread>     // std::jthread
#include <atomic>     // std::atomic<bool> for stop flags
#include <chrono>     // durations for try_acquire_for
#include <mutex>      // std::mutex — used in ThrottledExecutor to protect the thread vector
#include <utility>    // std::forward, std::move

// Conspicuously absent from the synchronization patterns themselves:
// #include <condition_variable> — not needed for latch, barrier, or semaphore workflows
```

---

## 8.10 Closing Mantra

> Stop writing mutexes and condition variables to answer the question "are we there yet?" — the standard library wrote the answer in 2020; you were just too busy maintaining your old bugs to notice.
