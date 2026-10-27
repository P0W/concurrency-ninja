# Chapter 4 — std::condition_variable

*Or: how to sleep properly without burning the house down.*

---

## Cold Open: The Lost Wakeup

Brendan shipped this on a Friday. Tests passed. It hangs in production roughly once a week.

```cpp
// broken — Brendan's Friday pipeline
#include <condition_variable>
#include <mutex>
#include <thread>

bool        ready = false;
std::mutex  mtx;
std::condition_variable cv;

void produce() {
    ready = true;       // (1) set flag — no lock
    cv.notify_one();    // (2) ring doorbell
}

void consume() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);      // (3) bare wait — no predicate
    if (ready) { /* process */ }
}
```

The race: if the scheduler runs `produce()` entirely before `consume()` reaches `cv.wait()`, the notification fires into an empty wait queue and vanishes. The consumer parks itself and waits for a bell that already rang. Forever.

Two bugs, one crash site. Here is the fix:

```cpp
// fixed
void produce() {
    { std::lock_guard<std::mutex> lk(mtx); ready = true; }  // state under lock
    cv.notify_one();                                         // notify after unlock
}

void consume() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [&] { return ready; });   // predicate: re-checks after every wakeup
    /* process — ready is guaranteed true here */
}
```

If `produce()` finishes before `consume()` calls `wait()`, the predicate fires immediately and the consumer never sleeps. No lost wakeup. Both bugs fixed in four lines.

---

## What a cv Is NOT / IS

**NOT:**
- A flag — it stores no state; missed notifications vanish.
- A POSIX signal — it only reaches threads already sleeping on the queue.
- A data queue — it queues *threads*, not values.
- A semaphore — no counter; `std::counting_semaphore` is in Chapter 8.

**IS:**
- A wait queue stapled to a mutex.
- A primitive that atomically **releases the mutex and parks the thread**, then **re-acquires before returning**.
- The cure for `while (!ready) sleep_for(1ms)`.

> **Intuition pump (real-world):** A cv is a doorbell, not a sticky note. When you ring a doorbell, whoever is standing at the door hears it. If nobody is at the door yet, the ring vanishes into the void — the house doesn't log it, and the person arriving five minutes later has no idea you were ever there. Ring the doorbell *before* the pizza actually exists and the customer opens the door, sees nothing, slams it, and spends the rest of the evening quietly losing faith in delivery services. The rule: don't ring it unless you've actually delivered the pizza, and make sure the pizza (`ready = true`) is behind the door *before* you ring.

---

## The Three Iron Rules

1. **Wait under `std::unique_lock<std::mutex>`** — cv needs to unlock/re-lock on your behalf.
2. **Always use the predicate form** — `cv.wait(lock, pred)`, never bare `cv.wait(lock)`.
3. **Modify state under the mutex before `notify_*`** — no exceptions.

### Rule 1 — Lock type matters

```cpp
std::mutex              mtx;
std::condition_variable cv;

// broken: lock_guard has no unlock(); won't compile with cv.wait()
std::lock_guard<std::mutex> bad_lock(mtx);
cv.wait(bad_lock);      // compile error — good, caught early

// fixed: unique_lock is unlockable
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [&] { return ready; });   // cv can unlock/re-lock this
```

Need a non-`std::mutex` lock? Use `std::condition_variable_any`. Shown later.

### Rule 2 — Predicate form vs bare form

Spurious wakeups are real — the C++ standard itself permits `wait()` to unblock without a matching notify ([thread.condition.condvar]). Stolen wakeups are real (another waiter grabs the item first). Both kill you without the predicate.

> **Intuition pump (real-world):** The dog barks at 3 AM for absolutely no reason. You don't leap out of bed, put on shoes, and check every door and window. You roll over and go back to sleep, because you know the dog is just being a dog. The predicate form `cv.wait(lock, pred)` is you rolling over: "woke up — checked — nope, nothing real — back to sleep." The bare `cv.wait(lock)` is the person who gets up every time, wanders the kitchen in the dark, finds nothing, and is exhausted by morning.

```cpp
// broken: spurious or stolen wakeup → processes garbage
cv.wait(lock);
use(shared_state);          // shared_state may not be ready

// fixed: predicate is a while-loop under the hood
cv.wait(lock, [&] { return shared_state.ready; });
use(shared_state);          // guaranteed: predicate was true when we got here
```

### Rule 3 — State under the lock, always

> **Intuition pump (real-world):** You texted "pizza's here!" to someone whose phone was off. They turned it on twenty minutes later, saw no notification, checked the door, found nothing, and went to bed hungry. You *did* send the message. It just fired into the abyss before the recipient was listening. Modifying `ready = true` outside the lock and then calling `notify_one()` is exactly this: the consumer checked `ready`, saw `false`, decided to sleep, and your notification arrived in the window between their check and their park — gone, heard by no one, leaving them sleeping forever in their apartment.

```cpp
// broken: lost wakeup
ready = true;               // set outside lock
cv.notify_one();            // consumer may have checked ready==false, decided to sleep,
                            // and is now sleeping — your notify hit nobody

// fixed
{ std::lock_guard<std::mutex> lk(mtx); ready = true; }
cv.notify_one();
```

---

## Notify Before or After Unlock?

Both are correct. The difference is latency under contention.

> **Intuition pump (real-world):** In a meeting with a talking stick, you hand the stick back *before* you shout "next speaker!" — not after. If you're still clutching the stick when you yell, the next speaker jolts awake, reaches for it, discovers you haven't let go yet, and has to sit back down. You've paid the full cost of waking them for zero progress. Unlock first. Then notify. The woken thread can grab the mutex immediately instead of immediately contending with you for it.

```cpp
// hurry-up-and-wait: notify while holding lock
// woken thread immediately blocks again waiting for YOU to release
{
    std::lock_guard<std::mutex> lk(mtx);
    ready = true;
    cv.notify_one();    // woken thread stalls on the mutex you still hold
}

// preferred hot path: notify after unlock
// woken thread can acquire the mutex immediately
{
    std::lock_guard<std::mutex> lk(mtx);
    ready = true;
}
cv.notify_one();        // mutex is free; woken thread runs without stalling
```

Correctness is identical **for long-lived objects whose lifetime clearly outlives the notification**. For short-lived or stack-allocated synchronization primitives — like `OneShot` below — the awakened thread can destroy the object before `notify_*` executes, leaving you calling a method on a destroyed condition variable. For those, notify while still holding the lock. The hot-path unlock-before-notify advice applies to objects you own and that outlive the call site (queues, services, member variables). Know which one you have.

---

## `condition_variable` vs `condition_variable_any`

One difference: `_any` works with any *Lockable*, not just `unique_lock<mutex>`. It carries a small internal-mutex overhead. Use the plain one with `std::mutex` unless you need flexibility.

> **Intuition pump (real-world):** `std::condition_variable` is a corded landline: it works, it's fast, it does exactly one thing, and it is physically incapable of working anywhere except plugged into a `std::mutex` wall socket. `std::condition_variable_any` is a smartphone: works with any lock type, goes anywhere, and — crucially — is the only one with the stop-token overload. The smartphone is slightly slower, but it's the one that actually rings when you tell it to stop.

The real reason to reach for `_any` in C++20 is the **stop-token overloads** — which plain `condition_variable` does not have:

```cpp
#include <condition_variable>
#include <stop_token>
#include <mutex>
#include <chrono>

std::mutex              mtx;
std::condition_variable_any cv_any;   // _any required; cv does NOT have this overload

// Wrap in a function so stop_token is a real parameter, not thin air.
void wait_for_data_or_stop(std::stop_token st, bool& data_ready) {
    // Atomically wakes on: predicate true OR stop requested — whichever comes first.
    std::unique_lock<std::mutex> lock(mtx);
    cv_any.wait_for(lock, st, std::chrono::seconds(5),
                    [&] { return data_ready; });
    // returns if data_ready==true, timeout elapsed, OR st.stop_requested()
}
```

`std::condition_variable` has no `stop_token` overload. End of story.

---

## `notify_one` vs `notify_all`

> **Intuition pump (real-world):** `notify_one` is calling one specific friend: you know they're available, you ring their phone, done. `notify_all` is sending a group text at midnight: everyone's phone buzzes, everyone squints at their screen in the dark, 63 of them read "pizza here" and think "not for me" and go back to sleep. You paid the full round-trip of waking 64 people to actually move one item. Use `notify_one` when one will do. Use `notify_all` when the state change genuinely applies to everyone — like a shutdown signal.

```cpp
// notify_one: exactly one waiter wakes. right call when one unit of work appeared.
queue_.push_back(std::move(value));
lock.unlock();
not_empty_.notify_one();    // one consumer can drain one item — wake one

// notify_all: every waiter wakes. right call when state change affects everyone.
{
    std::lock_guard<std::mutex> lk(mtx_);
    closed_ = true;
}
cv_.notify_all();           // all blocked threads need to know we're shutting down
```

`notify_all` on a per-item push with 64 consumers: 64 threads wake, 63 find the predicate false, go back to sleep. You paid 64 context switches for one item. That is the thundering herd. Use `notify_one` for single-resource wakeups.

---

## Production Snippet 1 — Bounded Blocking Queue with `close()`

The workhorse. Study it. You will write some variant of this everywhere.

```cpp
#include <condition_variable>
#include <deque>
#include <mutex>
#include <optional>
#include <stdexcept>
#include <type_traits>

template <typename T>
class BoundedBlockingQueue {
    static_assert(std::is_nothrow_move_constructible_v<T>,
                  "T must be nothrow-movable so pop() cannot lose items or strand waiters");
public:
    explicit BoundedBlockingQueue(std::size_t capacity) : capacity_(capacity) {
        if (capacity == 0) throw std::invalid_argument("capacity must be > 0");
    }

    // Returns false if the queue was closed before item could be pushed.
    bool push(T value) {
        std::unique_lock<std::mutex> lock(mtx_);
        not_full_.wait(lock, [&] { return queue_.size() < capacity_ || closed_; });
        if (closed_) return false;
        queue_.push_back(std::move(value));
        lock.unlock();
        not_empty_.notify_one();    // one slot filled → wake one consumer
        return true;
    }

    // Returns nullopt when the queue is closed and fully drained.
    std::optional<T> pop() {
        std::unique_lock<std::mutex> lock(mtx_);
        not_empty_.wait(lock, [&] { return !queue_.empty() || closed_; });
        if (queue_.empty()) return std::nullopt;    // closed + drained
        T value = std::move(queue_.front());
        queue_.pop_front();
        lock.unlock();
        not_full_.notify_one();     // one slot freed → wake one producer
        return value;
    }

    // Close: unblocks all push() and pop() callers.
    void close() {
        { std::lock_guard<std::mutex> lk(mtx_); closed_ = true; }
        not_full_.notify_all();     // wake all producers  (state change = everyone)
        not_empty_.notify_all();    // wake all consumers  (state change = everyone)
    }

private:
    mutable std::mutex          mtx_;
    std::condition_variable     not_full_;   // separate cv per condition — never share
    std::condition_variable     not_empty_;
    std::deque<T>               queue_;
    std::size_t                 capacity_;
    bool                        closed_ = false;
};
```

Design notes compressed:
- Two cvs, not one — one cv for two unrelated conditions means you wake the wrong waiter.
- `close()` uses `notify_all` — shutdown is relevant to every blocked thread.
- `push`/`pop` use `notify_one` — one item, one possible waiter can proceed.
- State modified under lock; notify called after unlock — hot path form.
- `std::optional<T>` return from `pop()` — type-safe "drained and closed" sentinel.

---

## Production Snippet 2 — One-Shot Rendezvous

Fire-and-wait: caller blocks until an async worker delivers a result. No `std::future` required.

```cpp
#include <condition_variable>
#include <mutex>
#include <optional>
#include <thread>

template <typename Result>
class OneShot {
public:
    // Notify inside the lock: the waiter may destroy this object the instant
    // it wakes, so we must not touch cv_ after unlocking.
    void set(Result r) {
        std::lock_guard<std::mutex> lk(mtx_);
        result_ = std::move(r);
        cv_.notify_one();   // under the lock — safe against immediate destruction
    }

    Result wait() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [&] { return result_.has_value(); });
        return std::move(*result_);
    }

private:
    std::mutex              mtx_;
    std::condition_variable cv_;
    std::optional<Result>   result_;    // optional IS the predicate state — no separate bool
};

// Usage
OneShot<int> shot;
std::thread([&]{ shot.set(42); }).detach();
int answer = shot.wait();   // blocks until set() is called
```

`result_.has_value()` doubles as the predicate. No separate `bool ready` that can drift out of sync with the payload.

---

## Production Snippet 3 — Interruptible Sleep (`cv_any` + `stop_token`, C++20)

Every background service needs "sleep N ms, but wake instantly on shutdown."

```cpp
#include <condition_variable>
#include <mutex>
#include <chrono>
#include <stop_token>
#include <thread>

class PeriodicWorker {
public:
    explicit PeriodicWorker(std::chrono::milliseconds interval)
        : interval_(interval)
        , thread_([this](std::stop_token st) { run(st); }) {}

private:
    void run(std::stop_token st) {
        while (!st.stop_requested()) {
            do_work();
            std::unique_lock<std::mutex> lock(mtx_);
            // Pure timed sleep: only wake reasons are timeout or stop. No predicate state.
            // The stop-token overload already handles the stop wakeup independently;
            // []{ return false; } means "no other condition to check — just sleep or stop."
            cv_.wait_for(lock, st, interval_, []{ return false; });
        }
    }

    void do_work() { /* ... */ }

    std::chrono::milliseconds   interval_;
    std::mutex                  mtx_;
    std::condition_variable_any cv_;        // _any: required for stop_token overload
    std::jthread                thread_;    // jthread calls request_stop() on destruct
};
```

Zero polling. Reacts to shutdown within microseconds *while parked in the wait*; a long-running `do_work()` must also be stop-aware to bound total latency. Full `stop_token`/`jthread` coverage in Chapter 9.

---

## Production Snippet 4 — Event Countdown

Wait for N concurrent operations to finish before proceeding.

```cpp
#include <condition_variable>
#include <mutex>
#include <cstddef>
#include <stdexcept>
#include <thread>

class CountdownLatch {
public:
    explicit CountdownLatch(std::size_t n) : count_(n) {
        if (n == 0) throw std::invalid_argument("count must be > 0");
    }

    void count_down() {
        std::lock_guard<std::mutex> lk(mtx_);
        if (count_ == 0) return;    // guard against over-decrement
        if (--count_ == 0) cv_.notify_all();    // under the lock: waiter may destroy us
    }

    void wait() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [&] { return count_ == 0; });
    }

private:
    std::mutex              mtx_;
    std::condition_variable cv_;
    std::size_t             count_;
};

// Usage
CountdownLatch latch(subsystems.size());
for (auto& s : subsystems) {
    std::thread([&s, &latch]{ s.init(); latch.count_down(); }).detach();
}
latch.wait();   // blocks until all subsystems finish init
```

On C++20 you can replace the whole class with `std::latch latch(4);` — see Chapter 8.

---

## Timed Waits

```cpp
#include <chrono>

std::unique_lock<std::mutex> lock(mtx);

// Relative timeout — returns true if pred fired, false if timed out
bool ok = cv.wait_for(lock, std::chrono::milliseconds(500),
                      [&] { return ready; });
// ok == pred() evaluated under the re-acquired lock after wakeup.
// Treat !ok as "predicate not satisfied" — not strictly "timed out" —
// and recheck state if you care which it was.
if (!ok) { /* timed out; ready is still false */ }

// Absolute deadline — same return semantics
auto deadline = std::chrono::steady_clock::now() + std::chrono::seconds(10);
bool ok2 = cv.wait_until(lock, deadline, [&] { return ready; });
```

**Always `steady_clock`, never `system_clock` for timeouts.** System clock can jump backward on NTP sync; your 10-second timeout becomes a 4-hour hang, silently.

```cpp
// broken: system_clock can jump — jumps backward → timeout can stretch far beyond the intended interval;
//         jumps forward → timeout can fire early, before the intended duration has elapsed
auto bad = std::chrono::system_clock::now() + std::chrono::seconds(10);
cv.wait_until(lock, bad, pred);

// fixed: steady_clock is monotonic
auto good = std::chrono::steady_clock::now() + std::chrono::seconds(10);
cv.wait_until(lock, good, pred);
```

---

## Debugging cv Hangs: Snippet Checklist

**Missing notify on error path** — the most common culprit in production hangs:

```cpp
// broken: exception skips notify; consumer waits forever
void produce() {
    std::lock_guard<std::mutex> lk(mtx_);
    result_ = do_work();    // throws? notify never called
    cv_.notify_one();
}

// fixed: set state (even failed state) and notify unconditionally
void produce() {
    try {
        auto val = do_work();
        std::lock_guard<std::mutex> lk(mtx_); result_ = std::move(val);
    } catch (...) {
        std::lock_guard<std::mutex> lk(mtx_); failed_ = true;
    }
    cv_.notify_one();   // always reached — no wait is orphaned (waiter checks result_ || failed_)
}
```

**Wrong mutex on producer side** — state changes invisible to the waiter:

```cpp
// broken: producer uses mtx_b, consumer waits on mtx_a — different locks
void producer() { std::lock_guard<std::mutex> lk(mtx_b); ready = true; }
void consumer() { std::unique_lock<std::mutex> lk(mtx_a);
                  cv.wait(lk, [&]{ return ready; }); }  // loops forever

// fixed: same mutex on both sides + notify the poor waiter
void producer() { { std::lock_guard<std::mutex> lk(mtx_a); ready = true; } cv.notify_one(); }
void consumer() { std::unique_lock<std::mutex> lk(mtx_a);
                  cv.wait(lk, [&]{ return ready; }); }
```

**Predicate capturing a destroyed object** — UB that looks like a hang:

```cpp
// broken: 'this' destroyed while thread still waits → dangling pointer in predicate
Worker::~Worker() { /* does not join thread_ */ }

// fixed: signal + join before members are destroyed
Worker::~Worker() {
    { std::lock_guard<std::mutex> lk(mtx_); shutdown_ = true; }
    cv_.notify_all();
    if (thread_.joinable()) thread_.join();  // thread exits before mtx_/cv_ destruct
}
```

---

## cv vs Sleep-Polling

```cpp
// broken: polling — burns CPU, 1ms reaction latency minimum
std::atomic<bool> ready{false};   // atomic belongs to the polling example only
while (!ready.load(std::memory_order_acquire))
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
// cost: ~0.1% CPU per thread, ≈1ms avg reaction latency

// fixed: cv — zero CPU while waiting, ~1–10µs reaction latency
// ready is a plain bool protected by mtx; producer: { lock_guard lk(mtx); ready=true; } cv.notify_one();
bool ready = false;
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [&] { return ready; });
// cost: 0% CPU while parked, ≈1–10µs wakeup latency
```

100 polling threads at 1ms burn meaningful CPU for zero benefit. Use cv.

---

## Anti-Patterns: The Hall of Shame

| Anti-Pattern | Consequence | Fix |
|---|---|---|
| `cv.wait(lock)` bare, no predicate | Spurious/stolen wakeup → UB or corrupt state | `cv.wait(lock, pred)`, always |
| Modify state outside the lock, then notify | Lost wakeup; consumer sleeps forever | Modify under the same mutex |
| `notify_all` on every item push | Thundering herd; 63 threads wake to do nothing | `notify_one` when one waiter can proceed |
| One cv for two unrelated conditions | `notify_one` wakes wrong waiter type; deadlock | One cv per condition |
| `condition_variable` with `lock_guard` | Compile error (lucky); or wrong lock type → UB | `unique_lock<mutex>` for `cv`; `_any` for other lock types |
| Forgetting `notify_all` on shutdown | Blocked threads hang forever on join | Set close flag under lock, `notify_all` both cvs |
| Waiting on a cv owned by a destroyed object | Undefined behavior | Join all waiting threads before destroying the mutex/cv |
| `notify_while_holding_lock` in hot path | Woken thread stalls — you still hold the mutex | Unlock before notify |

---

## Muscle-Memory Lines

```cpp
// --- The trio ---
std::mutex              mtx;
std::condition_variable cv;
bool                    ready = false;

// --- Wait (predicate form, always) ---
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [&] { return ready; });

// --- Notify (state under lock, notify outside) ---
{ std::lock_guard<std::mutex> lk(mtx); ready = true; }
cv.notify_one();

// --- Shutdown broadcast ---
{ std::lock_guard<std::mutex> lk(mtx); closed_ = true; }
cv.notify_all();

// --- Timed wait (steady_clock) ---
bool fired = cv.wait_for(lock, std::chrono::milliseconds(200), [&] { return ready; });

// --- Interruptible sleep (C++20, cv_any only; st is a std::stop_token in scope) ---
cv_any.wait_for(lock, st, std::chrono::seconds(5), [&] { return ready; });
```

---

> **Three Iron Rules — commit to muscle memory**
>
> 1. `std::unique_lock<std::mutex>` — or `condition_variable_any` for other lock types.
> 2. `cv.wait(lock, pred)` — never the bare form. Never.
> 3. Modify shared state under the mutex, before `notify_*`.
>
> Break any one of these and you own a 2 AM hang that only reproduces under load.

---

## Closing Mantra

*The cv does not remember your notification; your predicate does.*
