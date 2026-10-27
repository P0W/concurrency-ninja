# Chapter 9 — Cooperative Cancellation: stop_token

---

## Cold Open: How We Used To Do It

`TerminateThread` (Win32). `pthread_cancel` (POSIX). `TerminateThread` and *asynchronous*
`pthread_cancel` stop a thread at an arbitrary instruction — mid-`malloc`, lock held, pointer
half-swapped. POSIX's *default* deferred cancellation fires only at cancellation points, but
in pure C POSIX it skips C++ destructors entirely, and on Linux (`glibc`/NPTL) it triggers
`_Unwind_ForcedUnwind` — a foreign-exception stack unwind that silently `std::terminate()`s
the moment it hits a `noexcept` boundary. The postmortem is always six words: *the heap lock
was left held.* C++20's answer: **the thread stops itself.**

```cpp
// The old way.  Filed under "never again."
// TerminateThread(hWorker, 0);  // global heap lock: now permanently locked. enjoy your hang.

// The C++20 way.
#include <stop_token>
#include <thread>

std::jthread worker{[](std::stop_token st) {
    while (!st.stop_requested()) {   // safe point: no locks held, invariants intact
        do_unit_of_work();
    }
    // RAII unwinds cleanly from here; destructors run; mutexes release
}};

// From any thread, at any time — atomic, idempotent, thread-safe:
worker.request_stop();
// jthread destructor: request_stop() then join().  Never std::terminate() for being unjoined.
```

The code is the moral. The thread checks the flag. The thread decides when to stop.
No one else gets a vote.

---

## Why Forced Cancellation Cannot Work

A thread might be anywhere: inside the allocator, halfway through a spinlock acquire,
mid-pointer-swap on a linked list. There is no recovery from killing it at those points.
Cooperative means the worker checks at points *it* defines as safe — end of an iteration,
before a sleep — then unwinds via RAII. The cost: cancellation is not instantaneous.
If the worker is sleeping for 30 seconds, you wait — unless you also hook the stop
to interrupt that sleep. That is what `stop_callback` is for.

> **Intuition pump (real-world):** In the navy, you don't shoot the cook in the middle of cooking dinner. You tap them on the shoulder and say "finish what you're doing, then stand down." They wrap up, plate everything safely, then stop. THAT is cooperative cancellation. The alternative — `TerminateThread` / asynchronous `pthread_cancel` — is shooting the cook mid-stir, with the soup still on the stove and their hand still in the pot. Even POSIX's default deferred `pthread_cancel` is less "polite request" and more "timed grenade": it detonates at the next cancellation point without calling C++ destructors, leaving RAII guards unrun and invariants broken. You won't starve tonight. You will, however, spend the next six hours cleaning the kitchen and explaining the incident to HR.

---

## The Three-Piece System

| Type | Role | Analogy |
|---|---|---|
| `std::stop_source` | Owns the bit. Calls `request_stop()`. | The manager who pushes the button |
| `std::stop_token` | Read-only observer. Cheap to copy. Pass into workers. | The "please stop" sticky note |
| `std::stop_callback` | RAII callback fired when stop is requested. Wakes sleepers. | The doorbell that rings every stopped thread |

All three share a reference-counted control block. All three live in `<stop_token>`.

> **Intuition pump (real-world):** The `stop_source` is the boss's panic button on the wall — `request_stop()` is the fist that slams it. The `stop_token` is the employee badge they all carry: they *check* whether the button has been pressed, and they do it at their own discretion, between tasks, when invariants are intact. The badge doesn't drain anyone's battery — `stop_requested()` is a cheap atomic load. Giving an employee a badge costs nothing. Giving them the panic button itself (`stop_source`) is a different conversation and typically a bad idea.

```cpp
#include <stop_token>
#include <condition_variable>
#include <mutex>
#include <chrono>

void three_piece_demo() {
    std::stop_source src;
    std::stop_token  tok = src.get_token();  // cheap copy; shares control block

    std::mutex mu;
    std::condition_variable cv;  // plain cv: no stop_token magic hiding the callback

    // RAII registration: fires when stop is requested; deregisters on destruction.
    // Lock mu inside the callback so notify_all() cannot fire between the predicate
    // check and the wait (lost wakeup), and declare wake BEFORE lock so mu is not
    // held when ~stop_callback() runs.
    std::stop_callback wake{tok, [&mu, &cv]{
        std::lock_guard lk{mu};
        cv.notify_all();
    }};

    std::unique_lock lock{mu};
    // wait_for returns true if stop fires (predicate became true), false on timeout
    cv.wait_for(lock, std::chrono::seconds{30}, [&tok]{ return tok.stop_requested(); });

    // From another thread: src.request_stop()
    //   → fires wake() → cv.notify_all() → predicate returns true → wait_for returns true → thread exits
}
```

`request_stop()` is idempotent. Calling it seventeen times equals calling it once.
`stop_possible()` returns false on a disengaged source (`std::nostopstate` tag) — useful
as a "I do not participate in cancellation" sentinel.

---

## `jthread` — Free Cancellation With Your Order

`jthread` = `std::thread` + built-in `stop_source` + destructor that calls
`request_stop()` then `join()` (never `terminate()`). If the callable's first
parameter is `std::stop_token`, it is wired in automatically.

> **Intuition pump (real-world):** `jthread` is the thread you wished `std::thread` was. It automatically signs the panic-button contract on hiring — you get a built-in `stop_source`, the stop token is wired in if you want it, and when it's fired it politely lets itself out: `request_stop()` then `join()`, in that order, on the way out the door. No `terminate()`. No dangling thread. No "oh right, I forgot to join." It's the senior hire who reads the employee handbook, shows up on time, and cleans their desk on the last day. `std::thread` is the intern who needs three separate reminders to not leave the heap lock held.

```cpp
#include <thread>

// stop_token injected as first arg automatically
std::jthread t{[](std::stop_token st, int n) {
    for (int i = 0; i < n && !st.stop_requested(); ++i)
        do_work(i);
}, 1'000'000};

t.request_stop();   // sets bit; fires any registered stop_callbacks
t.join();           // or let ~jthread do it
```

If the callable does not accept `stop_token`, `jthread` still compiles and works —
you just lose the auto-injection. Both forms are correct.

---

## Polling Pattern — CPU-Bound Loops

```cpp
#include <stop_token>
#include <cstddef>

void crunch(std::stop_token st, float* data, std::size_t rows, std::size_t cols) {
    for (std::size_t r = 0; r < rows; ++r) {
        if (r % 64 == 0 && st.stop_requested()) return;  // check every 64 rows
        for (std::size_t c = 0; c < cols; ++c)
            process(data[r * cols + c]);
    }
}
```

Checking every iteration of a tight inner loop adds a measurable atomic load per op.
Every 64 rows is cheap. N is your worst-case cancellation latency divided by row time.
Profile before picking N. "I'll check at the top of the outer loop" is only safe if
one iteration is fast.

---

## Wake-Me-When-Stopped — `sleep_for` Does Not Care

> **Intuition pump (real-world):** You yelled at the sleeping employee "wake up, we're closing!" but they had industrial-grade earplugs in. `std::this_thread::sleep_for()` has no slot for a stop token — it is constitutionally deaf to cancellation requests. The employee will wake up when they feel like it, probably 30 seconds from now, right after the building has been locked and the lights turned off. Use `condition_variable_any::wait_for(lock, stop_token, dur, pred)` instead — that's the same nap, but with a phone-call-from-the-boss override that cuts through any earplugs.

```cpp
// BROKEN: destructor blocks up to 30 seconds on shutdown
void bad_worker(std::stop_token st) {
    while (!st.stop_requested()) {
        do_poll();
        std::this_thread::sleep_for(std::chrono::seconds{30}); // uninterruptible; deaf to stop
    }
}
```

```cpp
// CORRECT: condition_variable_any has stop_token overloads; condition_variable does NOT
#include <condition_variable>
#include <mutex>

void good_worker(std::stop_token st) {
    std::mutex mu;
    std::condition_variable_any cv;
    std::unique_lock lock{mu};

    while (!st.stop_requested()) {
        lock.unlock(); do_poll(); lock.lock();
        // Returns false immediately when stop is requested — instant wakeup
        cv.wait_for(lock, st, std::chrono::seconds{30}, []{ return false; });
    }
}
```

Use `condition_variable_any`. Not `condition_variable`. Only `_any` has the overloads.
This is the most common "works on my machine" bug in C++20 concurrency code.

---

## `stop_callback` for Waking Any Sleeper

When `condition_variable_any` is not available — semaphore, `std::future`, platform wait —
wire a `stop_callback` to nudge whatever you are blocked on.

> **Intuition pump (real-world):** `stop_callback` is the doorbell rigged directly to the panic button. The moment the boss slams `request_stop()`, every employee who registered a callback gets an immediate buzz — they don't have to wait for their next polling check at the top of the loop. Sleeping on a semaphore? Wire the callback to `release()` it. Parked on a `std::future`? Wire it to set a flag and notify. The callback fires on the caller's thread, synchronously, during `request_stop()`. This is how you convert "I'll notice in up to 30 seconds" into "I notice right now."

```cpp
#include <stop_token>
#include <condition_variable>
#include <mutex>
#include <chrono>

void interruptible_sleep(std::stop_token st, std::chrono::milliseconds dur) {
    std::mutex mu;
    std::condition_variable cv;

    // Declare AFTER mu and cv (destructs before them) and BEFORE lock (so mu is
    // already unlocked when ~stop_callback blocks on an in-flight callback).
    // Acquiring mu inside the callback prevents a lost wakeup between predicate and wait.
    std::stop_callback wake{st, [&mu, &cv]{
        std::lock_guard lk{mu};
        cv.notify_all();
    }};

    std::unique_lock lock{mu};
    cv.wait_for(lock, dur, [&st]{ return st.stop_requested(); });
    // exits at dur OR when stop fires, whichever is first
}
```

Reverse-declaration-order is the lifetime rule: `wake` destructs before `cv` and `mu`.
The `stop_callback` destructor blocks until any concurrent callback invocation finishes
before it deregisters — so `cv` is always valid when the callback touches it.

---

## Production Snippet 1 — Interruptible HTTP Polling Loop

Polls every 30 seconds. `request_stop()` wakes the inter-poll sleep instantly; the destructor
does not hang. Note: `fetch()` itself takes no stop token here — if an HTTP call can block
indefinitely, pair it with a request-level timeout or a cancellable HTTP API.

```cpp
#include <stop_token>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <string>
#include <functional>

class HttpPoller {
public:
    using Callback = std::function<void(std::string)>;

    explicit HttpPoller(std::string url, Callback on_result)
        : url_{std::move(url)}
        , on_result_{std::move(on_result)}
        , worker_{[this](std::stop_token st) { run(st); }}  // jthread LAST — all members exist by now
    {}

    ~HttpPoller() = default;  // ~jthread: request_stop() then join() — instant wakeup

private:
    void run(std::stop_token st) {
        // Local waker captures mu_ and cv_. Safe: join() in ~jthread ensures
        // this thread exits before any member (including mu_ and cv_) is destroyed.
        std::stop_callback waker{st, [this]{
            std::lock_guard lk{mu_};
            cv_.notify_all();
        }};

        while (!st.stop_requested()) {
            if (auto r = fetch(url_); !r.empty() && !st.stop_requested())
                on_result_(std::move(r));   // skip callback if stop was requested during fetch

            std::unique_lock lock{mu_};
            cv_.wait_for(lock, std::chrono::seconds{30},
                         [&st]{ return st.stop_requested(); });
        }
    }

    static std::string fetch(const std::string&) { return {}; }  // WinHTTP / libcurl here

    std::string             url_;
    Callback                on_result_;
    std::mutex              mu_;
    std::condition_variable cv_;
    std::jthread            worker_;  // MUST be declared last
};
```

`worker_` last is load-bearing: its constructor starts a thread. Anything initialized
after `worker_` in the member list would not yet exist when `run()` first executes.
Destruction is reverse-declaration-order: `worker_` first → `request_stop()` fires
`waker` → `cv_.notify_all()` → thread sees stop and exits → `join()` returns →
then `cv_`, `mu_`, `on_result_`, `url_` destruct. Everything in the right order,
for free, by language rules.

---

## Production Snippet 2 — Graceful Service Shutdown

Multiple `jthread` workers. `shutdown()` is idempotent — safe to call from a destructor.
Concurrent callers: the first caller does all the work; a second concurrent caller returns
as soon as it loses the `stopped_` race, while joining may still be in progress. External
lifetime coordination is the caller's responsibility.

```cpp
#include <stop_token>
#include <thread>
#include <vector>
#include <atomic>
#include <chrono>
#include <utility>

class IndexingService {
public:
    explicit IndexingService(int n) {
        workers_.reserve(n);
        for (int i = 0; i < n; ++i)
            workers_.emplace_back([this](std::stop_token st, int id) {
                run(st, id);
            }, i);
    }

    void shutdown() {
        if (stopped_.exchange(true)) return;     // guard: only the first caller joins
        for (auto& w : workers_) w.request_stop();  // signal all — O(n), non-blocking
        for (auto& w : workers_) w.join();           // wait all
    }

    ~IndexingService() { shutdown(); }  // safe: exchange returns true, no double-join

private:
    void run(std::stop_token st, int /*id*/) {
        while (!st.stop_requested()) {
            fetch_and_index_one_item();
            if (st.stop_requested()) break;  // check after slow op, not just at top
        }
    }

    void fetch_and_index_one_item() {
        std::this_thread::sleep_for(std::chrono::milliseconds{10}); // placeholder
    }

    std::vector<std::jthread>  workers_;
    std::atomic<bool>          stopped_{false};  // guards against double-join; see shutdown()
};
```

The double-check after `fetch_and_index_one_item` matters. If that call takes 500ms
and you only check at the top of the loop, you burn one extra round-trip after
cancellation. For a service that must shut down in under two seconds, that matters.

---

## Production Snippet 3 — Propagating Stop Through a Chain + Merging Two Tokens

Cancellation flows naturally: pass `stop_token` by value down the call stack.

```cpp
#include <stop_token>
#include <optional>
#include <string>
#include <string_view>
#include <functional>
#include <chrono>
#include <condition_variable>
#include <mutex>
#include <thread>

std::optional<std::string> db_query(std::string_view sql, std::stop_token st);
std::optional<std::string> post_process(std::string_view raw, std::stop_token st);

std::optional<std::string> run_search(std::string_view query, std::stop_token st) {
    if (st.stop_requested()) return std::nullopt;
    auto row = db_query("SELECT body FROM idx WHERE term=?", st);
    if (!row || st.stop_requested()) return std::nullopt;
    return post_process(*row, st);
}
```

Two independent cancellation sources (user cancel + deadline timeout)?
The standard has no `merge`. Build it:

```cpp
// Fires when EITHER t1 OR t2 fires
struct merged_stop {
    merged_stop(std::stop_token t1, std::stop_token t2)
        : cb1_{t1, [this]{ src_.request_stop(); }}  // src_ init'd before cb1_, cb2_
        , cb2_{t2, [this]{ src_.request_stop(); }}
    {}
    std::stop_token token() const { return src_.get_token(); }
private:
    std::stop_source src_;
    // std::function adds one alloc; fine for shutdown paths
    std::stop_callback<std::function<void()>> cb1_, cb2_;
};

void work_with_deadline(std::stop_token user_st, std::chrono::seconds timeout) {
    std::stop_source deadline_src;
    std::jthread timer{[&deadline_src, timeout](std::stop_token st) {
        // Interruptible wait: wakes immediately if the work finishes and
        // jthread destructor calls request_stop() on this timer thread.
        std::mutex mu;
        std::condition_variable_any cv;
        std::unique_lock lock{mu};
        cv.wait_for(lock, st, timeout, []{ return false; });
        if (!st.stop_requested()) deadline_src.request_stop();
    }};
    merged_stop both{user_st, deadline_src.get_token()};
    run_search("some query", both.token());
}
```

Note `src_` is declared before `cb1_` and `cb2_` in `merged_stop`. The callbacks
capture `this` (via `src_`); they must not fire before `src_` is constructed or
after it is destroyed. Member order enforces this.

---

## Manual `stop_source` Without `jthread`

`std::async` does not propagate cancellation. Wire your own.

```cpp
#include <stop_token>
#include <future>

std::stop_source src;

auto fut = std::async(std::launch::async, [st = src.get_token()] {
    int count = 0;
    while (!st.stop_requested()) { do_work(); ++count; }
    return count;
});

std::this_thread::sleep_for(std::chrono::milliseconds{100});
src.request_stop();     // cooperative signal; does NOT kill the task
int n = fut.get();      // blocks until the task sees stop and returns
```

`future.get()` waits however long the task takes to observe the stop. If the task
is stuck in a 30-second network call that never checks `stop_requested()`, you wait
30 seconds. The signal is cooperative, not preemptive — wire your blocking calls too.

Same pattern for thread pools: pool owns a `stop_source`; passes its token with
every submitted task; calls `request_stop()` at shutdown. Tasks that check the
token exit cleanly; tasks that ignore it run to completion.

---

## `stop_callback` Lifetime Rules — Two Traps

**Trap 1 — fires synchronously at construction if stop is already requested:**

```cpp
std::stop_source src;
src.request_stop();   // bit already set before we construct the callback

// Lambda runs HERE, synchronously, inside this constructor call
std::stop_callback cb{src.get_token(), []{ do_cleanup(); }};
// do_cleanup() has already returned by this line
```

Do not construct a `stop_callback` while holding a lock that the callback acquires.
The callback fires in the constructor. That lock is held. Deadlock.

**Trap 2 — destructor blocks until any concurrent invocation finishes:**

```cpp
// request_stop() races with ~stop_callback on another thread?
// The destructor waits for the callback to return before deregistering.
// This prevents use-after-free on captured locals.
// It is correct. It is also a latency surprise if the callback is slow.
```

Both traps vanish with one rule: declare `stop_callback` after everything it captures;
let reverse-destruction-order handle lifetimes automatically.

---

## `stop_callback` Pitfalls — The Two That Actually Kill You

**Pitfall 1 — Constructing a callback while holding a lock the callback also acquires:**

```cpp
std::mutex m;
std::stop_source src;
src.request_stop();   // bit already set before cb is constructed

{
    std::lock_guard lk{m};
    // stop already requested → constructor fires the callback RIGHT NOW,
    // synchronously, on this thread, while lk is still held.
    std::stop_callback cb{src.get_token(), [&m]{
        std::lock_guard inner{m};  // DEADLOCK: m is already held by this thread
        do_cleanup();
    }};
}
```

The same hazard exists even when stop is not yet requested: a concurrent `request_stop()`
that races with construction may invoke the callback before the constructor returns, still
on the calling thread. Rule: never construct a `stop_callback` while holding a lock the
callback acquires.

**Pitfall 2 — Slow or blocking callbacks:**

`request_stop()` invokes all registered callbacks **synchronously**, on the caller's thread,
before it returns. A callback that sleeps, does I/O, or blocks on a contended lock stalls
every call to `request_stop()` for its full duration.

```cpp
// BAD: callback does slow I/O; request_stop() blocks until flush() returns
std::stop_callback cb{tok, []{
    flush_to_disk();   // could take hundreds of milliseconds
}};
```

Keep callbacks minimal: set a flag, notify a CV, post to a queue. The heavy work belongs
in the worker thread, not in the callback.

---

## Anti-Patterns

```cpp
// BAD: destructor stalls up to 30 seconds
std::jthread t{[](std::stop_token st) {
    while (!st.stop_requested())
        std::this_thread::sleep_for(std::chrono::seconds{30}); // uninterruptible
}};
// FIX: condition_variable_any + stop_token overload, or stop_callback to nudge the wait
```

```cpp
// BAD: atomic load on every iteration of a 100M-iteration loop
for (std::size_t i = 0; i < 100'000'000; ++i) {
    if (st.stop_requested()) return;  // ~100M atomic loads
    tiny_op(i);
}
// FIX: check every 1024 iterations; tune to your latency budget
```

```cpp
// BAD: source and token bundled together and passed by value
struct Ctx { std::stop_source src; std::stop_token tok; };
// All copies share the same control block.
// src.request_stop() on any copy cancels all copies.
// Usually a bug dressed as convenience.
// FIX: source owns (one place), token observes (many copies). Keep them separate.
```

```cpp
// BAD: swallowing all exceptions in a worker loop
try { long_running_op(); }
catch (...) {}   // discards cancellation signals, error codes, everything
while (true) {}  // now runs forever; your service never shuts down
// FIX: catch specific exceptions. Re-check stop_requested() in catch blocks.
```

- **`condition_variable` (not `_any`) + `stop_token`:** the `wait(lock, st, pred)` overload
  does not exist on plain `condition_variable`. Compile error if you pass `st` directly;
  mysterious non-wakeup if you just check `st.stop_requested()` in the predicate with nobody
  ringing the doorbell. Use `_any`.
- **`jthread` declared first in a class with a mutex:** thread starts before the mutex
  initializes. Data race. Declare `jthread` last. Always.
- **Forgetting `request_stop()` on manually-managed workers:** they run forever.
  SCM timeout kills the process. Root-cause analysis takes two days. Ship it anyway.

---

## Compiler Support

| Compiler | Minimum | Notes |
|---|---|---|
| MSVC | 19.29 (VS 2019 16.9) | Full: `jthread`, `stop_token`, `cv_any` overloads |
| GCC | 11 | Full |
| Clang | 14 (libstdc++) / 18–20 (libc++) | With `libstdc++` needs GCC 11+ headers; `libc++` added `jthread` in 18 (`-fexperimental-library`) and enabled it by default in 20 |

```cpp
#if __cpp_lib_jthread >= 201911L
    std::jthread t{[](std::stop_token st){ while (!st.stop_requested()) work(); }};
#else
    // C++17 fallback — needs: <atomic> <condition_variable> <mutex> <thread>
    std::mutex             mu;
    std::atomic<bool>      stop{false};
    std::condition_variable cv;
    std::thread t{[&]{
        std::unique_lock<std::mutex> lk{mu};
        cv.wait(lk, [&]{ return stop.load(); });
    }};
    // shutdown: { std::lock_guard<std::mutex> lk{mu}; stop.store(true); } cv.notify_all(); t.join();
#endif
```

---

## One-Screen Cheat Sheet

```
stop_source ──get_token()──► stop_token ──stop_requested()──► bool

stop_source::request_stop()
  ├── sets atomic bit (idempotent)
  ├── fires all registered stop_callbacks (unspecified order, caller's thread)
  └── subsequent calls: no-op

stop_callback<F>{token, callable}
  ├── construction: if stop already requested → callable fires synchronously RIGHT NOW
  ├── ~stop_callback: blocks until any concurrent invocation finishes, then deregisters
  └── callbacks fire synchronously on the request_stop() caller — keep them short;
      do NOT construct a stop_callback while holding a lock the callback acquires

jthread
  ├── constructor: if callable takes (stop_token, ...) → token is auto-injected
  ├── request_stop(): forwards to internal stop_source
  └── destructor: request_stop() → join()  (in that order, never terminate() for being unjoined)

Blocking wait patterns (pick one):
  condition_variable_any::wait[_for](lock, stop_token, pred)                    ← idiomatic C++20
  stop_callback{st, [&]{ std::lock_guard g{mu}; cv.notify_all(); }}             ← lock mu so notify can't race wait

Class member & wait-scope declaration order:
  std::mutex, std::condition_variable[_any], ..., std::jthread                  ← jthread LAST in class
  mu & cv (outer) → stop_callback (locks mu) → unique_lock (inner wait scope)   ← unlock mu BEFORE ~stop_callback
```

---

## Closing Mantra

*A stop_token is a polite request; cooperative cancellation is the only kind that
doesn't leave mutexes on the floor.*
