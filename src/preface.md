# Preface: Why This Book Exists

---

## The Bug That Started This

It was a Tuesday. Six months of clean logs. Then a customer with 16 cores filed a ticket:
"the counter is wrong." Not off-by-one wrong. *Sometimes negative* wrong. The counter was
`unsigned int`.

The code:

```cpp
// Shared. No lock. Seemed fine in testing.
unsigned int g_processed = 0;

void worker() {
    // ... do work ...
    ++g_processed;   // load, add, store. Three instructions. Not atomic.
}
```

You added a mutex, the crashes stopped, you shipped the fix, you told yourself you understood
what happened. You didn't.

What happened: the compiler kept `g_processed` in a register across iterations because nothing
in the abstract machine said it could change. The CPU had uncommitted stores in a write buffer.
Two threads handed each other garbage. The mutex papered over it. The real bug was a
**data race**, which is **undefined behavior**, which means the standard gave the optimizer
permission to do anything — and "anything" is not a synonym for "the wrong number."

Here's what the fix actually is:

```cpp
// Before — data race, undefined behavior, technically not a program
unsigned int g_processed = 0;
void worker() { ++g_processed; }

// After — correct, and the overhead is immeasurable on x86
std::atomic<unsigned int> g_processed{0};
void worker() { g_processed.fetch_add(1, std::memory_order_relaxed); }
```

There is a second story. A different team's benchmark plateaued at 8 threads. Algorithmic
tuning changed nothing. Eventually someone noticed two counters were adjacent struct fields,
sharing a cache line. Every write to one evicted the other from every other core's cache.

```cpp
#include <atomic>
#include <cstdint>
#include <new>   // std::hardware_destructive_interference_size

// Before — "false sharing". Looks innocent. Kills throughput.
struct Stats {
    std::atomic<uint64_t> completed;   // \
    std::atomic<uint64_t> errors;      //  } same 64-byte cache line
};

// After — each field owns its cache line
struct Stats {
    alignas(std::hardware_destructive_interference_size)
        std::atomic<uint64_t> completed;
    alignas(std::hardware_destructive_interference_size)
        std::atomic<uint64_t> errors;
};
```

The fix was two lines. The throughput went from 4x to 12x. The algorithmic work was noise.

Neither team was incompetent. Both were burned by the gap between intuition and how concurrent
hardware actually works. This book closes that gap.

---

## Who This Is For

You already know C++. You know RAII, smart pointers, move semantics, lambdas. You can read
this without reaching for a reference:

```cpp
// Illustrative — Queue and Task are your types, not standard library.
auto safe_queue = std::make_unique<Queue<Task>>();
auto fut = std::async(std::launch::async,
    [q = std::move(safe_queue)]() mutable { return q->drain(); });
```

What you **don't** have is a working mental model for threading. Your concurrency knowledge
is a collection of incantations that seem to work — until they don't, under load, on a
machine with more cores, during a demo.

**This book is not:**
- A C++ tutorial (read Meyers first, come back here)
- A systems-programming-from-assembly treatise
- An academic memory model proof

**This book is:**
- The bridge between "I know C++" and "I write correct concurrent C++"
- Practical patterns with working, tested code
- The vocabulary to review concurrent code and know when something's wrong

---

## What's In It

| # | Chapter | One-liner |
|---|---------|-----------|
| 1 | The Memory Model | Why your intuition is a liability and what to replace it with |
| 2 | `std::thread` and Friends | Launch, join, detach — and why detach will betray you |
| 3 | Mutexes and Locks | `scoped_lock`, lock ordering, and the deadlock two `lock_guard` calls apart |
| 4 | Condition Variables | The predicate pattern, spurious wakeups, and the lost-wakeup that tests won't find |
| 5 | Atomics | Six memory orders, when `seq_cst` is right (usually), when it isn't |
| 6 | Lock-Free Data Structures | ABA, hazard pointers, and the case for not writing your own queue |
| 7 | `std::future` and `std::promise` | Task-based parallelism, its costs, and what `launch::async` actually promises |
| 8 | Thread Pools | Why raw threads don't scale, work stealing, and false sharing in your task queue |
| 9 | False Sharing | The $64 problem, `hardware_destructive_interference_size`, and lying benchmarks |
| 10 | Synchronization Patterns | Latches, barriers, semaphores, `call_once`, double-checked locking done right |
| 11 | Testing and Debugging | TSan, Helgrind, stress tests, and the uncomfortable truth about what tests can't prove |
| 12 | Pitfalls Catalogue | 40 named, reproducible concurrent bugs. Bookmark this chapter. |

---

## What's Not In It

| Topic | Why it's out |
|-------|-------------|
| Coroutines (`co_await`, promise types) | A separate book. A longer, sadder book. |
| `std::execution` / P2300 executors | Not in the standard yet. Moving target. |
| GPU, SIMD, AVX intrinsics | Specialized domain, wrong shelf. |
| Distributed systems, message queues | Different failure modes, different guarantees, different book. |

The scope is deliberate. Narrow and deep beats wide and shallow.

---

## How to Read It

Chapters build. Chapter 1 (memory model) is the foundation for Chapter 5 (atomics) is the
foundation for Chapter 6 (lock-free). Don't skip Chapter 1. Everyone who skips Chapter 1
later wishes they hadn't.

Each chapter stands alone as a reference. Six months from now, debugging a condition variable,
turn to Chapter 4 without re-reading the first three.

**Bookmark Chapter 12.** When code review feels wrong but you can't name why, that's where
you go. Naming the bug is the first step to not writing it again.

> **Pro tip:** Skim Chapter 12 before Chapter 1. Not to learn — to inventory. You'll remember
> "right, there's a section on lost wakeups" when you need it later.

---

## Compiler and Standard

**C++20.** Not because C++17 is bad — because these are in C++20 and pretending otherwise
hurts:

```cpp
std::jthread          // auto-joins + stop_token. Replaces 90% of thread lifecycle boilerplate.
std::latch            // single-use countdown. Clean.
std::barrier          // reusable. Also clean.
std::counting_semaphore<N>  // explicit capacity, unlike condition_variable soup.
std::atomic_ref<T>    // atomicity without changing the storage type.
```

Tested on:

| Compiler | Min version | Flag |
|----------|-------------|------|
| MSVC | 19.30 (VS 2022) | `/std:c++20` |
| GCC | 11 | `-std=c++20` |
| Clang | 13 | `-std=c++20` |

No `#pragma omp`. No Boost. No `__sync_fetch_and_add`. No vendor extensions.

Where a snippet is most useful under ThreadSanitizer, that's stated in prose near the snippet —
never hidden in a code block tag you might miss:

```
clang++ -std=c++20 -fsanitize=thread -g example.cpp
```

> **Pitfall:** `std::atomic<double>` is not guaranteed to be lock-free by the standard —
> not in C++20, not in any prior version. Check `std::atomic<double>::is_always_lock_free`
> before committing to a design that assumes it is. On some platforms you get a mutex in
> a trench coat.

---

## The One Rule

Read this twice.

**A data race is undefined behavior.**

Not "you get a stale read." Not "implementation-defined." *Undefined.* The standard tells the
compiler: assume no data races exist. If yours does, the optimizer is not malfunctioning — it
is faithfully optimizing a program you didn't write.

```cpp
#include <atomic>
#include <string>
#include <thread>

// This is a data race. Both threads touch `ready` without synchronization.
bool ready = false;
std::string data;

std::jthread producer([&] {
    data = "hello";
    ready = true;       // store — no fence, no release
});

std::jthread consumer([&] {
    while (!ready) {}   // spin — no acquire
    use(data);          // may see empty string. May loop forever.
                        // Compiler may hoist `ready` out of the loop entirely.
                        // All of this is conforming.
});
// Both jthreads auto-join here. The race still fires before that.
```

The optimizer sees `ready` is never written in the consumer's translation unit (with
optimizations on), concludes it's permanently `false`, converts `while (!ready)` to
`while (true)`, and you have an infinite loop. TSan instruments actual memory accesses at
runtime and would likely catch the race — but only if both conflicting accesses execute.
Races on unexercised code paths are invisible to it. And your release binary, built with
`-O2` and no TSan, sees no synchronization and the optimizer is free to do exactly this.
This is not a compiler bug. This is your bug.

The correct version:

```cpp
std::atomic<bool> ready{false};
std::string data;

std::jthread producer([&] {
    data = "hello";
    ready.store(true, std::memory_order_release);   // publishes data
});

std::jthread consumer([&] {
    while (!ready.load(std::memory_order_acquire)) {}  // sees the publication
    use(data);   // guaranteed to see "hello"
});
```

`release` + `acquire` creates a *synchronizes-with* edge. When the consumer observes
`ready == true`, the standard guarantees it sees everything written before the `release`.
Not maybe. Guaranteed. That's the contract you buy. The cost is architecture-dependent and
typically tiny: on x86-64 the TSO memory model gives acquire/release semantics to ordinary
loads and stores, so it compiles to plain `mov` instructions; on ARM it uses lightweight
`LDAR`/`STLR` acquire/release instructions rather than a full barrier. Not expensive.
Not using it is.

> **Pitfall:** "It's fine, it's just a bool / int / pointer." An `int` load is not
> guaranteed atomic by the standard on any architecture. On x86 aligned loads usually
> are — usually — but "usually atomic on my architecture when aligned" is not a
> correctness argument, it's a prayer. Use `std::atomic<T>`. The overhead is
> immeasurable. The race is not.

One more thing that surprises people:

```cpp
// Looks safe. Is not.
std::atomic<int> x{0};
std::atomic<int> y{0};

// Thread 1:
x.store(1, std::memory_order_relaxed);
y.store(1, std::memory_order_relaxed);

// Thread 2:
while (y.load(std::memory_order_relaxed) != 1) {}
assert(x.load(std::memory_order_relaxed) == 1);   // Can FAIL.
```

`relaxed` operations are atomic — no data race — but provide no ordering guarantees between
*different* variables. Thread 2 can observe `y == 1` without yet seeing `x == 1`. Chapter 5
explains exactly why. For now: `relaxed` is only safe for independent counters. Everything
else needs a real memory order, and Chapter 1 gives you the framework to know which one.

---

## Muscle Memory

By the time you finish this book, these should be reflexes:

| When you see... | Ask... |
|-----------------|--------|
| A shared mutable field | "What synchronization owns this?" |
| A mutex | "Is the lock scope as narrow as possible?" |
| `condition_variable::wait` | "Is there a predicate? Does it check the actual invariant?" |
| An atomic | "Is the memory order correct, or just `seq_cst` by default?" |
| `shared_ptr` across threads | "Who documents the lifetime protocol?" |
| A `bool` flag between threads | "Is this `std::atomic<bool>`, or am I writing a bug?" |

Not a checklist. A reflex. The goal is to look at concurrent code and immediately see its
synchronization structure — the same way you look at a pointer and immediately ask "who owns
this?" That's what this book builds.

---

## The Acknowledgement

Concurrency is the one part of C++ where smart engineers routinely write code they don't
understand. Not from laziness — the mental model is genuinely hard to build, the memory model
was designed for compiler writers not application programmers, and the existing literature is
either formal-and-impenetrable or readable-and-wrong.

The engineers who get this right aren't smarter. They've just been wrong in visible ways more
often, and accumulated the specific failures that teach the specific lessons.

This book gives you those lessons without requiring you to burn down a production system first.

---

*A data race is not a bug you fix — it is a contract you violated.*
