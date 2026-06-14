# Chapter 5 — std::call_once / std::once_flag

## The Bouncer at a One-Night-Only Club

One person lights the candle. Everyone else waits outside. The candle is never lit twice.
That is `std::call_once`. Engineers spent a decade convinced they could replace the bouncer
with a hand-written trick. The trick killed production systems.

> **Intuition pump (real-world):** Imagine a bouncer at a one-night-only pop-up club. A hundred people show up at the door. He lets exactly one person in to do the setup. The other 99 stand outside staring at their phones. Once the setup person is done, the bouncer steps aside and everyone walks in. Nobody does the setup twice — not because the bouncer is smart, but because he has a laminated card that says "done" and he checks it every time. `std::call_once` is that bouncer. He doesn't forget. He doesn't let two people in "just to be safe." And unlike your hand-rolled version, he has never once been bribed by a race condition.

---

## Cold Open — The Bug That "Worked" on x86

```cpp
// BROKEN. Runs fine on x86 dev box. Corrupts memory on ARM prod server.
class Config {
    static Config*    instance_;  // plain pointer — not atomic
    static std::mutex mtx_;
public:
    static Config* get() {
        if (instance_ == nullptr) {               // (A) data race: no sync with (C)
            std::lock_guard<std::mutex> lk(mtx_);
            if (instance_ == nullptr) {           // (B) ok, under lock
                instance_ = new Config();         // (C) optimizer may reorder:
                                                  //     publish pointer BEFORE
                                                  //     object is fully constructed.
                                                  //     Thread seeing (A) non-null
                                                  //     dereferences garbage.
            }
        }
        return instance_;
    }
};
```

The write at `(C)` is not atomic. The compiler and CPU are allowed to publish the pointer
before finishing construction. Thread 2 sees a non-null pointer and dereferences a
half-built object. Undefined behavior. x86's strong memory model hides this for years.
ARM exposes it on day one in production. The formal defect is a data race per [intro.races]/21 — conflicting accesses to a non-atomic pointer with no synchronization between the lock-free read and the under-lock write. Reordering of the publish is what you observe when the UB bites; the race is why it is UB at all.

The fix is eight lines:

```cpp
// FIXED.
#include <mutex>

class Config {
    static std::once_flag flag_;
    static Config*        instance_;
    static void init() { instance_ = new Config(); }
public:
    static Config* get() {
        std::call_once(flag_, init);
        return instance_;
    }
};
```

`call_once` guarantees that when `init` returns, every thread that subsequently passes
through `call_once` sees all writes performed by `init`. No partial construction.
No data race. No memory order expertise required.

---

## The Problem in One Sentence

*Initialize X exactly once, even if 100 threads race — and make every thread see the
fully-constructed result.*

That requires:
- Exactly one thread runs the initializer.
- All other threads **block** until it completes.
- Completion publishes all writes with the correct memory ordering.
- The fast path (after init) touches no locks.

A bare `bool` fails (data race). A naively added `std::atomic<bool>` removes the
data race, but it is not a complete once primitive unless paired with correct
release/acquire publication and CAS-based or lock-based arbitration for
exactly-once execution. Hand-rolled DCLP requires expert-level memory-order
wrangling. The standard library solved this in 2011.

---

## Three Solutions, Ranked

| Solution | Use when | Cost |
|---|---|---|
| Function-local `static` | One global, no runtime args | Fastest; one acquire-load |
| `std::call_once` | Per-instance, per-key, any callable | Same fast path as static |
| `std::atomic<std::shared_ptr<T>>` | Need shared ownership + CAS | Multiple threads may construct; one wins |

### Solution 1 — Meyers Singleton (function-local static)

Use this first. C++11 guarantees thread-safe initialization via [stmt.dcl]/4 (N4861).

> **Intuition pump (real-world):** The receptionist's coffee mug. There is exactly one. It exists. It has always existed. Anyone who walks up to the desk can use it. Nobody creates a second one. Nobody argues about who made it. C++11 looked at this situation, said "yes, we promise this is thread-safe," and delivered. If your singleton is global and has no runtime arguments, this is the entire solution — one `static` local, job done, go home.

```cpp
#include <string>

struct AppConfig { std::string env; int timeout_ms; };

AppConfig load_from_disk();  // may throw — flag not set on throw, next call retries

const AppConfig& config() {
    static AppConfig c = load_from_disk();  // exactly one thread runs this
    return c;                               // all others block until done
}
```

If `load_from_disk()` throws, `c` is not initialized and the next caller retries.
Same semantics as `call_once`. Use `config()` from a million threads — the fast path
is a branch on a guard variable already in L1 cache.

The thread-safe-initialization guarantee is [stmt.dcl]/4, not [thread.once]. One edge-case
difference: recursive re-entry into a magic static — calling `config()` from inside
`load_from_disk()` — is undefined behavior per [stmt.dcl]/4. `call_once` handles the
equivalent re-entry differently: the recursive call deadlocks (the flag is in "executing"
state per [thread.once.callonce]/3), which is hostile but specified. For the common
non-recursive case, they produce identical results; cite [stmt.dcl]/4 when you need the
chapter and verse.

### Solution 2 — std::call_once (per-instance lazy init)

```cpp
#include <mutex>
#include <memory>
#include <string>

class ModelRunner {
    std::once_flag              flag_;
    std::unique_ptr<OnnxModel>  model_;
public:
    explicit ModelRunner(std::string path) : path_(std::move(path)) {}

    OnnxModel& get() {
        std::call_once(flag_, [this] {
            model_ = std::make_unique<OnnxModel>(path_);
        });
        return *model_;
    }
private:
    std::string path_;
};
```

Each `ModelRunner` instance has its own flag. Two instances initialize concurrently
without interference. The flag is a member — it is stable, not copyable, not movable.

### Solution 3 — std::atomic shared_ptr (reference; detail in Ch. 11)

```cpp
#include <atomic>
#include <memory>

// C++20: std::atomic<std::shared_ptr<T>> has native member functions.
// Multiple threads may construct; exactly one wins the CAS.
// Losers discard their copy. Fine for cheap stateless objects.
std::atomic<std::shared_ptr<HeavyThing>> g_thing;

std::shared_ptr<HeavyThing> get_thing() {
    auto p = g_thing.load(std::memory_order_acquire);
    if (!p) {
        auto candidate = std::make_shared<HeavyThing>();
        // CAS: if g_thing is still null, publish candidate.
        // On failure, p is updated with the winner's value.
        if (g_thing.compare_exchange_strong(p, candidate,
                std::memory_order_release, std::memory_order_acquire)) {
            return candidate;
        }
    }
    return p;
}
```

Rarely needed. If you are reaching for this, ask whether `call_once` would do.
Usually it would. This pattern executes the constructor up to N times under contention;
only the winning instance is published, the rest are destroyed. Safe only when construction
is cheap **and** free of externally-visible side effects (file opens, registrations, counter
bumps, GPU allocs). For anything with side effects, use `call_once`.

---

## std::once_flag Basics

Not copyable. Not movable. Default-constructed = "not yet called."

```cpp
#include <mutex>

std::once_flag f;
// std::once_flag f2 = f;             // error: deleted copy ctor
// std::once_flag f3 = std::move(f);  // error: no move ctor (implicitly suppressed; copy is deleted)

// Stable after creation — pass by reference, or store as a member.
void use(std::once_flag& flag) {
    std::call_once(flag, [] { /* init */ });
}
```

One flag answers one question: *did this initialization happen?*
Two independent initializations need two flags.

---

## std::call_once Semantics

```
callable completes normally  →  flag marked done; all blocked threads unblock
callable throws              →  flag NOT marked done; next caller retries
                                (The standard calls this "exceptional return" — [thread.once.callonce]/3.)
```

### Retryable Startup

```cpp
#include <mutex>
#include <stdexcept>
#include <iostream>
#include <string>
#include <functional>

static int g_attempts = 0;

void connect(std::string& out) {
    if (++g_attempts < 3)
        throw std::runtime_error("service down");
    out = "connected";
}

void demo() {
    std::once_flag flag;
    std::string    conn;

    for (int i = 0; i < 4; ++i) {
        try {
            std::call_once(flag, connect, std::ref(conn));
        } catch (const std::exception& e) {
            std::cout << "attempt " << i << " failed: " << e.what() << "\n";
        }
    }
    // attempt 0 failed: service down
    // attempt 1 failed: service down
    // attempt 2: succeeds, flag set
    // attempt 3: fast path, no-op
    std::cout << conn << "\n";  // "connected"
}
```

If you want *permanent failure* (no retry), catch inside the callable and set a
separate error flag. That is your call, not the library's.

---

## Production Snippet 1 — Lazy Model Loader

The code is the lesson. Read the comments.

```cpp
#include <mutex>
#include <memory>
#include <string>
#include <vector>
#include <stdexcept>

struct OnnxModel {
    explicit OnnxModel(const std::string& path) {
        if (path.empty()) throw std::runtime_error("empty model path");
        // ... load weights, allocate GPU memory, warm up kernels ...
    }
    std::vector<float> run(const std::vector<float>& input) const { return {}; }
};

class InferenceEngine {
    std::string                model_path_;
    std::once_flag             load_flag_;
    std::unique_ptr<OnnxModel> model_;

    // Runs at most once. Throws → flag not set → next predict() retries.
    void ensure_loaded() {
        std::call_once(load_flag_, [this] {
            model_ = std::make_unique<OnnxModel>(model_path_);
        });
    }

public:
    explicit InferenceEngine(std::string path)
        : model_path_(std::move(path)) {}

    // Thread-safe. 100 threads can call simultaneously.
    // One loads the model. 99 wait. All 100 proceed after.
    std::vector<float> predict(const std::vector<float>& input) {
        ensure_loaded();        // no-op after first successful load
        return model_->run(input);
    }
};
```

Each `InferenceEngine` instance has its own `load_flag_`. A text model and an image
model can lazy-initialize concurrently without sharing state.

---

## Production Snippet 2 — Registry of Per-Key Singletons

One initialized object per string key. Exactly-once per key, even under concurrent access.
The tricky part: `once_flag` is neither copyable nor movable, so any struct containing
one inherits that restriction. A `std::vector<Entry>` will refuse to compile the moment
it tries to copy or move elements during a resize. `std::unordered_map` is node-based —
rehash does *not* move element objects, so that particular fear is overblown — but
inserting a non-movable `Entry` by value is still awkward enough to invite mistakes.
Solution: heap-allocate each entry via `unique_ptr`. The pointer is movable; the
`Entry` never relocates. Clean ownership, no surprises, same result.

```cpp
#include <mutex>
#include <memory>
#include <string>
#include <functional>
#include <unordered_map>

template<typename T>
class OnceRegistry {
    struct Entry {
        std::once_flag        flag;    // non-movable — heap-allocate so Entry never relocates
        std::shared_ptr<T>    value;
    };

    std::mutex                                          map_mtx_;
    std::unordered_map<std::string, std::unique_ptr<Entry>> entries_;

    // Returns a stable pointer to the Entry for 'key', creating it if absent.
    // map_mtx_ held only for this lookup — not during the expensive init.
    Entry* get_entry(const std::string& key) {
        std::lock_guard<std::mutex> lk(map_mtx_);
        auto [it, inserted] = entries_.emplace(key, nullptr);
        if (inserted) it->second = std::make_unique<Entry>();
        return it->second.get();
    }

public:
    // Thread-safe. Different keys initialize concurrently.
    // Same key: exactly one thread runs factory(); others wait.
    std::shared_ptr<T> get(const std::string& key, std::function<T()> factory) {
        Entry* e = get_entry(key);                       // brief map lock
        std::call_once(e->flag, [&] {                    // no map lock held here
            e->value = std::make_shared<T>(factory());
        });
        return e->value;
    }
};

// Usage:
// OnceRegistry<DbConnection> db;
// auto conn = db.get("shard-7", [] { return DbConnection("shard-7:5432"); });
```

`map_mtx_` protects only the map structure (insertion/lookup). The actual factory call
runs without any global lock. Two threads requesting different keys initialize in
parallel. Two threads requesting the same key: one initializes, one waits on
`call_once`'s internal mechanism, not on `map_mtx_`.

---

## Anti-Pattern — DCLP Done Wrong (Again, So You Recognize It)

> **Intuition pump (real-world):** You see a new milk carton in the fridge, so you grab it for your cereal. But it's empty — your roommate put the carton on the shelf *before* pouring the milk into it. That is DCLP without a memory barrier: the pointer is published before the object is fully constructed. Thread 2 sees a non-null `g_widget` and cheerfully dereferences garbage. The naive double-check *does* prevent two threads from both calling `new` (the second `if (!g_widget)` inside the lock handles that). What it cannot prevent is Thread 1 reordering the pointer write ahead of the constructor stores, so Thread 2 reads a live pointer to a half-built object. x86's strong memory model hides this from you for years. ARM shows you on the first day in production. The formal defect is a data race per [intro.races]/21 — conflicting accesses to a non-atomic pointer with no synchronization between the lock-free read and the under-lock write. Reordering is the observable symptom; the race is what makes the program undefined.

```cpp
// BROKEN — the version your colleague will submit in a code review.
static Widget*    g_widget = nullptr;
static std::mutex g_mtx;

Widget* get_widget() {
    if (g_widget) return g_widget;          // (1) unsynchronized read — data race
    std::lock_guard<std::mutex> lk(g_mtx);
    if (!g_widget)
        g_widget = new Widget();            // (2) write races with (1)
    return g_widget;
}
```

```cpp
// "FIXED" with atomics — correct but please don't.
static std::atomic<Widget*> g_widget{nullptr};
static std::mutex           g_mtx;

Widget* get_widget() {
    Widget* p = g_widget.load(std::memory_order_acquire);  // acquire: see all writes
    if (!p) {
        std::lock_guard<std::mutex> lk(g_mtx);
        p = g_widget.load(std::memory_order_relaxed);
        if (!p) {
            p = new Widget();
            g_widget.store(p, std::memory_order_release);  // release: publish writes
        }
    }
    return p;
}
```

```cpp
// ACTUALLY FIXED — four lines.
static std::once_flag g_flag;
static Widget*        g_widget = nullptr;

Widget* get_widget() {
    std::call_once(g_flag, [] { g_widget = new Widget(); });
    return g_widget;
}
```

The atomic version is correct. It is also fifteen lines of memory-order subtlety that
the next maintainer will "simplify" into the broken version. The `call_once` version
is four lines and cannot be simplified into anything dangerous.

---

## What You Cannot Do

**Reset a flag** — `once_flag` has no `reset()`. It is a one-way switch.

> **Intuition pump (real-world):** You can't un-bake a cake. The eggs are in there. The flour is in there. The heat has happened. If you needed a cake that could be un-baked and re-baked on demand, what you actually needed was a recipe card and a state machine — not a cake. Same deal here: if you need "run once, then run again on command," that is a different abstraction entirely. A version counter. Object replacement. A state machine with an explicit reset transition. What you do *not* need is `once_flag::reset()`, which does not exist and, frankly, should not.

```cpp
std::once_flag f;
std::call_once(f, do_init);
f = std::once_flag{};  // error: operator= deleted
```

If you need "init on first access, reinit on command" — that is a different problem.
Use a version counter or replace the object.

**Pass a flag by value** — non-copyable, non-movable.

```cpp
void bad(std::once_flag f) { ... }          // error: deleted copy ctor
void good(std::once_flag& f) { ... }        // correct
```

**Use two callables on one flag and expect both to run.**

```cpp
std::once_flag f;
std::call_once(f, init_db);      // runs; flag set
std::call_once(f, init_cache);   // SILENT NO-OP. Cache never initialized.
                                 // No error. No exception. Nothing.
```

This is the nastiest failure mode: silent non-execution. One flag, one event.
Two inits need two flags.

**Use across processes** — `once_flag` is process-local.

```cpp
// once_flag in shared memory: NOT supported. Undefined behavior.
// For cross-process once-semantics: named mutex, file lock, or DB transaction.
```

**Assume the flag is set after a throw.**

```cpp
std::once_flag f;
try { std::call_once(f, []{ throw std::runtime_error("nope"); }); }
catch (...) {}
// f is NOT set. Next call_once will invoke the callable again.
```

---

## Pitfall Table

| Pitfall | What Happens | Fix |
|---|---|---|
| Two callables on one flag | Second callable silently skipped | One flag per init event |
| Throw → assume flag set | Flag stays "not done"; next caller retries | Catch + store error inside callable if no-retry needed |
| Flag (or struct with flag) in `std::vector` | Does not compile — `once_flag` is non-movable; `vector` tries to move elements on resize | Heap-allocate entries with `unique_ptr`; `unordered_map` is node-based and won't move elements, but non-piecewise insertion is still awkward |
| Pass flag by value | Does not compile | Pass by `std::once_flag&` |
| Cross-process coordination | Other processes see nothing | OS named synchronization primitive |
| "Reset" for reinit | No reset exists | Redesign; version counter or object replacement |
| Reading flag state | No public accessor | Trust `call_once`; do not inspect the flag |
| Flag outlived by its threads | Dangling reference, UB | Tie flag lifetime to object; join threads before destruction |

---

## Performance

Fast path after init: **one acquire-load + predictable branch**.

```
; Conceptual x86-64 fast path — magic-static guard variable (Itanium ABI)
; This is the function-local-static path; call_once's fast path is similar
; but implementation-defined (single acquire-load on GCC/Clang/glibc;
; InitOnceExecuteOnce on MSVC — comparable but not identical codegen).
movzx  eax, byte ptr [guard_var]   ; acquire-load
test   eax, eax
jnz    .done                       ; predicted taken after first init
; ... slow path: lock, check, call, unlock ...
.done:
```

Sub-nanosecond on a warm cache. On GCC/Clang with the Itanium ABI (shown above),
`call_once`'s fast path is a comparable acquire-load-and-branch. On MSVC, `call_once`
dispatches through `InitOnceExecuteOnce`, whose fast path differs. Neither is measurable
overhead in any real workload.

If you are calling `call_once` inside a tight inner loop a million times per second,
the answer is not to avoid `call_once` — it is to call it once *before* the loop and
cache the result.

---

## Full Compilable Reference

```cpp
// ch05_call_once.cpp
// cl /std:c++20 /W4 ch05_call_once.cpp              (MSVC 19.3x)
// g++ -std=c++20 -Wall -pthread ch05_call_once.cpp  (GCC 11+, Clang 13+)

#include <mutex>
#include <memory>
#include <string>
#include <functional>
#include <iostream>
#include <stdexcept>
#include <thread>
#include <vector>

// --- Pattern 1: Meyers singleton -------------------------------------------
struct AppConfig { std::string env = "production"; int timeout_ms = 5000; };

const AppConfig& config() {
    static AppConfig c;  // thread-safe C++11 init
    return c;
}

// --- Pattern 2: per-instance lazy init -------------------------------------
class HeavyService {
    std::once_flag       flag_;
    std::unique_ptr<int> resource_;

    void do_init() {
        resource_ = std::make_unique<int>(42);
        std::cout << "[init] thread " << std::this_thread::get_id() << "\n";
    }
public:
    int get() {
        std::call_once(flag_, &HeavyService::do_init, this);
        return *resource_;
    }
};

// --- Pattern 3: retryable init ---------------------------------------------
static int g_try = 0;

void flaky(std::string& out) {
    if (++g_try < 3) throw std::runtime_error("not ready");
    out = "connected";
}

void demo_retry() {
    std::once_flag flag;
    std::string    result;
    for (int i = 0; i < 4; ++i) {
        try { std::call_once(flag, flaky, std::ref(result)); }
        catch (const std::exception& e) {
            std::cout << "try " << i << ": " << e.what() << "\n";
        }
    }
    std::cout << "result: " << result << "\n";
}

int main() {
    std::cout << config().env << "\n";

    HeavyService svc;
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i)
        threads.emplace_back([&svc]{ svc.get(); });
    for (auto& t : threads) t.join();
    std::cout << "value: " << svc.get() << "\n";

    demo_retry();
}
// Expected:
//   production
//   [init] thread <one thread id>       <- exactly once
//   value: 42
//   try 0: not ready
//   try 1: not ready
//   result: connected
```

---

*Write it once, write it right, and let the standard library do the bouncing.*
