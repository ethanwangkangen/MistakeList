# C++ / HFT Interview Prep — Mistake Ledger

Running record of mistakes, misconceptions, and weak areas from interview practice.
**Part 1 = C++ depth** — knowledge gaps by theme, heatmap on top.
**Part 2 = LeetCode** — bug taxonomy with counts, identification failures, per-problem log, pre-submit checklist.

**Update protocol:** new LC bug → classify into §2.1, increment count, one-liner example, add to §2.4. Wrong-approach event → row in §2.2. New C++ gap → its theme in Part 1. Recurring across ≥2 sessions → promote to a priority table.

> Condensed 2026-07-29 from the 07-13 edition + five unpushed sessions (07-19, 07-25, 07-26, 07-29). Verbose entries compressed to wrong→correct cores; no distinct mistake dropped.

---

# Part 1 — C++ Depth

## 1.0 Knowledge-Gap Heatmap

| Area | Status | Notes |
|---|---|---|
| Initialization taxonomy (value/default/aggregate, copy- vs direct-init, `explicit`) | 🔴 weak | 3+ hits, #1 study item. Meyers 7 & 27 |
| Systems numbers (cache latencies, mispredict cost) | 🔴 memorize cold | Understood *why*, can't produce numbers |
| Custom-container mechanics (growth, lifetime, allocator contracts) | 🔴 weak | Two full vector builds (07-25, 07-26); growth-from-zero recurred across both |
| Concurrency & memory model | 🟡 improving | Memory-order reasoning now strong (13/14 fill-in); gaps are in the *library* surface (guards, tags, thread lifecycle) and in race-vs-UB taxonomy |
| Precise-semantics precision ("80% answers") | 🟡 shaky | `inline`, RAII scope, RVO/NRVO, `throw;` vs `throw e;` |
| Standard library edge cases | 🟡 shaky | `variant`, `map::operator[]`, `string_view` lifetime, strict weak ordering |
| Move semantics & value categories | 🟢 mostly solid | Edge cases: const-move, SSO, moved-from terminology |
| Templates & generic code | 🟢 mostly solid | ADL swap, `if constexpr` were the gaps |
| Class mechanics, virtual dispatch, exceptions | 🟢 mostly solid | Terminology slips; ctor-throw rules to lock |

## 1.1 Recurring / High-Priority (drill first)

| # | Topic | Trap in one line | Freq |
|---|---|---|---|
| 1 | Signed/unsigned comparison & unsigned underflow | `i < v.size()` with `int i`; `size()-2` wraps at n≤1; `capacity_-1` wraps at 0. Conversion goes UP to unsigned, so `-1 < size()` is FALSE | 🔁🔁🔁🔁 **most frequent item in the ledger; every LC session** |
| 2 | Growth-from-zero in custom containers | `capacity_*2` (or `*3`) stays 0; `size_ >= capacity_-1` wraps to SIZE_MAX. Fix: `capacity_ ? capacity_*2 : 1` | 🔁🔁 07-25 and 07-26, two builds running |
| 3 | `map::operator[]` silently inserts | Bare read mutates; poisons min/max folds; default-constructs members with no NSDMI | 🔁🔁🔁 3 hits → reflex: `find()` before every map read |
| 4 | Initialization taxonomy (`new S`, value/default/aggregate) | No mental model | ⭐ #1 study item |
| 5 | Negative modulo | C++ `%` truncates toward zero: `-27 % 26 == -1`, not 25 (Python differs). Guard: `((x % m) + m) % m`. Same idiom for cyclic indices `(i - k + n) % n` | 🔁🔁🔁 wraparound bit three times in different costumes (cyclic row shifts, circular arrays, alphabet rotation) |
| 6 | `std::move` on `const` → silent copy | `const T&&` can't bind to `T&&` → copy ctor | ⭐ 2× |
| 7 | Lifetime extension — two-part rule | (a) ANY reference **directly bound** to a temporary extends it (`const&`, `T&&`, `auto&&` all extend — "only const&" is folklore; non-const lvalue refs just can't *bind*); (b) extension stops at the first function-return-by-reference | ⭐ + 🔁 |
| 8 | Cache latency + branch-mispredict numbers | L1~4 / L2~12 / L3~40 / RAM~200 cyc; mispredict ~15–20 cyc; ~50 L1 hits per RAM access | ⭐ **3rd consecutive fail — pure recall, sticky-note it** |
| 9 | `optional` `*` vs `.value()` | `*opt` empty = UB; `.value()` throws | ⭐ high-freq HFT |
| 10 | ADL `swap` idiom | `using std::swap; swap(a,b);` — qualified call defeats custom swap | ⭐ high-freq HFT |
| 11 | SSO defeats move | Short-string move costs the same as a copy | ⭐ high-freq HFT |
| 12 | `string_view` lifetime + not null-terminated | View of a temporary dangles at the semicolon; `data()` has no `'\0'` | ⭐ high-freq HFT |
| 13 | Strict weak ordering | `<=` comparator in `std::sort` is UB, not just wrong | ⭐ trap question |

## 1.2 Future Study TODO (flagged, not yet drilled)

- **`std::function`** — confirmed cold 07-10. Type erasure, SBO + heap-alloc threshold, call overhead vs raw fn ptr / templated callable, HFT alternatives (`function_ref`, templates, CRTP), `std::move_only_function` (C++23).
- **Exceptions** — scattered partials, no unified model. Guarantees (nothrow/strong/basic), unwinding cost model (zero-cost tables, why HFT bans them hot-path), `noexcept` × move (`move_if_noexcept`), function-try-blocks, `exception_ptr`, rethrow rules.

## Theme: Move Semantics & Value Categories

- **`std::move` on `const`** (05-27, 06-28) — yields `const T&&`; no move ctor accepts it, so overload resolution falls back to **copy**. `std::move` does not strip const.
- **SSO defeats move** (07-02) — long-string move = O(1) pointer steal; short-string move = copy cost, chars live inline.
- **Pessimizing move** (07-02) — `return std::move(x)` turns an elidable prvalue into a non-elidable xvalue, killing guaranteed copy elision. `-Wpessimizing-move`.
- **NRVO output counting** (06-28) — wrong "124", correct **"133"**: return of a named local is an implicit move; initializing from the returned rvalue is a second move. `A b = foo()` is copy-init *syntax* calling a ctor, not `operator=`.
- **Moved-from state** (07-02) — the term is **valid but unspecified**, not "indeterminate" (that means uninitialized memory).
- **Sink idiom** — `void set(std::string n){ member_ = std::move(n); }` when the function stores a copy anyway. Not "when I want to modify."
- **Rvalue refs DO extend temporaries** (07-10, confident-wrong) — see §1.1 #7. C++23 P2718 extends all temporaries in a range-for initializer (pre-C++23 `for (x : make().member())` dangled).
- ✅ Correct: `std::forward` vs `std::move` on a forwarding ref (`T` deduces `int&` for an lvalue).

## Theme: Initialization & Lifetime

- **`new S` vs `new S()`** — `new S` default-initializes → scalar members **indeterminate**; `new S()` value-initializes → zeros. ✅ resolved 07-10.
- **Init taxonomy across storage** — `S s = S();` value-init (zero then NSDMIs); `S s;` at namespace/static scope zero-inits; `S s;` inside a function default-inits → members without NSDMIs indeterminate. **Only the automatic case gives garbage.**
- **Designated initializers** must appear in declaration order (C++20); `{.z=3,.y=2,.x=1}` does not compile.
- **`explicit` / copy-init vs direct-init** — no precise model yet. Meyers 7 & 27.
- **Lifetime extension scope** — follows the **directly-bound** temporary only; `const std::string& r = temp.getName();` returning a member ref dangles.
- **SIOF** — cross-TU static init order is unspecified. Fix: Construct-On-First-Use (function-local static, thread-safe since C++11). `constinit` is the tool for **mutable** globals (asserts constant-init, compile error otherwise); `const` may be dynamically initialized; `constexpr` is const + constant-initialized.
- **`static` — six uses** ✅ (statics locals, data members, member fns, free fns, globals, C++23 static lambdas). In-class static member needs an out-of-class definition unless `inline static` (C++17).
- **Block-scope bare `int x;`** is the only indeterminate case; `int x{}` / `int x = int()` zero everywhere.
- **Default arg referencing another parameter** — `void foo(int a, int b = a)` does not compile; other params aren't in scope at the call site.
- ✅ Returning `const std::string&` or `string_view` to a local both dangle.

## Theme: Containers & STL

- **vector reallocation total work** — geometric series **< 2N**, not ~3N → amortized O(1). Growth: libstdc++/libc++ 2×, MSVC 1.5×. Sub-2 factor lets the allocator **reuse freed blocks** (their sum can exceed the next request); at exactly 2× it never can.
- **`emplace_back` real gotcha** — not exception safety. It forwards to **`explicit`** ctors: `vector<vector<int>> v; v.emplace_back(10);` silently builds a 10-element vector; `push_back(10)` won't compile.
- **`emplace_back(new Derived())`** — no destructor runs at all; the danger is a **leak** if reallocation throws before a `unique_ptr` adopts the pointer.
- **`vector<bool>`** — bit-packed specialization; `operator[]` returns a **proxy**: `auto b = v[i]` deduces the proxy, `&v[i]` is illegal, no contiguous `.data()`. `sizeof` is still ~24B (object, not heap data).
- **Container per-instance overhead** — empty `unordered_map` ~56B on GCC → `vector<unordered_map>(1e6)` ≈ 56 MB before storing anything. Rough sizes: vector ~24B, string ~32B, unordered_map ~56B, map node ~48B+.
- **Erase-inside-loop** — `it = v.erase(it)` while keeping `++it` skips every element after an erasure. Increment only in the else-branch, or `std::erase(v, val)` (C++20).
- **`s += 'x'` vs `s = s + 'x'`** — `+=` amortized O(1) → O(n) total; `s + 'x'` builds a fresh temporary each iteration → **O(n²)**.
- **`map::operator[]`** (07-10) — knew it default-inserts but then evaluated `0 > 0` as true. Execution slip: after stating what an expression does, *evaluate the branch with the concrete value*.
- **Iterator invalidation precision** — vector: total only on **reallocation**; with spare capacity, insert invalidates at-and-after only. Deque: insert at either end invalidates **all iterators** but **no pointers/references** (fixed blocks never move). Unordered: iterators die on **rehash**, ptrs/refs never (nodes pinned).
- **`string_view` is not null-terminated** — `printf("%s", sv.data())` over-reads. Use `%.*s` with the length.
- **`optional`** — `*opt` on empty is UB (no check, fast); `.value()` throws `bad_optional_access`. Mirrors `vector::operator[]` vs `.at()`.

## Theme: Templates & Generic Code

- **ADL `swap`** — `std::swap(a,b)` in generic code defeats a cheaper custom swap. Idiom: `using std::swap; swap(a,b);` — the canonical customization-point pattern.
- **`if constexpr`** — the discarded branch is **not instantiated**, so it may be ill-formed for that type. A plain `if` type-checks both branches.
- **Template overload ambiguity** is a **compile error**, not UB. Meyers 24–26.
- ✅ `S<T,T>` vs `S<T,int>` for `S<int,int>` is ambiguous. `decltype(x)` → `int`, `decltype((x))` → `int&`. Reference collapsing: any `&` collapses to `&`.

## Theme: Casting & Type Punning

- **`reinterpret_cast` does not use memcpy** — zero codegen, just retypes a pointer. That's why it breaks strict aliasing. Value punning: `memcpy` (pre-C++20) or `std::bit_cast` (C++20); both compile to zero instructions.
- **Strict aliasing exemptions** — `char*`, `unsigned char*`, `std::byte*` may alias anything; reading through any other mismatched pointer is UB (may serve a stale register).

## Theme: Class Mechanics & Exceptions

- **ODR violation ≠ linker error** — only duplicate **non-inline** definitions give a clean error. Two TUs defining the same class/inline function *differently* is silent UB.
- **Constructor throw** — the object's destructor does **not** run (it never fully existed); already-constructed members are unwound. Destructors are implicitly `noexcept` since C++11 → a throwing dtor calls `std::terminate`.
- **shared_ptr cycle** is a **reference cycle**, not a "deadlock". Fix: `weak_ptr` on one side.
- **Destructor-only class + `std::move` → double-free** — a user-declared dtor suppresses *move* members but copy members are still generated, so `std::move` silently copies; two owners, dtor twice. Rule of Five. C++'s failure mode is "compiles, quietly wrong," almost never "throws."
- **One vptr in single inheritance** at offset 0, repointed during each ctor. Multiple vptrs only under multiple inheritance. Dispatch = two dependent loads + indirect call.
- **Slicing** — `f(Base b)` copy-constructs a fresh Base; every ctor sets the new object's vptr to its own vtable. The vptr is never copied.
- **`shared_ptr<Base>{new Derived}` survives a non-virtual base dtor** ✅ — the control block type-erases a deleter for the *constructor argument type*. `unique_ptr<Base>` deleting through `Base*` is UB.
- **Diamond** — `d.B::x` / `d.C::x`; `d.A::x` is still ambiguous. Virtual inheritance costs a runtime **offset indirection**, not "a vtable"; static_cast down from a virtual base is banned.
- **`alignas(64)`** forces `sizeof` to 64 (array contiguity) — the false-sharing fix made concrete. Pre-C++17 plain `new` only guaranteed ~16B alignment.
- ✅ `delete` vs `delete[]`; virtual default args bound **statically** (Derived body, Base default); virtual call from a ctor resolves to Base (pure virtual = UB); non-virtual base dtor + delete-through-base = UB.

## Theme: Raw Memory, Object Lifetime & Allocators

- **Assignment into uninitialized storage** — `data_[size_] = std::move(e);` on raw memory is UB: move-*assignment* reads/releases the destination's existing state. Must placement-new. Why it "worked": trivial types emit the same store; fresh OS pages are zeroed; UB is a license, not an obligation. ASan won't catch it (lifetime, not bounds); MSan will.
- **`move_if_noexcept` belongs in the reallocation loop**, not on a by-value `push_back` parameter (that one already gives the strong guarantee for free, since `size_` is untouched if construction throws). A throwing move mid-realloc leaves the old buffer half-gutted with no rollback — this is *why* move ctors should be `noexcept`.
- **Per-element `delete` inside one allocation** — `data_ + i` was never returned by an allocation → heap corruption. Unfuse the back if you unfused the front: explicit dtor calls in **reverse order** + one `operator delete(data_)`. Pairing never mixes: `new`/`delete`, `new[]`/`delete[]`, `operator new`/`operator delete`, `malloc`/`free`.
- **`allocator::deallocate(ptr, n)`** — `n` must equal the `allocate` count: for a vector the **capacity, not the size**. Wrong `n` is UB. Bug pattern: deallocating the old buffer with the *new* capacity after overwriting `cap_`. Caller-remembers-size is what lets pool/arena allocators skip per-allocation metadata.
- **`deallocate(nullptr, 0)`** (07-26) — called on every construction (ctor delegates to `reserve`, which deallocates the null `data_`) and again in the copy ctor. `deallocate` requires a pointer returned by `allocate`. Guard with `if (data_)`.

## Theme: Lambdas

- **`mutable`** is only for modifying **by-value** captures; reference captures never need it.
- **`[=]` captures `this`**, not members by value — `[=]{ return id_*2; }` is `this->id_` through a captured pointer → dangling under deferred execution. Fix `[*this]` (C++17) or `[id = id_]`. C++20 deprecated implicit this-capture via `[=]`.
- ✅ `[&count]` returned from a factory dangles; by-value `[count]` works.

## Theme: Concurrency & Memory Model

*Now the most heavily drilled area (Williams ch.5–8 + 07-26 quiz block). Memory-*ordering* reasoning is strong; the gaps are the library surface and the race-vs-UB taxonomy.*

**Memory model — misconceptions (07-26 quiz, 4 misses):**
- **"Stack locals can't be in data races"** — wrong, and wrongly equated with `thread_local`. Stack memory is fully shareable via pointers/references; races on it are UB. Only `thread_local` (and genuinely unshared locals) are per-thread by construction.
- **All-atomic check-then-increment called a "data race"** — atomics **never** data race. That bug is a pure **logic race** (lost updates; the limit can be exceeded between the check-load and the inner load). Wrong-answer ≠ UB; keep the two axes separate.
- **Mutex + non-atomic flag** — verdict right, but missed that the *unlocked* read of a non-atomic `ready` is itself a data race. **A mutex synchronizes only when BOTH sides lock.**
- **"Relaxed is fine, we only ship on x86"** — couldn't refute. Correct: the **compiler** optimizes against the C++ abstract machine, not the ISA, and may reorder plain accesses around relaxed atomics on any target.

**Memory ordering — 13/14 on the fill-in drill**, including the ping-pong trap (producer's wait needs acquire because the subsequent slot write must be ordered after the consumer's read).
- **Sole miscalibration, twice:** over-strengthening. Chose `acq_rel` for a pure event-counter `fetch_add` (relaxed is correct), and `acquire` to load the thread's **own** index in an SPSC queue (relaxed — single writer, nothing to synchronize with). The litmus question *"does any plain memory depend on this load from another thread?"* is applied in the positive direction but not the negative one.
- **Release/acquire happens-before** — a release-store/acquire-load pair *synchronizes-with* → *happens-before*: everything sequenced before the release (including non-atomic `data=42`) is visible after the acquire observes the store. Both relaxed → the flag itself is race-free but the edge vanishes → non-atomic access is a **data race → UB**, not a "stale read." That distinction is the interview discriminator.
- **Mutex "visibility"** (07-26, asked and resolved) — unlock = release, lock = acquire; the happens-before edge is **per-mutex-object**. Motivated by compiler reordering, store buffers, stale cache lines. Same structure as the SPSC release-store/acquire-load pair.
- **`compare_exchange_weak`** — permits spurious **FALSE** (failing without exchanging even when current == expected), never a wrong true. LL/SC reservations can be lost incidentally. Weak in retry loops, strong for one-shot logic.
- **False sharing** — two independent atomics on one 64B line: MESI ownership is per-*line*, so each write invalidates the other core's copy → coherence ping-pong. Fix `alignas(hardware_destructive_interference_size)`. Performance bug, not correctness.

**Locks & guards (07-26 quiz, 6 misses):**
- **"Contended" means another thread already holds the lock.** Uncontended ≈ 20 ns, a single atomic RMW entirely in userspace. Contended = futex syscall + park + context switch, ≈ 1–10 µs. glibc spins adaptively before parking.
- **`lock_guard` vs `scoped_lock`** — identical codegen for a single mutex (`scoped_lock` has a single-mutex partial specialization). The only reason to keep `lock_guard` around: a **zero-argument `scoped_lock` compiles and locks nothing**.
- **Tag arguments** — `lock_guard`: `adopt_lock` only, tag **last**. `scoped_lock`: `adopt_lock` only, tag **FIRST**. `unique_lock`: `adopt_lock` / `defer_lock` / `try_to_lock`.
- **`unique_lock` movability** — needed for returning a lock from a function, storing one in a container, and `cv.wait`. UB: moving one to another thread, whose destructor then unlocks from a non-owning thread.
- **`scoped_lock` with multiple mutexes uses `std::lock`** (try-and-back-off) → **order doesn't matter**; it exists to prevent multi-mutex deadlock. `lock_guard` is the order-dependent one.
- ✅ **Sharp catch (07-26):** `std::scoped_lock<std::mutex>(m);` parses as a **declaration of a variable named `m`** that shadows the mutex and locks nothing — sharper than the usual "temporary dies immediately" folklore, which needs brace syntax.

**Threads & futures:**
- **An exception escaping a thread's entry function calls `std::terminate`** (07-26, didn't know). To capture it: `std::async` / `packaged_task` / `promise`, then rethrow at `.get()`.
- **`hardware_concurrency()` is unreliable** (07-26, couldn't justify) — it's a hint, may return **0**, counts SMT threads not physical cores, ignores cgroup/container CPU quota, ignores affinity masks.
- **`std::thread` argument copies** (partial) — args are **decay-copied then passed as rvalues**, so binding to `std::string&` **fails to compile**; `const std::string&` compiles and silently binds to the internal copy. Use `std::ref` for genuine by-reference.
- **`jthread`** (partial) — gave auto-join, missed **`stop_token` cooperative cancellation**.
- **`std::async` default policy** (half) — had the famous half ✅ (the returned future's destructor blocks). Missed: no policy = `async|deferred`, and a **deferred task runs only on `.get()`/`.wait()`** — never call get, work never executes. Fire-and-forget needs explicit `std::launch::async`.
- **Condition variables** (partial) — had spurious wakeups + the notify-while-holding trade-off. Missed that the predicate loop also guards **missed/stolen wakeups**. `wait` **atomically releases the mutex and sleeps**, re-acquires before returning; the predicate always runs under the lock.
- **`priority_queue` with a capturing-lambda comparator** (07-19) — declared without passing the lambda to the constructor. Capturing lambdas are **not default-constructible**, even in C++20; the comparator object must be passed in.

**SPSC queue build (07-26):**
- **`if (headCurr = tailCurr)`** — assignment instead of `==`. The condition evaluates `tailCurr`, so `pop` spuriously returns nullopt whenever tail ≠ 0, and reads an empty queue's slot when tail == 0. Caught free by `-Wall` (`-Wparentheses`) — **compiler warnings are not yet part of the build habit.**
- **Atomic snapshot hygiene** — loaded `tailCurr`/`headCurr` into locals, then used the atomics directly (`(tail_+1) % N`, `buf_[tail_]`); each such use is an implicit **seq_cst** load and the snapshot locals sat dead. Benign here only because each index has a single writer; a real re-read bug in MPMC. **Rule: load each atomic once per operation into a local and use only the local.**

## Theme: Systems-Level Performance & Numbers

- **Cache latencies (MEMORIZE):** L1 ~4 cyc, L2 ~12, L3 ~40, RAM ~200. Branch mispredict ~15–20 cyc. ~50 L1 hits per RAM access. *Third consecutive fail — pure recall item.*
- **Stack vs heap at the CPU level**, **`vector` out of capacity on `push_back`**, **`new` vs placement new** — flagged unsure, see the vector/allocator theme above.
- ✅ `sizeof` of a class with one virtual + one int = 16 (vptr + int + padding). `sizeof("hello")` = 6 (includes `'\0'`). C array decays to a pointer when passed; `std::array` passes as an object and keeps its size.

## Theme: Precise Semantics (imprecise-answer cluster)

- **`inline`** — permits multiple identical definitions across TUs (an ODR/linkage keyword); it is *not* a request to inline.
- **`std::move`** — an unconditional cast to rvalue reference. Moves nothing itself.
- **`mutable`** — allows modification inside a `const` member function (caches, mutexes); the lambda meaning is separate.
- **RAII** — resource lifetime tied to **object** lifetime, released in the destructor at scope exit.
- **Copy elision vs RVO vs NRVO** — elision is the general permission; RVO (returning a prvalue) is **guaranteed** since C++17; NRVO (returning a named local) remains optional.
- **`unique_ptr` in a vector** — movable but not copyable, so the vector must move on realloc; the move ctor must be `noexcept` or it copies (impossible) / falls back.
- **`throw;` vs `throw e;`** — bare `throw;` rethrows the current exception preserving its dynamic type; `throw e;` **slices** to the static type.
- **`static` at namespace scope** = internal linkage.

## Theme: Standard Library Gaps

- **`std::variant` vs `union`** — variant is type-safe and knows its active alternative; a raw union does not. (Deeper variant study still TODO.)
- **`std::array<int,0>`** — valid, `size()==0`, `data()` unspecified, `begin()==end()`; empty `vector<int>` is heap-free too but dynamic.
- **Hash map collision strategies** (partial) — separate chaining (what libstdc++ `unordered_map` mandates via bucket + node pointers) vs open addressing (linear/quadratic probing, robin hood); the standard's iterator/reference-stability guarantees force chaining.
- **`std::terminate` vs `std::abort`** — `terminate` is the handler invoked on unhandled exception / throwing dtor / escaping thread exception; it *calls* `abort` by default but is user-replaceable.
- **Derived type without RTTI** — a manual type tag / enum in the base, or CRTP; not `dynamic_cast`.
- **Strict weak ordering** — a `<=` comparator in `std::sort` is **UB**, not merely wrong (can run off the end of the range).
- **`unordered_map` custom key** — needs both a `std::hash` specialization (or functor) **and** `operator==`; the hash/equality invariant is: equal keys must hash equal. Violating it silently loses lookups.
- **`unique_ptr` deleter storage** — EBO (empty base optimization), not type erasure; that's why a stateless-deleter `unique_ptr` is pointer-sized while `shared_ptr` type-erases in the control block.
- **`make_shared` vs `shared_ptr(new T)`** — one allocation instead of two, better locality; downside is that the object's storage cannot be freed until the last **weak_ptr** dies.
- **`volatile`** — for memory-mapped I/O and signal handlers; it prevents *compiler* elision/reordering of accesses but provides **no** atomicity and **no** cross-thread ordering. Not a threading tool.

## Theme: Integer Semantics & Conversions

- **Signed/unsigned comparison: conversion goes UP** — the signed operand converts to unsigned, so `-1 < v.size()` is **false**. See §1.1 #1.
- **Integral promotion** — `uint8_t + uint8_t` is an **`int`**; small types promote before arithmetic.
- **`auto` deduces the narrow type** (07-25) — `auto r = arr[i] - arr[i-1];` gives `int`, so `r * r` overflows **before** any widening at the assignment. Widening at the assignment does not rescue an `int * int` product. Declare `long long r` explicitly.
- **Unsigned wraparound in loop bounds** (07-25) — `for (int i = 0; i < d.size() - 2; ++i)` — `size()` is unsigned, so `size()-2` wraps for n ≤ 1. Fix: `int n = d.size(); i + 2 < n`. The boundary is n ≤ 1, not n < 3 (n = 2 gives exactly 0).
- **Negative modulo** — see §1.1 #5.

## Theme: OS & Systems Fundamentals

- **`fork()` return values** (07-25) — returns the **child's PID to the parent**, **0 to the child**, **−1 to the parent** on failure. `getppid()` is what returns the parent's PID.
- **Dangling pointer vs memory leak** (07-25) — defined "dangling" as unfreed memory whose pointer went out of scope; that is a **leak**. A **dangling pointer** is a live pointer whose pointee's lifetime has ended: use-after-free, a pointer/reference to a destroyed local, an iterator invalidated by reallocation. *Leak = memory alive, pointer gone. Dangling = pointer alive, memory gone.*
- **Unsynchronized concurrent writes to a global** (07-25 MCQ) — picked "overwritten by whichever thread called the function last." Correct: **the final value is unpredictable** — call order does not determine store order, a read-modify-write can lose an update entirely, and an unsynchronized concurrent write is a **data race (UB)**, not well-defined last-writer-wins.

## Theme: Custom Container Builds (vector ×2, shared_ptr) — 07-25, 07-26

**Growth from zero — 🔁 the standing item (both builds, two days apart):**
- 07-25 (debug round, broken Vector): `capacity_ = capacity_ * 2` stays 0 for a default-constructed vector → `new int[0]`, then `data_[size_++] = v` writes out of bounds on the **first** `push_back`. Fix: `capacity_ = capacity_ ? capacity_ * 2 : 1`.
- 07-26 (own implementation, **recurrence**): `if (size_ >= capacity_ - 1)` — `capacity_ - 1` underflows to `SIZE_MAX` when `capacity_ == 0` (moved-from vector, or `shrink_to_fit` on empty), so growth never triggers and placement-new writes through `nullptr`; `capacity_ * 3` is also still 0. Fix: `if (size_ == capacity_) reserve(capacity_ ? capacity_ * 2 : 1);`
- **Second occurrence in two days → standing checklist item.**

**Copy assignment (07-25 debug round — declared it correct, missed all three defects):**
1. `memcpy(data_, other.data_, other.size_)` copies `size_` **bytes**, not `size_ * sizeof(int)`.
2. The old `data_` is never deleted → leak on every assignment.
3. No self-assignment guard / copy-and-swap.

**Destruction & bounds (07-26):**
- `pop_back()` wrote `data_[size_].~Element(); size_--;` — destroys the slot **one past** the last live element (uninitialized memory) and never destroys index 0. Confirmed under ASan: garbage dtor call then SEGV. Correct: `data_[--size_].~Element();` plus an empty guard (otherwise `size_` wraps to SIZE_MAX).
- `at()` used `if (index > size_)` instead of `>=`, so `at(size())` returns garbage instead of throwing.

**Syntax & signatures (07-26):**
- Wrote `&vector operator=(...)` instead of `vector& operator=(...)`; the error cascades so every later member sees `vector` as the template argument type.
- Copy-and-swap parameter taken as `const vector other` — `std::move` on a const by-value param yields `const vector&&`, which cannot bind to `vector&&`. **Copy-and-swap takes its parameter non-const, by value.**

**Design-level misconceptions (07-26):**
- `move_if_noexcept` used inside a **copy** path — safe only by accident because the source was const.
- Move ctor / move-assign not marked `noexcept`, which makes the class's own `move_if_noexcept` fall back to copying when `Element` is itself this vector.
- `reserve` allowed to **shrink** and had no exception guarantee — a throwing element ctor leaks the temp buffer *and* the already-constructed elements.
- `deallocate(nullptr, 0)` on every construction — see the allocator theme.

**Earlier cluster (May–June):** `Element` vs `T` and `forward<Element>` vs `forward<Args>` [A]; `size_`/`capacity_` with no NSDMIs; missing const `operator[]`; missing deallocation in `reserve`/dtor; dead try-catch around a noexcept dtor. shared_ptr: control block deleted while holding its own mutex (UB); refcount race between pointer copy and increment → `atomic<size_t>`.

---

# Part 2 — LeetCode

## 2.WEAK — Weak Areas & Study Priority

⚠️ = self-identified; ◆ = ledger evidence.

**Paradigm gaps (highest ROI first):**
1. ⚠️◆ **Greedy category recognition** — execution is fine; recognizing *which* category and proving it fast is the gap. Weakest sub-type was regret/heap greedy — **now improving: LC 1642 (07-29) regret-greedy identified solo and solved clean.** Retention was the old failure (630 stalled despite the 871 install).
2. ◆ **Difference array + prefix sum retrieval** — **NEW, promoted 07-29.** Not a knowledge gap: both are catalogued (recognition-catalog #30, the June prefix-sums writeup) and both re-derive fine once named. The failure is **retrieval** — LC 2381 was worked with sorted-endpoint binary search before the diff array surfaced. Different failure mode from a bug class; the fix is a rehearsed trigger phrase, not a checklist item. See §2.3.
3. ⚠️◆ **Bitmask DP (submask/assignment flavor)** — stalled on 1723 (Jun) and 1655 (Jul), same "what do the bits index" stall. Drill set **1655 → 698 → 2305**. Deferred by request in recent sessions.
4. ◆ **Fenwick / segment tree + coordinate compression** (one bundle) — largely untouched; unlocks the same P90 problems. 2839, 327, 2407 all KIV'd at this wall.
5. ⚠️ **Tries** — untouched; 421 → 1707 queued. Explicitly deferred 07-2x in favour of dp/prefix/greedy/array/window/graph.
6. ◆ **Cyclic / wraparound index handling** — **promoted after three hits**: findMinimumShifts (cyclic row shifts), the Squarepoint torch-cone recall, and the alphabet rotation in 2381. Two named reframes now installed (see §2.3).
7. ◆ **Permutation-counting DP** — small family, cheap to close: 629 → 920 → 1359.

**Cross-cutting point-losers:**
- ◆ **Signed/unsigned discipline** — the single most frequent item in the whole ledger; recurred in *every problem* of several sessions. See §1.1 #1.
- ◆ **Axis-swap reframe** — the dominant identification failure: subarrays→elements, values→indices, array→occurrence-space (2555, 2818, 828, 2831).
- ◆ **Recurrence transcription** (category B) — correct math, wrong code. Hand-trace one 4-element example.
- ◆ **State-dimension sizing** — when to add a dimension (3977 `(node,power)`; 787 `[node][stops]`; 1631 minimax distance). *Improving: 787 and 1631 both framed correctly first try.*
- ◆ **Type-width discipline** (category G).
- ◆ **Reconstruction / traceback** — DPs asking for the answer, not its value.

**Deprioritized (know they exist, don't drill):** digit DP, convex-hull trick / Li Chao, suffix automata, most interval DP.

**Strengths (calibration):** monotonic-stack contribution counting, Kadane, binary-search-on-answer, exchange-argument skeleton, Dijkstra habits, sliding window (1838 "standard"), circular-array reframes.

## 2.0 How to read this part

§2.1 = *execution* bugs (categories + counts + examples). §2.2 = *identification* failures (wrong approach / stall / under-dimensioned state) — a different skill, tracked separately. §2.3 = idioms & checklists. §2.4 = per-problem log. §2.5 = pre-submit checklist.

**Headline diagnosis (stable ~3 months):** algorithmic derivation is fast and usually correct; the bug mass is **transcription fidelity** (correct math → incorrect code) and **fixed-convention recall** (comparator directions, argument orders, index domains, signed/unsigned). Identification failures cluster on **axis-swap reframes**, **state dimensioning**, and — newly — **retrieval of already-catalogued techniques**.

## 2.1 Bug Taxonomy — running counts + examples

| ID | Category | Count | Trend | Signature |
|---|---|---|---|---|
| A | Wrong identifier / variable mix-up | 7 | 🔁 steady | Right algorithm, wrong name plugged in |
| B | Formula inversion / transcription | 9 | 🔴 highest-risk | Derivation right in comments, coded term flipped — **silent wrong answer** |
| C | Base case / initialization | 12 | 🔺 | Only the one truly-free state may be 0; allocation sizing included |
| D | Off-by-one / index-domain translation | 6 | 🔁 steady | 0-indexed input vs 1-indexed dp; boundary-vs-element indexing |
| E | Ordering / direction of operations | 14 | 🔴 rising | Pop direction, comparator direction, read-before-pop, missing tie-break, guard asymmetry |
| F | Missing operation / dropped constraint | 11 | 🔴 format-amplified | A term or requirement never makes it into code |
| G | Numeric type / sentinel / overflow / precision | 14 | 🔴 **top-2 with E** | int*int, `auto` narrowing, unsigned wrap, sentinel width, float division, negative modulo |
| H | Variable shadowing | 2 | — | `int l = mid` inside a loop; structured binding shadowing a parameter |
| I | Over-engineering / wrong complexity or memory class | 11 | 🔺 | Extra state, redundant containers, rescans, memo where none needed |
| J | API convention misuse | 5 | 🔁 4+ sessions | `lower_bound`/`upper_bound` comparator argument order |
| K | State hygiene across calls | 1 | — | Mutated shared state leaks into the next feasibility call |
| L | Parameter/reference propagation | 1 | — | Out-param by value; callee mutates its own copy |
| M | Typo class the compiler would have caught | 1 | new 07-26 | `if (a = b)` for `==`; `-Wall`/`-Wparentheses` not in the build habit |

**Format-sensitivity note:** the distribution is format-dependent. In spec-implementation (OA-simulation) format, E/G transfer intact, B/C/D go dormant (no derived formulas or DP base cases), and **F explodes** (constraint-dense prose). The historical F count was suppressed by an algorithm-heavy diet, not by low propensity. Optiver-style OAs contain both formats — the checklist must cover both halves.

### A — Wrong identifier / variable mix-up
LC 862 `prefix` vs `prefixSum` · LC 1340 min↔max on the left jump boundary · LC 3956 `m-1` where `k-1` was meant · LC 1626 transition `dp[j] + dp[i]` instead of `dp[j] + players[i].score` · teleport-grid Dijkstra read a loop-invariant `distances[...][k]` instead of `[i]` · custom vector `Element` vs `T`, `forward<Element>` vs `forward<Args>` · findMinimumShifts (07-2x) named rows `m` / cols `n`, inverted vs the statement.
**Countermeasure:** dedicated 60-second post-coding pass on *identifiers only*, no logic.

### B — Formula inversion / transcription — HIGHEST RISK
Silent wrong answers, no crash. Two have survived a correction cycle.
- LC 862 derived `prefix[j] <= prefix[i] - k`, coded `k - prefix[i]`.
- LC 2444 counting term used the wrong anchor **and** the wrong extreme.
- LC 828 subtracted **dp values** where the derivation called for **position gaps**.
- LC 1092 stored the DP string by lexicographic comparison; the criterion was length.
- LC 3956 and LC 813 — prefix-sum direction inverted, independently, same day.
- LC 2334 span coded `i - st[j] + 1`; the true left boundary is the previous stack entry (exclusive).
- SquirrelResearch expiry compared a **duration to an absolute time**; the rot instant is `hide_timestamp + time_to_expire`. Countermeasure: compute `expires_at` once at insert.
- **LC 1834** (07-2x) used `min(nextIdleTime, arrival)` for current time, **dragging the clock backwards**; the clock should only jump forward to the next arrival when the heap is empty.
**Countermeasure:** hand-trace ONE 4-element example for any counting/contribution/transition formula before running.

### C — Base case / initialization
LC 1335 `dp[i][0]=0` for all i (only `dp[0][0]` is free) · LC 115 `dp[0][j]=1` for j>0 · mirror-path DP: start cell must not apply redirect logic, mirror on destination ⇒ 0 · LC 992 frequency array missing the `+1`, and `clear()` used as a reset (it zeroes the **size**, not the values — needed `assign(n+1,0)`) · LC 862 deque loop started at i=0 while reading `prefixSum[i-1]` · custom vector `size_`/`capacity_` without NSDMIs · `unordered_map` min-fold corrupted by `operator[]` default-inserting 0 (the phantom 0 wins every `min`) · LC 3977 `distances[source][power]` never zeroed · LC 1478 over-broad base row marked an infeasible state feasible · SquirrelResearch unguarded `caches_[id]` default-constructed a Cache with an indeterminate member · LC 2218 inner dimension `k` where the loop writes up to `j=k` (ASan heap-buffer-overflow) · **findMinimumShifts** (07-2x) accumulator declared `minPerColumn(n, LONG_MAX)` then **summed into** — signed overflow UB; a sum initializes to 0.
**Rules:** only the single truly-free state gets 0, everything else starts unreachable · never bare-read a map inside a min/max fold · always initialize a Dijkstra source cell (2D too) · if a state is derivable from the recurrence, don't hand-write it as a base case · after any DP allocation, eyeball **both** dimensions against the loop bounds as pairs.

### D — Off-by-one / index-domain translation
LC 2008 0-based search result into a 1-indexed dp (idiom: `int pos = it - v.begin()` used directly, `dp[0]=0` absorbing the no-predecessor case) · LC 1335 `j < min(i,d+1)` where `j <= min(i,d)` · LC 3956 deque entry `i-1` instead of `i-l` · LC 862 length is `i - j` with prefix indices, not `i - prefixIndex + 1` · LC 2334 exclusive-fencepost span is `right - left - 1` · general resistance to the **n+1 boundary-indexed convention**.

### E — Ordering / direction of operations
- LC 862 deque pop direction reversed against a correctly designed invariant.
- Min-cost-path Dijkstra: comparator `p1.second < p2.second` gives a **max-heap** → degrades toward O(VE), TLE.
- LC 1334 relaxation skip used strict `<`; `<=` keeps the queue tight.
- LC 2402 room heap ordered by end time only — needs a tie-break on room number.
- LC 2334 read `st.back()` **before** popping `curr` — the boundary read was `curr` itself.
- LC 3977 / **LC 1976 (retention miss, 4 days later)** — staleness skip written `>=`, evicting the **live** entry. **The asymmetry to lock: on pop, equal = owner → process (strict `>`); on relax, equal = no improvement → skip (`>=`).**
- LC 2528 diff-array consume-index alignment; cancellation must accumulate with `-=` at the endpoint, never `=`.
- LC 2547 inner split loop walked j **forward** while the segment grows **leftward** — the freq map held the wrong element set at every evaluation.
- SquirrelResearch ×3: expiry branch inversion · weight bound `< 0` where the spec required `<= 0` · scanned bottom-up for "first not-full" instead of top-down for "highest non-empty" (wrong direction **and** wrong property).
- **findMinimumShifts** (07-2x): the −1 sentinel from plain sweeps left the pre-first / post-last cells unhandled. Two valid completions: explicit wrap branches (`j+cols-last` / `cols-j+first`), or seed `prev = last-cols`, `next = first+cols` so `L <= j <= R` never wraps.
- **LC 787** (07-2x): `priority_queue` tie-breaker `e1.stops < e2.stops` inverted for a `>`-comparator (it prioritizes MORE stops) — **inert**, because the `[node][stops]` table makes pop order non-load-bearing. The whole tie-break line is dead weight. Logged because the direction error is real even where the consequence isn't.
- **Custom vector** (07-26): `pop_back` destroying `data_[size_]` before decrementing; `at()` using `>` instead of `>=`.
**Pattern:** the invariant is designed correctly; the mechanical realization (which end, which direction, which order) flips. Same genus as B, applied to operations.

### F — Missing operation / dropped constraint
LC 3956 transition missing `+ prefix[i]` · LC 312 missing the boundary multiplication for the last-popped balloon · LC 1793 dropped "must contain index k" · LC 1334 missing the stale-node skip · SquirrelResearch ×4 (settling rule entirely absent; "or the cache empties" stop clause never coded → `max_element` on an empty range; **missing `return` ×2** in bool functions, the second one a same-session recurrence) · LC 1888 the entire free-rotation operation never entered the model (degenerate dp dimensions were the tell) · **findMinimumShifts** (07-2x) copy-paste: the right-sweep wrote into `nearestFromLeft`, leaving `nearestFromRight` all −1 · **LC 1834** (07-2x) sorted `tasks` in place, destroying the original indices needed for the output.

### G — Numeric type / sentinel / overflow / precision
- LC 3956 `INT_MIN` sentinel against a `long long` accumulation → needed `LLONG_MIN`.
- LC 3956 **and** LC 813: `numeric_limits<double>::min()` is the smallest **positive** double. Use `lowest()`.
- Mirror-path DP: two near-MOD `int`s added before `% MOD`.
- LC 115 `long long` overflow from binomial growth — fixed by **tightening loop bounds**, not clamping.
- LC 2334 `threshold / (float)len` exceeds float's 24-bit mantissa → restructure as integer cross-multiply.
- LC 2528 **narrowing at a function boundary** (`long long mid` → `int` parameter) plus `vector<int>` holding `long long` deltas.
- LC 2439 `accumulate(b, e, 0)` — the literal `0` sets the accumulator type; sum reached ~1e14.
- LC 1976 `vector<int>` distances with costs to 2e11; `road.cost + dist` overflows *before* the comparison.
- SquirrelResearch ×2: `size < cap/2` integer truncation; then `static_cast<bool>(size())/cap < 0.5` — cast bound to the wrong subexpression **and** to `bool`. Both passed all visible examples. Fix eliminates the class: `2*size < cap`.
- **07-25:** `auto r = arr[i]-arr[i-1]` then `r*r` into a `long long` — `auto` deduces `int`, the multiply overflows first.
- **07-25:** `i < d.size()-2` wraps for n ≤ 1.
- **LC 918** (07-2x): `accumulate(..., 0LL)` result assigned into an `int` — the overflow guard defeated by the narrowing on the next token. Safe within constraints, but inconsistent.
- **LC 2381** (07-29): net alphabet shift can be negative → `shift % 26` is negative → `'a' + negative`. Guard `((x % 26) + 26) % 26`, then rotate in letter-space: `'a' + (s[i]-'a'+net) % 26`.
**Rules:** sentinels match the accumulator width · `lowest()` not `min()` for floats · widen **before** multiplying, and never rely on the assignment to widen · prefer integer cross-multiplication over division/float · **a value must not narrow as it crosses a function boundary** · any subtraction feeding a `%` gets the `+m` guard. Discipline, not blanket-casting: glance at constraints, mark what can exceed 2.1e9, type *those* `long long` and justify aloud. Blanket `long long` reads as not understanding types — a smell in HFT interviews specifically.

### H — Variable shadowing
LC 2517 `int l = mid;` inside the binary-search loop declared a dead local instead of assigning → infinite loop · **LC 787** (07-2x) structured binding `auto [n,c,s]` shadowed the parameter `int n`.
Watch every `type name =` inside a loop body.

### I — Over-engineering / wrong complexity or memory class
LC 2444 redundant resets + a special case the general formula already covered · LC 802 an extra `unordered_set` where the 3-color array already encoded it · LC 895 O(n) scan inside `push` (missed one-entry-per-push) · LC 2334 full-stack rescan → O(n²) TLE · minCost-string memoized disjoint subranges (dp written, never read) + `vector<unordered_map>` MLE + `string` passed by value · teleport-grid fan-out coded before the complexity was multiplied out · LC 1334 `unordered_map` adjacency for contiguous ids · **findMinimumShifts** stored two full rows×cols `int` matrices (~80 MB at the ceiling) where folding each row into one totals array suffices · **LC 918** two O(n) DP arrays where scalars suffice · **LC 2134** duplicated the array where modular indexing `r % n` gives O(1) space · **LC 1631** reallocated the visited grid on every binary-search iteration (~20×) plus `vector<bool>` overhead, and checked the target at pop instead of push.
**Countermeasures:** before adding a container or state var, name the fact it tracks — if an existing structure encodes it, don't · multiply out worst-case fan-out before coding · feed monotonic structures a sorted adversarial input mentally.

### J — API convention misuse
`lower_bound`/`upper_bound` custom-comparator argument order — **the single most repeated convention bug** (LC 1751 + 3 other sessions): `lower_bound` calls `comp(element, value)`; `upper_bound` calls `comp(value, element)`. Mnemonic: **the thing tested for smallness comes first.** Compiles fine, silently wrong on struct-field searches.
**Countermeasure:** recite the convention line before writing the comparator.

### K — State hygiene across calls
LC 1631 mutated the grid inside the feasibility check, corrupting every later `check(mid)`. Heuristic: visited-array when arrival direction doesn't matter; backtrack/restore when the path itself matters. Memo arrays need explicit reset across test cases.

### L — Parameter/reference propagation
SquirrelResearch: capacity out-param taken **by value**; the callee decremented its own copy → infinite loop. Fix `int&`. Distinct from K (K = state leaks forward; L = state fails to propagate back).

### M — Typo class the compiler would have caught
SPSC `pop()`: `if (headCurr = tailCurr)` for `==`. **`-Wall -Wextra` is free; build it into the habit.**

## 2.2 Identification Failures — approach selection & reframing

| Problem | Reached for | Actually was | Trigger to install |
|---|---|---|---|
| LC 2555 | hashmaps + binary search | sliding window over sorted positions | sorted array + "window of length L" → two-pointers FIRST |
| LC 2439 | pairwise local averaging greedy | binary search on answer + prefix feasibility | "minimize the maximum" → BSoA; adversarially test any local greedy before coding |
| LC 895 | heap + O(n) scan in push | freq map + stack per frequency level | design problems: every op O(1)/O(log n) **by construction**; a scan inside push means the paradigm is wrong |
| LC 1494 | stalled on mask meaning | `dp[mask]` = min semesters for that exact completed set | bitmask DP: state = subset achieved; path-independence is the point |
| LC 992 | doubted the window invariant | `atMost(K) − atMost(K−1)` | "exactly K" on windows → difference of two atMost passes |
| LC 828 | one previous position per char | two previous positions + position-gap contribution | contribution counting: the unit is (left choices) × (right choices) in **positions** |
| LC 2818 | "map element to max score" | contribution counting + greedy spend — **two independent stages** | subarray outcome decided by one "best" element → per-element win-count; then look for a second stage |
| LC 3928 | "Dijkstra twice from each node?" | one Dijkstra on a layered graph (2n nodes) | per-node mode/state → add a graph layer, not a second pass |
| LC 1478 | continuous mailbox position as the DP choice | sort → contiguous blocks (exchange argument) → partition DP over split points, closed-form median cost | geometric problem on a line → sort, prove contiguity by exchange, THEN partition DP with precomputed `cost(l,r)`. Family: 410, 813, 1959 |
| LC 3977 | single dist-per-node Dijkstra | `(node, power)` state | Dijkstra with a carried resource → "can a worse-primary arrival with a better resource be needed later?" If yes, it's a dimension |
| LC 813 vs 1043 | conflated the two k's | 813: k caps **group count**; 1043: k caps **group length** | when k appears, say out loud what k bounds before designing state |
| LC 630 (revisit) | "is there even a greedy?" | regret/heap greedy: deadline-sort + take-if-fits + evict the longest | **retention failure** — 871's mechanism was never generalized into a trigger. Proof sentence cold: "dropping courses converts count→slack; slack converts back at ≤1-for-1, so it never profits." (The invariant is **count**, not time) |
| LC 1866 | no independent state | permutation-counting DP: insert by rank, leftmost slot visible | comparison-defined counting → insert by rank, count insertion positions by effect. Drill 629 → 920 → 1359 |
| LC 1976 | one PQ pop per shortest path | augmented Dijkstra carrying `ways[]`: `<` overwrites, `==` accumulates, propagate only once a distance is FINAL | "shortest path + something *about* the shortest paths" → augmented Dijkstra. The staleness check becomes a **correctness** guard, not an optimization |
| LC 2831 | "how do I query deletions-in-range fast" | occurrence-space window (group indices by value); or stale-max `winLen − maxFreq ≤ k` | axis-swap flavor #3 (array → occurrence space). Key templates on the **formula**, not the operation verb — "deletion" failed to retrieve a template filed under "replacement" |
| LC 1888 | splice free rotation into a suffix DP | flips ∘ rotations **commute** → rotate first: min over n rotations of mismatch count; rotations = length-n windows of `s+s` | a global re-alignment op cannot be a local DP transition. Anchor the pattern to **absolute** indices of `s+s` so per-char verdicts survive window motion |
| **LC 2381** (07-29) | two direction-split lists sorted by start + binary search for coverage | **difference array** (+1 at `l`, −1 at `r+1`, one prefix pass) | **RETRIEVAL failure, not a knowledge gap** — catalogued twice already. Trigger to rehearse: *"many range updates, then read the whole array once" → difference array; "range sum queried repeatedly" → prefix sum; interleaved updates and reads → Fenwick.* (The corrected binary-search route is also valid: decouple starts and ends, coverage = `#starts ≤ i` − `#ends < i`.) |
| **LC 1871** (07-29) | O(n log n) sorted reachable-index vector + `lower_bound` | O(n) prefix-count over the window `[i-maxJump, i-minJump]` | reachability with a *range* of predecessors → dp + prefix sum. It's safe because the window's right edge `i-minJump` is strictly `< i` (minJump ≥ 1), so every prefix value queried is already computed |
| **LC 1631** (07-2x) | BSoA + BFS feasibility (correct, but slow) | minimax Dijkstra — `dist` = min over paths of the **max** edge, relax via `min(dist, max(dist_cur, edgeDiff))`; or Kruskal/union-find until endpoints connect | same "augment what distance MEANS" insight as 787. Single pass beats binary-search-plus-search when the answer *is* a path statistic |
| Recurring | prose instinct → no state | the prose WAS the state definition | enumerate the choices at position i first; the state is whatever those choices need to know |
| Recurring | element-indexed prefix DP | boundary-indexed (n+1) | prefix DPs index **boundaries**; size n+1, answer at `dp[n]` |
| **Bitmask DP** ⚠️ FLAGGED | stalls on "what does the mask index" | mask = the ≤~20 **target** set; iterate resources, each ORs in a submask | traceback (`parent[]`) only if the answer needs *which* resources |

**Solid recognitions (calibration):** Kadane instant · BSoA reliable · knapsack connection self-identified (871) · exchange-argument skeleton installed · **regret greedy now self-identified (1642, 07-29)** · **greedy + difference array self-identified before coding (995, 07-29)** · circular-array reframe pair named correctly, and binary-search-on-window-size correctly *rejected* for 2134 (the window size is fixed, nothing monotonic to search).

## 2.3 Idioms & Checklists to Drill

**Two pre-passes before submitting:**
1. **Type-width pass** [G] — read the constraints, compute the max of every accumulator and product; anything past 2.1e9 is `long long`, including **function parameters** and **containers of deltas**. Justify each aloud. Indices and single small elements stay `int`.
2. **Draw-the-array pass** [D, E] — for anything spatial (difference arrays, windows, monotonic spans, 2D DP), sketch 6–8 cells and mark: the interval for a sample index, where each delta is written, where consumed, where it cancels. Walk one concrete element through.

**Difference array — O(1) range update, offline read.** `diff[L] += v; diff[R+1] -= v;` then one prefix-sum pass. Sticky-note intuition: leave a note at `L` saying "from here add v" and at `R+1` saying "stop"; walk the line once carrying a running total, and what you're carrying at `i` is what `i` was owed. Size `n+1` so `R+1` is in bounds at `R = n-1` (this also removes any `if (R+1 < n)` guard). `long long` when values accumulate. **Decision rule: all updates before all reads → difference array; interleaved point-updates and range-reads → Fenwick; interleaved range-updates and range-reads → lazy segment tree.** Canonical: 2528, 1109, 370, 1094, 2381, 995.

**Prefix sum — O(1) range read.** The exact inverse: prefix sum turns *range reads* into two point lookups; a difference array turns *range writes* into two point writes. Uses: range sums; subarray-sum-equals-k with a hashmap (and the divisible-by-k / mod variants); prefix XOR / product / GCD; frequency prefixes for "how many of char c in `s[i..j]`"; **reachability over a window of predecessors** (1871).

**Modular / cyclic index arithmetic.** C++ `%` truncates toward zero → negative operands give negative results. Guard: `((x % m) + m) % m`. Cyclic step backward: `(i - k + n) % n`. Alphabet rotation: rebase to 0..25 first, `'a' + (c - 'a' + net) % 26`. **Any subtraction feeding a `%` gets the `+m`.**

**Circular arrays — the named pair.** (a) **Complement / total-minus-min** — no index wrap needed, works when the answer is a *sum* over the whole array minus an interior interval (918: max circular subarray = max(Kadane, total − minKadane), with the all-negative guard `smallest == total`). (b) **Duplicate the array (or index mod n)** — for wrap-straddling *windows* (2134, 1610, 1888's rotations as windows of `s+s`). Fixed-size window ⇒ nothing monotonic to binary search.

**Binary-search-on-answer ritual.** State the monotonicity sentence aloud before coding: "if x is feasible, every x' < x is feasible; if x is infeasible, x+1 is too." Then bounds, then an O(n) greedy `feasible(x)`. The bugs live in the check, not the search. Template: `mid = l + (r - l + 1)/2` for the right-bound form; any `int l = ...` inside the loop is shadowing [H].

**Dijkstra checklist.** `priority_queue<pair<long long,int>, vector<...>, greater<>>` with `{dist, node}` — `greater<>` on that pair order gives the min-heap free. Pop → `if (d > dist[u]) continue;` — **strict `>`, never `>=`**. Push only on strict improvement. `long long` distances. Initialize the source cell (2D too).
- **State sizing first:** a single dist-per-node suffices only if no carried resource gates future edges. If a worse-primary arrival with a better resource can be needed later (Pareto non-dominance), the resource is a dimension: `(node, resource)` table for a small range, or a `settled[node]` frontier prune for a large one. Never a second pass. Seen as `(node, power)` in 3977, `[node][stops]` in 787.
- **Augment what "distance" MEANS**, not just the state: minimax paths relax with `min(dist, max(dist_cur, w))` (1631); path *counts* relax with `<` overwrite / `==` accumulate (1976).
- **Early return at target** is valid only when the PQ order makes the first target-pop optimal on all keys.

**Regret / heap greedy.** Trigger: a sequential budget + commitments you'd sometimes want to un-take + "the worst held item" is one comparable number. Take everything greedily; when the budget breaks, evict the worst held item from a heap. 871, 630, 1642 (bricks vs ladders: always spend bricks, and when they run out, promote the largest past climb to a ladder).

**Monotonic stack — evaluate at pop time.** An element's full span-as-minimum is known exactly when popped. Left boundary = the element below it after popping (exclusive); right = the incoming index (exclusive); span = `right - left - 1`. Sentinel at the end to flush. Never rescan the live stack.

**Contribution counting — strictness asymmetry.** `(i − left[i]) × (right[i] − i)`. With duplicates, symmetric strictness double-counts: strict on one side, or-equal on the other. Max-over-spans doesn't care; sum-over-counts does.

**Bitmask DP over a small target set** (⚠️ weak). Recognition: target set ≤ ~20 that you cover by assigning resources — the size constraint IS the tell. State: `dp[mask]` over the **target** set, never the resources. Iterate resources one at a time; each ORs in a reachable submask. Submask loop `for (int sub = S; sub; sub = (sub-1) & S)` — and `sub = 0` is valid, handle it outside. `need[sub] = need[sub & (sub-1)] + quantity[__builtin_ctz(sub)]`. Traceback only if the answer needs *which* resources (1723 yes, 1655 no). Why not greedy: assignment constrains future assignments non-locally.

**DP state-derivation procedure.** (1) Parameterize the question — "exactly/at most k of P" puts P's count in the state. (2) Sufficiency test — what must the future know? List the leaks. (3) **Eliminate before enlarging** — for each leak try reordering processing (position → rank/sorted/reverse), canonicalizing (relabel to ranks), or conditioning on a special element. Only a leak surviving all three becomes a dimension. Dividing question: *does the future need this intrinsically, or is it an artifact of my processing order?* (4) Size it against constraints.

**Permutation-counting DP.** Trigger: counting arrangements where the property is defined purely by **comparisons**. Relabel to ranks (bijection ⇒ identities drop from the state), build by inserting in rank order, count insertion positions by effect. 1866: `dp[i][j] = dp[i-1][j-1] + (i-1)*dp[i-1][j]`. Dies the moment magnitudes matter.

**Smaller conventions.** `lower_bound` → `comp(element, value)`, `upper_bound` → `comp(value, element)` ("the thing tested for smallness comes first"); returns first ≥ / first >. Exactly-K on windows = `atMost(K) − atMost(K−1)`. 0-indexed input → 1-indexed dp: use `pos = it - v.begin()` directly with `dp[0]` as the empty-prefix base. Knapsack 2D→1D: iterate capacity **in reverse**. Grids in DP: flat `vector<int>` with manual indexing beats `vector<vector<int>>`. `static int directions[][]` inside a member function is a smell — drop `static` or use `constexpr`.

**Process discipline — patch vs refactor.** When the state, comparator, and relaxation are right, diagnose whether the failures are 1-line bugs BEFORE rewriting the approach. 3977 was two 1-line patches from correct; a full refactor was explored unnecessarily. Tell: if the shape is sound and only edge cases fail, you have bugs, not the wrong algorithm.

**Retrieval discipline (new 07-29).** A catalogued technique that doesn't fire is a *retrieval* failure, and no pre-submit checklist catches it — checklists run after you've chosen an approach. The fix is rehearsing trigger phrases in the constraint-reading pass, not adding a check. Before coding anything with ranges: say which of {prefix sum, difference array, Fenwick} the update/read pattern implies.

## 2.4 Per-Problem Log (chronological)

Format: `LC # (date) — bugs [category] / notes`. Older entries compressed to their distinct content.

**May–June:** 1793 dropped must-contain-k [F] · 862 identifier, formula inversion, deque start, length formula [A,B,C,D] · 1340 min↔max [A] · 3956 identifier, prefix direction, deque entry, sentinel width, missing term [A,B,D,F,G] · 1626 self-compounding transition [A] · 1335 base row + loop bound [C,D] · 115 base case + binomial overflow [C,G] · 992 array sizing + `clear()` misuse [C] · 2008 index-domain [D] · 1334 relaxation strictness, missing stale skip, map adjacency [E,F,I] · 2402 heap tie-break [E] · 802 redundant set [I] · 895 scan-inside-push [I,§2.2] · 1092 wrong comparison criterion [B] · 828 dp-values vs position gaps [B,§2.2] · 2444 wrong anchor + redundant state [B,I] · 2555 / 2818 / 1494 / 992 / 927 identification rows [§2.2] · 2517 shadowing [H] · 1723 bitmask stall [§2.2] · 2439 greedy → BSoA [§2.2] · 813 vs 1043 k-semantics [§2.2] · teleport-grid + mirror-path + minCost-string clusters [A,C,G,I].

**July 1–13:** 1751 comparator order [J] · 871 knapsack + regret greedy installed · 673/813/494/1043 state-dimension drill · 2402 / 2334 (span formula, full-stack rescan, float precision, read-before-pop) [B,E,G,I] · 3977 state augmentation + uninitialized source + `>=` staleness [C,E,§2.2] · 2528 BSoA + difference array **first learned**; consume alignment [E], `vector<int>` deltas and `int` parameter narrowing [G] · 1655 bitmask paused (drill 1655 → 698 → 2305) · 630 revisit — retention failure [§2.2] · 2407 segment-tree KIV · 1866 permutation-counting installed · 1478 partition-DP reduction + over-broad base [C,§2.2] · 2218 allocation sizing [C] + amortized-over-sum complexity lesson · 1976 augmented Dijkstra, `>=` retention miss, int distances [E,G,§2.2] · 2439 revisit `accumulate(...,0)` [G] · 2831 occurrence-space reframe [§2.2] · 1888 walked (rotation cannot be a DP transition) [F,§2.2] · 2547 accumulation direction [E] · 2919 grading never delivered — **still open** · 2560/2517/2064/2318/2266/2088 solved clean · **SquirrelResearch (07-13)** first spec-implementation rep: 12 bugs across B/C/E/F/G/L, zero algorithmic errors, both "fragile" design choices verified correct. Lesson for both parties: **expand all collapsed examples before writing anything — examples are spec.**

**July 19–26 (C++ / systems block):**
- **07-19** — `priority_queue` with a capturing-lambda comparator not passed to the ctor.
- **07-25** — Squarepoint R1 mock: dangling-pointer vs leak definition; `fork()` return values; threading MCQ (last-caller-wins vs unpredictable/UB). Broken-Vector debug round: missed all three copy-assign defects and the growth-from-zero bug. Own code: `auto` narrowing before `r*r` [G]; `size()-2` unsigned wrap [G].
- **07-26** — own vector: growth-from-zero **recurrence**, `pop_back` destroy order [E], `at()` `>` vs `>=` [E], `&vector operator=` syntax, const copy-and-swap parameter, `deallocate(nullptr,0)`, four design-level misconceptions (`move_if_noexcept` in a copy path, missing `noexcept` on moves, shrinking `reserve`, no exception guarantee). Concurrency quiz: 6 misses (thread-exception → terminate; `hardware_concurrency` caveats; "contended" cost model; guard codegen; tag positions; `unique_lock` movability) + 2 partials (thread arg decay-copy, `jthread` stop_token). Memory-order fill-in **13/14**, sole miss = over-strengthening. Memory-model quiz: 4 misses (stack locals and races; atomics-never-data-race; mutex needs BOTH sides; abstract machine vs x86). SPSC queue: `=` for `==` [M], atomic snapshot hygiene, over-strong ordering on its own index. ✅ Sharp catch: `scoped_lock<mutex>(m);` is a shadowing declaration.

**July 27–29 (LeetCode block, 1780–1900 band):**
- **findMinimumShifts** (past-OA recall, cyclic row shifts) — 5 bugs: copy-paste into the wrong sweep array [F], `LONG_MAX` init on a sum [C], unhandled pre-first/post-last cells from a −1 sentinel [E], rows/cols naming inverted vs the statement [A], two full matrices ~80 MB where per-row folding suffices [I].
- **2401** — could not solve (pairwise-AND-zero subarray); flagged for review, no attempt details.
- **1911** solved easily · **2444** (~2090) and **1856** (~2051) solved · **421** skipped (trie deferred).
- **1834** (~1798) — right structure (sort + min-heap), two bugs: sorted in place destroying output indices [F]; `min(nextIdle, arrival)` dragging the clock backwards [B]. Asked for the finished solution rather than completing it.
- **918** (~1777) — correct first try (max/min Kadane + total−min, all-negative guard). Notes: `0LL` accumulate assigned into `int` [G]; two DP arrays where scalars suffice [I]; signed/unsigned.
- **2134** (~1748) — correct (duplicate array + fixed-width window counting zeros). Doubling unnecessary — `r % n` gives O(1) space [I].
- **787** (~1789) — correct `[node][stops]` augmentation + lazy stale skip. Inverted-but-inert PQ tie-break [E], shadowed parameter [H], signed/unsigned.
- **1888** (~1885) — had the rotation-as-doubling insight and the window scaffold, stalled on parity handling and asked for the full solution. Key points: anchor mismatches to a FIXED absolute-index pattern; `min(mismatch, n−mismatch)` covers both targets free; odd-n guard `n%2==0 || l%2==0`; O(1)-space version needs no doubling.
- **1631** (~1948) — solved via BSoA + BFS. "Why so slow" answered: per-iteration visited reallocation, `vector<bool>`, target check at pop [I]. Taught minimax Dijkstra and Kruskal alternatives [§2.2].
- **1838** (07-29, ~1876) — solved, "sliding pattern standard", no issues.
- **1642** (07-29, ~1844) — solved; **regret greedy self-identified**.
- **1871** (07-29, ~1896) — solved O(n log n) with a sorted reachable-index vector + `lower_bound`; asked afterwards about the O(n) prefix-sum form. Notes: computed `int sz = s.size()` then wrote `i < s.size()` anyway [G]; dead commented-out `cout`.
- **2381** (07-29, ~1793) — correctly judged O(n·m) too slow; proposed sorted-by-start lists + binary search (corrected to decoupled starts/ends). **Difference array not retrieved** despite being catalogued twice → §2.2 + §2.WEAK #2. Implementation bugs: no alphabet wraparound at all, and negative modulo [G]. Two separate diff arrays where one signed array suffices; `sz+1` sizing would drop the bounds guard; second dead debug print in two problems.
- **995** (07-29, ~1835) — **correct on the first draft**, greedy + difference array both self-identified before coding. Notes: the `-1` impossibility check runs after the bookkeeping instead of before; kept the `if (i+k < sz)` guard instead of `sz+1` sizing; tracked a full flip count where only parity matters; two-clause condition reducible to `nums[i] ^ parity`. O(1)-space variants taught (sentinel in `nums`, or a deque of flip positions).
- **Session note (07-29):** difference array appeared twice in one session and was absent the first time, present the second — the retrieval fix worked within the session. Signed/unsigned discipline clean on the last two problems for the first time in weeks.

**Open / unresolved:** 2919 grading never delivered · 2401 unsolved · 1888 implementation unconfirmed · 2407 segment-tree implementation KIV · 1655 → 698 → 2305 bitmask drill not started · 421 → 1707 trie pair deferred · 629 → 920 → 1359 permutation drill queued · 2839, 327 waiting on Fenwick/segment trees.

## 2.5 Pre-Submit Checklist (~90 seconds, every problem)

0. **Approach retrieval** (before coding, not after): for anything with ranges, say which of {prefix sum, difference array, Fenwick, lazy segtree} the update/read pattern implies. For anything cyclic, name which circular reframe applies.
1. **Identifiers only** [A/H] — read every name against intent; any `type name =` inside a loop body is a shadowing suspect.
2. **Formula trace** [B] — hand-trace one 4-element example for any counting/contribution/transition formula. Non-negotiable; this class is silent.
3. **Comparators** [E] — heap direction (`<` in a PQ comparator = MAX-heap); strict `<` for sort; tie-break needed when keys collide? On pop equal = owner (strict `>`); on relax equal = skip (`>=`).
4. **Numerics** [G] — widen before `*`; `auto` deduces the narrow type; sentinel matches accumulator width; `lowest()` not `min()`; integer cross-multiply over division; **every subtraction feeding a `%` gets the `+m` guard**.
5. **Signedness** [G] — every loop over a container: hoist `int n = c.size()` or use `size_t`. Never `int i < c.size()`, never `c.size() - k` without a guard. *(The most frequent item in this ledger.)*
6. **Boundaries** [C/D] — first iteration (reads at −1?), last (sentinel flush?), empty structure, dp base = only the truly-free state, `n+1` sizing for boundary-indexed and difference arrays.
7. **Conventions** [J] — recite `lower_bound`/`upper_bound` argument order before writing; no bare `m[key]` reads inside min/max folds.
8. **State hygiene** [K/L] — anything mutated that survives into the next call? Any out-param taken by value?
9. **Complexity sanity** [I] — name the worst-case input; justify every container by a fact nothing else tracks.
10. **Spec fidelity** [E/F — spec-implementation format] — EXPAND ALL COLLAPSED EXAMPLES FIRST; examples are spec. Numbered rule list with boundary strictness before coding; map each rule to its implementing line; every bool function returns on all paths.
11. **Cleanup** — dead debug prints, unused variables, commented-out code. Ten seconds; it is a code-review signal in interviews. *(Two dead `cout`s in two problems on 07-29.)*
12. **Build habit** — `-Wall -Wextra` is free and would have caught the `=`/`==` typo [M].

---

*Last updated: 2026-07-29 · condensed rebuild + five sessions merged (07-19 lambda-comparator; 07-25 Squarepoint R1 mock + broken-Vector debug; 07-26 own vector, concurrency quiz, memory-model quiz, SPSC queue; 07-27–29 LeetCode 1780–1900 band ×11 problems). New: category M; §2.WEAK #2 difference-array/prefix-sum **retrieval** gap and #6 cyclic-index handling; §1.1 renumbered with signed/unsigned at #1 and growth-from-zero at #2; negative-modulo idiom added; checklist items 0, 5, 11, 12 added. Counts now E 14, G 14, C 12, F 11, I 11, B 9.*

*Trend: identification and paradigm selection are improving (regret greedy and greedy+diff-array both self-identified on 07-29; state augmentation correct first try on 787 and 1631). The residue is mechanical — signed/unsigned, modulo sign, sizing conventions — plus retrieval of techniques already written down. Next: the O(n) rewrite of 1871, the 1655 → 698 → 2305 bitmask drill, and a Fenwick/segment-tree unlock.*
