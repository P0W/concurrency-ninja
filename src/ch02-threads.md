# Chapter 2 — std::thread & std::jthread

> *All `#include` directives shown in first snippet of each section. Later snippets note `// includes elided` when headers are unchanged.*

---

## 2.1 Cold Open: std::terminate Enters the Chat

```cpp
#include <thread>
#include <stdexcept>

class DataLoader {
public:
    DataLoader()
        : m_thread([this] { loadData(); })   // thread is live
    {
        initResources();                      // throws std::runtime_error
        // ^ unwinds constructor
        // ^ m_thread destructor runs
        // ^ m_thread is joinable
        // ^ §[thread.thread.destr]: "calls std::terminate"
        // ^ your process is gone. happy friday.
    }
private:
    void loadData()    { /* runs concurrently, uses this */ }
    void initResources() { throw std::runtime_error("bad config"); }
    std::thread m_thread;
};
```

Don't launch threads in constructors that can throw. Don't. That's the whole lesson.

There are two hazards, not one. The obvious one: if `initResources()` throws, `m_thread`'s destructor runs on a joinable thread and calls `std::terminate`. The subtler one: `loadData()` starts executing *before* the constructor body runs, so it races with `initResources()` over any state that `initResources()` is supposed to set up. A `start()` method called after full construction eliminates both.

> **Pitfall:** If anything after `std::thread` construction can throw, you will call `std::terminate`. Declare `std::jthread` last among all members, and assign it only after every throwing operation and every invariant is satisfied — typically as the final statement of the constructor body. A separate `start()` method called after full construction is the safest pattern.

---

## 2.2 Two States, One Rule

A thread object is either **joinable** (a live OS thread is attached) or **not joinable** (default-constructed, moved-from, joined, or detached). That is the entire state machine.

```cpp
#include <thread>
#include <cassert>

void noop() {}

int main() {
    std::thread t;
    assert(!t.joinable());    // default: not joinable

    t = std::thread(noop);
    assert(t.joinable());     // live thread: joinable

    t.join();
    assert(!t.joinable());    // joined: not joinable

    std::thread t2(noop);
    t2.detach();
    assert(!t2.joinable());   // detached: not joinable (and now untracked)
}
```

Destructor of a joinable `std::thread` → `std::terminate`. The committee had three options: silent detach (use-after-free), silent join (deadlock during unwind), or terminate. Terminate is the loudest and therefore the most honest.

> **Intuition pump (real-world):** You hired a contractor to renovate the kitchen. Job's not done. You decide to just… walk out of the building without paying them, firing them, or even acknowledging their existence. You make eye contact with security on the way out. Security calls the cops. The cops are `std::terminate`. Your process is gone. The contractor is also gone. Nobody got closure. Don't walk out on joinable threads.

---

## 2.3 std::thread: The Full API

```cpp
#include <thread>
#include <string>
#include <iostream>

void work(int id) {
    std::cout << "thread " << std::this_thread::get_id() << " id=" << id << "\n";
}

int main() {
    // Construction: callable + decay-copied args
    std::thread t(work, 42);

    (void)t.joinable();        // true — live thread attached
    (void)t.get_id();          // std::thread::id of the OS thread
    (void)t.native_handle();   // pthread_t / HANDLE — use for platform APIs only

    t.join();            // block until thread exits; t is now not-joinable
    // t.join();         // would throw std::system_error — already joined

    std::thread t2(work, 99);
    t2.detach();         // release ownership; OS cleans up; you never hear from it again
    // t2.join();        // std::system_error — not joinable after detach

    // Move-only: transfer ownership
    std::thread t3(work, 1);
    std::thread t4 = std::move(t3);   // t3 is now not-joinable
    t4.join();

    // hardware hint (can return 0 — see §2.11)
    unsigned int cpus = std::thread::hardware_concurrency();
    (void)cpus;
}
```

### Member functions: pointer-to-member syntax

```cpp
// includes elided
struct Processor {
    void run()           { /* ... */ }
    void process(int n)  { /* ... */ }
};

Processor p;
std::thread t1(&Processor::run,     &p);        // no args
std::thread t2(&Processor::process, &p, 42);    // with args
t1.join(); t2.join();
// &p is a raw pointer. If p dies before t1/t2 finish: use-after-free.
```

---

## 2.4 Argument Passing: Copy, Move, Ref

Everything passed to `std::thread` is **decay-copied** into internal storage. The thread cannot hold dangling stack references by accident — unless you explicitly ask it to with `std::ref`.

```cpp
#include <thread>
#include <functional>
#include <string>
#include <utility>

void byCopy (std::string  s) { /* owns a copy  */ }
void byRef  (std::string& s) { s += "_done";       }  // modifies caller's string

int main() {
    std::string data = "payload";

    std::jthread t1(byCopy, data);               // COPY: data can die, thread is safe
    std::jthread t2(byCopy, std::move(data));     // MOVE: data is gutted after this line
    // data is now in a valid-but-unspecified state

    std::string data2 = "mutable";
    std::jthread t3(byRef, std::ref(data2));      // REF: thread holds pointer to data2
    // data2 MUST outlive t3. jthread destructor joins, so here it's fine.
    // Move t3 out of this scope and you have a dangling reference.
}
```

> **Pitfall:** `std::ref` opts you out of the decay-copy safety net. The compiler will not warn you when the referent dies. The runtime will not stop you. You get a use-after-free and a stack trace you cannot explain on a Monday morning.

> **Intuition pump (real-world):** Passing `std::ref(x)` to a thread is like handing a copy of your house keys to a contractor and saying "let yourself in whenever." You then go on holiday. Sometimes you get home and the fridge is repainted. Sometimes you get home and the house is on fire. Occasionally you don't come home to a house at all — the contractor walked in after you'd sold it. The lifetime is YOUR problem. The compiler's hands are clean.

---

## 2.5 std::jthread — What std::thread Should Have Been

> **Intuition pump (real-world):** `std::thread` is the old contractor arrangement — you hire them, and if anything goes wrong mid-job, you have to personally track them down, file the paperwork, and formally terminate the engagement or your building's insurance is void. `std::jthread` is the upgraded HR contract: it has "request stop" built in, it auto-files all the termination paperwork on the way out, and the destructor handles the handshake whether you remembered to or not. It is the HR upgrade you always wanted and never got until C++20. Use it. Always. The only reason to reach for `std::thread` in new code is "I need to ship to C++17" — which is fine, but be honest with yourself about it.

Same header (`<thread>`), C++20. Two additions that matter enormously: **auto-join in the destructor** and **built-in stop token**.

### Side-by-side: manual vs automatic

```cpp
#include <thread>

void work() { /* ... */ }

void demo() {
    // --- std::thread: you manage everything ---
    {
        std::thread t(work);
        // ... if an exception fires here, t's destructor calls std::terminate
        t.join();   // must not forget this
    }

    // --- std::jthread: destructor joins for you ---
    {
        std::jthread t(work);
        // exception here? destructor still joins. no terminate.
    }   // <- join() called automatically
}
```

### Stop token: injected automatically when callable is invocable with std::stop_token first

```cpp
#include <thread>
#include <stop_token>
#include <chrono>

std::jthread t([](std::stop_token st, int n) {
    for (int i = 0; i < n && !st.stop_requested(); ++i) {
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}, 100);
// Only 'n=100' passed at construction — stop_token is injected by jthread.

t.request_stop();   // signal from outside; stop_requested() becomes true
// destructor: request_stop() (idempotent) then join()
```

All `stop_token`, `stop_source`, and `stop_callback` member functions on the same stop state are thread-safe by spec ([stoptoken.intro]); no external mutex is required around `request_stop()` ↔ `stop_requested()`.

### Comparison table

| Feature | `std::thread` | `std::jthread` |
|---|---|---|
| Standard | C++11 | C++20 |
| Header | `<thread>` | `<thread>` |
| Destructor while joinable | `std::terminate` | `request_stop()` + `join()` |
| Built-in cancellation | No | `std::stop_source` / `std::stop_token` |
| Auto stop-token injection | N/A | Yes — prepended when callable is invocable with `std::stop_token` as first argument |
| `request_stop()` | N/A | Yes |
| `get_stop_source()` | N/A | Yes |
| Move-only | Yes | Yes |
| `joinable()`, `join()`, `detach()` | Yes | Yes |
| `get_id()`, `native_handle()` | Yes | Yes |
| **Default choice** | Only for C++17 compat | **Yes, always** |

---

## 2.6 std::this_thread

```cpp
#include <thread>
#include <chrono>
#include <iostream>

void threadSelf() {
    std::this_thread::get_id();     // this thread's id

    std::this_thread::yield();      // hint to scheduler: run someone else
                                    // no guarantees; on an idle core: no-op

    std::this_thread::sleep_for(std::chrono::milliseconds(50));

    auto t = std::chrono::steady_clock::now() + std::chrono::seconds(1);
    std::this_thread::sleep_until(t);
}
```

### The sleep_for cancellation trap

```cpp
#include <thread>
#include <stop_token>
#include <chrono>

void doWork() {}   // stub

// BROKEN: sleep_for ignores stop tokens
void badWorker(std::stop_token st) {
    while (!st.stop_requested()) {
        doWork();
        std::this_thread::sleep_for(std::chrono::seconds(30)); // uninterruptible
    }
    // shutdown latency: 0–30s. uniformly distributed. enjoy your SLA breach.
}

// FIXED: condition_variable_any::wait_for wakes on stop OR timeout
#include <condition_variable>
#include <mutex>

void goodWorker(std::stop_token st) {
    std::mutex mtx;
    std::condition_variable_any cv;
    std::unique_lock lock(mtx);
    while (!st.stop_requested()) {
        lock.unlock();
        doWork();
        lock.lock();
        // mtx is ceremonial here — condition_variable_any::wait_for requires a Lock& by
        // signature; the stop-token overload's interrupt machinery does the real work.
        // Nothing is protected by mtx; it exists to satisfy the API contract.
        cv.wait_for(lock, st, std::chrono::seconds(30), []{ return false; });
        // ^ returns immediately when stop is requested. shutdown in <1ms.
    }
}
```

> **Pro tip:** `sleep_for` in a cancellation-sensitive loop is always wrong. Use `condition_variable_any::wait_for` with a stop token (Ch.4) or `semaphore::try_acquire_for` (Ch.8).

---

## 2.7 Member Declaration Order: The Shutdown Timebomb

Members are destroyed in **reverse declaration order**. Your thread uses other members. If those members die first, you have a use-after-free.

```cpp
#include <thread>
#include <vector>
#include <atomic>
#include <stop_token>

static void process(std::vector<int>&) {}   // stub

// BROKEN: m_thread declared first → destroyed LAST
//         m_data declared last  → destroyed FIRST
//         thread can still touch m_data after it is destroyed → use-after-free
class Broken {
    std::jthread            m_thread;   // destroyed last  (declared first)
    std::vector<int>        m_data;     // destroyed first (declared last)
public:
    Broken() {
        m_thread = std::jthread([this](std::stop_token st) {
            while (!st.stop_requested()) { process(m_data); }
        });
    }
    // Destruction: m_data destroyed first, THEN m_thread requests stop + joins.
    // Thread may be mid-process(m_data) when m_data is already gone.
};

// FIXED: m_thread declared last → destroyed FIRST
//        m_data still alive when destructor requests stop and joins
class Fixed {
    std::vector<int>        m_data;     // destroyed last  (declared first)
    std::jthread            m_thread;   // destroyed first (declared last)
public:
    Fixed() {
        m_thread = std::jthread([this](std::stop_token st) {
            while (!st.stop_requested()) { process(m_data); }
        });
    }
    // Destruction: m_thread requests stop → thread exits → m_data destroyed. Clean.
};
```

**Takeaway:** `std::jthread` last. Always. Non-negotiable.

> **Intuition pump (real-world):** In a class destructor, members die in *reverse* declaration order — last declared, first destroyed. Declare `m_jthread` first (as a naive "put the important stuff at the top" instinct suggests), and when the destructor runs, your worker is still alive and thrashing while `m_data`, `m_queue`, `m_config` — all the things it's trying to touch — are already packing their boxes and leaving. The thread is reading a ghost. Declare `m_jthread` last. It dies first. Everything it depends on is still in the building. Then the rest of the class follows it out in an orderly fashion. Reverse declaration order is not a quirk; it is the rule, and if you fight it you lose.

---

## 2.8 Exceptions From Threads

Uncaught exception in a thread body → `std::terminate`. Your `main`'s `catch(...)` cannot help you.

```cpp
#include <thread>
#include <exception>
#include <stdexcept>
#include <iostream>

void doWork() { throw std::runtime_error("oops"); }   // stub that throws

int main() {
    // BROKEN: exception escapes → std::terminate (commented out — don't run)
    // std::jthread bad([]{ throw std::runtime_error("bye"); });

    // FIXED option 1: catch at the top of the thread
    std::jthread safe([] {
        try { doWork(); }
        catch (const std::exception& e) { std::cerr << "thread: " << e.what() << "\n"; }
        catch (...) { std::cerr << "thread: unknown exception\n"; }
    });

    // FIXED option 2: transport the exception to the caller via exception_ptr
    std::exception_ptr eptr;
    {
        std::jthread transporting([&eptr] {
            try { doWork(); }
            catch (...) { eptr = std::current_exception(); }
        });
    }   // jthread destructor joins here — safe to read eptr after this point

    if (eptr) std::rethrow_exception(eptr);
    // For the full promise/future version, see Ch.7.
}
```

---

## 2.9 Production Snippet — HeartbeatMonitor

The right way to build a self-managing periodic worker. One snippet. No wrong version — the note is enough.

> **Note:** The wrong version uses `sleep_for` in the loop and `std::atomic<bool>` for cancellation. Shutdown latency: up to the sleep duration. Your deployment pipeline will hate it. Don't.

```cpp
#include <thread>
#include <stop_token>
#include <condition_variable>
#include <mutex>
#include <chrono>
#include <string>
#include <utility>
#include <iostream>

class HeartbeatMonitor {
public:
    explicit HeartbeatMonitor(std::string endpoint)
        : m_endpoint(std::move(endpoint))
    {
        // Assigned last — after m_endpoint, m_mtx, m_cv are initialized.
        // std::jthread injects stop_token automatically (first param).
        m_thread = std::jthread([this](std::stop_token st) { run(st); });
    }
    // Destructor = default.
    // Destruction order: m_thread first (last declared) → request_stop() + join()
    //                    m_cv, m_mtx, m_endpoint destroyed after thread exits.
    // Shutdown while parked in wait: sub-millisecond.
    // Total shutdown latency also includes the current ping() call's duration.

private:
    void run(std::stop_token st) {
        std::unique_lock lock(m_mtx);
        while (!st.stop_requested()) {
            lock.unlock();
            ping();
            lock.lock();
            // Wakes on stop request OR after interval — not a fixed sleep.
            m_cv.wait_for(lock, st, std::chrono::seconds(5), []{ return false; });
        }
    }

    void ping() { std::cout << "ping: " << m_endpoint << "\n"; }

    std::string                 m_endpoint;
    std::mutex                  m_mtx;
    std::condition_variable_any m_cv;
    std::jthread                m_thread;   // LAST
};
```

---

## 2.10 Production Snippet — RAII Thread Group

```cpp
#include <thread>
#include <vector>
#include <functional>
#include <iterator>
#include <type_traits>
#include <algorithm>
#include <numeric>
#include <atomic>
#include <iostream>

class ThreadGroup {
public:
    ThreadGroup(std::size_t n, std::function<void(std::size_t)> fn) {
        m_threads.reserve(n);
        for (std::size_t i = 0; i < n; ++i)
            m_threads.emplace_back([fn, i]{ fn(i); });
    }
    // jthread vector destructor joins all threads — even during stack unwind.
    ThreadGroup(const ThreadGroup&) = delete;
    ThreadGroup& operator=(const ThreadGroup&) = delete;
private:
    std::vector<std::jthread> m_threads;
};

template<class It, class F>
void parallelForEach(It first, It last, F fn) {
    static_assert(std::is_base_of_v<std::random_access_iterator_tag,
                  typename std::iterator_traits<It>::iterator_category>,
                  "parallelForEach requires random-access iterators; "
                  "for forward iterators, materialize into a vector first");
    const std::size_t n = static_cast<std::size_t>(std::distance(first, last));
    if (n == 0) return;
    const std::size_t nt = std::max(1u, std::thread::hardware_concurrency());
    const std::size_t chunk = (n + nt - 1) / nt;
    ThreadGroup g(std::min(nt, n), [&](std::size_t idx) {
        auto b = std::next(first, static_cast<std::ptrdiff_t>(idx * chunk));
        auto e = std::next(first, static_cast<std::ptrdiff_t>(std::min((idx+1)*chunk, n)));
        std::for_each(b, e, fn);
    });
}   // ThreadGroup destructor joins here

int main() {
    std::vector<int> v(1000);
    std::iota(v.begin(), v.end(), 0);
    std::atomic<long long> sum{0};
    parallelForEach(v.begin(), v.end(), [&](int x){
        sum.fetch_add(x, std::memory_order_relaxed);
    });
    std::cout << sum << "\n";  // 499500
}
```

- `std::vector<std::jthread>` + `emplace_back` + `reserve` = correct. `emplace_back` constructs the move-only `std::jthread` in place; `push_back(t)` with an lvalue does not compile (jthread is not copyable), while `push_back(std::move(t))` does compile.
- Exceptions inside `fn` → `std::terminate`. For propagation, use `std::async` or `std::promise` (Ch.7).

---

## 2.11 hardware_concurrency: Handle With Care

```cpp
#include <thread>
#include <algorithm>

// Can return 0. Standard says "not computable or not well defined".
// In containers: often returns the HOST cpu count, not the cgroup limit.
// Treat as a hint. Always defend.
unsigned int threadCount(unsigned int cap = 64) {
    unsigned int n = std::thread::hardware_concurrency();
    return std::clamp(n > 0 ? n : 4u, 1u, cap);
}
```

Add a config override. Your laptop has 16 cores. Your prod container has 2. `hardware_concurrency()` will lie to you and not feel bad about it.

> **Intuition pump (real-world):** `hardware_concurrency()` is the OS wandering into the room, glancing at the ceiling, mumbling "umm… 8? maybe? could be 0, I don't know your life" and wandering back out. It's a hint, not a contract. In a container it'll gleefully report the host machine's 96-core count while your cgroup is throttled to 2. Always clamp it. Always add a config override. The OS is doing its best. Its best is not good enough for a thread pool sizing decision.

---

## 2.12 Detach: Just Don't

`detach()` means: "I am giving up all ability to know if this thread is alive, finished, crashed, or currently deleting your database."

```cpp
#include <thread>
#include <string>
#include <chrono>

void use(const std::string&) {}   // stub

void bad() {
    std::string msg = "hello";
    std::thread t([&msg]{           // captures reference to msg
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        use(msg);                   // msg is destroyed. use-after-free.
    });
    t.detach();
}   // msg destroyed here. thread still running. boom.
```

The one tolerable use: a crash watchdog that uses only OS primitives, captures no C++ objects, and you accept it will be killed by the OS at process exit with no cleanup. Everything else: use `std::jthread`.

> **Intuition pump (real-world):** `detach()` is firing the contractor without paperwork. They're still in your house. You've changed the locks but forgot they have the garage code. You don't know what they're painting. You don't know if they're accessing your files. You don't know if they've subcontracted to someone with worse judgment. Then your process exits — the OS sheriff arrives, evicts everyone mid-paint-job, brushes still mid-stroke, no final coat. RIP the half-painted wall. RIP your invariants. RIP any resource they were holding. "Fire and forget" is a missile metaphor, and missiles don't hold `std::vector` references.

> **Pitfall:** `detach()` is not "fire and forget". It is "fire and pray". Use a thread pool instead.

---

## 2.13 Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `std::thread` destroyed while joinable | `std::terminate` | Use `std::jthread`, or call `join()`/`detach()` first |
| Thread lambda captures local by ref; caller returns | Use-after-free | Capture by value, or ensure structural lifetime |
| Thread/jthread declared before resources it uses | Use-after-free at shutdown | Declare thread member **last** |
| `join()` called twice | `std::system_error` | Check `joinable()` first |
| `std::ref` on local; jthread moved out of scope | Dangling reference | Only `std::ref` when referent structurally outlives all consumers |
| `detach()` + lambda with captures | Use-after-free | Don't detach with captures |
| `sleep_for` in stop-sensitive loop | Shutdown stalls for sleep duration | `condition_variable_any::wait_for` with stop token |
| Uncaught exception in thread body | `std::terminate` | Wrap in `try/catch(...)` or use `exception_ptr` |
| `std::thread` launched in constructor body that throws | `std::terminate` | Use `start()` method after full construction |
| `hardware_concurrency()` used as exact thread count | Returns 0 in some envs, host count in containers | Defend with fallback + config override |
| Member function thread with raw `this` | Use-after-free if object dies first | Ensure structural lifetime or use `shared_ptr` |

---

## 2.14 Closing Mantra

> **Declare your jthread last. Join your thread before your data dies. Everything else is commentary.**
