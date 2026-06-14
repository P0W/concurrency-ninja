# Acknowledgements

No book is written in a vacuum. The ideas, intuitions, and rigor in *Concurrency Ninja* stand on a long lineage of people who made the C++ memory model and concurrency primitives explainable in the first place. The expression is mine; the field is everyone's.

## Inspirations

The way concurrency is *taught* in this book — analogies, error patterns, the "ABBA" deadlock framing, the release/acquire-as-handshake mental model — owes a debt to:

- **Anthony Williams** — *C++ Concurrency in Action* (2nd ed.) is the foundational textbook for the modern C++ concurrency surface. If you finish this book and want a thicker reference, read his next.
- **Herb Sutter** — the *atomic<> Weapons* talk series (CppCon 2012/2013) is the single best public explanation of the memory model. If you've ever heard someone say "release/acquire is the hand-off," they got it from him.
- **Bryan Cantrill** — his writing and talks on real-world systems failures shaped the "production stories first, theory second" framing of every cold open in this book.
- **Arthur O'Dwyer** — his blog (*Quuxplusone Blog*) on `atomic<shared_ptr<T>>`, ABI corners, and the mechanics of `alignas` were invaluable when writing the production-patterns chapter.
- **Mara Bos** — *Rust Atomics and Locks*. The C++ memory model is shared with Rust's; her diagrams and explanations transfer almost verbatim and are often clearer than the C++ versions.
- **Hans Boehm** and **Sarita Adve** — the original memory-model papers that gave the standard its semantics. Heavy reading; necessary if you ever want to argue about consume ordering at a standards meeting.

## Standards used

The technical claims throughout this book are checked against:

- **ISO/IEC 14882:2020** (C++20), working draft **N4861** — https://timsong-cpp.github.io/cppwp/n4861/
- **ISO/IEC 14882:2023** (C++23), working draft **N4950** — https://timsong-cpp.github.io/cppwp/n4950/
- **P0660R10** — *Cooperative interruption for thread destruction*, the design paper behind `std::jthread` and `std::stop_token`.

Where the text says "the standard guarantees X," the citation lives in one of those documents. Where it says "in practice on x86" or "typical implementations," it means exactly that — implementation behavior, not a standard guarantee.

## Tooling

- **[mdBook](https://rust-lang.github.io/mdBook/)** — the static-site generator that turns the source markdown into the live site at [p0w.github.io/concurrency-ninja](https://p0w.github.io/concurrency-ninja/). The Rust ecosystem made this possible; the book is layered on their work.
- **GitHub Pages** + **GitHub Actions** — hosting and CI, free to the world.

## If you finished this book, read next

A short, opinionated reading list:

| Topic | Read this |
|---|---|
| The 800-page treatment of every primitive | Williams, *C++ Concurrency in Action* (2nd ed.) |
| The memory model, deeper | Sutter's *atomic<> Weapons* (CppCon 2012/2013, both talks on YouTube) |
| The memory model, even deeper (cross-language) | Bos, *Rust Atomics and Locks* (free online: marabos.nl/atomics) |
| Lock-free queues, hazard pointers, real-world performance | Dmitry Vyukov's 1024cores.net |
| What the future of C++ concurrency looks like | P2300 (sender/receiver, the executors proposal) and the CppCon talks tracking it |
| Production debugging stories | Bryan Cantrill's talks — pick any of them |

## Thanks

If you spotted an errata, opened an issue, or improved this book in any way through a PR — your name belongs here. Pull requests welcome at https://github.com/P0W/concurrency-ninja.
