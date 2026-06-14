# Chapter 12 — The Pitfalls Catalog

Bookmark this. Laminate it. Every entry below has caused a production outage somewhere. They are sorted by theme, not by how badly they will hurt you — they will all hurt you.

Format per entry: name, one-line cause, `// broken`, `// fixed`. Code is the explanation.

---

## Section 1: Lifetime & Ownership

### Destroying a Joinable `std::thread`

A `std::thread` whose destructor runs while it is still joinable calls `std::terminate`. No stack trace. Just death. Prefer `std::jthread`, which joins in its destructor.

```cpp
// broken
{
    std::thread t([] { do_work(); });
    if (early_exit) return;     // t still joinable → std::terminate
}

// fixed
{
    std::jthread t([] { do_work(); }); // joins automatically on scope exit
}
```

*See Ch. 2.*

---

### Capturing Locals by Reference into a Thread Lambda

The thread outlives the function frame; the captured reference points at destroyed storage.

```cpp
// broken — thread is detached; cfg destroyed when start() returns; thread keeps reading it
void start() {
    std::string cfg = load();
    std::thread t([&cfg] { process(cfg); });
    t.detach(); // cfg gone; thread runs on; use-after-free
}

// fixed — lambda owns its data by value
void start() {
    std::thread([cfg = load()] { process(cfg); }).detach(); // cfg moved into lambda
}
```

*See Ch. 2.*

---

### Member Declaration Order — `jthread` Before Its Dependencies

Members are destroyed in reverse declaration order. `jthread` declared *before* its dependencies is destroyed *last* — it joins after the mutex and queue it uses are already gone.

> **Intuition pump (real-world):** In a destructor, members are dismissed in reverse declaration order — it's last hired, first fired. Put the `jthread` first in the struct and your worker thread is last to be dismissed. The problem: the office furniture (mutex, queue, flags) gets carted out from under the still-running worker first. The worker comes back to its desk and finds an empty room with bare concrete where the mutex used to be. This is use-after-free with extra paperwork. Put the `jthread` *last* in the struct. It gets dismissed *first*. By the time the movers touch anything else, the worker has already clocked out and gone home.

```cpp
// broken — jthread declared FIRST; destroyed LAST; joins after mutex is already destroyed
struct Server {
    std::jthread      m_worker;   // destroyed last → joins against an already-dead mutex
    std::mutex        m_mutex;    // destroyed first
    std::queue<Task>  m_queue;    // destroyed second
};

// fixed — jthread LAST; destroyed FIRST; joins before anything else tears down
struct Server {
    std::mutex        m_mutex;
    std::queue<Task>  m_queue;
    std::jthread      m_worker;   // LAST → destroyed FIRST → joins cleanly
};
```

*See Ch. 3.*

---

### Detaching a Thread That Touches Local State

`detach()` is "fire and forget" only if the thread owns everything it touches. Reference to anything with bounded lifetime → use-after-free.

```cpp
// broken
void fire(RequestData data) {
    std::thread t([&data] { send(data); });
    t.detach(); // data destroyed on return; thread keeps reading it
}

// fixed
void fire(RequestData data) {
    auto sp = std::make_shared<RequestData>(std::move(data));
    std::thread([sp] { send(*sp); }).detach(); // shared_ptr keeps data alive
}
```

---

### Discarding the Future from `std::async`

A discarded `std::future` obtained from `std::async` **blocks in its destructor** until the task finishes. Your "fire and forget" is synchronous.

```cpp
// broken
void handle(Request r) {
    std::async(std::launch::async, [r] { process(r); }); // future discarded → blocks here
}

// fixed
class Handler {
    std::vector<std::future<void>> m_active;
public:
    void handle(Request r) {
        m_active.push_back(std::async(std::launch::async, [r] { process(r); }));
    }
};
```

*See Ch. 6.*

---

### Storing `future<future<T>>` and Forgetting the Inner One

Only futures from `std::async(launch::async, ...)` block in their destructors. If you discard the inner future returned by `get()`, its blocking destructor fires immediately on that line.

```cpp
// broken — inner future discarded; its blocking destructor fires right here
auto outer = std::async(std::launch::async, []() {
    return std::async(std::launch::async, compute); // returns a future<int>
});
outer.get(); // returns inner future<int> as rvalue; immediately destroyed → blocks here

// fixed — keep the inner future alive until you actually need its result
auto outer = std::async(std::launch::async, []() {
    return std::async(std::launch::async, compute);
});
auto inner = outer.get(); // inner future kept alive
int result = inner.get(); // consumed here, when you choose
```

Flatten the chain, or use coroutines. If you see `future<future<T>>`, redesign.

*See Ch. 6.*

---

## Section 2: Mutexes

### Lock-Without-RAII

Manual `lock()`/`unlock()` leaves the mutex locked permanently on any exception or early return added later.

```cpp
// broken
void update() {
    m_mutex.lock();
    if (bad()) { m_mutex.unlock(); return; } // add one more return and forget unlock
    do_work();
    m_mutex.unlock();
}

// fixed
void update() {
    std::lock_guard lk(m_mutex); // unlocks on any exit path, always
    if (bad()) return;
    do_work();
}
```

---

### ABBA Deadlock

Thread A locks mutex 1 then 2. Thread B locks mutex 2 then 1. Under the right scheduling: permanent deadlock.

```cpp
// broken — concurrent transfer(x,y) on T1 and transfer(y,x) on T2:
//   T1 locks x.mtx, then waits on y.mtx
//   T2 locks y.mtx, then waits on x.mtx  → deadlock
void transfer(Account& a, Account& b, int n) {
    std::lock_guard lk1(a.mtx);
    std::lock_guard lk2(b.mtx);
    a.balance -= n; b.balance += n;
}

// fixed
void transfer(Account& a, Account& b, int n) {
    if (&a == &b) return;                    // self-transfer: nothing to do; same mutex twice = UB
    std::scoped_lock lk(a.mtx, b.mtx);      // acquires both atomically, deadlock-free
    a.balance -= n; b.balance += n;
}
```

*See Ch. 4.*

---

### Holding a Lock Across a Callback / I/O / External Call

The callback may block indefinitely or try to acquire the same mutex. Copy, release, then call.

> **Intuition pump (real-world):** You locked the kitchen, walked inside, and ordered DoorDash from inside the kitchen with the kitchen lock still held. The DoorDash driver needs to drop the order at the kitchen counter. You're in the kitchen. The kitchen is locked. The driver stands outside for 45 minutes. Meanwhile, nobody else in the building can cook. This is obvious when described with takeout food. It is somehow not obvious at 11pm when a `std::function<void()>` is passed into a method that already holds `m_mutex`. Release the lock, *then* call the callback.

```cpp
// broken
void dispatch(std::function<void()> cb) {
    std::lock_guard lk(m_mutex);
    m_active = true;
    cb(); // cb might acquire m_mutex; or just stall for 5 seconds
}

// fixed
void dispatch(std::function<void()> cb) {
    { std::lock_guard lk(m_mutex); m_active = true; }
    cb(); // lock released; cb can do whatever it likes
}
```

---

### Recursive Locking on a Non-Recursive Mutex

`std::mutex` is not recursive. Locking it twice on the same thread is undefined behavior — usually an immediate deadlock.

```cpp
// broken
void outer() {
    std::lock_guard lk(m_mutex);
    inner(); // inner() also locks m_mutex → UB / deadlock
}

// fixed — extract an _unlocked helper; caller holds the lock
void outer() {
    std::lock_guard lk(m_mutex);
    inner_locked(); // precondition: m_mutex already held
}
void inner_locked() { /* no lock acquisition */ }
```

---

### Releasing a Lock You Don't Own

`std::unique_lock` is move-only — its copy constructor is `= delete`, so `auto lk2 = lk` is a compile error, not a sneaky runtime surprise. The language enforces single ownership at compile time, which is the good news.

The *real* danger is bypassing RAII entirely: calling `m_mutex.unlock()` directly without actually holding the lock is undefined behavior at the `std::mutex` level — two callers can end up in the critical section simultaneously. (`unique_lock::unlock()` is safer in that it tracks ownership and throws `std::system_error` if you call it unlocked, but getting to that point means you've already lost the thread.)

```cpp
// compile error — unique_lock is not copyable; the compiler enforces single ownership here
std::unique_lock lk(m_mutex);
auto lk2 = lk;  // error: use of deleted function; nothing to debug at runtime

// fixed — transfer ownership explicitly
std::unique_lock lk(m_mutex);
auto lk2 = std::move(lk);  // lk is now empty and unlocked; lk2 holds the mutex

// broken — calling unlock() directly on a mutex this thread does not own is UB
// (precondition violation per [N4861] §32.5.4.2.2/24; two threads can be in the critical
//  section simultaneously — the kind of bug that shows up only under load, at 3 a.m.)
void release_for(std::mutex& m) { m.unlock(); }   // caller never held the lock

// fixed — only the owning RAII guard releases the mutex
{ std::lock_guard lk(m); /* critical section */ } // unlock on scope exit, with ownership
```

---

### `notify_*` While Holding the Lock (Performance)

Not a correctness bug. The woken thread immediately blocks trying to reacquire the mutex the notifier still holds — a wasted context switch.

```cpp
// suboptimal
{ std::lock_guard lk(m_mutex); m_ready = true; m_cv.notify_one(); }

// better — release lock, then notify
{ std::lock_guard lk(m_mutex); m_ready = true; }
m_cv.notify_one(); // woken thread acquires mutex immediately; no stall
```

---

### Locking Inside a Destructor

Acquiring a lock in the destructor does not save you if you never joined the worker. The destructor completes, the object is destroyed, and the still-running worker thread accesses the now-dead members — use-after-free.

```cpp
// broken — worker is running; destructor sets the flag but returns without joining;
//          ~thread() on a joinable thread calls std::terminate
struct MyService {
    std::mutex   m_mutex;
    bool         m_shutdown{false};
    std::thread  m_worker{[this]{
        while (true) {
            { std::lock_guard lk(m_mutex); if (m_shutdown) return; }
            do_work();
        }
    }};
    ~MyService() {
        std::lock_guard lk(m_mutex);
        m_shutdown = true;
        // no join; ~thread() runs while m_worker is joinable → std::terminate
    }
};

// fixed — jthread declared LAST so its destructor runs first and joins
struct MyService {
    std::mutex   m_mutex;
    bool         m_shutdown{false};
    std::jthread m_worker{[this](std::stop_token st){
        while (!st.stop_requested()) {
            { std::lock_guard lk(m_mutex); if (m_shutdown) return; }
            do_work();
        }
    }};
    ~MyService() {
        { std::lock_guard lk(m_mutex); m_shutdown = true; }
        // ~jthread requests stop and joins; worker exits cleanly
    }
};
```

---

## Section 3: Condition Variables

### `wait` Without Predicate

Bare `wait()` returns spuriously. It also misses notifications that arrive before the wait call. The two-argument form is the only correct form.

```cpp
// broken — spurious wakeup processes garbage; early notify causes infinite sleep
m_cv.wait(lk);
process(m_data);

// fixed — predicate handles both spurious wakeup and lost wakeup
m_cv.wait(lk, [this] { return m_ready; });
process(m_data);
```

*See Ch. 5.*

---

### Modifying State Outside the Lock Then Notifying

The waiter checks the condition (false), you set it (no lock), you notify, then the waiter sleeps — missing the notification.

```cpp
// broken — classic lost wakeup
m_ready = true;       // no lock held
m_cv.notify_one();    // waiter may already be past the check and heading to sleep

// fixed — modify under lock, notify after
{ std::lock_guard lk(m_mutex); m_ready = true; }
m_cv.notify_one();
```

*See Ch. 5.*

---

### One CV for Two Conditions

`notify_one()` picks an arbitrary waiter. If consumers and drain-watchers share a CV, they steal each other's signals.

```cpp
// broken — one cv, two conditions
std::condition_variable m_cv; // consumers wait here AND drain-watcher waits here
// notify_one() may wake the wrong one every time

// fixed — one cv per condition
std::condition_variable m_item_ready_cv;
std::condition_variable m_queue_empty_cv;
```

---

### `notify_all` When `notify_one` Suffices

One item added, one consumer can proceed, but every sleeping thread wakes up, contends on the mutex, finds nothing to do, and goes back to sleep. The thundering herd scales quadratically with thread count.

```cpp
// broken — O(threads) context switches per item
{ std::lock_guard lk(m_mutex); m_queue.push(item); }
m_cv.notify_all(); // wakes 127 threads; 126 go immediately back to sleep

// fixed
{ std::lock_guard lk(m_mutex); m_queue.push(item); }
m_cv.notify_one();
```

---

### `sleep_for` in a Worker Loop

`sleep_for` ignores cancellation. Shutdown blocks for the full duration, every time, forever.

```cpp
// broken
while (!m_stop) {
    do_work();
    std::this_thread::sleep_for(5s); // stop_token cannot wake this
}

// fixed — m_cv must be std::condition_variable_any, not std::condition_variable;
// only condition_variable_any has the stop_token wait_for overload
while (!st.stop_requested()) {
    do_work();
    std::unique_lock lk(m_mutex);
    m_cv.wait_for(lk, st, 5s, [] { return false; }); // wakes immediately on stop
}
```

*See Ch. 9.*

---

## Section 4: Atomics & Memory Model

### `volatile` for Synchronization

`volatile` prevents the compiler from eliminating reads/writes to one variable. It provides zero synchronization with respect to other memory. Using it for inter-thread flags is undefined behavior.

> **Intuition pump (real-world):** Putting a "do not autocomplete" sticky note on the whiteboard does not stop two people from walking up and writing on it simultaneously. The note tells the janitor not to erase the board. It tells no one else anything at all. `volatile` is that sticky note: it prevents the *compiler* from optimizing away the read. It has no opinion on two threads racing to write. For that you need `std::atomic` (the whiteboard with a turn-taking protocol) or `std::mutex` (the whiteboard locked in a room with a bouncer). The sticky note is not a bouncer.

```cpp
// broken — UB; no ordering guarantee; optimizer still reorders surrounding stores
volatile bool ready = false;
data = 42; ready = true; // producer
while (!ready) {} use(data); // consumer may see ready=true, data=0

// fixed
std::atomic<bool> ready{false};
data.store(42, relaxed); ready.store(true, release); ready.notify_all(); // producer
ready.wait(false, acquire); use(data.load(relaxed)); // consumer
```

---

### Load-then-Store Instead of Atomic RMW

Two separate atomic operations leave a race window between them. Another thread writes in that gap; one increment disappears.

```cpp
// broken — read-modify-write with a race window in the middle
int old = counter.load();
counter.store(old + 1); // another thread wrote here; your increment is lost

// fixed
counter.fetch_add(1, std::memory_order_relaxed); // single atomic RMW; no race window
```

---

### `relaxed` Everywhere Because "x86 Is seq_cst"

x86's TSO hides missing acquire/release pairs. ARM, POWER, and RISC-V do not. Code that works in CI (x86) can break on customer hardware (ARM).

```cpp
// broken on ARM — relaxed store/load; no ordering; consumer may see flag before data
data.store(42, relaxed); flag.store(true, relaxed);   // producer
if (flag.load(relaxed)) use(data.load(relaxed));       // consumer: data may still be 0

// fixed — release/acquire pair establishes happens-before on all architectures
data.store(42, relaxed); flag.store(true, release);    // producer
if (flag.load(acquire)) use(data.load(relaxed));       // consumer: data is visible
```

*See Ch. 7.*

---

### CAS Failure Order Cannot Be `release`

`compare_exchange_*` takes a success order and a failure order. Using `release` or `acq_rel` as the failure order is undefined behavior — a failed CAS wrote nothing, so release semantics are meaningless. Not all implementations diagnose this; some silently downgrade.

```cpp
// broken — UB; failure order may not be release or acq_rel
val.compare_exchange_strong(exp, des,
    std::memory_order_acq_rel,
    std::memory_order_release); // ← undefined behavior

// fixed — failure order must be ≤ success order, and never release/acq_rel
val.compare_exchange_strong(exp, des,
    std::memory_order_acq_rel,
    std::memory_order_acquire); // acquire on failure is correct
```

---

### Treating `compare_exchange_weak` Spurious Failure as Meaningful

`compare_exchange_weak` may fail even when `*this == expected`. That is not the other thread changing the value — that is hardware load-linked/store-conditional noise. If your logic branches on failure assuming contention, it will misfire.

```cpp
// broken — spurious failure triggers contention-handling logic incorrectly
if (!val.compare_exchange_weak(exp, des))
    handle_contention(); // called on spurious failure; wrong

// fixed — one-shot conditional transition: use strong; no retry loop needed
if (!val.compare_exchange_strong(exp, des))
    { /* exp now holds current value; transition was not applicable */ }

// if a retry loop IS needed, recompute des from the updated exp on each failure;
// blindly retrying with the original des stores a stale value over any state:
// auto exp = val.load();
// while (should_transition(exp)) {
//     auto des = compute_desired(exp);           // derived from current exp
//     if (val.compare_exchange_weak(exp, des))   // exp updated on failure
//         break;
// }
```

---

### Busy-Wait Spin on an Atomic

`while (!done) {}` burns 100% of a CPU core. On a VM it may prevent the flag-setting thread from being scheduled — the spin defeats itself.

```cpp
// broken
while (!done.load(std::memory_order_acquire)) {} // 100% CPU; may starve the setter

// fixed — C++20 atomic wait; OS-efficient blocking (futex / WaitOnAddress)
// producer must pair the store with notify: done.store(true, release); done.notify_one();
done.wait(false, std::memory_order_acquire);
```

---

### Assuming `atomic<T>` Is Lock-Free for Arbitrary `T`

Large types use a hidden internal lock. Your "lock-free" structure is not.

```cpp
// broken assumption — may use internal locks on platforms without 128-bit CAS
std::atomic<std::pair<uint64_t, uint64_t>> state;

// verify at compile time
static_assert(std::atomic<uint64_t>::is_always_lock_free);
// if is_always_lock_free is false, encode your state in a single machine word
```

---

## Section 5: Lock-Free Patterns

### ABA in a CAS Loop

Thread reads head=A. Another thread pops A, frees it, allocates a new node that gets address A, pushes it. First thread's CAS succeeds — but A is now a completely different node. Linked-list corruption follows.

```cpp
// broken — CAS sees address A; does not know it's a different A
Node* old_head = head.load();
// ...thread 2 pops old_head, frees it, pushes new node at same address...
head.compare_exchange_weak(old_head, old_head->next); // succeeds; wrong node unlinked

// fixed — tagged pointer: version counter in high bits prevents false match
struct TaggedPtr { Node* ptr; uintptr_t tag; };
std::atomic<TaggedPtr> head;
// CAS fails if tag differs, even if ptr address matches

// NOTE: a tagged pointer prevents the CAS from accepting a recycled address, but it does
// NOT make old_head->next safe to read after another thread may have popped and freed
// old_head. Pair tagged pointers with a safe reclamation scheme — hazard pointers, epoch-
// based reclamation (RCU), or reference counting — before dereferencing any removed node.
```

*See Ch. 10.*

---

### Hand-Rolled DCLP Without Fences

The pointer becoming non-null can be observed before the constructor finishes on weakly-ordered CPUs.

```cpp
// broken — no release/acquire; pointer visible before object is constructed
Singleton* g_p = nullptr;
if (!g_p) { std::lock_guard lk(g_m); if (!g_p) g_p = new Singleton(); }
return *g_p; // may dereference partially-constructed object on ARM

// fixed — function-local static; C++11 guarantees thread-safe init with correct fences
Singleton& get() { static Singleton inst; return inst; }
```

---

### SPSC Ring Buffer Used With Multiple Producers

SPSC algorithms are correct only with exactly one producer. Two producers both read the same write index, both write to the same slot. One write is silently lost.

```cpp
// broken — two threads calling push() on an SPSC queue; silent data loss
std::thread p1([&] { for (auto x : batch1) q.push(x); });
std::thread p2([&] { for (auto x : batch2) q.push(x); }); // SPSC; data race

// fixed — serialize producers with a mutex; or use a true MPSC queue
std::mutex push_mutex;
std::thread p1([&] { for (auto x : batch1) { std::lock_guard lk(push_mutex); q.push(x); } });
std::thread p2([&] { for (auto x : batch2) { std::lock_guard lk(push_mutex); q.push(x); } });
```

---

### Padding Without Alignment

`sizeof(T) == 64` is not enough. The hardcoded `char pad[60]` encodes an assumption: cache lines are 64 bytes. On x86 that is usually true. On Apple Silicon (M-series) and POWER9+, cache lines are 128 bytes — so two elements 64 bytes apart land on *the same* 128-byte cache line and trigger genuine false sharing. The padding did its job for one architecture and betrayed you on another.

`alignas(std::hardware_destructive_interference_size)` fixes both problems at once: it aligns each element to the target's actual destructive-interference size *and* implicitly rounds `sizeof(T)` up to match, so the struct self-describes its own stride on every supported platform.

```cpp
// broken — sizeof=64 but hardcoded for 64-byte cache lines;
// on 128-byte-line CPUs (Apple M-series, POWER9+) adjacent elements
// share a cache line and false-share despite the padding
struct Counter { std::atomic<int> val; char pad[60]; }; // alignof == 4, not cache-line size

// fixed — alignas adapts to the target architecture's actual cache-line size
struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<int> val;
};
static_assert(alignof(Counter) >= std::hardware_destructive_interference_size);
```

---

### False Sharing Between Two Independent Atomics

Two unrelated atomics declared adjacent share a cache line. Every write by one core invalidates the other — serializing logically independent operations.

```cpp
// broken — a and b share a cache line; writes to a stall readers of b
std::atomic<int> a{0};
std::atomic<int> b{0}; // adjacent in memory; same cache line

// fixed
struct alignas(std::hardware_destructive_interference_size) Padded {
    std::atomic<int> val{};
};
Padded a, b; // each on its own cache line
```

*See Ch. 8.*

---

### Missing Consumer-Side `acquire` to Pair a Producer-Side `release`

`release` on the producer store creates a happens-before edge only when the consumer loads with `acquire`. `relaxed` on the consumer side breaks the guarantee — invisible on x86, wrong on ARM.

```cpp
// broken — relaxed load; no happens-before; data may be stale on ARM
data = 42;
flag.store(true, std::memory_order_release);    // producer
if (flag.load(std::memory_order_relaxed))        // consumer: relaxed; synchronization broken
    use(data);

// fixed
if (flag.load(std::memory_order_acquire))        // acquire pairs with the release store
    use(data);                                   // data is now visible; happens-before established
```

---

## Section 6: Futures & Promises

### `std::async` Without `std::launch::async`

The default policy allows deferred execution — the task runs on the calling thread when `get()` is called. Your "background task" is synchronous. This can also deadlock if the calling thread is waiting for the deferred task to complete while the task waits for the same thread.

```cpp
// broken — may run deferred; blocks the caller on get()
auto f = std::async(task); // implementation may choose std::launch::deferred

// fixed
auto f = std::async(std::launch::async, task); // guaranteed separate thread
```

---

### Calling `get()` Twice on the Same `future`

`get()` is destructive. The second call throws `std::future_error::no_state`. Store the result; call `get()` once.

```cpp
// broken
auto f = std::async(std::launch::async, compute);
auto a = f.get(); // ok
auto b = f.get(); // throws future_error::no_state

// fixed — call get() once; use shared_future for multiple consumers
auto sf = std::async(std::launch::async, compute).share();
auto a = sf.get(); auto b = sf.get(); // shared_future: repeated get() is fine
```

---

### `set_value` Twice on the Same `promise`

A `promise` can be satisfied only once. A second `set_value` throws `future_error::promise_already_satisfied`. Ensure exactly one code path fulfills the promise.

```cpp
// broken — both success path and cleanup block call set_value
p.set_value(result);
p.set_value(result); // throws on second call

// fixed — atomic guard ensures exactly one fulfillment
if (!m_fulfilled.exchange(true)) p.set_value(result);
```

---

### Letting a `promise` Destruct Without Setting It

Destroying a promise without calling `set_value` or `set_exception` stores `broken_promise` in the shared state. The caller's `get()` throws an opaque internal exception instead of a meaningful error.

```cpp
// broken — exception in do_work() destroys p; caller gets broken_promise
void worker(std::promise<int> p) {
    p.set_value(do_work()); // throws; p destroyed; caller gets future_error
}

// fixed — always satisfy the promise
void worker(std::promise<int> p) {
    try { p.set_value(do_work()); }
    catch (...) { p.set_exception(std::current_exception()); }
}
```

---

### Submitting a `packaged_task` Without `std::move`

`std::packaged_task` is move-only. A by-value sink parameter is a valid pattern for a move-only type, but an lvalue caller must use `std::move` — passing the lvalue directly triggers a copy, which is deleted.

```cpp
// broken — lvalue passed without std::move; copy attempted; compile error
void submit(std::packaged_task<int()> task) { ... }
submit(my_task); // error: call to deleted copy constructor

// fixed — transfer ownership explicitly
submit(std::move(my_task)); // move the task; my_task is now empty
```

---

### `future<T&>` With a Short-Lived Referent

`future<T&>::get()` returns a reference. The referent's lifetime is not managed by the future. If it goes out of scope before `get()`, the reference dangles.

```cpp
// broken — local destroyed; future holds a dangling reference
std::promise<int&> p;
{ int local = 42; p.set_value(local); } // local destroyed here
int& ref = p.get_future().get();        // ref is dangling; UB on access

// fixed — store by value; future<int> owns the result
std::promise<int> p;
{ int local = 42; p.set_value(local); } // value copied into shared state
int val = p.get_future().get();         // val is a safe copy
```

---

## Section 7: Cancellation

### `sleep_for` Instead of CV-With-Stop-Token

Already in Section 3. One more time because engineers keep doing it: `sleep_for` is uninterruptible. Use `condition_variable_any::wait_for` with a `stop_token`.

```cpp
// broken
std::this_thread::sleep_for(10s); // stop_token does nothing here

// fixed
std::condition_variable_any cv;
std::unique_lock lk(m);
cv.wait_for(lk, stop_token, 10s, [] { return false; });
```

---

### Not Propagating `stop_token` Through the Call Chain

The stop token stops at the first level. Every sub-call that doesn't receive it runs to completion. Shutdown latency = sum of all un-cancelled sub-operations.

```cpp
// broken — stop_token not forwarded; sub-tasks run forever
void top(std::stop_token st) {
    while (!st.stop_requested())
        process_batch(); // no stop_token; ignores cancellation
}

// fixed — thread the stop_token through every level
void top(std::stop_token st) {
    while (!st.stop_requested())
        process_batch(st);
}
void process_batch(std::stop_token st) {
    for (auto& item : items()) {
        if (st.stop_requested()) return;
        handle(item, st);
    }
}
```

---

### Swallowing Exceptions in Workers

An empty `catch(...)` swallows the real exception. The promise destructs unsatisfied, storing `broken_promise` in the shared state. The caller's `get()` throws `std::future_error` immediately — it does not block forever, but it gets an opaque internal error instead of the original exception.

```cpp
// broken — exception swallowed; p destructs unsatisfied; caller gets broken_promise
// (future::get() throws std::future_error, not the original exception)
void worker(std::promise<Result> p) {
    try { p.set_value(do_work()); }
    catch (...) {} // p goes out of scope; broken_promise stored; original exception lost
}

// fixed — propagate the exception to the caller through the promise
void worker(std::promise<Result> p) {
    try { p.set_value(do_work()); }
    catch (...) { p.set_exception(std::current_exception()); }
}
```

## Section 8: Thread-Local & False Sharing

### `thread_local std::map<>` on a Hot Path

Every thread that first touches the variable gets its own heap-allocated `std::map`. 64 threads → 64 separate map constructions at first access, each with a latency spike.

```cpp
// broken — expensive per-thread heap allocation on every new thread's first call
int lookup(int k) {
    thread_local std::map<int, int> cache; // full std::map constructed per thread
    auto it = cache.find(k);
    return it != cache.end() ? it->second : (cache[k] = compute(k));
}

// better — thread_local unordered_map is still per-thread but cheaper to construct
int lookup(int k) {
    thread_local std::unordered_map<int, int> cache;
    auto [it, inserted] = cache.emplace(k, 0);
    if (inserted) it->second = compute(k);
    return it->second;
}
```

---

### `thread_local` Referenced from a Global Destructor

On most implementations, `thread_local` storage for the main thread is destroyed before global-static destructors run. A global destructor that reads `thread_local` data touches memory that is already gone.

```cpp
// broken — t_buf is thread_local; may be destroyed before ~Logger() runs
thread_local std::string t_buf;
struct Logger {
    ~Logger() { flush(t_buf); } // t_buf lifetime may have already ended; UB
} g_logger;

// fixed — flush before thread_local storage is torn down; destructor is a no-op
thread_local std::string t_buf;
struct Logger {
    void flush_current_thread() { flush(t_buf); } // call this before thread exits
    ~Logger() {} // nothing to do; threads flushed their own buffers during shutdown
} g_logger;
```

---

### Per-Thread Atomics Without `alignas` → False Sharing

Per-thread counters in a plain array sit in adjacent cache lines. Writes by thread N+1 invalidate the cache entry for thread N. The per-thread design is correct; the layout defeats it.

```cpp
// broken — tightly packed; adjacent entries share a cache line
std::atomic<int> counters[kMaxThreads]; // false sharing across threads

// fixed
struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<int> val{};
};
Counter counters[kMaxThreads]; // each Counter owns a full cache line
```

---

### Padding `std::vector<T>` Elements and Expecting Alignment

`std::vector` aligns its heap allocation to `alignof(T)`. Without `alignas`, `alignof(T)` is 4 — the vector base has no cache-line alignment guarantee. Worse, the hardcoded `char pad[60]` bakes in the assumption that cache lines are 64 bytes. On 128-byte-line CPUs (Apple Silicon, POWER9+), two elements that are 64 bytes apart occupy the *same* cache line — genuine false sharing, no matter how much padding you added.

```cpp
// broken — sizeof=64, alignof=4; hardcoded for 64-byte cache lines only;
// on 128-byte-line CPUs adjacent elements share a cache line
struct Padded { std::atomic<int> val; char pad[60]; }; // alignof == 4
std::vector<Padded> counters(N);  // base alignment == 4; padding assumes 64-byte lines

// fixed — alignas propagates into vector: allocator uses alignof(T) for the heap block
struct alignas(std::hardware_destructive_interference_size) Padded {
    std::atomic<int> val;
};
static_assert(sizeof(Padded) == std::hardware_destructive_interference_size);
std::vector<Padded> counters(N); // each element on its own cache line, any architecture
```

---

## Section 9: Design-Level Smells

### Recursive Mutex "For Convenience"

A `std::recursive_mutex` allows re-entry, which means a function can acquire the lock while the object's invariants are broken (a caller is mid-modification). The re-entrant code sees garbage state.

```cpp
// smell — recursive_mutex papers over a design problem
std::recursive_mutex m_mutex; // why does this need re-entrancy?

// fix — extract _locked variant; document the precondition
void outer()         { std::lock_guard lk(m_mutex); inner_locked(); }
void inner_locked()  { /* precondition: m_mutex held by caller */ }
```

---

### Lock Every Method Without Thinking About Granularity

Technically thread-safe. Practically single-threaded. All parallelism serialized on one lock.

```cpp
// smell
int  read()  { std::lock_guard lk(m_mutex); return m_data; }
void write(int x) { std::lock_guard lk(m_mutex); m_data = x; }
// every operation serialized; adding threads makes it slower

// consider — readers don't need to exclude each other
std::shared_mutex m_rwmutex;
int  read()       { std::shared_lock lk(m_rwmutex); return m_data; }
void write(int x) { std::unique_lock lk(m_rwmutex); m_data = x; }
```

---

### One Giant Global Mutex

```cpp
// smell — entire subsystem serializes on one lock
std::mutex g_lock;
void anything() { std::lock_guard lk(g_lock); ... }
```

Performance collapses at scale. Partition state by ownership. Use per-shard locks. Design most operations to require no lock at all.

---

### `shared_mutex` Cargo-Cult

`shared_mutex` is only beneficial when reads are long relative to lock-acquisition cost and writes are rare. For a single integer read, the bookkeeping costs more than a plain mutex. Under write pressure, reader-preferring implementations starve writers.

```cpp
// smell — shared_mutex on a trivially-short critical section
std::shared_mutex m;
int get() { std::shared_lock lk(m); return m_x; } // overhead > benefit for int read
```

Measure first. Plain `std::mutex` is often faster for small critical sections.

---

### Thread Per Request

```cpp
// smell
void on_request(Request r) {
    std::thread([r] { handle(r); }).detach(); // one OS thread per request; collapses at scale
}

// fix — submit to a bounded thread pool
void on_request(Request r) { g_pool.submit([r] { handle(r); }); }
```

Default stack per thread: 1–8 MB. At 10,000 requests: up to 80 GB virtual address space reserved. The OS scheduler cries.

---

### Inheriting from `std::thread`

`std::thread` has no virtual destructor, no virtual methods, and is not designed as a base class. Compose instead.

```cpp
// smell
struct NamedThread : std::thread { std::string name; }; // no virtual dtor; slicing; wrong

// fixed — contain a jthread as a member
struct NamedThread { std::string name; std::jthread thread; };
```

---

### Spinlocks in User-Space

You do not own the scheduler. The lock holder can be preempted at any point. Every waiting thread spins at 100% CPU until the holder is rescheduled — which may take milliseconds on a loaded system.

> **Intuition pump (real-world):** You set up camp at the post office counter at 8am and refuse to leave even when the clerk goes on their lunch break. You're not "optimizing for low latency when the clerk returns." You're standing at an empty counter burning your own time — and your body is physically blocking everyone else from even approaching — while the clerk is in the break room eating a sandwich and watching their phone. That's a user-space spinlock: you are consuming 100% of a CPU core, preempting no one, making no progress, while the OS scheduler weeps and the lock-holder finishes their sandwich at their own pace.

```cpp
// smell — user-space spinlock
while (flag.exchange(true, acquire)) {} // spinning; lock holder may be preempted right now
flag.store(false, release);

// fix — std::mutex spins briefly then yields to the OS; use it
std::lock_guard lk(m_mutex);
```

---

## Section 10: Diagnostic & Debugging Traps

### "Works in Debug, Races in Release"

The optimizer widened your race window. `volatile` does not fix this. Neither does `-O0` in production.

```cpp
// the optimizer found your race:
// Debug:   load X, compute, store X (slow; race window < scheduling quantum)
// Release: load X, [reorder], store X (fast; race window > scheduling quantum)
// fix: correct synchronization, not a build-type workaround
```

*See Ch. 7.*

---

### "Works on x86, Breaks on ARM"

TSO has been lying to you. x86's strong memory model hides missing acquire/release pairs that ARM exposes. Your CI passing on x86 is not evidence of correctness.

```cpp
// "works" on x86 (TSO orders stores for you); fails on ARM (weak model)
data = 42;
ready.store(true, std::memory_order_relaxed); // missing release; ARM sees stale data

// fix: write to the C++ memory model, not to x86's TSO accident
ready.store(true, std::memory_order_release);
```

*See Ch. 7.*

---

### "TSan Reports a Race but It Never Fails Without TSan"

TSan is right. A narrow race that never triggers in testing is a wide race waiting for production load, a different compiler, or a different CPU.

```cpp
// TSan report: data race on m_cache at foo.cpp:42
// "but it works fine!" — narrow window; not zero; fix it
// TSan has ~0% false positive rate on well-formed programs
```

---

### Adding `printf` Makes the Bug Disappear

`printf` acquires an internal lock and makes a syscall. This changes scheduling timing and introduces implicit barriers. The race is still there; the `printf` is an accidental band-aid.

```cpp
// broken investigation
printf("DEBUG\n"); // hides the race by changing timing
// removing it brings the race back

// correct investigation: use TSan; reproduce under load; fix the synchronization
```

---

### Flaky Test Marked `FLAKY` and Forgotten

A concurrent test that fails 5% of the time has a race that fails 5% of the time. At production scale with real load it fails 30% of the time. A flaky concurrent test is a production incident waiting to ship.

```cpp
// wrong response
// FLAKY: sometimes fails under load — TODO: investigate
TEST(MyTest, ConcurrentAccess) { ... }

// correct response: reproduce with increased threads; run under TSan; fix the race
// do NOT mark concurrent tests FLAKY and move on
```

---

## How to Develop the Instinct

- **TSan in CI on every PR.** Not occasionally. Every PR. TSan catches ~95% of the races in this catalog when code paths are exercised. No other single tool comes close.
- **Ask "where's the happens-before?" in every review.** When you see a shared mutable variable, demand a concrete answer: "the release store in `publish()` happens-before the acquire load in `consume()`." If neither the author nor you can name it, there is no happens-before.
- **Code-review checklist:** (1) Is every shared variable protected by a named mechanism — mutex, atomic, or immutable-after-construction? (2) Are all `std::thread` members declared last in the class? (3) Does every `wait()` call have a predicate? (4) Are acquire/release pairs matched and justified? (5) Is the stop token threaded through every cancellable operation?
- **Reproduce under load before closing any bug.** Run concurrent tests with 10× the thread count, under TSan, in a tight loop for 30 seconds. A race that does not reproduce under load is a race you have not yet reproduced, not a race that does not exist.

---

*If you cannot name the happens-before, there is no happens-before.*
