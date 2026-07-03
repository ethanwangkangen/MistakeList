# C++ / HFT Interview Prep — Mistake Ledger

Running record of mistakes, misconceptions, and weak areas from interview practice.
**Part 1 = C++ depth Q&A** — knowledge gaps grouped by theme, with a heatmap on top.
**Part 2 = LeetCode** — bug taxonomy with running counts and examples, identification failures, per-problem log, pre-submit checklist.

**Update protocol (per session):**
1. Every new LeetCode bug → classify into a taxonomy category (§2.1), increment its count, add a one-liner example there, and add the problem to the per-problem log (§2.4).
2. Every "reached for the wrong approach" event → row in §2.2 with root cause + trigger heuristic.
3. New C++ gaps → under their theme in Part 1; refresh the heatmap status if a theme moves.
4. Anything recurring across ≥2 sessions → promote to the relevant priority table.

> Rebuilt 2026-07-03 from ~25 sessions spanning May–July 2026, including the 152-question C++ marathon (2026-06-19), the June LeetCode debugging cluster, the paradigm-installation sessions (Jun 16 / Jun 26 / Jun 30), and the Jul 1 DP/greedy drill. Best-effort: all graded-mistake sessions mined via keyword search + chronological page-through.

---

# Part 1 — C++ Depth

## 1.0 Knowledge-Gap Heatmap

| Area | Status | Notes |
|---|---|---|
| Initialization taxonomy (value/default/aggregate, copy- vs direct-init, `explicit`) | 🔴 weak | Came up 3+ times, flagged #1 study item. Meyers 7 & 27 |
| Systems numbers (cache latencies, mispredict cost) | 🔴 memorize cold | Understood *why*, couldn't produce numbers |
| Concurrency & memory model | 🟡 in progress | Studying Williams; no graded mistakes yet — expect entries here |
| Precise-semantics precision ("80% answers") | 🟡 shaky | Many small fixes: `inline`, RAII scope, RVO/NRVO, `throw;` vs `throw e;` |
| Standard library edge cases | 🟡 shaky | `variant`, `map::operator[]`, `string_view` lifetime, strict weak ordering |
| Move semantics & value categories | 🟢 mostly solid | Edge cases remain: const-move, SSO, moved-from terminology |
| Templates & generic code | 🟢 mostly solid | ADL swap and `if constexpr` were the gaps; collapsing/decltype solid |
| Class mechanics, virtual dispatch, exceptions | 🟢 mostly solid | Terminology slips (ref cycle ≠ deadlock); ctor-throw rules to lock in |

## 1.1 Recurring / High-Priority (drill these first)

| # | Topic | The trap in one line | Frequency |
|---|---|---|---|
| 1 | `needExpand` / `capacity_ - 1` unsigned boundary | Underflow when `capacity_==0`; off-by-one at growth boundary | 🔁🔁 multi-session |
| 2 | `std::move` on `const` → silent copy | `const T&&` can't bind to `T&&`, falls back to copy ctor | ⭐ came up 2× |
| 3 | Initialization taxonomy (`new S`, value/default/aggregate) | No mental model; came up 3+ times | ⭐ flagged as #1 study item |
| 4 | `optional` `*` vs `.value()` | `*opt` empty = UB; `.value()` throws | ⭐ high-freq HFT |
| 5 | ADL `swap` idiom | `using std::swap; swap(a,b);` unqualified or custom swap never wins | ⭐ high-freq HFT |
| 6 | SSO defeats move | Short-string move costs same as copy | ⭐ high-freq HFT |
| 7 | Lifetime extension scope | Stops at first function-return-by-reference → dangles | ⭐ high-freq HFT |
| 8 | Cache latency + branch-mispredict numbers | L1~4/L2~12/L3~40/RAM~200 cyc; mispredict ~15–20 cyc | ⭐ memorize cold |
| 9 | `map::operator[]` silently inserts | Bare read mutates the map; no `operator[]` on const map — has also caused a real LC bug (min-accumulation corrupted by default-insert, see §2.1-C) | ⭐ high-freq HFT + 🔁 |
| 10 | `string_view` lifetime | View of a temporary dangles at the semicolon | ⭐ high-freq HFT |
| 11 | Strict weak ordering | `<=` comparator in `std::sort` is UB, not just wrong | ⭐ trap question |

*(Coding-side recurrences — identifier swaps, formula transcription, `lower_bound`/`upper_bound` conventions, monotonic-structure ordering — live in Part 2 §2.1, which is their system of record.)*

## Theme: Move Semantics & Value Categories

### `std::move` on a `const` object → silent copy — (2026-05-27, again 2026-06-28)
- **Wrong:** Said the rvalue-ref overload wins when you `std::move` a const object.
- **Correct:** `std::move` on `const std::string` yields `const std::string&&`. No move ctor accepts `const T&&` (moving would steal from a const), so overload resolution falls back to the **copy** ctor. Compiles, silently copies. `std::move` does *not* strip const.

### SSO defeats move — 2026-07-02
- **Wrong:** Reasoned move is always cheaper than copy.
- **Correct:** Long-string move = O(1) (steal heap pointer). **Short-string move = same cost as a copy** — SSO stores chars *inline* in the object, no pointer to steal, bytes must be physically copied. Move is not universally cheaper.

### `std::move` on a return value (pessimizing move) — 2026-07-02
- **Sharpened (got the conclusion, tightened mechanism):** `return std::move(x)` turns an elidable **prvalue** into a non-elidable **xvalue**, disabling guaranteed copy elision (C++17). Strictly no better, sometimes worse → `-Wpessimizing-move`. For cheap movable types the assembly is often identical, but you trade a language *guarantee* for an optimizer's *maybe*; for a type with a non-trivial move it becomes real work.

### NRVO disabled — output counting — (2026-06-28)
- **Wrong:** Said output was "124" (copy + assignment).
- **Correct:** It's "133". A return of a named local is an **implicit move** (since C++11) → move ctor. Then initializing from the returned rvalue is another move ctor. No copy, no assignment. `A b = foo()` is copy-init *syntax* but calls a constructor, not `operator=`.

### Moved-from state terminology — 2026-07-02
- **Wrong:** Called it "indeterminate."
- **Correct:** The term is **valid but unspecified**. "Indeterminate" = *uninitialized memory*, a different concept. You may call any no-precondition op (assign, `clear`, `size`); just can't assume the value.

### Pass-by-value-and-move (sink) — reinforced across sessions
- **The idiom:** `void set(std::string n){ member_ = std::move(n); }` — use **when the function stores/owns a copy anyway**; lets caller move in for free (rvalue → 2 cheap moves, 0 copies). `const&` always copies for the rvalue case. Not "when I want to modify." Read-only → `const&` still wins.

### `std::forward` vs `std::move` with forwarding reference — (2026-06-28, correct)
- Given `wrapper(T&& arg)` called with an lvalue: `T` deduces to `int&`; `std::forward<T>(arg)` yields `int&`, `std::move(arg)` yields `int&&`. ✅ (logged as a *correct* answer for reference.)

## Theme: Initialization & Lifetime

### `new S` vs `new S()` — (2026-06-28)
- **Wrong:** Thought `new S` zero-initializes.
- **Correct:** `new S` **default-initializes** — for a non-class/aggregate scalar member, that leaves it **indeterminate** (garbage). `new S()` value-initializes (zeros). This is the classic trap.

### Initialization taxonomy across storage — (2026-06-28)
- **Flagged as #1 study gap (no mental model, came up 3+ times).** The reference model:
  - `S s2 = S();` → **value init**: zero-init then default member initializers apply.
  - `S s3;` at **namespace/static scope** → zero-init first (static storage).
  - `S s4;` **inside a function** (automatic) → default init; members without default member initializers are **indeterminate**. Only this case gives garbage.

### Designated initializer ordering — (2026-06-28)
- **Wrong:** Said `{.z=3, .y=2, .x=1}` compiles in C++20.
- **Correct:** Does **not** compile. C++20 designated initializers must appear in **declaration order**.

### Explicit / copy-init vs direct-init — (2026-06-28)
- **Gap flagged:** No precise model for when `explicit` blocks a conversion and copy-init (`T x = ...`) vs direct-init (`T x(...)`) differ. Study Meyers items 7 & 27.

### Lifetime extension scope — 2026-07-02
- **Wrong:** Didn't know how far `const&` lifetime extension reaches.
- **Correct:** Extension applies **only to the temporary bound directly**. `const std::string& r = temp.getName();` where `getName()` returns a reference to a member → `r` dangles. Extension does NOT reach through a function returning a reference into the temporary. Rule: *follows the directly-bound temporary, stops at first function-return-by-reference.*

### Static initialization order fiasco — 2026-07-02
- **Wrong:** Left blank.
- **Correct:** Init order of statics across *different* TUs is **unspecified** → may use a not-yet-constructed object. Fix: **Construct-On-First-Use** — function-local `static` via accessor (`A& getA(){ static A a; return a; }`). Thread-safe since C++11 ("magic statics").

### `static` keyword — full taxonomy — (2026-06-18, mostly solid)
- You correctly identified **six** uses (Claude initially listed four): static locals, static data members, static member functions, static free functions (internal linkage), static globals (internal linkage), static lambdas (C++23). Related rules locked in: in-class `static` member needs an out-of-class definition unless `inline static` (C++17, avoids the ODR linker error); static storage duration → zero-init, static locals additionally dynamically initialized on first pass.

### Value init taxonomy (block-scope bare `int x;`) — 2026-07-02 (mostly correct)
- Only bare `int x;` at **block scope** is indeterminate. `int x{}`, `int x = int()` are zero everywhere; namespace/static scope is always zero.

### Default arg referencing another parameter — (2026-06-28)
- **Wrong:** Guessed `void foo(int a, int b = a){}` compiles.
- **Correct:** Doesn't compile. Default args are evaluated at the call site where other parameters aren't in scope.

### Dangling reference / `string_view` — (2026-06-28, correct)
- Returning `const std::string&` to a local dangles once the function returns; returning `std::string_view` of a local has the **same** problem. ✅

## Theme: Containers & STL

### `std::vector` reallocation total work + growth factor — 2026-07-02
- **Wrong:** Said total copy/move work is ~3N.
- **Correct:** Geometric series N + N/2 + N/4 + … **< 2N** → amortized O(1). Growth factors: **libstdc++ & libc++ = 2×, MSVC = 1.5×.** Reason for <2×: a sub-2 factor lets the allocator **reuse previously freed blocks** (their sum can exceed the next request); at exactly 2× the new block always exceeds the sum of all freed blocks, so no reuse.

### `emplace_back` real gotcha — 2026-07-02
- **Wrong:** Cited exception safety as the trap.
- **Correct:** Exception safety is fine (strong guarantee holds). Real trap: `emplace_back` forwards to constructors **including `explicit` ones** → `vector<vector<int>> v; v.emplace_back(10);` silently builds a **10-element vector**, while `push_back(10)` fails to compile. Bypasses the explicit-conversion guard.

### `emplace_back(new Derived())` failure mode — (2026-06-28)
- **Wrong:** Thought the wrong destructor gets called.
- **Correct:** No destructor runs at all. The danger is a **memory leak** — if the vector's reallocation throws before the raw pointer is adopted by a `unique_ptr`, the `new Derived()` is leaked because nothing owns it yet.

### `std::vector<bool>` — 2026-07-02 (root cause known, symptoms missed)
- **Correct root cause known** ("not really a vector"). **Concrete symptoms to add:** bit-packed specialization; `operator[]` returns a **proxy**, not `bool&`. (1) `auto b = v[i];` deduces the *proxy*; (2) can't take `bool* p = &v[i];`; (3) no contiguous `.data()` → breaks C-API interop. `sizeof(vector<bool>)` is still the normal ~24 bytes (three pointers) — `sizeof` measures the object, not the heap data.

### Container per-instance overhead — (2026-06-23, from an MLE)
- An **empty `unordered_map` is ~56 bytes** on GCC. `vector<unordered_map<...>> dp(n)` with n=1e6 → ~56 MB before storing anything. Know rough empty-container footprints (vector ~24B, string ~32B w/ SSO buffer, unordered_map ~56B, map node ~48B+) when sizing memo structures.

## Theme: `std::optional` / `std::variant`

### `operator*` vs `.value()` — 2026-07-02
- **Wrong:** Unsure.
- **Correct:** `*opt` on empty = **undefined behavior** (no check, no branch — fast). `.value()` on empty = **throws `std::bad_optional_access`**. Hot path: verify once with `if (opt)`, then use `*opt`. Mirrors `vector::operator[]` vs `.at()`.

## Theme: Templates & Generic Code

### ADL `swap` idiom — 2026-07-02
- **Wrong:** Unsure.
- **Correct:** Writing `std::swap(a,b)` in generic code forces the std version and defeats a type's cheaper custom `swap`. Idiom: `using std::swap; swap(a, b);` — unqualified call lets **ADL** find the type's own `swap`, with `std::swap` as fallback. Canonical customization-point pattern (same as `std::begin`/`begin`).

### `if constexpr` — 2026-07-02
- **Wrong:** Stated only "the true branch compiles" — missed the point.
- **Correct:** The **discarded branch is not instantiated**, so it may contain code that would be *ill-formed* for that type. A normal `if` type-checks *both* branches → hard error on the dead branch. E.g. `if constexpr (is_pointer_v<T>) return *t; else return t;` — plain `if` fails to compile `*t` when `T=int`.

### Template overload resolution (T vs T& vs T&&) — (2026-06-28)
- **Wrong:** Overload ambiguity case. Also mis-set as UB.
- **Correct:** Ambiguous overload resolution is a **compile error**, not UB. Study partial ordering + forwarding-reference deduction (Meyers 24–26).

### Partial specialization ambiguity — (2026-06-10, self-caught Claude's error)
- Reference note: `S<T,T>` vs `S<T,int>` for `S<int,int>` is **ambiguous**, not a clean pick. (You correctly caught this — kept as a known-solid point.)

### `decltype` vs `decltype((x))` — 2026-07-02 (correct)
- `decltype(x)` → `int` (entity type). `decltype((x))` → `int&` (parenthesized = expression; lvalue int expression → `int&`).

### Reference collapsing — 2026-07-02 (correct)
- `T=int&` in `T&&` → `int&`. Table: `& &`→`&`, `& &&`→`&`, `&& &`→`&`, `&& &&`→`&&`. Any `&` collapses to `&`.

## Theme: Casting & Type Punning

### `reinterpret_cast` does NOT use memcpy — (2026-06-19)
- **Wrong:** Said `reinterpret_cast` is implemented using `memcpy`.
- **Correct:** It changes the type of a pointer/reference with **zero codegen** — copies nothing. That's *why* it's dangerous and can violate strict aliasing. For value type-punning use `memcpy` (pre-C++20) or `std::bit_cast` (C++20). Dereferencing a `reinterpret_cast`ed pointer is only safe for `char*`/`unsigned char*`/`std::byte*`, casting back to the original type, or layout-compatible types.

### Strict aliasing exemptions — (2026-06-19, mostly correct)
- Exempt aliasing types are `char*`, `unsigned char*`, `std::byte*` (not `const char*` specifically — it's the underlying char family). They may alias any type; others reading through a mismatched pointer is UB (compiler may serve a stale register value).

## Theme: Class Mechanics & Exceptions

### ODR violation is not always a linker error — (2026-06-19)
- **Wrong:** Said ODR violation → linkage error.
- **Correct:** Only duplicate **non-inline** function definitions give a clean linker error. Two TUs defining the same class/inline function *differently* is **silent UB** — the linker picks one, you get bizarre runtime bugs.

### Constructor throw / destructor rules — (2026-06-28)
- **Gap flagged:** If a constructor throws, the object's destructor does **not** run (the object never fully existed); already-constructed member sub-objects *are* unwound. Core RAII rule. Destructors are implicitly `noexcept` since C++11 — a throwing destructor calls `std::terminate`.

### `delete p` vs `delete[] p` — (2026-06-19, correct)
- `new int[10]` must be freed with `delete[]`. Wrong form is UB (skips element destructors / heap corruption). ✅

### shared_ptr reference cycle — terminology — (2026-06-28)
- **Wrong:** Called two `shared_ptr`s pointing at each other a "deadlock."
- **Correct:** It's a **reference cycle** / circular reference — refcounts never reach zero → leak. Deadlock is a concurrency (lock-wait) concept. Fix: `weak_ptr` on one side.

### Virtual function default arguments — (2026-06-28, correct)
- `Base* p = new Derived; p->f();` with different default args → runs **Derived's body** but with **Base's default arg** ("Derived 10"). Default args are bound **statically** (by the pointer's static type); the function body dispatches dynamically. ✅

### Virtual call from constructor — (2026-06-28, correct)
- During `Base` construction the vptr points at `Base`'s vtable, so a virtual call resolves to `Base`'s version; a **pure** virtual called from the ctor is UB. ✅

### Non-virtual base destructor + delete-through-base — 2026-07-02 (correct)
- `Base* p = new Derived; delete p;` with a non-virtual base dtor is **UB**; only `~Base` runs (static dispatch), `~Derived` skipped → derived resources leak.

## Theme: Lambdas

### `mutable` not needed for by-reference capture — (2026-06-28)
- **Wrong:** Thought `[&count]` needs `mutable` to modify `count`.
- **Correct:** Modifying through a **reference** capture never needs `mutable`. `mutable` is only for modifying **by-value** captures (which are const by default inside the lambda body).

### Reference-capture dangling — (2026-06-28, correct)
- `[&count]` returned from a factory dangles once the enclosing function returns; switching to by-value `[count]` copies and works as a counter. ✅

## Theme: Concurrency & Memory Model

*Concurrency was deliberately paused during several Q&A sessions ("haven't studied yet"). Being addressed via Williams (ch. 5 → 6.2 → 7 → 3.2, 3.3, 8.2). Add mistakes here as they surface — currently the highest-priority prep gap.*

### `scoped_lock` vs `lock_guard` — (2026-06-28)
- **Wrong:** Said `std::scoped_lock lock(m1,m2)` vs `lock(m2,m1)` risks deadlock.
- **Correct:** `scoped_lock` with multiple mutexes uses `std::lock` internally (try-and-back-off) → **order doesn't matter**, it exists specifically to prevent multi-mutex deadlock. `lock_guard` is the sequential, order-dependent one.

*(Memory ordering, false sharing, SPSC — studied via Williams ch.5/6; no graded mistakes logged yet, mostly correct in study sessions. Watch for: seq_cst-vs-acquire/release reasoning, memcmp gotcha for `atomic<UDT>`, `compare_exchange_weak` spurious failure.)*

## Theme: Systems-Level Performance & Numbers

*Flagged in the 152-question session (2026-06-19) as a whole category: "you understand *why* things are fast/slow but can't produce the numbers or specifics." Memorize these cold.*

### Cache latency numbers (didn't know — MEMORIZE)
- L1 hit ~4 cycles, L2 ~12, L3 ~40, main memory ~200. A RAM miss is ~50× an L1 hit. This single fact drives every container/layout decision in HFT.

### Branch misprediction cost (didn't know)
- ~15–20 cycles on modern x86 (pipeline flush + restart). Motivates `[[likely]]`/`[[unlikely]]`, branchless code, PGO.

### Stack vs heap at the CPU level — (Q120, "unsure")
- Stack alloc = a subtraction on `rsp`, one instruction, ~free. Heap = `malloc`/`operator new`: walks free lists, may syscall (`mmap`/`brk`), takes locks in MT. Orders of magnitude slower + unpredictable → why HFT avoids heap on the hot path.

### `std::vector` out of capacity on `push_back` — (Q122)
- **Wrong:** Said "heap overflow."
- **Correct:** It **reallocates** — new buffer (~2× capacity), move/copy elements over, destroy old, free old. Invalidates all iterators/pointers/refs. Why `reserve()` matters, and where `noexcept` move vs copy is chosen.

### `new` vs placement new — (Q125)
- **Wrong:** Said `new` is slower "because it constructs objects as well" (implying placement new doesn't construct).
- **Correct:** Both construct. `new` = **allocate + construct**; placement new = **construct only** at memory you already own. `new` is slower because of the *allocation* step (heap walk, possible syscall, locking), not construction.

### `sizeof` a class with one virtual + one int — (Q121, correct)
- vptr (8) + int (4) + 4 padding = 16 on x86-64. ✅

### `sizeof("hello")` — (152-session)
- **Wrong:** Said 8 (thought pointer).
- **Correct:** It's **6**. String literals are `const char[N+1]` arrays, not pointers; `sizeof` doesn't decay arrays.

### C array vs `std::array` passing — (Q50)
- **Wrong:** Had the decay direction backwards.
- **Correct:** The **C array** `int arr[N]` decays to `int*`, losing size. `std::array<int,N>` is a real object — `N` is part of the type, retains size, can be passed by ref / copied / returned.

## Theme: Precise Semantics (imprecise-answer cluster from 152-session)

*The consistent failure mode flagged: "80% answers where interviewers want 100%." Each is a 5-minute fix; there are just many.*

### `inline` — actual meaning
- **Wrong:** "allows static declarations as definitions" / thought it's about optimization.
- **Correct:** It's an **ODR mechanism** telling the linker to merge identical definitions across TUs. Nothing to do with inlining/optimization directly. C++17 extended it to variables (`inline` variables).

### `std::move` — precise
- **Imprecise:** "casts to rvalue."
- **Precise:** an unconditional `static_cast<T&&>`. Generates **zero** assembly; the move ctor/assignment does the real work.

### `mutable` — primary use
- **Gap:** Only knew the lambda use.
- **Correct:** Primary use is allowing modification of a class member inside a `const` method (e.g. `mutable std::mutex`, cached values). (Also: not needed for by-reference lambda capture — see Lambdas theme.)

### RAII — scope
- **Imprecise:** Scoped it to smart pointers only.
- **Correct:** General principle of tying *any* resource's lifetime to object lifetime — mutexes, file handles, sockets, DB connections.

### `constexpr if` false branch — (repeat of if-constexpr gap)
- **Missed:** The false branch is **completely discarded** and need not even compile for that instantiation. This is what replaced many SFINAE patterns.

### Copy elision vs RVO vs NRVO
- **Imprecise:** "no copy is made" but couldn't distinguish.
- **Correct:** Copy elision is the umbrella. **RVO** = prvalue return, guaranteed since C++17. **NRVO** = named local return, still optional/non-guaranteed.

### `unique_ptr` in a vector
- **Imprecise:** Said can't copy *or* move.
- **Correct:** Can't **copy** (right). But move works — so `sort`, `emplace_back`, moving out, anything move-only is fine.

### `throw` vs `throw e` (rethrow)
- **Gap flagged:** `throw;` rethrows the current exception preserving its dynamic type; `throw e;` copies and can **slice** a polymorphic exception. Use bare `throw;` to rethrow.

## Theme: Standard Library Gaps (from 152-session)

### `std::variant` vs `union` — (didn't know variant)
- `union`: no type safety, you track the active member, reading the wrong member is UB. `std::variant`: type-safe tagged union, knows its active type, throws `std::bad_variant_access` on wrong access, works with `std::visit`. Cost: stores a type index → slightly larger. Dispatch on type via `std::visit`.

### `std::map::operator[]` insertion danger
- **Correct rule to lock:** `operator[]` **silently inserts** a default-constructed value if the key is missing — so a bare read `m[key]` mutates the map, and `operator[]` doesn't exist on a `const map`. Use `at()` (throws) or `find()` (iterator) for lookups. **This has bitten in live coding** — see §2.1-C (default-inserted 0 corrupting a min-accumulation).

### `string_view` lifetime danger
- `std::string_view sv = std::string("temp");` dangles — the temporary dies at the semicolon and `sv` points to freed memory. `const std::string&` extends the temporary; `string_view` does **not**.

### `std::array<int,0>` vs empty `std::vector<int>` — (Q99)
- **Imprecise:** Said the vector "maintains a pointer to heap memory."
- **Correct:** An empty `vector` typically does **not** heap-allocate until first insertion (just three null pointers). Real difference: `array<int,0>` is a compile-time fixed empty container that can never hold anything; the vector can grow.

### Hash map collision strategies — (Q97, partial)
- Two approaches: **chaining** (list per bucket — simple, handles high load factor, but pointer chasing kills cache) vs **open addressing** (probe for next slot — cache-friendly contiguous array, but degrades at high load factor, deletion needs tombstones). HFT answer: open addressing + linear probing, keep load factor ~0.5–0.7.

### `std::terminate` vs `std::abort` — (Q123, didn't know)
- `terminate` calls the terminate handler (default → `abort`), customizable via `set_terminate` (one hook, e.g. logging). `abort` immediately kills the process, no destructors / `atexit`.

### Determining derived type without RTTI — (Q124, didn't know)
- Store a **type enum/tag** in the base, set in each derived ctor, compare the tag. `dynamic_cast` is too slow for HFT; an int compare is ~1 cycle.

### Strict weak ordering — (didn't know)
- `std::sort`'s comparator must be irreflexive (`comp(a,a)` false), asymmetric, transitive. Using `<=` instead of `<` violates irreflexivity and is **UB**, not just wrong output. Classic trap.

### `std::unordered_map` custom key
- Needs **both** a hash (`std::hash` specialization or template param) **and** `operator==`. Rehash invalidates iterators but not references/pointers to elements.

### `volatile` — precise (pre-concurrency framing)
- Forces the compiler not to optimize away a variable's reads/writes and not to reorder *that variable's* accesses. **Not** sufficient for threading: no atomicity, no cross-thread ordering. Use `std::atomic`. (Full thread version in Concurrency theme once studied.)

---

# Part 2 — LeetCode

## 2.0 How to read this part

- **§2.1** is the system of record for *execution* bugs: categories with running counts and every logged example. New bug → find its category → increment count → add the one-liner.
- **§2.2** is the system of record for *identification* failures: reached for the wrong approach, stalled on the reframe, or under-dimensioned the state. This is a different skill from §2.1 and is tracked separately on purpose.
- **§2.3** = idioms/checklists to drill (the reusable fixes).
- **§2.4** = chronological per-problem log (the raw data feeding everything above).
- **§2.5** = the pre-submit checklist derived from the taxonomy. Run it every time.

**Headline diagnosis (stable across ~2 months):** algorithmic derivation is fast and usually correct; the bug mass is in **transcription fidelity** (correct math → incorrect code) and **fixed-convention recall** (comparator directions, argument orders, index domains). Identification failures cluster around **axis-swap reframes** (values→indices, subarrays→elements) and **state dimensioning**.

## 2.1 Bug Taxonomy — running counts + examples

Historic baseline distribution (June 20 taxonomy doc): wrong identifier ~35%, base case/init ~25%, placement/ordering ~15%, missing operations ~10%, lifetime/resource ~8%, syntax ~7%. Counts below are **logged instances** from mined sessions (May 18 – Jul 3); update them as new bugs land.

| ID | Category | Count | Trend | One-line signature |
|---|---|---|---|---|
| A | Wrong identifier / variable mix-up (incl. min↔max swap) | 7 | 🔁 steady | Right algorithm, wrong name plugged in |
| B | Formula inversion / transcription error | 7 | 🔴 highest-risk | Derivation correct in comments, coded term flipped — **silent wrong answer** |
| C | Base case / initialization | 8 | 🔁 steady | Only the one truly-free state may be 0; everything else unreachable |
| D | Off-by-one / index-domain translation | 6 | 🔁 steady | 0-indexed input vs 1-indexed dp; boundary-vs-element indexing |
| E | Ordering / direction of operations | 5 | 🔺 rising (2× on Jul 3) | Pop direction, comparator direction, read-before-pop, missing tie-break |
| F | Missing operation / dropped constraint | 4 | — | A term or requirement from the derivation never makes it into code |
| G | Numeric type / sentinel / overflow / precision | 7 | 🔁 steady | int*int, INT_MIN-as-LLONG-sentinel, double::min(), float division |
| H | Variable shadowing | 1 | — | `int l = mid` inside a loop re-declares instead of assigns |
| I | Over-engineering / wrong complexity or memory class | 7 | 🔁 steady | Extra state, redundant containers, rescans, memo where none needed |
| J | API convention misuse (fixed calling conventions) | 5 | 🔁 4+ sessions | `lower_bound`/`upper_bound` comparator argument order |
| K | State hygiene across calls | 1 | — | Mutated shared state leaks into the next feasibility call / test case |

### A — Wrong identifier / variable mix-up (7 logged)
- LC 862: wrote `prefix` where the variable was `prefixSum`.
- LC 1340: `min` instead of `max` for the left jump boundary (min/max swap).
- LC 3956: loop variable `m-1` where `k-1` was meant.
- LC 1626: transition `dp[j] + dp[i]` instead of `dp[j] + players[i].score` — compounded an in-progress dp value into itself.
- Teleport-grid Dijkstra: final answer loop read `distances[...][k]` (loop-invariant!) instead of `[i]`.
- Custom `vector<T>`: used `Element` where the template param was `T`; `std::forward<Element>` instead of `std::forward<Args>` in `emplace_back`.
- **Countermeasure:** dedicated 60-second post-coding pass checking *identifiers only*, no logic (installed 2026-06-23).

### B — Formula inversion / transcription error (7 logged) — HIGHEST RISK CLASS
These produce plausible wrong answers with no crash and no obvious trace. Two have *survived a correction cycle*.
- LC 862: derived `prefix[j] <= prefix[i] - k` in the comment, coded `k - prefix[i]` — persisted through one review pass.
- LC 2444: counting term `r - max(lastMinK, lastMaxK) + 1` instead of `min(lastMinK, lastMaxK) - lastBadIndex` (wrong anchor *and* wrong extreme).
- LC 828: recurrence subtracted **dp values** where the derivation called for **position gaps**.
- LC 1092: stored the DP-table string by lexicographic comparison where length comparison was the criterion.
- LC 3956: prefix-sum direction inverted in the transition.
- LC 813: prefix-sum direction inverted again, independently, same day.
- LC 2334: span of a stack minimum coded as `i - st[j] + 1`; the true left boundary is the previous stack entry (exclusive).
- **Countermeasure:** for any counting/contribution formula, hand-trace ONE 4-element example before running (this is what caught 2444).

### C — Base case / initialization (8 logged)
- LC 1335: initialized `dp[i][0] = 0` for **all** i; only `dp[0][0]` is genuinely free — the rest must be unreachable (`INT_MAX`).
- LC 115: base case `dp[0][j] = 1` for j > 0 (empty source can't produce a non-empty target — should be 0).
- Mirror-path DP: two base-case misses — `grid[0][0]` being a mirror must not apply redirect logic (robot starts there, never "enters"), and a mirror on the destination cell ⇒ answer 0.
- LC 992: frequency array sized without the `+1` (values go up to n), and `freq.clear()` used to "reset" — it zeroes the **size**, not the values (`assign(n+1, 0)` was needed).
- LC 862: monotonic-deque loop started at i=0 while reading `prefixSum[i-1]` → indexed `[-1]`.
- Custom `vector<T>`: `size_`/`capacity_` with no default member initializers → UB on first use.
- `unordered_map` min-accumulation corrupted by `operator[]` **default-inserting 0** on a missing key — the phantom 0 wins every `min()`. (Direct LC manifestation of Part 1 §Standard-Library `map::operator[]` trap.)
- LC 3977: `distances[source][power]` never initialized to 0 in a 2D `(node,power)` Dijkstra table. Combined with a `>=` staleness skip, the source skipped itself → whole search returned `{-1,-1}`. Base-case/init wearing a Dijkstra costume.
- **Rule to lock:** *only the single truly-free state gets 0; everything else starts unreachable.* And *never bare-read a map inside a min/max fold.* And *always initialize the source cell of a Dijkstra dist table (2D too).*

### D — Off-by-one / index-domain translation (6 logged)
- LC 2008: after binary search returned 0-based rides index j, used `dp[j]` where the 1-indexed dp needed `dp[j+1]`. Cleaner idiom: `int pos = it - rides.begin()` used directly as dp index, `dp[0]=0` absorbing the no-predecessor case.
- LC 1335: loop bound `j < min(i, d+1)` where `j <= min(i, d)` was correct.
- LC 3956: deque window entry point `i-1` instead of `i-l`.
- LC 862: result length `i - prefixIndex + 1` overcounts; with prefix indices the length is `i - j`.
- LC 2334: exclusive-boundary span (length between two exclusive fenceposts is `right - left - 1`).
- Prefix-sum DPs generally: repeated resistance to the **n+1 boundary-indexed convention** ("boundaries, not elements") — surfaced multiple times in the Jun 26 session before sticking.

### E — Ordering / direction of operations (5 logged, 2 on Jul 3)
- LC 862: monotonic-deque pop direction reversed relative to the increasing invariant that was *correctly designed*.
- Min-cost-path Dijkstra: PQ comparator `p1.second < p2.second` → **max-heap**; not a constant-factor bug, it degrades O(E log V) toward O(VE) → TLE.
- LC 1334: relaxation skip used strict `<` (still pushes equal-cost duplicates); `<=` keeps the queue tight.
- LC 2402: booked-rooms heap ordered by end time only — simultaneous frees need a **secondary tie-break on room number** (rule: lowest-numbered available room).
- LC 2334: read `st.back()` as the left boundary **before** popping `curr` — the boundary read was `curr` itself. Pop first, then peek.
- LC 3977: staleness skip written as `if (t >= dist[cell]) continue;` → evicts the **live** entry, not just stale duplicates. A popped entry carries the exact value written into its cell at push time, so equality means "owner," not "stale." Must be strict `>` (or equivalently `!=`, since a cell only ever decreases so stale ⇒ strictly greater). `>=` collapsed the whole search to `{-1,-1}` by skipping the source.
- **Pattern:** the invariant is designed correctly; the mechanical realization (which end, which direction, which order) flips. Same genus as B, applied to operations instead of formulas.

### F — Missing operation / dropped constraint (4 logged)
- LC 3956: transition missing the `+ prefix[i]` term entirely.
- LC 312: coins for the last-popped balloon missing the boundary multiplication `nums[i-1] * nums[k] * nums[j+1]`.
- LC 1793: dropped the constraint that the subarray must **contain index k**.
- LC 1334: missing stale-node skip (`if (dist > distances[city]) continue;`) after popping.

### G — Numeric type / sentinel / overflow / precision (7 logged)
- LC 3956: `INT_MIN` sentinel where the accumulation is `long long` → needed `LLONG_MIN`.
- LC 3956 **and again** LC 813 (independently, same day): `numeric_limits<double>::min()` is the smallest **positive** double, not the most negative. Use `lowest()` or `-max()`.
- Mirror-path DP: adding two values each near MOD in `int` → overflow before the `% MOD`. Widen to `long long` first.
- LC 115: `long long` overflow from explosive binomial growth in intermediate dp cells — the fix was **tightening loop bounds** to skip cells that can't contribute (clamping was the wrong fix; you correctly pushed back on it).
- LC 2334: `threshold / static_cast<float>(len)` with threshold up to 1e9 exceeds float's 24-bit mantissa → silent precision loss. **Restructure as integer cross-multiply:** `(long long)elem * len > threshold`.
- General multi-session: `int * int` intermediate before assignment to `long long` — cast an operand *before* the multiply.
- **Rules to lock:** sentinels match the accumulator's width; `lowest()` not `min()` for floating point; widen before multiplying; prefer integer cross-multiplication over any division/float comparison.

### H — Variable shadowing (1 logged — distinct category by design)
- LC 2517 (tastiness): `int l = mid;` inside the binary-search loop declared a **new** dead local instead of assigning the outer `l` → infinite loop / wrong convergence. Distinct from A: the name is *right*, the declaration is the bug. Watch every `type name =` inside loop bodies.

### I — Over-engineering / wrong complexity or memory class (7 logged)
- LC 2444: unnecessary tracking-variable resets + a special case for `minK == maxK` that the general formula already handled.
- LC 802: added an `unordered_set` for cycle detection when the 3-color `nodes` array already encoded exactly that state.
- LC 895: heap solution with an O(n) linear scan inside `push` — missed the *one-entry-per-push* paradigm (each push is its own record; never scan).
- LC 2334: rescanned the entire stack at every index → O(n²), TLE on strictly-increasing input. The idiom is **evaluate at pop time**, when an element's full span is known.
- minCost-string (recursive): memoized a recursion whose subranges are **disjoint** — the dp was written and never read; plus `vector<unordered_map>` (~56B per empty map × 1e6 → MLE) plus `string` passed **by value** down the recursion (and unused).
- Teleport-grid Dijkstra: O(m²·n²·k) teleport fan-out coded before the complexity was checked — no micro-optimization could save it; needed virtual-node-per-price-level restructuring.
- LC 1334 (minor): `unordered_map` adjacency for contiguous 0..n-1 nodes where `vector<vector<Road>>` is strictly better.
- **Countermeasures:** before adding a container/state var, name the fact it tracks — if an existing structure already encodes it, don't. Before coding a graph/DP with fan-out, multiply out the worst case. Feed monotonic structures a sorted adversarial input mentally.

### J — API convention misuse (5 logged, ≥4 separate sessions)
- `lower_bound`/`upper_bound` custom-comparator argument order — **the single most repeated convention bug** (LC 1751 and at least three other sessions):
  - `lower_bound(first, last, value, comp)` calls `comp(element, value)` — "first element NOT less than value."
  - `upper_bound(first, last, value, comp)` calls `comp(value, element)` — "first element that value is less than."
  - Mnemonic: **the thing being tested for smallness comes first.** Compiles fine with the wrong order; silently wrong results on heterogeneous (struct-field) searches.
- LC 862: uncertainty about what `upper_bound` returns relative to the target (strictly-greater position).
- **Countermeasure:** never write these comparators from memory mid-problem; recite the convention line first, then write.

### K — State hygiene across calls (1 logged)
- LC 1631: the grid was permanently mutated inside the binary-search feasibility check, corrupting every subsequent `check(mid)`. Heuristic installed: **visited-array when arrival direction doesn't affect the answer; backtrack/restore when the path itself matters.** Related discipline: memo arrays need explicit reset across LC test cases (`memset`/`assign`), since statics persist.

## 2.2 Identification Failures — approach selection & reframing

*The other half of the skill. Each row: what was reached for, what it actually was, the root cause, and the trigger to install.*

| Problem | Reached for | Actually was | Root cause | Trigger to install |
|---|---|---|---|---|
| LC 2555 | hashmaps + binary search | sliding window over sorted positions | thinking in **values**, not **indices**, on a sorted array | sorted array + "segment/window of length L" → try two-pointers/window FIRST |
| LC 2439 | pairwise local averaging greedy | binary search on answer + prefix feasibility | greedy locality never stress-tested | "minimize the maximum" → BSoA reflex; adversarial-test any local greedy before coding |
| LC 895 | heap + O(n) scan in push | freq map + stack-per-frequency-level | DS-design paradigm gap: **one entry per push** | design problems: every op should be O(1)/O(log n) *by construction*; a scan inside push/pop means the paradigm is wrong |
| LC 1494 | stalled on bitmask state meaning | dp[mask] = min semesters for that exact completed-set | state semantics: **which** set is done matters, not how you got there | bitmask DP: state = subset achieved; path-independence is the whole point |
| LC 992 | (correctly doubted the window invariant) | atMost(K) − atMost(K−1) | "exactly-K" transform not yet installed | "exactly K" on windows → difference of two atMost() passes |
| LC 828 | one previous position per char | two previous positions + position-gap contribution | state under-dimensioned; contribution counted in dp-values not gaps | contribution counting: the unit is (left-choices) × (right-choices) in **positions** |
| LC 927 | (insight correct) | three-pointer simultaneous walk | multi-pointer lockstep implementation, not identification | implementation drill, see §2.3 |
| LC 2818 | stalled on "map element to max score" | contribution counting (zone of dominance, prime-score key) + greedy spend — **two independent stages** | didn't see the problem *factors* into two known moves | when a subarray's outcome is decided by one "best" element under a ranking → per-element win-count = contribution counting; then look for a second, separate stage |
| LC 3928 | "Dijkstra twice from each node?" | single Dijkstra on a layered graph (2n nodes: empty/carrying) | state augmentation reframe (layer = carried state) | per-node mode/state → add a graph layer, not a second pass |
| LC 3977 | single dist-per-node Dijkstra | `(node, power)` state — power is a real dimension | Pareto non-dominance: a worse-time arrival with MORE power isn't dominated (power can gate later edges), so neither coordinate alone dominates | Dijkstra with a carried resource → ask "can a worse-primary arrival with better-resource ever be needed later?" If yes, resource is a state dimension (or a `settled[node]=best-resource` frontier prune if the resource range is large) |
| Teleport-grid | coded the fan-out directly | virtual nodes per price level | complexity not multiplied out before coding | write the edge-count formula BEFORE implementing any fan-out construction |
| LC 813 vs 1043 | conflated the two k's | 813: k caps **group count**; 1043: k caps **group length** | imprecise structural read of the constraint | when k appears, say out loud what k bounds before designing state |
| Recurring | prose instinct → no state | the prose WAS the state definition | NL→state conversion gap ("all different possible total rewards" = the dp axis) | procedure: enumerate the choices at position i first; the state is whatever those choices need to know |
| Recurring | — | prefix-sum ↔ subarray-sum link | needs prompting to connect "subarray sum" to prefix differences | "subarray sum/count with condition" → prefix quantity + hashmap/sorted structure over prefixes |
| Recurring | element-indexed prefix DP | boundary-indexed (n+1) convention | resistance to fencepost indexing | prefix DPs index **boundaries**, not elements; size n+1, answer at dp[n] |

**Solid recognitions (for contrast, keep calibrated):** Kadane (LC 2606) instant; BSoA (LC 2616, and LC 2439 on second look) reliable; knapsack connection (LC 871) self-identified; exchange-argument skeleton installed; LC 2551/1899 solved fast but with prior exposure — freshness there unverified.

**Deferred topics:** LC 2839 (coordinate-compressed segment trees) — revisit after studying segment trees / BIT.

## 2.3 Idioms & Checklists to Drill

**Difference array — O(1) range-update + point-query (learned 2026-07-03, LC 2528):**
Turns "add `v` to every index in `[L,R]`" into two O(1) edits: `diff[L] += v; diff[R+1] -= v;`. A running prefix sum over `diff` (left-to-right) reconstructs each position's live total — each interval's `+v` turns ON at `L`, OFF at `R+1`, so at position `i` the running sum = sum of all `v` whose interval covers `i`. Range-update + point-query, both O(1) amortized, one pass. Avoids the O(n·r) rescan (category I) when many overlapping range-adds accumulate during a sweep. Two traps: size `diff` as `n+1` so `diff[R+1]` at `R=n-1` is in-bounds (C/D off-by-one); `long long` throughout when values accumulate (G). Canonical use: feasibility checks in binary-search-on-answer where each forced action adds power/coverage over a forward range (LC 2528, 1109, 370, 1094).

**Binary-search-on-answer verification ritual (reinforced 2026-07-03):**
Before coding BSoA, state the monotonicity sentence out loud: "if target `x` is feasible, every `x' < x` is feasible (same solution already clears the lower bar); if `x` infeasible, `x+1` also infeasible (needs ≥ as many resources)." That one argument is the license. Then: bounds (lo = current worst, hi = worst + budget), and an O(n) greedy `feasible(x)`. The check is where the bugs live, not the search.

**Monotonic stack — pop-time evaluation (LC 84 / 907 / 1856 / 2334 family):**
An element's full span-as-minimum is known exactly when it's **popped**. Never rescan the live stack. Left boundary = element *below* it after popping (exclusive); right boundary = the incoming index (exclusive); span = `right - left - 1`. Append a sentinel (e.g. −1 / 0) to flush the stack at the end.

**Contribution counting — strict/non-strict asymmetry:**
Count per element: `(i − left[i]) × (right[i] − i)`. With duplicates, symmetric strictness double-counts: use **strictly** on one side, **or-equal** on the other (e.g. previous strictly-smaller, next smaller-or-equal; or the problem's stated tie-break, as in 2818's index rule). Max-over-spans (histogram) doesn't care; sum-over-counts does. Shared engine: *element + maximal zone of dominance*; the knobs are (a) max vs sum, (b) tie-break strictness.

**Dijkstra checklist (assembled from 4 sessions of bugs):**
`priority_queue<pair<long long,int>, vector<...>, greater<>>` with `{dist, node}` — greater<> on that pair order gives the min-heap free (no hand-written comparator to invert). Pop → `if (d > dist[u]) continue;` (stale skip — **strict `>`, never `>=`**; the popped entry equals its cell by construction, so `>=` evicts the live owner). Push only on strict improvement `nd < dist[v]`. `long long` distances. `vector<vector<>>` adjacency for contiguous node ids. **Initialize the source cell** (`dist[source]=0`, and `dist[source][fullResource]=0` for augmented state) before the loop.
- **State sizing (decide FIRST):** single dist-per-node suffices only if no carried resource can gate future edges. If a worse-primary-key arrival with a better secondary resource can be needed later (Pareto non-dominance), the resource is a **state dimension** `(node, resource)` — table if the resource range is small, or a `settled[node] = best-resource-finalized` frontier prune (pop in primary-then-resource order, skip `resource <= settled[node]`) if the range is large. Never a second pass.
- **Early return at target** is valid when the PQ order makes the first target-pop already optimal on all keys (e.g. `{time, -power}` → first pop is min-time, max-power).

**Process discipline — patch vs refactor:** when a design has the right *state, comparator, and relaxation*, diagnose whether failures are 1-line bugs (uninit cell, wrong inequality) BEFORE rewriting the approach. LC 3977 was two 1-line patches from correct; a full `settled[]`/Pareto refactor was explored unnecessarily. Cost: real time. Tell: if the shape is sound and only edge cases fail, you have bugs, not the wrong algorithm.

**`lower_bound` / `upper_bound` conventions:**
lower → `comp(element, value)`; upper → `comp(value, element)`; "the thing tested for smallness comes first." Returns: lower = first ≥, upper = first >. For struct fields, the heterogeneous comparator's parameter order must match the function, not your intuition.

**Binary search hygiene:** right-bound template uses `mid = l + (r - l + 1) / 2` (avoid the l==r-1 stall); assignments in the loop are `l = mid;` — any `int l = ...` inside the loop is shadowing (H).

**Exactly-K on windows:** `exactly(K) = atMost(K) − atMost(K−1)`.

**0-indexed input → 1-indexed dp:** compute `pos = it - v.begin()` and use it directly as the dp index with `dp[0]` as the empty-prefix base; do not juggle a `j`/`j+1` pair.

**Knapsack 1D collapse:** 2D `dp[i][w]` → 1D by iterating capacity **in reverse** (forward reuses this round's items).

**Grids in DP:** flat `vector<int>` or global C array with manual indexing over `vector<vector<int>>` when bounds are known (cache locality; from the heap/stack session).

## 2.4 Per-Problem Log (chronological, append-only)

Format: `LC # (date) — bugs [category letters] / identification notes`

- **LC 1793** (05-18) — dropped must-contain-k constraint [F].
- **LC 1631** (05-18) — grid mutated across feasibility calls [K]; visited-vs-backtrack heuristic installed.
- **LC 1092** (05-18) — lexicographic vs length comparison in dp table [B].
- **LC 115** (05-19) — base case dp[0][j]=1 [C]; binomial-growth overflow, fixed by tightening loop bounds (pushed back correctly on clamping) [G].
- **LC 312** (05-19) — missing boundary multiplication in last-balloon coins [F].
- **LC 813** (06-04) — interval-DP + prefix-sum debugging session; base-case/loop-bound framework installed (domains → base from recurrence bottom-out → bounds follow).
- **LC 1335** (06-10) — dp[i][0]=0 for all i [C]; loop bound off-by-one [D].
- **LC 1626** (06-15) — dp[j]+dp[i] self-compounding transition [A].
- **LC 1499 / 1937 / 630** (06-16) — paradigm session: recurrence derivation, algebraic separation, greedy hypothesis stress-testing; no execution bugs logged.
- **LC 3956** (06-17) — wrong loop var m-1/k-1 [A]; prefix direction [B]; missing +prefix[i] [F]; deque entry i-1 vs i-l [D]; INT_MIN sentinel [G]; double::min() [G].
- **LC 813** (06-17, revisit) — double::min() repeated independently [G]; prefix direction again [B].
- **LC 3928** (06-17) — identification: layered-graph single Dijkstra, not "Dijkstra twice" [§2.2]. Dijkstra habits installed: lazy deletion, long long.
- **LC 2267** (06-18) — memoization added cleanly; tri-state memo encoding + open>remaining pruning installed; `inline static` / memset-across-testcases discipline (feeds [K]).
- **Min-cost-path w/ reversals** (06-20) — inverted PQ comparator → max-heap TLE [E]. (Correctly rebutted the stale-skip misdiagnosis — the dist-check already covered it.)
- **Mirror-path count** (06-20) — MOD overflow in int [G]; start-cell-mirror and destination-mirror base cases [C ×2]. (Correctly argued direction belongs in pull-DP state.)
- **LC 3815** (06-20) — DS selection Q&A (map vs unordered_map for ordered max) — no bugs.
- **LC 2008** (06-21) — dp[j] vs dp[j+1] index translation [D]; `auto ride =` copies a vector per iteration (perf hygiene).
- **minCost-string** (06-23) — string by value down recursion (unused) + vector<unordered_map> footprint → MLE; memo on disjoint subranges (written, never read) [I ×3, G-adjacent].
- **LC 895** (06-23) — O(n) scan in push; one-entry-per-push paradigm [I, §2.2].
- **LC 927** (06-23) — insight right, 3-pointer lockstep implementation stalled [§2.2].
- **LC 1340** (06-23) — min-for-max left boundary [A]; spurious leftMax/rightMax blocking logic [I-adjacent].
- **LC 828** (06-23) — dp-values instead of position gaps [B]; needed two prev positions per char [§2.2].
- **LC 802** (06-23) — redundant unordered_set beside 3-color array [I].
- **LC 3395** (06-23) — worked; no graded bugs logged.
- **LC 1334** (06-24) — missing stale skip [F]; `<` vs `<=` relaxation [E]; unordered_map adjacency [I-minor].
- **Teleport-grid Dijkstra** (06-24) — final loop [k] vs [i] [A]; uncounted O(m²n²k) fan-out → restructure via virtual nodes [I, §2.2].
- **LC 2517** (06-25) — `int l = mid` shadowing [H]; right-bound mid formula locked.
- **LC 1494** (06-25) — bitmask state semantics stall [§2.2].
- **LC 992** (06-25) — `if (have == k)` counting guard (count all valid windows) [B-adjacent]; freq sizing n+1 + `.clear()` misuse [C, D]; exactly-K transform installed [§2.2].
- **Paradigm marathon** (06-26: 2606, 828-deep, 2281, 2616, 2640, div-deletions, 2818) — contribution-counting engine + strict/non-strict installed; boundary-indexing resistance surfaced repeatedly [D]; unordered_map default-insert corrupting min [C]; NL→state conversion gap named [§2.2]; 2818 two-stage factorization stall [§2.2].
- **LC 862** (06-28) — prefix vs prefixSum [A]; i=0 reading [-1] [C]; k−prefix[i] inversion, survived a review pass [B]; length i−j not i−idx+1 [D]; deque pop direction [E]; upper_bound semantics uncertainty [J].
- **LC 2444** (06-29) — counting formula wrong anchor+extreme [B]; unnecessary resets + minK==maxK special case [I].
- **LC 2555** (06-29) — hashmap+BS where sliding window on sorted positions [§2.2].
- **LC 2439** (06-29) — local pairwise greedy → BSoA on counterexample [§2.2].
- **LC 2551 / 1899** (06-29) — solved fast, prior exposure (freshness unverified).
- **LC 1751** (07-01) — lower_bound comparator argument order [J].
- **LC 871** (07-01) — knapsack connection self-identified; lazy-greedy (heap regret) + 2D→1D reverse-iteration collapse installed.
- **LC 673 / 813 / 494 / 1043** (07-01) — DP state-dimension drill; 813-vs-1043 k-semantics conflation [§2.2]. (Also correctly caught the "index vs accumulate" heuristic's knapsack counterexample.)
- **LC 2402** (07-03) — heap comparator missing room-number tie-break for equal end times [E].
- **LC 2334** (07-03) — span i−st[j]+1 vs previous-entry boundary [B/D]; full-stack rescan O(n²) [I]; float threshold precision → integer cross-multiply [G]; read st.back() before popping curr [E].
- **LC 3977** (07-03) — Minimum Time to Reach Target With Limited Power (state-augmented Dijkstra). Uninitialized `distances[source][power]=0` [C]; `>=` staleness skip evicting the live equal-time entry (must be strict `>` or `!=`) [E]; initially under-dimensioned state (single dist/node) before recognizing power must be a `(node,power)` dimension [§2.2, state-sizing]. Process note: approach was sound throughout — needed two 1-line patches, not the `settled[]`/Pareto refactor that was explored. See §2.3 Dijkstra checklist additions.
- **LC 2528** (07-03) — Maximize the Minimum Powered City. BSoA correctly self-identified (monotonic feasibility). **Technique learned:** difference array for O(1) range-update during the greedy `feasible(x)` sweep — was the missing piece (didn't know it). Greedy placement = push forced stations as far right as still covers the deficient city (`[i, i+r]`), maximizing forward reach. See §2.3 difference-array + BSoA-ritual idioms. No bug logged; new tool acquired.

**Custom-implementation cluster (vector / shared_ptr from scratch, May–June):**
`Element` vs `T`, `forward<Element>` [A]; `size_`/`capacity_` uninitialized [C]; `needExpand` / `capacity_-1` unsigned underflow at capacity 0 [D/G — multi-session, systematic]; missing const `operator[]`; missing deallocation in `reserve`/destructor; no downsize guard in `reserve` → overflow; `deallocate(nullptr,…)` from ctor path; dead try-catch around noexcept dtor; shared_ptr: control block deleted while holding its own mutex (UB); refcount race between pointer copy and increment → `atomic<size_t>`.

## 2.5 Pre-Submit Checklist (run every problem — ~90 seconds)

1. **Identifiers only** (A/H): read every variable name against intent; any `type name =` inside a loop body is a shadowing suspect.
2. **Formula trace** (B): any counting/contribution/transition formula → hand-trace one 4-element example. Non-negotiable; this class is silent.
3. **Comparators** (E): heap direction (`<` in a pq comparator = MAX-heap); strict `<` for sort; tie-break needed when keys can collide?
4. **Numerics** (G): widen before `*`; sentinel matches accumulator width; `lowest()` never `min()` for floats; integer cross-multiply over division/float.
5. **Boundaries** (C/D): first iteration (reads at −1?), last (sentinel flush?), empty structure, dp base = only the truly-free state, n+1 sizing for boundary-indexed prefix arrays.
6. **Conventions** (J): lower/upper_bound comparator order recited before written; no bare `m[key]` reads inside min/max folds.
7. **State hygiene** (K): anything mutated that survives into the next call/iteration/test case?
8. **Complexity sanity** (I): worst-case input named (sorted/increasing array for stacks, dense fan-out for graphs); every container/state var justified by a fact nothing else tracks.

---

*Last updated: 2026-07-03 (LC 3977, 2528 appended) · Sessions mined: ~25 (May 18 – Jul 3) · Logged execution bugs: ~54 · Logged identification events: 15 · Techniques logged: difference array, BSoA ritual*
