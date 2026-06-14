# Chapter 7 — Futures: async / future / promise / packaged_task

> "The future is already here — it's just blocked on `get()`."

---

## 7.1 Cold Open: The Great Async Outage of Q3

Team rewrites batch processor with "modern C++ async." Latency increases 40%.

```cpp
// BROKEN — looks parallel, IS sequential
void process_batch(const std::vector<int>& ids)
{
    for (int id : ids)
        std::async(process_item, id);  // future discarded; dtor BLOCKS at this semicolon
    // Result: serial execution + thread-creation overhead. Worse than a plain for-loop.
}
```

The destructor of a future returned by `std::async` blocks until the task finishes. Discard the future, pay the wait, every iteration, in sequence.

Bonus bug: no launch policy. The default (`async | deferred`) lets the implementation choose. If it picks `async` and the future is discarded, the dtor blocks — serializing the loop. If it picks `deferred` and the future is discarded, the task is **silently never executed**. Either way: wrong.

```cpp
// FIXED
void process_batch(const std::vector<int>& ids)
{
    std::vector<std::future<void>> fv;
    fv.reserve(ids.size());
    for (int id : ids)
        fv.push_back(std::async(std::launch::async, process_item, id));
    for (auto& f : fv) f.get();   // collect here, not in the launch loop
}
```

Two rules. Keep the future. Specify `std::launch::async`. This chapter explains why.

---

## 7.2 The Mental Model: One Box, Four Handles

The shared state is a heap-allocated, reference-counted box: holds a value-or-exception and a condvar. Everything else is a handle to that box.

A future is a coat-check ticket. A promise is the attendant who stuffs your coat in the slot. A packaged_task is an attendant who also knows the procedure. `std::async` is the entire coat-check as a vending machine.

| Handle | Role | One-shot? |
|---|---|---|
| `std::future<T>` | Consumer. `get()` moves value out. | Yes — one `get()`. |
| `std::promise<T>` | Producer. Manual `set_value` / `set_exception`. | Yes — one `set_*`. |
| `std::packaged_task<R(Args)>` | Callable with a built-in promise. | Yes — invoke once. |
| `std::async(policy, f, args...)` | Spawns task, returns future. | — |

All four pieces in one snippet:

> **Note:** The blocking destructor is exclusive to futures returned by `std::async(std::launch::async, ...)`. Futures from `std::promise` or `std::packaged_task` have normal, non-blocking destructors. Don't conflate them.

```cpp
#include <future>
#include <thread>
#include <string>
#include <memory>

void all_four_pieces()
{
    // --- promise + future ---
    std::promise<int> p;
    std::future<int>  f = p.get_future();
    std::thread([p = std::move(p)]() mutable { p.set_value(42); }).detach();
    (void)f.get();                      // 42; future is now invalid

    // --- packaged_task ---
    std::packaged_task<int(int)> task([](int x) { return x * x; });
    std::future<int> f2 = task.get_future();
    std::thread(std::move(task), 7).detach();
    (void)f2.get();                     // 49

    // --- async ---
    auto f3 = std::async(std::launch::async, []{ return std::string("done"); });
    (void)f3.get();                     // "done"

    // --- shared_future (broadcast) ---
    std::promise<std::string> sp;
    std::shared_future<std::string> sf = sp.get_future().share();
    std::thread([sf]{ (void)sf.get(); }).detach();  // reader 1
    std::thread([sf]{ (void)sf.get(); }).detach();  // reader 2
    sp.set_value("broadcast");          // both unblock
}
```

---

## 7.3 `std::future<T>` API

> **Intuition pump (real-world):** `std::future<T>` is your coat-check ticket. You handed over your jacket and got a numbered stub. Don't pester the attendant every five seconds asking if it's ready — they're working on it. `future.get()` is when you finally walk up, show the ticket, and they hand you the coat. If it's not ready, you stand there until it is. The ticket is one-shot: try to claim the coat a second time and they look at you like you've lost the plot entirely. You have.

```cpp
#include <future>
#include <chrono>
#include <stdexcept>
using namespace std::chrono_literals;

void future_api_tour()
{
    std::promise<int> p;
    std::future<int>  f = p.get_future();

    f.valid();          // true — has shared state
    f.wait_for(0s);     // future_status::timeout (not ready yet)

    p.set_value(99);

    f.wait_for(0s);     // future_status::ready
    f.wait();           // no-op: already ready; future still valid

    int v = f.get();    // 99; moves value out; INVALIDATES future
    (void)v;
    f.valid();          // false — do not call get() again

    // Exception propagation: thrown in worker, rethrown on get().
    auto bad = std::async(std::launch::async, []() -> int {
        throw std::runtime_error("boom");
        return 0;
    });
    try   { bad.get(); }
    catch (const std::runtime_error&) { /* caught here, in the calling thread */ }
}
```

`get()` moves the value out. The future dies. `wait()` lets you block without consuming it. `wait_for(0s)` is a non-blocking poll. `future_status::deferred` means the task hasn't started — it runs on the first thread to call `get()` or `wait()`; timed waits (`wait_for`/`wait_until`) return `future_status::deferred` without running the task. That's the deferred footgun in action.

---

## 7.4 `std::shared_future<T>`: One Result, Many Readers

`future` is move-only; one `get()`. `shared_future` is copyable; for object types `get()` returns `const T&` (reference specializations return `T&`; `shared_future<void>::get()` returns nothing); callable from any number of threads any number of times.

> **Intuition pump (real-world):** `std::shared_future` is the photocopier for your coat-check ticket. You got the original, ran it through the copier, handed copies to five friends. Each friend independently walks to the counter, shows their copy, and gets told the same answer — the coat that was retrieved once, returned by `const&` every subsequent time. Handing the original `future` around is *not* this: `get()` moves the value out and the ticket self-destructs. Call `.share()` first, then distribute. It is a two-second fix that will save you one very confusing `future_error(no_state)` at midnight.

```cpp
#include <future>
#include <thread>
#include <string>
#include <vector>

void shared_future_broadcast()
{
    std::promise<std::string> p;
    std::shared_future<std::string> sf = p.get_future().share();
    // share() consumes the unique future; sf is now the broadcast handle.

    std::vector<std::thread> readers;
    for (int i = 0; i < 6; ++i)
        readers.emplace_back([sf]() {
            const std::string& v = sf.get();  // all six block until set_value
            (void)v;
        });

    p.set_value("leader=node-3");       // all six unblock

    for (auto& t : readers) t.join();
}
```

Use when: leader election result, config reload complete, any "wait until we know X" pattern with N consumers.

---

## 7.5 `std::promise<T>`: Manual Producer

`promise` is the write side. Pull the future out first. Then set from wherever the answer comes from.

> **Intuition pump (real-world):** `std::promise<T>` is the coat-check attendant. Their job is to eventually stuff something into the slot matching your ticket. Maybe a coat (`set_value`). Maybe a soggy note that reads "I spilled coffee on your jacket" (`set_exception`). Either way, one job, one shot, the slot gets filled exactly once. If they clock out without filling the slot — destructor fires, no `set_*` called — the shared state becomes ready with a `broken_promise` exception. Everyone waiting unblocks; when they call `get()`, it throws `future_error(broken_promise)`. Note: `wait()` and `wait_for()` do not throw — they only unblock. The exception is rethrown exclusively by `get()`. "Broken promise" in production logs is diagnostic gold: some code path died before it could answer. Find it. It wasn't an accident.

```cpp
#include <future>
#include <thread>
#include <stdexcept>

void promise_basics()
{
    // --- set_value ---
    {
        std::promise<int> p;
        auto f = p.get_future();
        std::thread([p = std::move(p)]() mutable { p.set_value(7); }).detach();
        (void)f.get();  // 7
    }

    // --- set_exception ---
    {
        std::promise<int> p;
        auto f = p.get_future();
        std::thread([p = std::move(p)]() mutable {
            p.set_exception(std::make_exception_ptr(std::runtime_error("failed")));
        }).detach();
        try   { f.get(); }
        catch (const std::runtime_error&) { /* rethrown here */ }
    }

    // --- broken promise: dtor fires without set_* ---
    {
        std::future<int> f;
        { std::promise<int> p; f = p.get_future(); }  // p destroyed, no set called
        try   { f.get(); }    // throws future_error(broken_promise)
        catch (const std::future_error&) { /* something died without answering */ }
    }
}
```

Broken promise in production logs = a code path died before it could answer. Diagnostic gold.

`promise<void>` carries no value — pure one-shot signal:

```cpp
std::promise<void> rdy;
auto sig = rdy.get_future();
std::thread([r = std::move(rdy)]() mutable {
    /* ... init work ... */
    r.set_value();          // "ready"
}).detach();
sig.get();                  // caller blocks here
```

---

## 7.6 `std::packaged_task`: Callable with a Built-In Promise

Wrap a callable. Pull the future. Decide when and where to invoke it — you own the executor.

> **Intuition pump (real-world):** `std::packaged_task` is a self-addressed, stamped envelope. You write the task inside, seal it, hand it to whoever — a thread, a pool worker, a test harness running it at 3 AM. They invoke it whenever. The result auto-mails back to the return address (the future) without you writing a single line of marshaling code. The envelope is move-only because you can't photocopy a self-addressed envelope and have both copies deliver to the same mailbox. Which, when you think about it, is just good postal policy and also the reason your `push_back(task)` doesn't compile.

```cpp
#include <future>
#include <thread>

void packaged_task_basics()
{
    std::packaged_task<int(int, int)> task([](int a, int b) { return a + b; });
    std::future<int> f = task.get_future();

    std::thread t(std::move(task), 20, 22);  // task is move-only; must move
    (void)f.get();   // 42
    t.join();
}
```

`packaged_task` is move-only. `std::function<void()>` requires copyable. The bridge: `shared_ptr`.

```cpp
// The thread-pool idiom in miniature:
auto pt = std::make_shared<std::packaged_task<int()>>([]{ return 1; });
std::future<int>      fut = pt->get_future();
std::function<void()>  fn = [pt]{ (*pt)(); };   // copyable via shared_ptr
// push fn into any queue; worker calls fn(); fut becomes ready.
```

This pattern appears in every C++ thread pool ever written. Section 7.10 shows the full version.

---

## 7.7 `std::async`: The Deceptively Simple One-Liner

### Footgun 1: discarded future = blocking destructor

> **Intuition pump (real-world):** You asked the chef to make a burger. They started cooking and handed you the receipt. You threw the receipt in the bin. The chef makes the burger anyway — but that receipt's **destructor** blocks until the burger is plated and ready. So you stand there for three minutes staring at the bin, wondering why your "async" kitchen is somehow slower than cooking everything yourself. The receipt was the future. The bin was `void`. The standing-and-waiting was the destructor. The lesson is written in the commit history of approximately every team that has touched `std::async` for the first time. Keep the receipt.

```cpp
// BROKEN — dtor of each temporary blocks; this is a serial loop
for (int id : ids)
    std::async(std::launch::async, work, id);
```

```cpp
// FIXED — store futures; all tasks run concurrently; collect at the end
std::vector<std::future<void>> fv;
for (int id : ids)
    fv.push_back(std::async(std::launch::async, work, id));
for (auto& f : fv) f.get();
```

### Footgun 2: missing launch policy

```cpp
// DANGEROUS — may run deferred on the caller's thread; no guarantees
auto f = std::async(heavy, data);

// CORRECT — must run asynchronously
auto f = std::async(std::launch::async, heavy, data);
```

| Policy | Behavior |
|---|---|
| `std::launch::async` | New thread or thread pool. Guaranteed async. |
| `std::launch::deferred` | Runs on the first thread to call `get()` or `wait()`. No new thread. Timed waits return `deferred` without running. |
| `async \| deferred` (default) | Implementation picks. You have no guarantees. |

> **Intuition pump (real-world):** The default launch policy is ordering from a restaurant without specifying delivery or pickup. The restaurant will absolutely choose "come get it yourself" (`deferred`) to save labour. If they do roll the dice and pick `async`, and you threw away the receipt, see the burger story above. Specifying `std::launch::async` is checking the delivery box. It is one token. The cost of skipping it has appeared in at least one incident review per team per year since C++11 shipped. Check the box.

Always specify. The default exists for lazy evaluation, not for "I want this to run in the background." If you wanted lazy, you'd write a lambda.

### Footgun 3: `wait_for` on a deferred future always returns `deferred`

```cpp
auto f = std::async(work);                   // default policy — may be deferred
auto s = f.wait_for(std::chrono::seconds(1));
// s might be future_status::deferred, never timeout or ready,
// because the task hasn't started and won't until you call get().
```

---

## 7.8 Production Snippet 1: Bounded Parallel Fan-Out

`counting_semaphore` caps in-flight tasks. The code IS the explanation.

```cpp
#include <future>
#include <semaphore>
#include <vector>
#include <string>
#include <stdexcept>

std::string fetch_url(const std::string& url)
{
    if (url.empty()) throw std::invalid_argument("empty url");
    return "body:" + url;
}

std::vector<std::string> fetch_all(const std::vector<std::string>& urls,
                                   std::ptrdiff_t max_in_flight = 16)
{
    if (max_in_flight <= 0)
        throw std::invalid_argument("max_in_flight must be > 0");  // semaphore ctor UB otherwise
    std::counting_semaphore<> slots(max_in_flight);
    std::vector<std::future<std::string>> futures;
    futures.reserve(urls.size());

    for (const auto& url : urls)
    {
        slots.acquire();    // block if max_in_flight tasks already running
        try
        {
            futures.push_back(std::async(std::launch::async,
                [&slots, url]() -> std::string {
                    struct Rel { std::counting_semaphore<>& s;
                                 ~Rel() { s.release(); } } _r{slots};
                    return fetch_url(url);
                }));
        }
        catch (...)
        {
            slots.release();  // async threw (e.g. std::system_error: out of threads); reclaim slot
            throw;
        }
    }

    std::vector<std::string> results;
    results.reserve(urls.size());
    for (auto& f : futures)
        results.push_back(f.get());   // first exception propagates
    return results;
}
```

Collect all errors instead of stopping at the first one:

```cpp
    std::vector<std::exception_ptr> errs;
    for (auto& f : futures)
    {
        try   { results.push_back(f.get()); }
        catch (...) { errs.push_back(std::current_exception()); results.emplace_back(); }
    }
    if (!errs.empty()) std::rethrow_exception(errs.front());
```

---

## 7.9 Production Snippet 2: `promise<void>` Init Signal

Constructor blocks until the worker is ready, or throws if the worker fails. Exception crosses the thread boundary without you writing any marshaling code.

```cpp
#include <future>
#include <thread>
#include <chrono>
#include <functional>
#include <stdexcept>

class Service
{
public:
    explicit Service(std::function<void()> on_init,
                     std::function<void()> on_tick)
    {
        std::promise<void> rdy;
        auto sig = rdy.get_future();

        worker_ = std::jthread(
            [on_init = std::move(on_init),
             on_tick = std::move(on_tick),
             p       = std::move(rdy)](std::stop_token st) mutable
            {
                try
                {
                    on_init();      // may throw; if so, propagates to ctor caller
                    p.set_value();  // "ready" — unblocks the constructor

                    while (!st.stop_requested()) on_tick();
                    // Note: one extra on_tick() may run if stop is requested
                    // between the check and the call. Acceptable for graceful shutdown;
                    // if on_tick must never run post-stop, pass the stop_token into it.
                }
                catch (...)
                {
                    // Guard: set_value may have already been called if on_tick threw.
                    try { p.set_exception(std::current_exception()); }
                    catch (const std::future_error&) { /* already satisfied */ }
                }
            });

        sig.get();  // blocks; rethrows worker's init exception if any
    }
    // jthread dtor: requests stop + joins automatically. Nothing to clean up.

private:
    std::jthread worker_;
};

void demo_service()
{
    try
    {
        Service svc(
            [] { /* bind socket — may throw */ },
            [] { std::this_thread::sleep_for(std::chrono::milliseconds(10)); });
        // ctor returned → worker is ready and running
    }
    catch (const std::exception& e) { (void)e; /* init failed */ }
}
```

---

## 7.10 Production Snippet 3: `ThreadPool::submit` Returning a Future

The `shared_ptr<packaged_task>` bridge makes a move-only callable live inside a copyable `std::function`. This is the canonical thread-pool idiom.

```cpp
#include <future>
#include <functional>
#include <memory>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <vector>
#include <type_traits>
#include <utility>      // std::move, std::forward, std::invoke
#include <stdexcept>

class ThreadPool
{
public:
    explicit ThreadPool(std::size_t n)
    {
        // If workers_.emplace_back() throws (OS thread limit, bad_alloc, etc.),
        // any threads already running would be destroyed unjoinable → std::terminate.
        // Catch, signal them to stop, drain, then rethrow. Your OS will thank you.
        try
        {
            for (std::size_t i = 0; i < n; ++i)
                workers_.emplace_back([this]{ drain(); });
        }
        catch (...)
        {
            { std::lock_guard lk(mu_); stop_ = true; }
            cv_.notify_all();
            for (auto& t : workers_)
                if (t.joinable()) t.join();
            throw;
        }
    }

    ~ThreadPool()
    {
        { std::lock_guard lk(mu_); stop_ = true; }
        cv_.notify_all();
        for (auto& t : workers_) t.join();
    }

    template<typename F, typename... Args>
    [[nodiscard]] auto submit(F&& f, Args&&... args)
        -> std::future<std::invoke_result_t<std::decay_t<F>, std::decay_t<Args>...>>
    {
        using R = std::invoke_result_t<std::decay_t<F>, std::decay_t<Args>...>;

        // Zero-argument callable with bound args.
        auto bound = [fn  = std::forward<F>(f),
                      ...a = std::forward<Args>(args)]() mutable -> R {
            return std::invoke(std::move(fn), std::move(a)...);
        };

        // packaged_task is move-only → shared_ptr makes it safe for std::function.
        auto pt  = std::make_shared<std::packaged_task<R()>>(std::move(bound));
        auto fut = pt->get_future();

        {
            std::lock_guard lk(mu_);
            if (stop_) throw std::runtime_error("pool stopped");
            q_.push([pt]{ (*pt)(); });  // lambda captures shared_ptr by value
        }
        cv_.notify_one();
        return fut;
    }

    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

private:
    void drain()
    {
        while (true)
        {
            std::function<void()> task;
            {
                std::unique_lock lk(mu_);
                cv_.wait(lk, [this]{ return stop_ || !q_.empty(); });
                if (stop_ && q_.empty()) return;
                task = std::move(q_.front()); q_.pop();
            }
            task();
        }
    }

    std::mutex                        mu_;
    std::condition_variable           cv_;
    std::queue<std::function<void()>> q_;
    std::vector<std::thread>          workers_;
    bool                              stop_ = false;
};

void demo_pool()
{
    ThreadPool pool(4);
    auto f1 = pool.submit([]{ return 42; });
    auto f2 = pool.submit([](std::string s){ return s + "!"; }, std::string("hi"));
    (void)f1.get();     // 42
    (void)f2.get();     // "hi!"
}
```

`[[nodiscard]]` on `submit` is not pedantry — it is the mechanical guard against the bug in Section 7.1. The compiler warns when you discard the future. It costs nothing. Always use it.

---

## 7.11 `when_all` / `when_any`: The Standard Doesn't Have Them

Proposed every three years since C++14. Not in C++20. File the bug report with the committee.

**Faking `when_all`** — a loop of `get()`. All tasks run in parallel; you collect sequentially:

```cpp
template<typename T>
std::vector<T> when_all(std::vector<std::future<T>>& fv)
{
    std::vector<T> out;
    out.reserve(fv.size());
    for (auto& f : fv) out.push_back(f.get());  // first exception propagates
    return out;
}
```

Suboptimal only when task latencies are wildly unequal and you care about seeing the fast ones early. Otherwise it's fine.

**Faking `when_any`** requires N watcher threads, each blocking in `get()`. You spawn O(N) threads whose entire job is to wait. This is ugly enough that if you genuinely need `when_any`, reach for Folly's `SemiFuture`, HPX's `future`, or standard coroutines (Chapter 12). The standard's future model is pull-based by design; push-based composition is not in `<future>`.

---

## 7.12 The Footguns Table

| Footgun | Symptom | Fix |
|---|---|---|
| Discard `async` future | Dtor blocks; sequential perf (**async-launched only**) | Store the future |
| No launch policy | async: dtor blocks; deferred: task silently dropped | Always `std::launch::async` |
| `get()` twice | `future_error(no_state)` | One `get()`; `shared_future` for multi-read |
| `set_value()` twice | `future_error(promise_already_satisfied)` | Write-once; audit all code paths |
| Destroy promise without setting | waiters unblock; `get()` throws `future_error(broken_promise)` | `set_*` on every path, including catch |
| `future<T&>` with dead referent | Dangling ref on `get()` | Use `future<T>` by value |
| Pass `packaged_task` by value | Compile error or empty task | `std::move()` it; `shared_ptr` for `std::function` |
| `wait_for` on deferred future | Always returns `deferred` | Force `launch::async` |

Code reminders for the three most expensive:

```cpp
// --- discard trap ---
std::async(std::launch::async, work);           // BLOCKS (dtor waits for task)
auto f = std::async(std::launch::async, work);  // OK — keep the future

// --- get() twice ---
auto f = std::async(std::launch::async, []{ return 1; });
f.get();    // fine
f.get();    // throws future_error(no_state)
// Fix: f.share() → shared_future → call get() as many times as needed

// --- set_value() twice ---
std::promise<int> p;
p.set_value(1);
p.set_value(2);  // throws future_error(promise_already_satisfied)
```

---

## 7.13 When NOT to Use Futures

- **High-frequency producer/consumer** — one heap allocation per shared state; use a condvar queue (Ch. 5) or lock-free ring buffer (Ch. 11).
- **Continuation chains** (`.then().then()`) — not in standard C++; use coroutines (Ch. 12) or Folly/HPX.
- **GPU / heterogeneous compute** — CUDA/SYCL have their own event types; `std::future` does not interop.
- **Multi-condition waits** — "wake me when A or B or timeout"; `std::condition_variable` with a compound predicate is more expressive.
- **Fire-and-forget** — the blocking destructor makes true fire-and-forget impossible with `std::async`; use a thread pool with a void queue.

Futures are for "one result, once, eventually, from one other place." Richer models need richer tools.

---

## 7.14 Summary

The entire ecosystem is one shared-state box with four different handles:

```
std::async          — you just want it to run somewhere
std::packaged_task  — you pick the executor
std::promise        — you wire everything manually
```

Rules:

- Always `std::launch::async`. Never trust the default.
- Keep the future. The discarded-future destructor blocks.
- `get()` once per `future`. `shared_future` for fan-out reads.
- `set_*` on every code path. A destroyed promise breaks its waiters.
- `shared_ptr<packaged_task>` is the bridge to `std::function`.

---

*Specify the launch policy; keep the future; call `get()` once — violate any of these and you have not written async code, you have written a sequential bug with extra thread-creation overhead.*
