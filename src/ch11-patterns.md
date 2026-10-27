# Chapter 11 — Production Patterns

Stop studying. Start building. Nine patterns, each liftable into a real codebase today.
Theory is compressed to bullets. Code dominates. Paste, adapt, ship.

---

## Portability Preamble

Every snippet needs this. Put it in a shared header and forget about it.

```cpp
#include <new>
#include <cstddef>

// Header-guard the symbol, not just the file: an inline constexpr defined twice
// in the SAME translation unit is a redefinition error. If two of these pattern
// headers land in one TU, this macro keeps exactly one definition.
#ifndef NINJA_CACHE_LINE_DEFINED
#define NINJA_CACHE_LINE_DEFINED
#ifdef __cpp_lib_hardware_interference_size
    inline constexpr std::size_t k_cache_line =
        std::hardware_destructive_interference_size;
#else
    inline constexpr std::size_t k_cache_line = 64;  // 64 on x86-64 / most ARM64 (128 on Apple Silicon)
#endif
#endif
```

---

## Pattern 1 — Lock-Free SPSC Ring Buffer

The fastest queue on the planet when you have exactly one producer thread and exactly one consumer
thread -- audio engines, packet rings, profiler buffers.

> **Intuition pump (real-world):** The bartender hands beers to the runner. Only the bartender puts drinks on the tray (producer); only the runner picks them off (consumer). Nobody else is allowed to touch the tray — not the dishwasher, not the other bartender, not you. Because only two people ever interact with it, no one needs to negotiate. They never wait on each other unless the tray is completely full or completely empty. The entire lock-free magic is just this contract: one person in, one person out. Break the contract — let two bartenders both reach for the tray at once — and you get concurrent data corruption instead of concurrent beers.

**API:** `try_push(T&&) -> bool`, `try_pop() -> optional<T>`, storage capacity must be a power of two
(usable capacity is one less; the empty slot distinguishes full from empty without extra state).

```cpp
// spsc_ring.h  ----------------------------------------------------------------
#pragma once
#include <atomic>
#include <optional>
#include <memory>
#include <cassert>
#include <new>
#include <type_traits>
#include <cstddef>
#include <utility>
#include <stdexcept>

#ifndef NINJA_CACHE_LINE_DEFINED
#define NINJA_CACHE_LINE_DEFINED
#ifdef __cpp_lib_hardware_interference_size
    inline constexpr std::size_t k_cache_line = std::hardware_destructive_interference_size;
#else
    inline constexpr std::size_t k_cache_line = 64;
#endif
#endif

// Exactly ONE thread calls try_push. Exactly ONE thread calls try_pop.
// Breaking this contract is UB. The compiler will not save you.
template <typename T>
class SpscRingBuffer {
    static_assert(std::is_move_constructible_v<T>);

    struct alignas(alignof(T)) Slot {
        std::aligned_storage_t<sizeof(T), alignof(T)> storage;
    };

public:
    explicit SpscRingBuffer(std::size_t cap)
        : cap_(cap), mask_(cap - 1), buf_(std::make_unique<Slot[]>(cap))
    {
        if (cap < 2 || (cap & (cap - 1)) != 0)
            throw std::invalid_argument("SpscRingBuffer: capacity must be power-of-two >= 2");
        assert(cap >= 2 && (cap & (cap - 1)) == 0);
    }

    ~SpscRingBuffer() {
        // No concurrent producer/consumer during destruction. If there is, the bug
        // is not in the ring; it is in the adult supervision.
        auto t = tail_.load(std::memory_order_relaxed);
        const auto h = head_.load(std::memory_order_relaxed);
        while (t != h) {
            auto* p = std::launder(reinterpret_cast<T*>(&buf_[t & mask_].storage));
            p->~T();
            ++t;
        }
    }

    SpscRingBuffer(const SpscRingBuffer&)            = delete;
    SpscRingBuffer& operator=(const SpscRingBuffer&) = delete;

    // PRODUCER THREAD ONLY.
    // val is only moved from when a slot is actually available.
    // If the ring is full (return false), val is left intact -- caller may retry.
    [[nodiscard]] bool try_push(T&& val) {
        const std::size_t h = head_.load(std::memory_order_relaxed);  // (A)
        if (h + 1 - tail_cache_ > mask_) {
            tail_cache_ = tail_.load(std::memory_order_acquire);       // (B)
            if (h + 1 - tail_cache_ > mask_) return false;             // full; val untouched
        }
        ::new (&buf_[h & mask_].storage) T(std::move(val));            // move only here
        head_.store(h + 1, std::memory_order_release);                  // (C)
        return true;
    }

    // CONSUMER THREAD ONLY.
    [[nodiscard]] std::optional<T> try_pop() {
        const std::size_t t = tail_.load(std::memory_order_relaxed);  // (D)
        if (head_cache_ == t) {
            head_cache_ = head_.load(std::memory_order_acquire);       // (E)
            if (head_cache_ == t) return std::nullopt;                 // empty
        }
        T* p = std::launder(reinterpret_cast<T*>(&buf_[t & mask_].storage));
        T   v(std::move(*p));
        p->~T();
        tail_.store(t + 1, std::memory_order_release);                 // (F)
        return v;
    }

    // Approximate. Do not use for correctness.
    // Load tail_ first so a concurrent pop cannot make t > h and underflow size_t.
    [[nodiscard]] std::size_t size_approx() const noexcept {
        const auto t = tail_.load(std::memory_order_acquire);
        const auto h = head_.load(std::memory_order_acquire);
        return h >= t ? (h - t) : 0;
    }

    [[nodiscard]] std::size_t capacity() const noexcept { return mask_; }  // usable slots

private:
    const std::size_t         cap_;
    const std::size_t         mask_;
    std::unique_ptr<Slot[]>   buf_;

    // Separate cache lines: head_ is written by producer, tail_ by consumer.
    // Same cache line = false sharing = halved throughput. Not optional.
    alignas(k_cache_line) std::atomic<std::size_t> head_{0};
    alignas(k_cache_line) std::atomic<std::size_t> tail_{0};

    // Local caches: each side avoids hammering the cross-thread atomic every op.
    // These are NOT shared -- each is touched by exactly one thread.
    alignas(k_cache_line) std::size_t tail_cache_{0};  // producer's stale view of tail
    alignas(k_cache_line) std::size_t head_cache_{0};  // consumer's stale view of head
};
```

### Usage

```cpp
#include <thread>
#include <cstdio>
#include <chrono>

int main() {
    SpscRingBuffer<int> ring(1024);   // power-of-two storage; 1023 usable slots

    std::jthread prod([&](std::stop_token st) {
        for (int i = 0; !st.stop_requested(); ) {
            // try_push takes T&&: val is only moved from when a slot is free.
            // On failure the int is untouched; spin and retry the same value.
            if (ring.try_push(std::move(i)))
                ++i;
        }
    });

    std::jthread cons([&](std::stop_token st) {
        long long sum = 0;
        while (!st.stop_requested()) {
            if (auto v = ring.try_pop()) sum += *v;
        }
        std::printf("sum = %lld\n", sum);
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    // jthreads request_stop + join on scope exit
}
```

### Performance Notes

- Same-core (shared L1): 300-600 M items/s for `int`.
- Cross-core, NUMA-local: 100-200 M items/s; limited by cache-coherency traffic.
- Without `alignas(k_cache_line)` on `head_`/`tail_`: false sharing halves throughput. The padding
  is load-bearing, not decorative.
- `size_approx()` is approximate and stale. Never gate correctness on it; only use for metrics.

### Memory Order Rationale

- **(A)** `relaxed` -- `head_` is owned by the producer; this load is private to the producer.
- **(B)** `acquire` -- synchronises with consumer's release at **(F)**; ensures the producer sees
  the consumer's completed reads of the slots it freed before advancing `tail_`.
- **(C)** `release` -- all slot writes above must be visible before `head_` advances; consumer's
  acquire at **(E)** pairs with this to see the completed slot.
- **(D)** `relaxed` -- `tail_` is owned by the consumer; this load is private to the consumer.
- **(E)** `acquire` -- pairs with producer's release at **(C)**; consumer sees slot fully written.
- **(F)** `release` -- slot destruction above must complete before `tail_` advances; producer's
  acquire at **(B)** pairs with this to see the slot is free.

### When NOT to Use

- Multiple concurrent producers **or** multiple concurrent consumers -- two threads calling
  `try_push` simultaneously race on `head_` with no mutual exclusion. Use a mutex-based MPMC
  queue or Vyukov's bounded MPMC.
- Producers/consumers hopping across threads without synchronization -- "single producer/consumer"
  requires either a dedicated thread or an explicit happens-before handoff (such as a mutex
  serializing callers, as shown in Chapter 12).
- Blocking push semantics needed -- SPSC is non-blocking by design. Add semaphores or switch to
  the bounded queue in Pattern 4.

---

## Pattern 2 — Thread Pool

You need to dispatch callable work units and get results back without managing threads manually.

> **Intuition pump (real-world):** The thread pool is the temp agency. You don't hire and fire a new contractor for every task — the spin-up cost for an OS thread is real, the stack allocation is real, and the scheduler doesn't care about your feelings. Instead, you have a pool of already-paid workers who sit in a lounge reading their phones, waiting for the queue to ring. New task arrives: the first available worker picks it up. Task done: worker returns to the lounge. No spin-up cost per task. No "each request spawns a thread" collapse at 10,000 concurrent users. The agency has a fixed headcount; you control how many temps you pay for.

**API:** `submit(F, Args...) -> future<R>`, `pending_count() -> size_t`, RAII shutdown in destructor.

```cpp
// thread_pool.h  -------------------------------------------------------------
#pragma once
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <queue>
#include <vector>
#include <memory>
#include <utility>
#include <stdexcept>
#include <type_traits>
#include <cstddef>

class ThreadPool {
public:
    explicit ThreadPool(std::size_t n) {
        if (n == 0) throw std::invalid_argument("ThreadPool: n must be > 0");
        workers_.reserve(n);
        for (std::size_t i = 0; i < n; ++i)
            workers_.emplace_back([this](std::stop_token st){ run(st); });
    }

    // Explicit destructor: signal stop_, wake all, then clear workers_.
    // workers_ is the last member so its destructor (jthread join) runs first,
    // before mu_, cv_, q_, stop_ are destroyed. All members are valid during join.
    ~ThreadPool() {
        { std::unique_lock lk(mu_); stop_ = true; }
        cv_.notify_all();
        workers_.clear();  // jthread dtor: request_stop() + join()
    }

    ThreadPool(const ThreadPool&)            = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    // Returns std::future<R>. Throws std::runtime_error if pool is stopping.
    //
    // DEADLOCK: calling future.get() from inside a submitted task while the
    // pool is saturated will deadlock. The waiting worker holds its slot;
    // the dependency can never run. Structure work as fire-and-collect,
    // not fire-and-wait-inline.
    template <typename F, typename... Args>
    [[nodiscard]] auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<std::decay_t<F>, std::decay_t<Args>...>>
    {
        using R = std::invoke_result_t<std::decay_t<F>, std::decay_t<Args>...>;

        // Bind args now (before the lock) so the lambda is self-contained.
        auto bound = [fn  = std::forward<F>(f),
                      ...a = std::forward<Args>(args)]() mutable -> R {
            return std::invoke(std::move(fn), std::move(a)...);
        };

        // packaged_task<R()> is move-only; std::function requires copyable.
        // Wrap in shared_ptr: copying the function copies the ptr, not the task.
        auto task = std::make_shared<std::packaged_task<R()>>(std::move(bound));
        auto fut  = task->get_future();

        {
            std::unique_lock lk(mu_);
            if (stop_) throw std::runtime_error("ThreadPool: shutting down");
            q_.push([task]{ (*task)(); });
        }
        cv_.notify_one();
        return fut;
    }

    [[nodiscard]] std::size_t pending_count() const {
        std::unique_lock lk(mu_);
        return q_.size();
    }

private:
    void run(std::stop_token st) {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock lk(mu_);
                // condition_variable_any::wait overload accepts a stop_token.
                // Returns false if the wait was interrupted by stop_requested()
                // (e.g. during constructor-throw unwind) rather than by the
                // predicate becoming true. In that case, exit cleanly.
                //
                // DRAIN VARIANT: replace `if (stop_) return;` below with
                // `if (stop_ && q_.empty()) return;` to finish queued tasks
                // before exiting. Documented policy: default = discard queued,
                // complete in-flight only.
                if (!cv_.wait(lk, st, [&]{ return stop_ || !q_.empty(); }))
                    return;  // stop token fired before predicate satisfied
                if (stop_) return;
                task = std::move(q_.front());
                q_.pop();
            }
            task();  // called OUTSIDE the lock -- tasks may call submit()
        }
    }

    // Member declaration order matters: workers_ must be last.
    // C++ destroys members in reverse declaration order.
    // workers_ destroyed first → jthreads join → all other members still valid.
    // If workers_ were declared before stop_, a constructor throw would destroy
    // stop_ first, leaving running threads reading a destroyed bool (UAF).
    mutable std::mutex                mu_;
    std::condition_variable_any       cv_;   // _any: supports stop_token wait overload
    std::queue<std::function<void()>> q_;
    bool                              stop_ = false;
    std::vector<std::jthread>         workers_;  // LAST: destroyed (joined) first
};
```

### Usage

```cpp
ThreadPool pool(std::max(1u, std::thread::hardware_concurrency()));

std::vector<std::future<int>> futs;
for (int i = 0; i < 64; ++i)
    futs.push_back(pool.submit([i]{ return i * i; }));

int total = 0;
for (auto& f : futs) total += f.get();
// pool destructs here -- workers join cleanly
```

### Exception Propagation Demo

```cpp
ThreadPool pool(2);

auto fut = pool.submit([]() -> int {
    throw std::runtime_error("something exploded");
    return 42;
});

try {
    int result = fut.get();   // rethrows here
} catch (const std::runtime_error& e) {
    // worker did not crash; exception was captured in packaged_task
    std::printf("caught: %s\n", e.what());
}
```

### Nested-Submit Deadlock — Minimal Reproducer

```cpp
ThreadPool pool(1);  // only one worker

auto outer = pool.submit([&pool] {
    // This submits a new task to the pool while the only worker is
    // busy running THIS lambda. inner.get() will never return.
    auto inner = pool.submit([]{ return 99; });
    return inner.get();  // DEADLOCK: inner can never run
});

outer.get();  // hangs forever
```

### Properties

- **FIFO dequeue.** Tasks are pulled from the queue in submission order (actual completion order
  across multiple workers still depends on OS scheduling). One mutex serialises all `submit()`
  calls; this is correct and fast enough for task rates below ~1 M/s. Above that, replace with a
  lock-free MPMC queue and a semaphore for sleeping workers.
- **Exception propagation.** If a task throws, the exception is captured in the `packaged_task`
  and rethrown on `future.get()`. Workers do not crash.
- **Shutdown policy.** Destructor signals stop; workers exit after their current task. Queued tasks
  become broken promises -- `future.get()` throws `std::future_error`. Change the `cv_.wait`
  predicate (see DRAIN VARIANT comment) to drain the queue before exiting instead.
- **Nested-submit deadlock.** A task that submits a subtask and calls `future.get()` on it will
  deadlock if the pool has no free workers. This is a fundamental property of fixed-size blocking
  pools, not a bug. Either size the pool to have slack, or restructure with continuations.

---

## Pattern 3 — Atomic Shared-Ptr Publish (Poor Man's RCU)

Routing tables, config snapshots, ML weights: change rarely, read on every request -- readers must
never block writers.

> **Intuition pump (real-world):** The museum's display case. The curators rarely swap out the artifact — maybe once a day, maybe once a week (that's your `publish()`). Visitors file past it constantly, all day, dozens at a time (that's your `snapshot()` readers). No visitor blocks another visitor. No visitor asks the curator's permission to look. When the curator does swap the artifact — at 2am, with gloves, while you're asleep — every visitor who grabbed a reference to the old artifact before the swap continues looking at it unmolested. The old artifact's ref count keeps it alive until the last visitor walks away. The new artifact becomes visible to anyone who arrives after the swap. Zero application-level reader locks. The `shared_ptr` ref counts are the velvet rope: they keep both artifacts alive for exactly as long as someone is looking.

**API:** `snapshot() -> shared_ptr<const T>` (acquire-load, lock-free*), `publish(shared_ptr<T>)` (release-store).

```cpp
// atomic_publish.h  ----------------------------------------------------------
#pragma once
#include <atomic>
#include <memory>
#include <unordered_map>
#include <string>
#include <cstdint>

// The data you publish. Must be immutable after construction --
// no mutex needed because nothing mutates it after publish().
struct RouteTable {
    std::unordered_map<std::string, std::string> routes;
    std::uint64_t version = 0;

    [[nodiscard]] std::string lookup(const std::string& dest) const {
        auto it = routes.find(dest);
        return it != routes.end() ? it->second : std::string{};
    }
};

class AtomicRouteStore {
public:
    AtomicRouteStore()
        : ptr_(std::make_shared<const RouteTable>()) {}

    // READERS: call this, use the snapshot, drop it.
    // Old snapshots stay alive until the last shared_ptr to them drops.
    // Do NOT cache snapshots across requests -- you pin old versions.
    [[nodiscard]] std::shared_ptr<const RouteTable> snapshot() const noexcept {
        return ptr_.load(std::memory_order_acquire);
    }

    // WRITER: build the new RouteTable first, then call this once.
    // Old readers continue to see the old version. New readers see the new one.
    void publish(std::shared_ptr<const RouteTable> next) {
        ptr_.store(std::move(next), std::memory_order_release);
    }

    void publish(std::unordered_map<std::string, std::string> routes,
                 std::uint64_t version)
    {
        auto t      = std::make_shared<RouteTable>();
        t->routes   = std::move(routes);
        t->version  = version;
        publish(std::move(t));
    }

private:
    // std::atomic<shared_ptr<T>> is C++20.
    // Call is_lock_free() at startup to verify hardware support.
    // On MSVC this uses a striped spinlock (fast, but not academically lock-free).
    std::atomic<std::shared_ptr<const RouteTable>> ptr_;
};
```

### Usage

```cpp
AtomicRouteStore store;

// Reader (any number of concurrent threads):
auto snap = store.snapshot();   // acquire-load, ~1 atomic op
auto hop  = snap->lookup("10.0.0.1");
// snap drops here; old RouteTable freed if no other readers hold it

// Writer (one at a time; build new version off the hot path):
store.publish({{"10.0.0.0/8","192.168.1.1"},{"0.0.0.0/0","192.168.1.254"}}, 42);
```

### Caveats

- **`is_lock_free()` may return false.** MSVC uses a hidden spinlock. Still faster than
  `std::mutex` for read-heavy workloads; just technically not lock-free. Check at startup with
  `ptr_.is_lock_free()` if your latency SLA depends on it.
- **Old versions pile up under long-lived snapshots.** A reader that holds `snap` for minutes
  keeps the old `RouteTable` alive for minutes. Document that snapshots are request-scoped,
  not session-scoped.
- **High write rate = bottleneck.** Every `publish()` is a heap allocation for a full new object.
  At thousands of writes per second this is measurable. At one write per second (the typical config
  refresh case) it is invisible.

---

## Pattern 4 — Producer/Consumer Pipeline with Back-Pressure

Multiple processing stages with different throughputs; slow stages must throttle fast ones, not
silently drop data or grow unbounded queues.

**Core primitive:** a blocking bounded queue -- `push()` blocks when full (back-pressure),
`pop()` blocks when empty, `close()` signals end-of-stream.

```cpp
// bounded_queue.h  -----------------------------------------------------------
#pragma once
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>
#include <stdexcept>
#include <type_traits>
#include <cstddef>

template <typename T>
class BoundedQueue {
    static_assert(std::is_nothrow_move_constructible_v<T>,
                  "T must be nothrow-movable so pop() cannot lose items or strand waiters");
public:
    explicit BoundedQueue(std::size_t cap) : cap_(cap) {
        if (cap == 0) throw std::invalid_argument("BoundedQueue: cap must be > 0");
    }

    // Blocks until space or closed. Returns false if closed before pushed.
    [[nodiscard]] bool push(T val) {
        std::unique_lock lk(mu_);
        not_full_.wait(lk, [&]{ return closed_ || q_.size() < cap_; });
        if (closed_) return false;
        q_.push(std::move(val));
        not_empty_.notify_one();
        return true;
    }

    // Blocks until item or queue closed+empty. Returns nullopt on closed+empty.
    [[nodiscard]] std::optional<T> pop() {
        std::unique_lock lk(mu_);
        not_empty_.wait(lk, [&]{ return closed_ || !q_.empty(); });
        if (q_.empty()) return std::nullopt;
        T v = std::move(q_.front()); q_.pop();
        not_full_.notify_one();
        return v;
    }

    // Signal end-of-stream. Unblocks all waiters.
    // Items already queued remain drainable until empty.
    void close() {
        std::unique_lock lk(mu_);
        closed_ = true;
        not_full_.notify_all();
        not_empty_.notify_all();
    }

    [[nodiscard]] std::size_t size() const {
        std::unique_lock lk(mu_); return q_.size();
    }

private:
    const std::size_t       cap_;
    mutable std::mutex      mu_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
    std::queue<T>           q_;
    bool                    closed_ = false;
};
```

### Pipeline Wiring

```cpp
// pipeline.cpp  --------------------------------------------------------------
#include <thread>
#include <vector>
#include <iostream>
#include <string>
#include <cstdint>   // std::uint64_t
#include <cstdio>    // std::printf (used in monitor below)
#include <chrono>    // std::chrono::seconds (used in monitor below)

struct Raw       { int id; std::string data; };
struct Processed { int id; std::size_t len; std::string result; };

void run_pipeline(std::size_t n_workers = 4) {
    if (n_workers == 0) n_workers = 1;
    BoundedQueue<Raw>       qa(256);   // producer  -> workers
    BoundedQueue<Processed> qb(128);   // workers   -> aggregator

    // Declare jthreads in REVERSE shutdown order.
    // C++ destroys locals in reverse declaration order.
    // We want: producer destroyed first (gets stop, closes qa),
    //   then qa_guard (closes qa if producer construction threw before closer joins workers),
    //   then closer (joins workers, closes qb),
    //   then workers (already joined by closer -- fast no-op),
    //   then aggregator (qb now closed, drains and exits).

    // Stage 3 — Aggregator (declared first = destroyed last)
    std::jthread aggregator([&] {
        std::uint64_t count = 0, bytes = 0;
        while (auto p = qb.pop()) { ++count; bytes += p->len; }
        std::cout << count << " items, " << bytes << " bytes\n";
    });

    // Guard Stage 2 + closer creation: if any worker or closer jthread constructor
    // throws std::system_error, close qa and qb so already-spawned threads and
    // aggregator unblock and join cleanly during stack unwinding.
    std::vector<std::jthread> workers;
    std::jthread closer;
    try {
        workers.reserve(n_workers);
        for (std::size_t w = 0; w < n_workers; ++w) {
            workers.emplace_back([&] {
                while (auto r = qa.pop()) {
                    Processed p;
                    p.id     = r->id;
                    p.len    = r->data.size();
                    p.result = std::string(r->data.rbegin(), r->data.rend());
                    if (!qb.push(std::move(p))) break;  // qb closed early
                }
            });
        }

        // Closer: waits for all workers to drain qa, then closes qb.
        closer = std::jthread([&] {
            for (auto& t : workers) if (t.joinable()) t.join();
            qb.close();  // signals aggregator: no more transformed items
        });
    } catch (...) {
        qa.close();
        qb.close();
        throw;
    }

    // Exception safety: declared AFTER closer (so ~QaGuard runs BEFORE closer's
    // destructor during stack unwinding!). If the producer jthread constructor
    // throws below, ~QaGuard closes qa first so workers exit and closer can join.
    struct QaGuard {
        BoundedQueue<Raw>& q;
        ~QaGuard() { q.close(); }  // idempotent; safe if producer already closed it
    } qa_guard{qa};

    // Stage 1 — Producer (declared last = destroyed first)
    std::jthread producer([&](std::stop_token st) {
        for (int i = 0; i < 10'000 && !st.stop_requested(); ++i)
            if (!qa.push({i, "payload_" + std::to_string(i)})) break;
        qa.close();  // always called; signals workers: no more input
    });

    // Explicitly wait for the finite producer to finish pushing all 10 000 items.
    // Without this, scope exit fires the producer's jthread destructor immediately,
    // which calls request_stop() before the loop completes — processing only a small
    // prefix of the intended input. For an infinite or externally-stopped producer,
    // remove this join and rely on request_stop() from the destructor instead.
    producer.join();

    // Destruction order on scope exit (reverse of declaration):
    //   producer  → already joined above; dtor is a no-op
    //   qa_guard  → qa.close() (idempotent; crucial if producer ctor threw!)
    //   closer    → joins workers (they exit when qa is closed+empty), closes qb
    //   workers   → already joined by closer; jthread dtors are no-ops
    //   aggregator→ qb.pop() returns nullopt; thread exits and joins
    //   qb, qa    → destroyed after all threads have exited
}
```

### Graceful Shutdown Rules

- **Declare in reverse shutdown order.** jthreads destroy in reverse declaration order. Declare
  aggregator first (destroyed last), producer last (destroyed first). Producer gets stop signal,
  closes `qa`; closer joins workers and closes `qb`; aggregator drains and exits.
- **For a finite producer, call `producer.join()` explicitly.** The jthread destructor requests
  stop before joining. For a loop bounded by a count (`i < 10'000`), that means scope exit
  cancels the loop after whatever few items raced through. `producer.join()` lets all items
  through, then hands off cleanly to the rest of the pipeline.
- **Guard source queues with RAII.** If the producer jthread constructor throws, no lambda ever
  runs, so `qa.close()` is never called. Workers block forever in `qa.pop()`. The `QaGuard`
  struct above closes `qa` unconditionally on scope exit; `close()` is idempotent so the
  normal path (producer closing it) and the exception path both work correctly.
- **Close source first, sink last.** Closing `qb` before workers finish draining `qa` causes
  workers' `qb.push()` calls to return `false` immediately, silently dropping every remaining
  in-flight item from `qa` on the floor.
- **The closing chain: producer closes qa → workers drain qa and exit → closer joins workers
  and closes qb → aggregator drains qb and exits.** The declaration order enforces this.
- **Queue depth = saturation signal.** `qa.size()` near capacity means workers are the bottleneck.
  `qb.size()` near capacity means the aggregator is the bottleneck. Emit both as metrics every
  second; alert on sustained near-capacity depth.
- **Back-pressure propagates automatically.** A slow aggregator fills `qb`; workers block on
  `qb.push()`; `qa` fills up right behind them; the producer blocks on `qa.push()`. No explicit
  throttle logic needed. This is why bounded beats unbounded.

### Queue-Depth Monitor

```cpp
// Run alongside the pipeline. Emit depth metrics every second.
// Wire 'depth_a' and 'depth_b' into your telemetry system; alert on
// sustained values near capacity -- that is your bottleneck signal.
std::jthread monitor([&](std::stop_token st) {
    while (!st.stop_requested()) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        const auto depth_a = qa.size();
        const auto depth_b = qb.size();
        std::printf("qa=%zu/%d  qb=%zu/%d\n",
                    depth_a, 256, depth_b, 128);
        // depth_a -> 256: workers are the bottleneck (add workers or optimise transform)
        // depth_b -> 128: aggregator is the bottleneck (parallelise aggregation)
        // both near 0:    pipeline not the bottleneck; look elsewhere
    }
});
```

---

## Pattern 5 — Read-Mostly Cache with TTL

You need a concurrent cache with high read throughput. LRU is wrong here: every LRU cache hit
mutates the recency list, which requires exclusive access, which means every "read" is secretly a
write and your shared_mutex is useless.

TTL-only eviction has zero per-read mutation. Multiple readers run concurrently under shared_lock.

**API:** `get(key) -> optional<V>`, `put(key, val)`, `get_or_load(key, factory) -> V`,
`evict(key)`, `cleanup()` (also called by a background jthread).

```cpp
// ttl_cache.h  ---------------------------------------------------------------
#pragma once
#include <mutex>
#include <shared_mutex>
#include <unordered_map>
#include <optional>
#include <functional>
#include <chrono>
#include <thread>
#include <stdexcept>
#include <utility>
#include <cstddef>

template <typename Key, typename Value>
class TtlCache {
    using Clock = std::chrono::steady_clock;
    using TP    = Clock::time_point;
    using Dur   = Clock::duration;

    struct Entry { Value val; TP expiry; };

public:
    explicit TtlCache(Dur ttl, Dur sweep = std::chrono::seconds(30))
        : ttl_(ttl)
    {
        sweeper_ = std::jthread([this, sweep](std::stop_token st) {
            sweep_loop(st, sweep);
        });
    }

    ~TtlCache() = default;  // jthread dtor: request_stop() + join()

    TtlCache(const TtlCache&)            = delete;
    TtlCache& operator=(const TtlCache&) = delete;

    // O(1) lookup. Multiple threads concurrent. No mutations on hit.
    [[nodiscard]] std::optional<Value> get(const Key& k) {
        {
            std::shared_lock lk(mu_);
            auto it = map_.find(k);
            if (it == map_.end())         return std::nullopt;  // miss
            if (it->second.expiry > Clock::now()) return it->second.val;  // hit (copy)
            // found but expired -- fall through to evict
        }
        // Upgrade to unique_lock to evict. Re-check: another thread may have
        // beaten us here and already evicted or refreshed the entry.
        {
            std::unique_lock lk(mu_);
            auto it = map_.find(k);
            if (it != map_.end() && it->second.expiry <= Clock::now())
                map_.erase(it);
        }
        return std::nullopt;
    }

    // Exclusive write. Resets TTL unconditionally.
    void put(const Key& k, Value v) {
        std::unique_lock lk(mu_);
        map_.insert_or_assign(k, Entry{std::move(v), Clock::now() + ttl_});
    }

    // Cache-aside: check -> miss -> factory (OUTSIDE lock) -> store -> return.
    // Factory is called without any lock held, so it may block freely.
    // Race: two threads may both miss and both call factory. Second write wins.
    // Safe only if factory is idempotent for a given key.
    template <typename Factory>
    [[nodiscard]] Value get_or_load(const Key& k, Factory&& f) {
        if (auto v = get(k)) return std::move(*v);
        Value fresh = std::invoke(std::forward<Factory>(f), k);
        put(k, fresh);       // store a copy
        return fresh;        // return the original
    }

    void evict(const Key& k) {
        std::unique_lock lk(mu_);
        map_.erase(k);
    }

    // Sweep all expired entries. Called by background thread and also
    // available for manual invocation (e.g. on memory pressure signal).
    void cleanup() {
        const auto now = Clock::now();
        std::unique_lock lk(mu_);
        for (auto it = map_.begin(); it != map_.end(); ) {
            if (it->second.expiry <= now) it = map_.erase(it);
            else ++it;
        }
    }

    [[nodiscard]] std::size_t size() const {
        std::shared_lock lk(mu_); return map_.size();
    }

private:
    void sweep_loop(std::stop_token st, Dur interval) {
        // Sleep in short increments so stop_requested() is noticed promptly.
        while (!st.stop_requested()) {
            const auto wake = Clock::now() + interval;
            while (!st.stop_requested() && Clock::now() < wake)
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            if (!st.stop_requested()) cleanup();
        }
    }

    mutable std::shared_mutex          mu_;
    std::unordered_map<Key, Entry>     map_;
    const Dur                          ttl_;
    std::jthread                       sweeper_;
};
```

### Usage

```cpp
using namespace std::chrono_literals;

TtlCache<std::string, std::string> cache(5min, 60s);

cache.put("user:42", "Alice");

// Concurrent reads: all acquire shared_lock simultaneously
auto name = cache.get("user:42");  // returns optional<string>("Alice")

// Cache-aside: factory runs outside the lock
auto val = cache.get_or_load("user:99", [](const std::string&) {
    return query_database("user:99");  // slow call, no lock held
});
```

### The `get_or_load` Race -- When It Matters

```cpp
// SAFE: factory is idempotent (same key -> same value, no side effects)
TtlCache<int, std::string> safe_cache(60s);
auto v = safe_cache.get_or_load(42, [](int id) {
    return fetch_username_from_db(id);  // two threads may call this for id=42
    // second write overwrites first -- both return correct value
});

// UNSAFE: factory has side effects (charges card, sends email, inserts row)
TtlCache<int, std::string> unsafe_cache(60s);
auto v2 = unsafe_cache.get_or_load(42, [](int id) {
    charge_credit_card(id);   // called TWICE on a concurrent miss -- BAD
    return "charged";
});
// Fix: use a per-key loading sentinel (a shared_future stored in the map)
// or ensure the factory is idempotent at the external system level.
```

### LRU vs. TTL -- Concrete Throughput Impact

```
Scenario: 8 threads, all readers, 1 writer thread, hot working set fits in cache.

LRU cache (exclusive lock on every read):
  Throughput ~= single_reader_throughput  (all 8 readers serialised)

TtlCache (shared_lock on reads):
  Throughput ~= 8 * single_reader_throughput  (8 readers concurrent)

The shared_mutex gains you reader parallelism only because get() has zero
writes on a cache hit. The moment you add an LRU list touch, the gain
evaporates and you are back to a regular mutex.
```

- **TTL on read = zero write operations = truly shared access.** `get()` under `shared_lock`
  touches no mutable state on a non-expired hit. N threads read simultaneously.
- **LRU on read = always writes the recency list = can never use shared_lock.** Every LRU hit
  requires moving a node in the eviction list. That write must be serialised. Your concurrent
  "reader" is actually an exclusive writer. Shared_mutex gives you nothing.
- **Lazy eviction (expired entry found during get) correctly upgrades to unique_lock with a
  re-check.** There is a gap between dropping shared_lock and acquiring unique_lock; another
  thread may have already cleaned up. The re-check prevents a double-erase.

---

## Advanced patterns

The first five patterns keep the lights on. These four are for when traffic arrives with a crowbar and a legal department.

---

## Pattern 6 — Lock-Based MPMC Bounded Queue

Many producers, many consumers, bounded memory, graceful shutdown; boring enough to be correct, fast enough to avoid summoning lock-free archaeology.

> **Intuition pump (real-world):** The airport TSA line. Many travelers (producers) approach from all directions and drop their bins on the belt. Many agents (consumers) stand at the end pulling bins off to inspect them. There is one shared conveyor belt, and it is protected by the queue of humanity itself — everyone takes turns at the belt; no one jumps the machinery. The belt is bounded: when it fills up, the travelers back up into a line and wait. That's your back-pressure. When it empties, the agents stand around looking bored and checking their phones. That's your consumers blocking on `pop()`. It is not glamorous. It is not lock-free. It handles every airline's combined passenger volume because it is simple and it does not fall over.

```cpp
// mpmc_bounded_queue.h  ------------------------------------------------------
#pragma once
#include <condition_variable>
#include <chrono>
#include <mutex>
#include <optional>
#include <stdexcept>
#include <type_traits>
#include <utility>
#include <vector>
#include <cstddef>

// Multi-producer, multi-consumer, bounded, move-only friendly.
// close() means: no new pushes; consumers drain queued items; then pop() returns nullopt.
template <typename T>
class MpmcBoundedQueue {
    static_assert(std::is_nothrow_move_constructible_v<T>,
                  "T must be nothrow-movable so pop_locked() cannot lose items or skip notify");
public:
    explicit MpmcBoundedQueue(std::size_t capacity)
        : ring_(capacity), capacity_(capacity)
    {
        if (capacity == 0) throw std::invalid_argument("MpmcBoundedQueue: capacity must be > 0");
    }

    MpmcBoundedQueue(const MpmcBoundedQueue&)            = delete;
    MpmcBoundedQueue& operator=(const MpmcBoundedQueue&) = delete;

    template <typename U>
    [[nodiscard]] bool try_push(U&& value) {
        std::unique_lock lk(mu_);
        if (closed_ || size_ == capacity_) return false;
        push_locked(std::forward<U>(value));
        lk.unlock();
        not_empty_.notify_one();
        return true;
    }

    // Blocks until space exists or the queue is closed.
    // Returns false if close() wins the race. Sorry, your task was not special.
    template <typename U>
    [[nodiscard]] bool push(U&& value) {
        std::unique_lock lk(mu_);
        not_full_.wait(lk, [&] { return closed_ || size_ < capacity_; });
        if (closed_) return false;
        push_locked(std::forward<U>(value));
        lk.unlock();
        not_empty_.notify_one();
        return true;
    }

    template <typename U, typename Rep, typename Period>
    [[nodiscard]] bool push_for(U&& value, const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock lk(mu_);
        if (!not_full_.wait_for(lk, timeout, [&] { return closed_ || size_ < capacity_; }))
            return false;
        if (closed_) return false;
        push_locked(std::forward<U>(value));
        lk.unlock();
        not_empty_.notify_one();
        return true;
    }

    [[nodiscard]] std::optional<T> try_pop() {
        std::unique_lock lk(mu_);
        if (size_ == 0) return std::nullopt;
        auto value = pop_locked();
        lk.unlock();
        not_full_.notify_one();
        return value;
    }

    // Blocks until an item exists, or close()+drained.
    [[nodiscard]] std::optional<T> pop() {
        std::unique_lock lk(mu_);
        not_empty_.wait(lk, [&] { return closed_ || size_ > 0; });
        if (size_ == 0) return std::nullopt;
        auto value = pop_locked();
        lk.unlock();
        not_full_.notify_one();
        return value;
    }

    template <typename Rep, typename Period>
    [[nodiscard]] std::optional<T> pop_for(const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock lk(mu_);
        if (!not_empty_.wait_for(lk, timeout, [&] { return closed_ || size_ > 0; }))
            return std::nullopt;
        if (size_ == 0) return std::nullopt;
        auto value = pop_locked();
        lk.unlock();
        not_full_.notify_one();
        return value;
    }

    void close() {
        std::unique_lock lk(mu_);
        if (closed_) return;
        closed_ = true;
        lk.unlock();
        not_empty_.notify_all();
        not_full_.notify_all();
    }

    [[nodiscard]] bool closed() const {
        std::unique_lock lk(mu_);
        return closed_;
    }

    [[nodiscard]] std::size_t size() const {
        std::unique_lock lk(mu_);
        return size_;
    }

private:
    template <typename U>
    void push_locked(U&& value) {
        ring_[tail_].emplace(std::forward<U>(value));
        tail_ = (tail_ + 1) % capacity_;
        ++size_;
    }

    [[nodiscard]] T pop_locked() {
        T value = std::move(*ring_[head_]);
        ring_[head_].reset();
        head_ = (head_ + 1) % capacity_;
        --size_;
        return value;
    }

    mutable std::mutex              mu_;
    std::condition_variable         not_empty_;
    std::condition_variable         not_full_;
    std::vector<std::optional<T>>   ring_;
    const std::size_t               capacity_;
    std::size_t                     head_ = 0;
    std::size_t                     tail_ = 0;
    std::size_t                     size_ = 0;
    bool                            closed_ = false;
};
```

- **Correctness beats cosplay.** One mutex protects `head_`, `tail_`, `size_`, and `closed_`; two condition variables prevent the classic "everyone woke up, nobody can move" circus.
- **Move-only types work.** `std::optional<T>` owns slots directly; failed `try_push()` does not move from your object. Tiny mercy, still invoiceable.
- **Shutdown drains.** `close()` rejects future pushes, wakes everyone, and lets consumers empty the queue before `nullopt` ends the loop.
- **Use this for thread-pool task queues.** If profiling says the mutex is the bottleneck, then read Dmitry Vyukov's bounded MPMC queue. Do not improvise one at 2 a.m. with vibes.

---

## Pattern 7 — Atomic Shared-Ptr Observable History

You need a lock-free-ish append-only history where readers can walk a stable snapshot while the writer keeps publishing new facts into the void.

```cpp
// observable_history.h  ------------------------------------------------------
#pragma once
#include <atomic>
#include <functional>
#include <memory>
#include <optional>
#include <utility>
#include <vector>
#include <cstdint>
#include <string>

// Single writer, many readers. Nodes are immutable after construction.
template <typename T>
class ObservableHistory {
public:
    struct Node {
        T value;
        std::shared_ptr<const Node> next;

        Node(T v, std::shared_ptr<const Node> n)
            : value(std::move(v)), next(std::move(n)) {}

        // Iterative destructor: the default shared_ptr chain is recursive — one stack frame per
        // node. A history grown to ~50k–100k entries blows the default 1 MiB Windows stack on
        // whichever poor thread drops the last reference (often a reader Snapshot, not the writer).
        // Why not check use_count() == 1? Because [util.smartptr.shared.obs]/16 warns use_count()
        // is a relaxed hint that does not synchronize with prior readers finishing traversal.
        // Instead, when *this* node's refcount genuinely hits zero (with full acq_rel sync),
        // we park 'next' in a thread-local drain queue so nested ~Node() calls flatten to O(1) stack.
        ~Node() {
            if (!next) return;
            thread_local bool draining = false;
            thread_local std::vector<std::shared_ptr<const Node>> garbage;
            garbage.push_back(std::move(next));
            if (draining) return;
            draining = true;
            while (!garbage.empty()) {
                auto cur = std::move(garbage.back());
                garbage.pop_back();
                cur.reset();  // if last owner, ~Node() just pushes cur->next onto 'garbage'
            }
            draining = false;
        }
    };

    using Snapshot = std::shared_ptr<const Node>;

    ObservableHistory() = default;
    ObservableHistory(const ObservableHistory&)            = delete;
    ObservableHistory& operator=(const ObservableHistory&) = delete;
    ~ObservableHistory() = default;

    // WRITER THREAD ONLY. Multiple writers need a CAS loop or a mutex.
    void prepend(T value) {
        auto old_head = head_.load(std::memory_order_acquire);
        auto node = std::make_shared<Node>(std::move(value), std::move(old_head));
        head_.store(std::move(node), std::memory_order_release);
    }

    // Readers take one atomic snapshot and then walk ordinary shared_ptr links.
    [[nodiscard]] Snapshot snapshot() const noexcept {
        return head_.load(std::memory_order_acquire);
    }

    template <typename F>
    void for_each(F&& f) const {
        for (auto p = snapshot(); p; p = p->next) {
            std::invoke(f, p->value);
        }
    }

    [[nodiscard]] std::vector<T> to_vector() const {
        std::vector<T> out;
        for_each([&](const T& v) { out.push_back(v); });
        return out;
    }

    [[nodiscard]] std::optional<T> latest() const {
        auto p = snapshot();
        if (!p) return std::nullopt;
        return p->value;
    }

private:
    std::atomic<std::shared_ptr<const Node>> head_{std::shared_ptr<const Node>{}};
};

struct Event {
    std::uint64_t sequence;
    std::string message;
};

void example() {
    ObservableHistory<Event> log;
    log.prepend({1, "started"});
    log.prepend({2, "still alive, suspicious"});

    log.for_each([](const Event& e) {
        // Newest first. If you need oldest first, reverse the vector.
        [[maybe_unused]] auto sequence = e.sequence;
        [[maybe_unused]] auto message  = e.message;
    });
}
```

- **No hazard pointers. No epochs. No ritual sacrifice.** `shared_ptr` reference counts keep nodes alive while a reader holds any link in the chain.
- **Publication is acquire/release.** Writer constructs the whole immutable node, then release-stores `head_`; readers acquire-load and see a fully built list.
- **Single writer is load-bearing.** Two writers using `store()` can lose one prepend. Use a CAS loop or a mutex if writers multiply like services after a reorg.
- **Great for observable history.** Event logs, undo stacks, audit trails, append-only time-series breadcrumbs. Terrible for high-rate telemetry where ref counts become the tax collector.
- **Long chains must destruct iteratively.** The default `shared_ptr` chain destructor recurses once per node — fifty thousand audit-log entries on the default 1 MiB Windows stack crashes whichever thread drops the last snapshot. Worse, checking `use_count() == 1` to unlink live nodes violates `[util.smartptr.shared.obs]/16` (a relaxed `use_count()` read does not synchronize with a reader that just finished walking `next`). Parking `std::move(next)` in a thread-local drain vector inside `~Node()` guarantees we only touch `next` *after* the node's own reference count has hit zero with full `acq_rel` synchronization, flattening recursion to O(1) stack depth.

---

## Pattern 8 — Sharded Counter Map / Striped Lock Aggregator

High-throughput aggregation wants many small locks, not one heroic mutex taking a beating like a shared printer.

```cpp
// sharded_counter_map.h  -----------------------------------------------------
#pragma once
#include <functional>
#include <memory>
#include <mutex>
#include <new>
#include <unordered_map>
#include <utility>
#include <cstddef>
#include <cstdint>

#ifndef NINJA_CACHE_LINE_DEFINED
#define NINJA_CACHE_LINE_DEFINED
#ifdef __cpp_lib_hardware_interference_size
    inline constexpr std::size_t k_cache_line = std::hardware_destructive_interference_size;
#else
    inline constexpr std::size_t k_cache_line = 64;
#endif
#endif

template <typename Key,
          typename Count = std::uint64_t,
          std::size_t Shards = 64,
          typename Hash = std::hash<Key>>
class ShardedCounterMap {
    static_assert(Shards > 0);

    struct alignas(k_cache_line) Shard {
        mutable std::mutex mu;
        std::unordered_map<Key, Count, Hash> counts;
    };

public:
    ShardedCounterMap()
        : shards_(std::make_unique<Shard[]>(Shards)) {}

    ShardedCounterMap(const ShardedCounterMap&)            = delete;
    ShardedCounterMap& operator=(const ShardedCounterMap&) = delete;

    void increment(const Key& key, Count delta = 1) {
        auto& s = shard_for(key);
        std::unique_lock lk(s.mu);
        s.counts[key] += delta;
    }

    void add(Key key, Count delta = 1) {
        auto& s = shard_for(key);
        std::unique_lock lk(s.mu);
        s.counts[std::move(key)] += delta;
    }

    [[nodiscard]] Count get(const Key& key) const {
        const auto& s = shard_for(key);
        std::unique_lock lk(s.mu);
        auto it = s.counts.find(key);
        return it == s.counts.end() ? Count{} : it->second;
    }

    void erase(const Key& key) {
        auto& s = shard_for(key);
        std::unique_lock lk(s.mu);
        s.counts.erase(key);
    }

    [[nodiscard]] std::unordered_map<Key, Count, Hash> snapshot() const {
        std::unordered_map<Key, Count, Hash> total;
        for (std::size_t i = 0; i < Shards; ++i) {
            const auto& s = shards_[i];
            std::unique_lock lk(s.mu);
            for (const auto& [key, count] : s.counts) {
                total[key] += count;
            }
        }
        return total;
    }

    void clear() {
        for (std::size_t i = 0; i < Shards; ++i) {
            auto& s = shards_[i];
            std::unique_lock lk(s.mu);
            s.counts.clear();
        }
    }

private:
    [[nodiscard]] Shard& shard_for(const Key& key) {
        return shards_[hash_(key) % Shards];
    }

    [[nodiscard]] const Shard& shard_for(const Key& key) const {
        return shards_[hash_(key) % Shards];
    }

    Hash hash_{};
    std::unique_ptr<Shard[]> shards_;
};
```

- **Writes scale until the hash says no.** Different keys hit different shard locks; the same hot key still serialises. Physics remains rude.
- **Aggregate reads are intentionally heavier.** `snapshot()` locks shards one at a time and merges. Perfect for once-per-second metrics scrapes; bad for every request.
- **Padding matters.** `alignas(k_cache_line)` keeps adjacent shard mutexes from false-sharing each other into sadness.
- **Real use:** per-second request counts by route, latency buckets by endpoint, cache-hit counters at Envoy/HAProxy scale. For exact global ordering, use something else and accept the bill.

---

## Pattern 9 — Future Continuations with `std::async`

Dependent async steps are readable when you compose them; the standard forgot `.then()`, so we build the tiny questionable bridge ourselves.

```cpp
// future_then.h  -------------------------------------------------------------
#pragma once
#include <future>
#include <functional>
#include <type_traits>
#include <utility>
#include <vector>

// then(future<T>, F) -> future<R>. The continuation runs after input.get().
// This is a teaching-sized adapter, not an executor. Manage expectations.
template <typename T, typename F>
[[nodiscard]] auto then(std::future<T> input, F&& f) {
    using Fn = std::decay_t<F>;

    if constexpr (std::is_void_v<T>) {
        using R = std::invoke_result_t<Fn>;
        return std::async(std::launch::async,
            [in = std::move(input), fn = Fn(std::forward<F>(f))]() mutable -> R {
                in.get();
                if constexpr (std::is_void_v<R>) {
                    std::invoke(std::move(fn));
                } else {
                    return std::invoke(std::move(fn));
                }
            });
    } else {
        using R = std::invoke_result_t<Fn, T>;
        return std::async(std::launch::async,
            [in = std::move(input), fn = Fn(std::forward<F>(f))]() mutable -> R {
                decltype(auto) value = in.get();  // propagates exceptions; preserves T& if reference future
                if constexpr (std::is_void_v<R>) {
                    std::invoke(std::move(fn), std::forward<T>(value));
                } else {
                    return std::invoke(std::move(fn), std::forward<T>(value));
                }
            });
    }
}

[[nodiscard]] std::future<std::vector<int>> fetch() {
    return std::async(std::launch::async, [] {
        return std::vector<int>{1, 2, 3, 4, 5};
    });
}

[[nodiscard]] int parse(std::vector<int> bytes) {
    int sum = 0;
    for (int v : bytes) sum += v;
    return sum;
}

[[nodiscard]] int transform(int parsed) {
    return parsed * 10;
}

void example() {
    auto result = then(then(fetch(), parse), transform);
    const int answer = result.get();  // 150, unless the universe files a bug
    [[maybe_unused]] auto published = answer;
}
```

- **Exceptions propagate naturally.** If `fetch()` throws, `input.get()` rethrows inside the continuation; the returned future carries that exception onward like a tiny legal notice.
- **The returned future's destructor blocks** ([futures.async]/5). Discarding an intermediate `then(...)` result at a semicolon does not let your caller move on — `~future()` blocks the discarding thread until the async task finishes, serialising what you thought was parallel work. Either chain every result or `.get()` it. A truly fire-and-forget continuation requires `std::thread{...}.detach()` or a real executor.
- **Each `then()` may spawn a thread.** `std::async(std::launch::async)` is deliberately blunt. Fine for low fan-out I/O chains; catastrophic for thousands of micro-stages.
- **No cancellation, no scheduler, no affinity.** This is not P2300, not coroutines, not a production executor. It is a clean bridge before you commit to the real machinery.
- **Real use:** serialising dependent async I/O steps -- fetch, parse, transform, store -- when blocking would be uglier and a coroutine migration is not on today's bingo card.

---

## The Pattern Menu

| Requirement | Reach for | Hard constraint |
|---|---|---|
| Fastest 1-to-1 message passing between two dedicated threads | `SpscRingBuffer<T>` | Exactly one producer, one consumer, forever |
| Dispatch callable work, get typed results back | `ThreadPool` | Never call `future.get()` from inside a submitted task |
| Config / routing / model weights: rare writes, reads on every request | `AtomicRouteStore` | Snapshots are request-scoped; long-lived snapshots pin old versions |
| Multi-stage processing with different stage throughputs | `BoundedQueue` pipeline | Close stages source-to-sink; never reverse the shutdown order |
| Concurrent read-heavy lookup with time-based expiry | `TtlCache` | Factory in `get_or_load` must be idempotent if concurrent misses are possible |
| Thread-pool-style work queue with many producers and consumers | `MpmcBoundedQueue<T>` | `close()` means drain queued work, reject new work, then `pop()` returns `nullopt` |
| Append-only event history with many readers and one writer | `ObservableHistory<T>` | Nodes are immutable; multiple writers require CAS or a mutex |
| High-throughput counters by key | `ShardedCounterMap<Key>` | Aggregation locks every shard; same hot key still serialises |
| Dependent async steps without coroutines/executors | `then(std::future<T>, F)` | Each continuation uses `std::async`; do not build a million-stage confetti cannon |

---

**The right primitives, in the right shape, in production beats clever ones in isolation.**
