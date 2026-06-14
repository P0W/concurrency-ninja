# Chapter 10 — Thread-Local Storage & False Sharing

*"No synchronization needed" is wonderful. "Each thread allocates its own 4 MB unordered_map" is less wonderful.*

---

## Part A — `thread_local`: Each Thread's Own Locker at the Gym

---

### 10.1 Cold Open

Dave added a cache. Startup tripled. This was Dave's cache:

```cpp
// thread_local + heavy constructor + 128-thread pool = 128 constructor calls at startup.
// Each thread warms up its OWN copy. Miss rate: 100% until warmed. Memory: 128x.
// A shared_mutex-guarded map would have been faster. Dave knows this now.
thread_local std::unordered_map<std::string, SchemaEntry> g_schema_cache;
```

The lesson: `thread_local` is not "shared but fast." It is "separate and fully paid for by each tenant."

---

### 10.2 What `thread_local` Does

Every thread that accesses the variable gets its own instance. Constructed on first access per thread (function-local), destroyed on thread exit. No synchronization needed — because no sharing.

```cpp
#include <atomic>
#include <cassert>
#include <thread>

thread_local int tls_x = 0;

int main()
{
    std::thread t([] {
        tls_x = 42;
        assert(tls_x == 42);  // t's copy
    });
    t.join();
    assert(tls_x == 0);       // main thread's copy, untouched
}
```

Two threads, two `tls_x`, zero fights. This is the entire value proposition.

> **Intuition pump (real-world):** Think of `thread_local` as the locker at the gym. Every member has their own. You don't know what's inside mine; I don't know what's inside yours; neither of us has to knock. There's no "lock it before you open it" ceremony because the design guarantee is structural — no one else can reach your locker. That's the entire security model. The moment someone hands another thread a pointer to their `thread_local` data, they've left their locker key on a park bench.

---

### 10.3 What `thread_local` Is Not

Not free. On Linux, dynamic TLS goes through `__tls_get_addr` — a function call. On Windows, DLL-resident `thread_local` uses `FlsGetValue`. Cache the pointer if you access it in a tight loop:

```cpp
void hot_loop()
{
    auto& buf = tls_scratch;          // one TLS lookup — here
    for (int i = 0; i < 1'000'000; ++i)
        process(buf);                 // no TLS lookup — buf is a plain reference
}
```

And: taking the address of a `thread_local` from thread A and handing it to thread B is a dangling pointer the moment A exits. The compiler will not warn you.

---

### 10.4 Production Use Case 1 — Per-Thread RNG

The textbook correct use. No locks, no contention, independent state per thread.

```cpp
#include <random>

thread_local std::mt19937 tls_rng{ std::random_device{}() };

inline int rand_int(int lo, int hi)
{
    return std::uniform_int_distribution<int>{lo, hi}(tls_rng);
}
// Compare to: global rng + mutex = every call serialized. Don't.
```

Each thread seeds its own generator independently. Horizontal scaling is perfect.

---

### 10.5 Production Use Case 2 — Per-Thread Scratch Buffer

Reusable staging buffer for serialization. Grows on demand, never shrinks (amortizes allocation). The reset-but-don't-shrink idiom:

```cpp
#include <vector>
#include <cstdint>
#include <span>

thread_local std::vector<uint8_t> tls_scratch;

// Returns a writable span of exactly `n` bytes.
// Allocates only when the buffer needs to grow past its high-water mark.
inline std::span<uint8_t> scratch(std::size_t n)
{
    if (tls_scratch.size() < n)
        tls_scratch.resize(n);   // grow only — never shrink
    // Do NOT call clear() — that would not release memory anyway,
    // but it would confuse callers. Give them a raw span instead.
    return { tls_scratch.data(), n };
}

// Usage
void encode_request(const Request& req)
{
    auto buf = scratch(req.max_encoded_size());
    std::size_t actual = req.encode_into(buf);
    socket_.write(buf.first(actual));
}
```

After warmup: zero allocations on the common path. The buffer lives at its high-water mark for the thread's lifetime.

---

### 10.6 Production Use Case 3 — Per-Thread Counter with Periodic Flush

Hot-path counter without a hot atomic. Accumulate locally, flush periodically. Teaser for the full sharded design in Part B.

```cpp
#include <atomic>
#include <cstdint>

inline std::atomic<uint64_t> g_requests{0};

thread_local uint64_t tls_requests = 0;

inline void on_request()
{
    if (++tls_requests >= 1'000) [[unlikely]]  // flush every 1 000 ops
    {
        g_requests.fetch_add(tls_requests, std::memory_order_relaxed);
        tls_requests = 0;
    }
}

// On thread exit you must flush the remainder — see pitfalls.
// A RAII guard or an atexit hook works. "I'll remember to do it manually"
// is not a strategy.
```

The hot path is a single non-atomic increment. The atomic runs ~0.1% as often.

---

### 10.7 `inline thread_local` — The C++17 ODR Fix

Pre-C++17: define in one `.cpp`, declare `extern thread_local` in header. Tedious.
C++17: `inline thread_local` at namespace scope — one definition across all TUs.

```cpp
// myutil.h — safe to include from 100 translation units
#pragma once
#include <vector>

// C++17: inline namespace-scope thread_local — one definition, all TUs agree.
// PRE-C++17 win: none; you needed extern thread_local in the header + a .cpp.
inline thread_local std::vector<char> tls_fmt_buf;

// Class template statics never needed a .cpp (templates instantiate per-TU
// since C++98). The C++17 benefit shown above is for non-template globals.
template<typename T>
class PerThread {
public:
    static T& local() { return inst_; }
private:
    static thread_local T inst_;
};
template<typename T> thread_local T PerThread<T>::inst_{};
```

---

### 10.8 Pitfalls

**Heavy constructor × thread pool = slow startup.**

> **Intuition pump (real-world):** A `thread_local std::unordered_map` is each thread getting their own full filing cabinet — with a hasp, a padlock, three keys on a lanyard, and a setup ritual involving a notary. Spawning 100 threads means assembling 100 filing cabinets at once. Nobody tells you this is happening; the first time each thread touches the variable the constructor fires, silently, as the thread-pool spins up. You put the code in production, startup jumps from 200ms to 8 seconds, and Dave spends two days blaming the database. It wasn't the database.

```cpp
// BROKEN — 64 threads, each pays construction cost on first access
thread_local BigExpensiveThing tls_thing;  // 64 constructions, 64 destructions

// FIXED — construct lazily with a cheap sentinel; or ask: do you need 64 copies?
thread_local BigExpensiveThing* tls_thing_ptr = nullptr;
inline BigExpensiveThing& get_thing() {
    if (!tls_thing_ptr) tls_thing_ptr = new BigExpensiveThing{};
    return *tls_thing_ptr;
    // Yes, this leaks on thread exit. Add a thread_local RAII guard if you care.
}
```

**Destructor timing: runs on thread exit, which may be after `main` returns for detached threads.**

```cpp
// BROKEN — destructor of tls_log may run after the global logger is destroyed
thread_local LogBuffer tls_log;    // destructor touches g_logger — boom

// FIXED option A: join all threads before main exits (simplest, always correct)
//   int main() { pool.join_all(); /* now static destructors run safely */ }

// FIXED option B: guard with a process-lifetime atomic flag
inline std::atomic<bool> g_alive{true};
struct SafeLogBuffer {
    ~SafeLogBuffer() {
        if (g_alive.load(std::memory_order_acquire))
            g_logger.flush(data_);   // safe: logger still alive
        // else: silently drop — beats crashing
    }
    // ... data_ ...
};
thread_local SafeLogBuffer tls_log;
// In main, before returning: g_alive.store(false, std::memory_order_release);
```

**DLL boundaries on Windows.** The old `__declspec(thread)` was genuinely fragile with `FreeLibrary` — TLS destructors were not called for threads that existed before the DLL loaded. Modern C++11 `thread_local` on MSVC uses CRT TLS callbacks that fire correctly on `DLL_THREAD_DETACH` and `DLL_PROCESS_DETACH`. The "do not use TLS in unloaded DLLs" rule is legacy advice for `__declspec(thread)`. For new code: `thread_local` in a DLL is fine **as long as the DLL outlives every thread that initialized its TLS variables**. Plugin-style DLLs that may be unloaded via `FreeLibrary` while other threads still hold live `thread_local` instances must join or stop those threads — and ensure all TLS destructors have run — before unloading. Just do not mix `__declspec(thread)` and `thread_local` on the same variable.

**Static destructor + `thread_local` = UB.**

```cpp
// BROKEN: thread_local on main thread is destroyed BEFORE static storage destructors.
// Accessing tls_x from a static destructor is UB.
struct Finalizer { ~Finalizer() { tls_x.flush(); } };  // DO NOT
static Finalizer fin;
```

**You cannot enumerate instances.** There is no `for each thread's tls_x`. If you need a global aggregate, register each thread's pointer in a mutex-guarded `std::vector` on first access and deregister on thread exit. Annoying but necessary.

---

## Part B — False Sharing: Two Atomics, One Cache Line, Zero Mercy

*False sharing is two people sharing a parking spot — fine until both have cars. Then every departure triggers a negotiation.*

---

### 10.9 The Phenomenon

Two atomics on the same 64-byte cache line. Thread A writes one; Thread B's copy of the line is invalidated. B re-fetches from L3. A's write invalidates B's line. They ping-pong. The atomics are not logically contended. The cache line is.

The underlying mechanism is MESI (Modified / Exclusive / Shared / Invalid). When Thread A writes `a`, its core transitions the line to **Modified** and broadcasts an **Invalidate** to every other core holding a copy. Thread B's line flips to **Invalid**. Before B can write `b` — which is on *the same line* — it must re-fetch to **Exclusive** state. That fetch pays L3 latency (or cross-socket QPI latency). Multiply by millions of iterations.

> **Intuition pump (real-world):** Imagine you and a coworker sit at opposite ends of a large open-plan office. Your job: continuously update Column A of a spreadsheet. Their job: Column B. Two different columns, two different people, zero logical overlap — completely independent work. Management, in their infinite wisdom, gave you ONE physical notebook to share. Both columns are on the **same page**. Only one person can hold the notebook at a time. You spend 90% of the day walking across the room, wrestling the notebook out of each other's hands, walking back, writing one number, and watching the other person stand up to come get it again. Throughput: abysmal. Frustration: maximum. The columns themselves don't care about each other. The shared physical substrate does.
>
> That's false sharing. Your atomics are `a` and `b` — independent work. The 64-byte cache line is the notebook — a shared physical transport both cores fight over.
>
> The fix — `alignas(64)` — gives each variable its own page. Now you each have a notebook on your own desk. You write in yours; your coworker writes in theirs. You never look at each other, never wait, never walk anywhere. The OS scheduler weeps with relief. Throughput goes from 1,200ms to 80ms. The columns still don't care about each other. They just finally have the privacy the architecture demands.

```cpp
// BROKEN — a and b share a cache line. Two threads hammering them
// ping-pong the line between cores. Order-of-magnitude slowdown.
struct Bad  { std::atomic<long> a, b; };

// FIXED — each atomic on its own cache line. No invalidation crosses threads.
struct Good { alignas(64) std::atomic<long> a;
              alignas(64) std::atomic<long> b; };
```

Here is a runnable benchmark so you can feel the pain personally:

```cpp
#include <atomic>
#include <chrono>
#include <thread>
#include <cstdio>

struct Bad  { std::atomic<long> a, b; };
struct Good { alignas(64) std::atomic<long> a; alignas(64) std::atomic<long> b; };

template<typename T>
long long bench(long n)
{
    T obj{};
    auto t0 = std::chrono::steady_clock::now();
    std::thread t1([&]{ for (long i=0;i<n;++i) obj.a.fetch_add(1,std::memory_order_relaxed); });
    std::thread t2([&]{ for (long i=0;i<n;++i) obj.b.fetch_add(1,std::memory_order_relaxed); });
    t1.join(); t2.join();
    return std::chrono::duration_cast<std::chrono::milliseconds>(
               std::chrono::steady_clock::now() - t0).count();
}

int main()
{
    constexpr long N = 50'000'000L;
    std::printf("Bad:  %lld ms\n", bench<Bad>(N));
    std::printf("Good: %lld ms\n", bench<Good>(N));
    // Typical output on a modern x86 laptop:
    //   Bad:  1 200 ms
    //   Good:    80 ms
    // On a 2-socket NUMA box the ratio is worse. "Order of magnitude" is a safe prior.
}
```

---

### 10.10 The C++17 Spelling

`<new>` ships two constants. Use them. Do not hardcode 64.

```cpp
#include <new>
#include <cstddef>

// Value-check the feature macro — some early libstdc++ implementations defined
// it without the correct value, producing unexpected constants.
#if defined(__cpp_lib_hardware_interference_size) && \
    (__cpp_lib_hardware_interference_size >= 201703L)
    // "stay at least this far apart" — avoid false sharing
    inline constexpr std::size_t kCacheLine = std::hardware_destructive_interference_size;
    // "stay within this" — exploit constructive sharing
    inline constexpr std::size_t kCacheShare = std::hardware_constructive_interference_size;
#else
    inline constexpr std::size_t kCacheLine  = 64;  // correct for x86-64 and most ARMv8
    inline constexpr std::size_t kCacheShare = 64;  // Apple Silicon effective: 128 for some ops
#endif
```

Paste this into a shared header. Use `kCacheLine` everywhere. When a CPU with 128-byte lines ships and your stdlib updates, you recompile and you are done.

> **Intuition pump (real-world):** `hardware_destructive_interference_size` is the official "minimum urinal gap" constant. The standard has looked at the existing hardware, done the math, and declared: "this far apart, and you won't accidentally bump elbows." Use it instead of hardcoding 64. Your hardcoded 64 is correct for x86 today. It was wrong for POWER9 yesterday and will be wrong for whatever Apple ships next year. The constant updates when you recompile on the new target. Your magic number does not.

---

### 10.11 Production Snippet — The Sharded Counter

Replaces a single `std::atomic<uint64_t>` that was a bottleneck. With 64 shards, each thread *usually* hits a distinct shard; collisions are possible but rare when the thread count is much smaller than `kShards`. Reads sum all shards. Classic high-throughput telemetry pattern.

```cpp
#include <atomic>
#include <array>
#include <thread>
#include <functional>   // std::hash
#include <cstdint>
#include <cstddef>
#include <new>

#if defined(__cpp_lib_hardware_interference_size) && \
    (__cpp_lib_hardware_interference_size >= 201703L)
    inline constexpr std::size_t kCacheLine = std::hardware_destructive_interference_size;
#else
    inline constexpr std::size_t kCacheLine = 64;
#endif

// One atomic per cache line — no two shards share a line.
struct alignas(kCacheLine) Shard {
    std::atomic<uint64_t> v{0};
    // Compiler pads struct to kCacheLine bytes because of alignas. Good.
};
static_assert(sizeof(Shard) >= kCacheLine);

inline constexpr std::size_t kShards = 64;  // power-of-two for cheap masking

class ShardedCounter {
public:
    void add(uint64_t n = 1) noexcept
    {
        const auto idx = std::hash<std::thread::id>{}(std::this_thread::get_id())
                         & (kShards - 1);
        shards_[idx].v.fetch_add(n, std::memory_order_relaxed);
    }

    uint64_t load() const noexcept
    {
        uint64_t sum = 0;
        for (const auto& s : shards_)
            sum += s.v.load(std::memory_order_relaxed);
        return sum;   // not linearizable — fine for telemetry
    }

    // Requires external quiescence: concurrent add() calls that land between individual
    // shard stores will have their increments silently discarded. Stop all producers
    // (or otherwise coordinate) before calling reset().
    void reset() noexcept
    {
        for (auto& s : shards_)
            s.v.store(0, std::memory_order_relaxed);
    }

private:
    std::array<Shard, kShards> shards_{};
};

// Usage
inline ShardedCounter g_requests;

void handle_request() {
    // ... do work ...
    g_requests.add();   // hits one shard; rarely contends when threads << kShards
}
```

`sizeof(ShardedCounter)` is 4 KB on most platforms. Create one per metric, not one per request.

> **Intuition pump (real-world):** The old design was one cashier with 16 people all trying to pay at once — every increment is a fight at the single counter, and the line wraps around the store. The sharded design is 16 cashiers: threads are spread across registers, so most of the time each thread has its own lane and no one waits. (Two threads can hash to the same shard — the design *reduces* contention, it does not eliminate it; with more threads than shards, collisions are guaranteed.) When someone asks "how many total items were sold today?" the manager walks each register and adds up the tapes. That sum happens once, slowly, when you call `load()`. The increment — the path that happens millions of times per second — costs nothing but a relaxed write to your own private lane. Classic tradeoff: make the hot path trivial; let the cold path pay the price.

---

### 10.12 Padding Within a Single Object

Different fields accessed by different threads must live on different lines.

```cpp
#include <atomic>
#include <new>

// BROKEN — doc_count written by many threads; score written by one.
// Both on the same line. Every count++ invalidates the line for whoever reads score.
struct IndexBad {
    std::atomic<uint64_t> doc_count{0};  // high-contention write
    float                 score{0.f};    // low-contention write
    uint32_t              flags{0};
};

// FIXED — isolate high-contention field to its own line.
struct IndexGood {
    alignas(kCacheLine) std::atomic<uint64_t> doc_count{0};  // line 0: many writers

    alignas(kCacheLine) float    score{0.f};   // line 1: one writer at a time
    uint32_t                     flags{0};     // shares line 1 — same thread, fine
    uint32_t                     _pad{0};      // documents grouping intent
};
```

The second `alignas(kCacheLine)` starts a new line for the low-contention group. `score` and `flags` share a line because they are accessed by the same thread — that is fine, even desirable (see §10.13).

---

### 10.13 Constructive Interference — The Rare Good Case

When two pieces of data are *always read together by the same thread*, force them onto one line. Fetching the first brings the second for free.

```cpp
#include <new>
#include <cstdint>
#include <cstddef>

// Key and value always fetched together on lookup. Pack them tight.
struct CacheEntry {
    uint64_t key;
    uint64_t value;
    uint32_t generation;
    uint32_t flags;
    // 24 bytes total — well under kCacheShare (64).
};

// The assert is the whole point: it fires if someone adds a fat field later.
static_assert(sizeof(CacheEntry) <= kCacheShare,
    "CacheEntry crossed a cache-line boundary; lookup will pay extra fetch");
```

Rule of thumb: `alignas(kCacheLine)` to keep hot-written-by-many-threads data *apart*; `static_assert(sizeof <= kCacheShare)` to keep hot-read-together data *together*.

---

### 10.14 Detecting False Sharing in Production

**Linux — `perf c2c`** (cache-to-cache): samples cross-core line bounces, reports the hot struct and the offending field.

```
perf c2c record -F 60000 -a --all-user -- ./your_binary
perf c2c report
```

**Windows — Intel VTune Profiler**: Memory Access analysis, "Bottom-up" view. False sharing hotspots annotated directly on source lines.

**Windows — ETW + WPA**: Enable PMC profile in WPR; correlate L2/L3 cache-miss events with thread IDs and virtual addresses in WPA. More manual than VTune, but no extra license.

Heuristic that saves you from opening any tool: if throughput does not scale past 2-4 threads despite lock-free code, false sharing is the first suspect.

---

### 10.15 Pitfalls

**`std::vector<T>` with `alignas(64)` T — base pointer alignment not guaranteed on pre-C++17 allocators.**

When `T` has `alignas(64)`, `sizeof(T)` is a multiple of 64. So contiguous array elements are *all* 64-byte aligned as long as element 0 is. The question is whether the allocator aligns element 0 correctly. Pre-C++17, `operator new` only guaranteed `__STDCPP_DEFAULT_NEW_ALIGNMENT__` (typically 8 or 16 bytes). The entire array could start on an 8-byte boundary, defeating every element.

C++17 introduced alignment-aware `operator new(size_t, align_val_t)`. Conforming C++17 toolchains (MSVC 19.3x+, GCC 11+, Clang 13+) automatically use it for over-aligned types, so `std::vector<Padded>` works correctly on the stated target toolchains. On older or non-conforming runtimes, verify:

```cpp
struct alignas(64) Padded { std::atomic<uint64_t> v{}; };

// On C++17-conforming toolchains: works correctly — allocator uses align_val_t.
std::vector<Padded> slots(64);
assert(reinterpret_cast<uintptr_t>(slots.data()) % 64 == 0);  // passes on MSVC/GCC/Clang targets

// Fallback for genuinely pre-C++17 / non-conforming runtimes:
// operator new[] without align_val_t cannot guarantee super-alignment.
// Use a platform aligned allocator instead.
#if defined(_WIN32)
    struct AlignedFree { void operator()(void* p) const noexcept { ::_aligned_free(p); } };
    auto slots2 = std::unique_ptr<Padded[], AlignedFree>(
        static_cast<Padded*>(::_aligned_malloc(64 * sizeof(Padded), 64)));
#else
    // C11 / POSIX: size must be a multiple of alignment
    struct FreeDeleter { void operator()(void* p) const noexcept { std::free(p); } };
    auto slots2 = std::unique_ptr<Padded[], FreeDeleter>(
        static_cast<Padded*>(std::aligned_alloc(64, 64 * sizeof(Padded))));
#endif
assert(slots2 && reinterpret_cast<uintptr_t>(slots2.get()) % 64 == 0);
```

**Padding bytes without alignment — the struct is 64 bytes but starts at a 16-byte boundary.**

```cpp
// BROKEN — has 64 bytes of storage; alignment is still 8.
// Adjacent instance shares a cache line with this one.
struct CounterWrong {
    std::atomic<uint64_t> v{};
    char _pad[56];        // 64 bytes total, but struct alignment == 8
};

// FIXED — alignas does both: forces start address AND pads size to multiple of 64.
struct alignas(64) CounterRight {
    std::atomic<uint64_t> v{};
    // Compiler inserts 56 bytes of padding automatically.
};
static_assert(sizeof(CounterRight) == 64);
```

**Over-padding cold data wastes cache.**

```cpp
// WRONG — this struct is accessed once at startup and never again.
// Padding it to 64 bytes wastes 87% of its cache line for no benefit.
struct alignas(64) ConfigRecord { uint32_t version; uint32_t flags; };

// RIGHT — alignas(64) only on data that is written by concurrent threads.
// For cold or read-once data, pack tightly.
struct ConfigRecord { uint32_t version; uint32_t flags; };
```

**Assuming 64 bytes everywhere.** Apple Silicon has 128-byte effective granularity for some coherence traffic. Graviton 3 prefetches in 256-byte sectors. Use `kCacheLine`, not the literal `64`. You wrote the fallback. Use it.

**False sharing on `thread_local` data.** OS TLS blocks may place adjacent `thread_local` variables of different threads near each other in memory. Small `thread_local` types (a single `int`) on adjacent threads can share a line. Use `alignas(kCacheLine)` on performance-sensitive `thread_local` state:

```cpp
// Each thread's flush-staging slot gets its own cache line.
// Without alignas, the runtime may pack these for different threads adjacently.
struct alignas(kCacheLine) FlushSlot { uint64_t pending = 0; };
thread_local FlushSlot tls_flush;
```

---

## Closing Mantra

*Thread-local means each thread pays its own rent; padding means they stop stealing each other's parking.*
