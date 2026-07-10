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

## 1.2 Future Study TODO (flagged weak, not yet drilled)

| Topic | Why flagged | Scope to cover |
|---|---|---|
| `std::function` | Flagged 2026-07-10; known related gap: type-erasure overhead | Type erasure mechanics, SBO (small buffer optimization), heap-alloc threshold, call overhead vs raw fn ptr / templated callable, when HFT code avoids it (`function_ref`, templates, CRTP alternatives), `std::move_only_function` (C++23) |
| Exceptions | Flagged 2026-07-10; scattered partial entries (ctor throw, `throw;` vs `throw e;`, dtor `noexcept`, `terminate` vs `abort`) but no unified model | Exception guarantees (nothrow/strong/basic), stack unwinding mechanics + cost model (zero-cost tables, why HFT bans them on hot paths), `noexcept` semantics & interaction with move (`move_if_noexcept`), function-try-blocks, exception_ptr, rethrow rules — consolidate the scattered entries into one drill |

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

## Theme: Raw Memory, Object Lifetime & Allocators (vector build) — 2026-07-10

### Assignment into uninitialized storage vs placement new
- **Wrong:** Proposed `data_[size_] = std::move(element);` in `push_back` writing into raw `operator new` memory.
- **Correct:** No `Element` object is alive at `data_[size_]` — move-*assignment* reads/releases the destination's existing state (garbage) → **UB**. Assignment requires a live object; construction creates one. Must placement-new: `new (data_ + size_) Element(std::move(element));`
- **Why it "worked" anyway (lock this in):** (1) trivial types — assignment and construction emit the identical store instruction; (2) fresh OS pages are zeroed, and zeroed bytes often masquerade as a valid empty object (`delete nullptr` is a no-op); (3) UB is a license, not an obligation — `-O0` today ≠ `-O3` tomorrow. "It worked" is not evidence of correctness. Detonates with recycled buffers holding stale pointers. ASan won't catch it (lifetime, not bounds); MSan or `constexpr` evaluation will.

### `move_if_noexcept` — where it belongs
- **Gap:** Asked whether `push_back`'s by-value parameter needs it. No — the parameter is already the callee's; if construction throws, `size_` is untouched → strong guarantee for free.
- **Correct placement:** the **reallocation loop** (old buffer → new buffer). A throwing move mid-loop leaves the old buffer half-gutted with no rollback → fall back to copying unless the move ctor is `noexcept`. This is *why* move ctors should be marked `noexcept`.

### Per-element `delete` on elements inside one allocation
- **Wrong:** Proposed calling `delete` on each element in `reserve` teardown to "destroy + free in one step."
- **Correct:** `delete p` = `p->~T()` + `operator delete(p)`. `data_ + i` was never returned by an allocation — the heap has no metadata for a mid-block pointer → heap corruption. Even `delete data_` is wrong (one dtor only, and pairs `delete` with raw-`operator new` memory). Pattern: explicit dtor calls (**reverse order**, mirroring construction) + **one** `operator delete(data_)`.
- **Rule:** `new T` fuses allocate+construct, so `delete` fuses destroy+deallocate. You unfused the front (raw alloc + placement new) → you must unfuse the back. Pairing never mixes: `new`/`delete`, `new[]`/`delete[]`, `operator new`/`operator delete`, `malloc`/`free`.

### `allocator::deallocate(ptr, n)` — sized deallocation contract
- **Learned:** `n` must equal the count passed to the matching `allocate` — for a vector, the **capacity, not the size**. Wrong `n` is **UB**, not a checked error (std::allocator may tolerate it by coincidence; a pool allocator puts the block on the wrong free list).
- **Why the interface wants the size back:** caller-remembers-size lets allocators (pools, arenas, size-class free lists) skip per-allocation metadata entirely — the perf rationale behind C++14 sized `operator delete`.
- **Bug pattern to watch:** in `reserve`, deallocating the *old* buffer with the *new* capacity after overwriting `cap_`. Save old capacity first. Destroy count (`size_`) ≠ deallocate count (`cap_`). Go through `allocator_traits` (`destroy`, `deallocate`).

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

## 2.WEAK — Weak Areas & Study Priority (consolidated 2026-07-03)

Ordered by ROI for the Aug 2026 application cycle. ⚠️ = self-identified; ◆ = surfaced from ledger evidence.

**Paradigm gaps (highest ROI first):**
1. ⚠️◆ **Greedy category recognition** — executing greedy is fine; recognizing *which* of the 13 categories and proving correctness fast is the gap. Weakest sub-type: regret/heap greedy (LC 871 family) — **confirmed 07-06: LC 630 revisit stalled despite 871 install; retention, not just recognition**. Keep forgetting the taxonomy. Broadest gap, appears everywhere.
2. ⚠️◆ **Bitmask DP (submask/assignment flavor)** — stalled on LC 1723 (Jun) and LC 1655 (Jul), same "what do the bits index" stall. Fix in progress. Drill set scoped: **1655 → 698 → 2305** (same submask move ×3). Distinguish single-bit-transition (2^m·m) from whole-submask-transition (3^m, needs `(sub-1)&mask` loop).
3. ◆ **Fenwick / segment tree + coordinate compression** (one bundle) — largely untouched; unlock the same P90 problems. LC 2839 deferred on this; LC 327 (Count of Range Sum) KIV'd for lack of BIT/merge-sort-counting; **LC 2407 (07-06) KIV'd at the same wall** — identification complete, implementation blocked on the structure. Near-perfect first drill problem when this is picked up.
4. ⚠️ **Tries** — untouched, narrower, faster to learn once. LC 421 → 1707 queued as the drill pair (07-06).
4b. ◆ **Permutation-counting DP** — new, surfaced 07-06 (LC 1866): insertion-by-rank + rank-bijection unknown. Small family, cheap to close: drill 629 → 920 → 1359. Rarer in HFT OAs than #1–3 — take for momentum, not priority.

**Cross-cutting skills (not paradigms, but recurring point-losers):**
5. ◆ **Axis-swap reframe** — the dominant *identification* failure (§2.2): subarrays→elements, values→indices (LC 2555, 2818, 828). Stalls when the problem needs flipping what you iterate over.
6. ◆ **Recurrence transcription** — category B, highest-frequency bug class (7 logged, 2 survived review). Correct math, wrong code. Countermeasure: hand-trace one 4-elem example before running.
7. ◆ **State-dimension sizing** — when to add a dimension (LC 3977 node vs (node,power); 813-vs-1043 what k bounds). Distinct from knowing the paradigm.
8. ◆ **Reconstruction / traceback** — DPs asking for the actual answer not its value (LC 1723 parent array). General weak spot.
9. ◆ **Type-width discipline** — category G, recurring (LC 2528 today). Constraint-glance pre-pass in §2.3.
10. ◆ **Prefix-sum ↔ subarray-sum link + boundary (n+1) indexing** — needs prompting to see; resisted fencepost convention repeatedly (Jun 26/28).

**Explicitly DEPRIORITIZED (know they exist, don't drill — low/zero ROI for HFT):**
digit DP, convex-hull-trick / Li Chao, suffix automata / heavy string algos, most interval DP.

**Strengths (for calibration — these were gaps, now solid):** monotonic-stack contribution counting (Jun), Kadane recognition, binary-search-on-answer, exchange-argument skeleton, Dijkstra habits (post-3977).

---

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
| C | Base case / initialization | 10 | 🔺 2× on Jul 6 | Only the one truly-free state may be 0; everything else unreachable — allocation sizing included |
| D | Off-by-one / index-domain translation | 6 | 🔁 steady | 0-indexed input vs 1-indexed dp; boundary-vs-element indexing |
| E | Ordering / direction of operations | 7 | 🔺 rising (2× Jul 3, 2× Jul 7) | Pop direction, comparator direction, read-before-pop, missing tie-break, guard asymmetry |
| F | Missing operation / dropped constraint | 5 | — | A term or requirement from the derivation never makes it into code |
| G | Numeric type / sentinel / overflow / precision | 9 | 🔴 2× on Jul 7 | int*int, INT_MIN-as-LLONG-sentinel, double::min(), float division, int accumulator/dist |
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

### C — Base case / initialization (10 logged, 2 on Jul 6)
- LC 1335: initialized `dp[i][0] = 0` for **all** i; only `dp[0][0]` is genuinely free — the rest must be unreachable (`INT_MAX`).
- LC 115: base case `dp[0][j] = 1` for j > 0 (empty source can't produce a non-empty target — should be 0).
- Mirror-path DP: two base-case misses — `grid[0][0]` being a mirror must not apply redirect logic (robot starts there, never "enters"), and a mirror on the destination cell ⇒ answer 0.
- LC 992: frequency array sized without the `+1` (values go up to n), and `freq.clear()` used to "reset" — it zeroes the **size**, not the values (`assign(n+1, 0)` was needed).
- LC 862: monotonic-deque loop started at i=0 while reading `prefixSum[i-1]` → indexed `[-1]`.
- Custom `vector<T>`: `size_`/`capacity_` with no default member initializers → UB on first use.
- `unordered_map` min-accumulation corrupted by `operator[]` **default-inserting 0** on a missing key — the phantom 0 wins every `min()`. (Direct LC manifestation of Part 1 §Standard-Library `map::operator[]` trap.)
- LC 3977: `distances[source][power]` never initialized to 0 in a 2D `(node,power)` Dijkstra table. Combined with a `>=` staleness skip, the source skipped itself → whole search returned `{-1,-1}`. Base-case/init wearing a Dijkstra costume.
- LC 1478: **over-broad base-case loop** — set `dp[1][j] = 0` for ALL j including `j=0` (one house, zero mailboxes = infeasible marked feasible) → DP covered the first house "for free," answer too small. `dp[1][j≥1]` was derivable from the recurrence anyway; hand-writing a second base-case row is where the slip crept in. Same genus as LC 1335's dp[i][0]=0.
- LC 2218: **allocation-size flavor** — `vector<int>(k, 0)` inner dimension while the loop writes `dp[i][j]` up to `j=k`; needs `k+1`. Outer dimension was correctly `n+1` *on the same line* — inconsistent sizing within one allocation statement is the tell. Caught by ASan ("right of region", small offset); `.at()` substitution pinpoints the subscript instantly.
- **Rule to lock:** *only the single truly-free state gets 0; everything else starts unreachable.* And *never bare-read a map inside a min/max fold.* And *always initialize the source cell of a Dijkstra dist table (2D too).* And *if a state is derivable from the recurrence, don't hand-write it as a base case.* And *after writing a DP allocation, immediately eyeball BOTH dimensions against the loop bounds as pairs (`j < k+1` ↔ `k+1` slots) — 10 seconds, kills the whole class.*

### D — Off-by-one / index-domain translation (6 logged)
- LC 2008: after binary search returned 0-based rides index j, used `dp[j]` where the 1-indexed dp needed `dp[j+1]`. Cleaner idiom: `int pos = it - rides.begin()` used directly as dp index, `dp[0]=0` absorbing the no-predecessor case.
- LC 1335: loop bound `j < min(i, d+1)` where `j <= min(i, d)` was correct.
- LC 3956: deque window entry point `i-1` instead of `i-l`.
- LC 862: result length `i - prefixIndex + 1` overcounts; with prefix indices the length is `i - j`.
- LC 2334: exclusive-boundary span (length between two exclusive fenceposts is `right - left - 1`).
- Prefix-sum DPs generally: repeated resistance to the **n+1 boundary-indexed convention** ("boundaries, not elements") — surfaced multiple times in the Jun 26 session before sticking.

### E — Ordering / direction of operations (7 logged, 2 on Jul 3, 2 on Jul 7)
- LC 862: monotonic-deque pop direction reversed relative to the increasing invariant that was *correctly designed*.
- Min-cost-path Dijkstra: PQ comparator `p1.second < p2.second` → **max-heap**; not a constant-factor bug, it degrades O(E log V) toward O(VE) → TLE.
- LC 1334: relaxation skip used strict `<` (still pushes equal-cost duplicates); `<=` keeps the queue tight.
- LC 2402: booked-rooms heap ordered by end time only — simultaneous frees need a **secondary tie-break on room number** (rule: lowest-numbered available room).
- LC 2334: read `st.back()` as the left boundary **before** popping `curr` — the boundary read was `curr` itself. Pop first, then peek.
- LC 3977: staleness skip written as `if (t >= dist[cell]) continue;` → evicts the **live** entry, not just stale duplicates. A popped entry carries the exact value written into its cell at push time, so equality means "owner," not "stale." Must be strict `>` (or equivalently `!=`, since a cell only ever decreases so stale ⇒ strictly greater). `>=` collapsed the whole search to `{-1,-1}` by skipping the source.
- LC 2528: **initial consume-index alignment error [Ethan's]** — first draft consumed the diff array at the wrong index; corrected to `adjustments[leftBound]`. The correct pairing: a station placed at slot `i+r` (to cover deficient city i) covers `[i, i+2r]`; consuming at `leftBound = j-r` and cancelling at `i+r+1` makes the turn-off fire exactly at city `j = i+2r+1` (window-relative, not absolute-index). This is an alignment bug → **draw-the-array pre-pass** (§2.3) is the countermeasure. *(Note: Claude later wrongly claimed the corrected `leftBound` version was still buggy and pushed "consume at i"; a 200k stress test proved the `leftBound` version correct. So Ethan's final code was right; the initial draft was the miss.)*
- LC 2528: cancellation accumulates with `-=` at the endpoint (`adjustments[i+r+1] -= add`), never `=`, so multiple cities cancelling at the same boundary don't overwrite each other. (Note: Ethan's passing submission already used `-=` correctly; general rule stands — diff arrays accumulate at endpoints.)
- LC 1976 (07-07) — **RETENTION MISS of the 3977 entry above.** Applied `>=` to the pop-time staleness check (source skipped itself, dist=0 == distances[0]) when the `>=` belonged on the *relaxation* skip. The asymmetry to lock: **on pop, equal = owner → process (strict `>`); on relax, equal = no improvement → skip (`>=`).** Second occurrence of the same confusion in 4 days; cross-ref 3977 + 1334.
- LC 2547 (07-07): inner split loop walks j **forward from 0** while the candidate segment is `[j..lastIdx]` — the freq map `adj` was built by *adding* elements as j advances, but advancing j **shrinks** the segment from the left, so `adj` held the wrong element set at every cost evaluation. Direction of accumulation must match direction of segment growth: iterate j from lastIdx downward (segment grows leftward, map only ever adds).
- **Pattern:** the invariant is designed correctly; the mechanical realization (which end, which direction, which order) flips. Same genus as B, applied to operations instead of formulas.

### F — Missing operation / dropped constraint (5 logged)
- LC 3956: transition missing the `+ prefix[i]` term entirely.
- LC 312: coins for the last-popped balloon missing the boundary multiplication `nums[i-1] * nums[k] * nums[j+1]`.
- LC 1793: dropped the constraint that the subarray must **contain index k**.
- LC 1334: missing stale-node skip (`if (dist > distances[city]) continue;`) after popping.
- LC 1888 (07-07): entire **type-2 operation (free rotation) never entered the model** — the suffix DP solved the no-rotation problem. Root cause was deeper than a dropped term: rotation is a *global re-alignment*, not a per-position choice, so it cannot be spliced into a local recurrence at all (see §2.2 row). Also symptomatic: the loop never read `s[i]`, and dp[i][0]/dp[i][1] were assigned identical expressions — a degenerate state dimension is a tell that the model is wrong.

### G — Numeric type / sentinel / overflow / precision (9 logged, 2 on Jul 7)
- LC 3956: `INT_MIN` sentinel where the accumulation is `long long` → needed `LLONG_MIN`.
- LC 3956 **and again** LC 813 (independently, same day): `numeric_limits<double>::min()` is the smallest **positive** double, not the most negative. Use `lowest()` or `-max()`.
- Mirror-path DP: adding two values each near MOD in `int` → overflow before the `% MOD`. Widen to `long long` first.
- LC 115: `long long` overflow from explosive binomial growth in intermediate dp cells — the fix was **tightening loop bounds** to skip cells that can't contribute (clamping was the wrong fix; you correctly pushed back on it).
- LC 2334: `threshold / static_cast<float>(len)` with threshold up to 1e9 exceeds float's 24-bit mantissa → silent precision loss. **Restructure as integer cross-multiply:** `(long long)elem * len > threshold`.
- LC 2528: **narrowing at a function boundary** — `mid` was `long long` in the caller but `valid(..., int minimumPower)` truncated every candidate above ~2.1e9 silently. Also `vector<int> adjustments` truncating `long long` deltas. Constraints: k ≤ 1e9, windowed power ≤ n·max = 1e5·1e5 = 1e10 — both exceed int.
- General multi-session: `int * int` intermediate before assignment to `long long` — cast an operand *before* the multiply.
- LC 2439 (07-07, revisit): `accumulate(begin, end, 0)` — the **literal `0` sets the accumulator type** to int; sum reaches ~1e14 (n=1e5 × 1e9). Use `0LL` — or here, `r = *max_element` (tighter bound, stays in int). Irony logged: `backLog` inside the checker was correctly widened, the line *above* the binary search wasn't. Constraint-glance pre-pass would have caught it.
- LC 1976 (07-07): `vector<int> distances` with path costs up to n·w ≈ 200·1e9 = 2e11; also `road.cost + dist` overflows in int *before* the comparison runs. Widen the dist table AND the PQ payload together.
- **Rules to lock:** sentinels match the accumulator's width; `lowest()` not `min()` for floating point; widen before multiplying; prefer integer cross-multiplication over any division/float comparison. **A value's type must not narrow as it crosses a function boundary — if a `long long` is passed in, the parameter is `long long`.** Discipline (interview-legitimate, NOT blanket-casting): before coding, glance at constraints, mark every quantity that can exceed 2.1e9 (accumulators + products are the triggers), type *those* `long long` and justify aloud; leave indices/`r`/single elements `int`. Blanket `long long` reads as not understanding types — a smell in HFT interviews specifically.

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
| LC 1478 | "where does the last mailbox go?" (continuous position as the DP choice) | sort → contiguous blocks (exchange argument) → partition DP over split points + closed-form median cost per block | the reduction *in front of* the DP wasn't automatic — prior partition-DP reps (word break, palindromes) hand you sequence order for free; here contiguity had to be earned | geometric/assignment problem on a line → sort, prove per-server blocks are contiguous via exchange argument, THEN it's partition DP with precomputed cost(l,r). Drill family: 410 (DP form), 813, 1959 |
| LC 3977 | single dist-per-node Dijkstra | `(node, power)` state — power is a real dimension | Pareto non-dominance: a worse-time arrival with MORE power isn't dominated (power can gate later edges), so neither coordinate alone dominates | Dijkstra with a carried resource → ask "can a worse-primary arrival with better-resource ever be needed later?" If yes, resource is a state dimension (or a `settled[node]=best-resource` frontier prune if the resource range is large) |
| Teleport-grid | coded the fan-out directly | virtual nodes per price level | complexity not multiplied out before coding | write the edge-count formula BEFORE implementing any fan-out construction |
| LC 813 vs 1043 | conflated the two k's | 813: k caps **group count**; 1043: k caps **group length** | imprecise structural read of the constraint | when k appears, say out loud what k bounds before designing state |
| Recurring | prose instinct → no state | the prose WAS the state definition | NL→state conversion gap ("all different possible total rewards" = the dp axis) | procedure: enumerate the choices at position i first; the state is whatever those choices need to know |
| Recurring | — | prefix-sum ↔ subarray-sum link | needs prompting to connect "subarray sum" to prefix differences | "subarray sum/count with condition" → prefix quantity + hashmap/sorted structure over prefixes |
| Recurring | element-indexed prefix DP | boundary-indexed (n+1) convention | resistance to fencepost indexing | prefix DPs index **boundaries**, not elements; size n+1, answer at dp[n] |
| **Bitmask DP over a small target set** ⚠️ FLAGGED WEAK — REVISIT | stalls on "what does the mask index" | mask = the ≤~20 target set (skills / customers / nodes); iterate the *resources* one at a time, each flips on a submask | state-design stall: recognizing bitmask is needed but not what the bits ARE | when a target set has size ≤ ~20 and you assign resources to cover/satisfy it → `dp[mask]` over the TARGET set; process resources one at a time; transition ORs in each resource's reachable submask. Traceback (`parent[]`) only if the answer needs *which* resources, not just feasibility/count |

| LC 630 (07-06 **revisit** — first seen 06-16; 871 regret-greedy self-identified 07-01) | "is there even a greedy?" guess, then stalled on the mechanism | regret/heap greedy: deadline-sort (exchange arg) + take-if-fits + evict current longest via max-heap | **retention/transfer failure**: 871's mechanism was installed but never generalized into a trigger; also twice asserted **time** (not **count**) as the invariant quantity in the correctness proof — the multi-swap is rejected because it costs count, not because it worsens time (it strictly improves time) | sequential budget + commitments you'd sometimes want to un-take + "worst held item" is one comparable number → heap regret greedy. Proof sentence to say cold: "dropping courses converts count→slack; slack converts back at ≤1-for-1 (the swap op proves it), so never profits" |
| LC 1866 | no independent state; recurrence supplied | permutation-counting DP: dp[i][j] via inserting the new **shortest** stick (rank order), leftmost slot = visible, other i−1 slots = hidden | new subfamily: didn't know insertion-by-rank or the rank-relabeling bijection that licenses dropping identities from the state (comparison-only property ⇒ ranks lossless) | counting arrangements with a comparison-defined property (visible/records/inversions) → insert by rank, count insertion positions by effect. Drill set: **629 → 920 → 1359** |

| LC 1976 (07-07) | one PQ pop per shortest path, counting pops at the target | **augmented Dijkstra**: carry ways[] through relaxation — `<` overwrites, `==` accumulates; a node's count may only propagate once its distance is FINAL (the staleness check is now a *correctness* guard, not an optimization) | had only ever used the staleness check as a perf nicety, so its correctness role was invisible; also misconceived path counting as path enumeration (answer can be exponential — the mod hint in the statement was the tell) | "shortest path + something *about* the shortest paths" (count / min-edge / reconstruction) → augmented Dijkstra: secondary quantity per node, equal-case branch, propagate-on-finalization. Alt frame: tight edges (dist[u]+w==dist[v]) form a DAG, accumulate in dist order. Consolidation: 1786 (assigned + solved same day) |
| LC 2831 (07-07) | "how do I query deletions-in-range fast" (reaching for a structure) | occurrence-space window: group indices by value, slide over each positions list; deletions = span − count. OR: stale-max window on the raw array (`winLen − maxFreq ≤ k`) | axis-swap flavor #3 (array→occurrence space) not yet named; AND the stale-max template was filed under "replacement ops" (424/1838) so "deletion" didn't retrieve it — key the template on the FORMULA `winLen − maxFreq ≤ k`, not the operation verb | one-value objective ("all equal" / freq-of-X) → per-value positions lists. When a window feels blocked on "compute X cheaply": 30-sec reframe check (different sequence to slide over?) BEFORE reaching for a structure. Stale max is safe because ans only improves when maxFreq genuinely increases |
| LC 1888 (07-07, walked — implementation unconfirmed) | splice free-rotation into the suffix DP as the "free" branch | flips ∘ rotations **commute** → all rotations first, then flips: answer = min over n rotations of mismatch count; rotations of s = length-n windows of s+s; anchor patterns "0101…"/"1010…" to ABSOLUTE indices of s+s so per-char verdicts survive window motion (min(diffA,diffB) is phase-swap invariant) | a global re-alignment op can't be a local DP transition — no state at index i captures "we rotated"; plus coordinate-anchoring trick unknown | free/cyclic op + optimize per rotation → s+s window; window criterion that seems position-dependent → anchor to absolute coordinates, track all phases. Related: rotation only matters for odd n here (even n: rotation just swaps pattern roles). Same coordinate-system genus as the n+1 prefix boundary convention |

**Solid recognitions (for contrast, keep calibrated):** Kadane (LC 2606) instant; BSoA (LC 2616, and LC 2439 on second look) reliable; knapsack connection (LC 871) self-identified; exchange-argument skeleton installed; LC 2551/1899 solved fast but with prior exposure — freshness there unverified.

**Deferred topics:** LC 2839 (coordinate-compressed segment trees) — revisit after studying segment trees / BIT.

## 2.3 Idioms & Checklists to Drill

**Two pre-passes to run before submitting (installed 2026-07-03, LC 2528):**

*(1) Type-width pass — for numeric/overflow bugs [category G].* Before coding: read the constraints, compute the max value of every accumulator and product. Anything that can exceed 2.1e9 (INT_MAX) is `long long`. Then check every function boundary: if a `long long` is passed in, the parameter is `long long` too (LC 2528's real bug was `int minimumPower` truncating a `long long mid`). Containers holding wide deltas are `long long` too (`vector<int> adjustments` was wrong). This is NOT blanket-casting — indices, `r`, single small elements stay `int`, and you justify each `long long` aloud. Trigger words in constraints: any bound ≥ ~1e5 that gets summed or multiplied.

*(2) Draw-the-array pass — for index/alignment bugs [categories D, E].* For anything spatial — difference arrays, sliding windows, monotonic-stack spans, 2D grid DP — sketch 6–8 cells on paper and mark: the coverage/window interval for a sample index, where each delta is written, where it's consumed, where it cancels. Walk one concrete element through. This catches fencepost errors, `[i-r,i+r]` vs `[i-r,i+r-1]`, and diff-array turn-on/turn-off misalignment. (It would NOT have caught LC 2528's bugs, which were type-width — different pre-pass. Match the pass to the failure class.)

**Bitmask DP over a small target set (⚠️ FLAGGED WEAK AREA — Ethan self-identified, revisit):**
*Recognition:* a target set of size ≤ ~20 (skills, customers, nodes to visit) that you cover/satisfy by assigning resources. The size-≤20 constraint IS the tell.
*State:* `dp[mask]` where mask = subset of the TARGET set achieved (NOT the resources). This is the step Ethan stalls on — "what do the bits index?" Answer: always the small target set.
*Process:* iterate resources (people / piles / nodes) one at a time; each resource, when applied, ORs some reachable submask into the current mask.
*Transition shape:* for each reachable `mask`, for each valid `sub` this resource can cover, `dp[mask | sub] = true` (or min-cost etc.).
*Traceback:* store a `parent[]` array ONLY if the answer needs *which* resources were chosen (LC 1723 needed it — count alone doesn't say who). If the answer is just feasibility/count (LC 1655 boolean), NO traceback — don't over-build.
*Two examples seen:*
  - **LC 1723 Smallest Sufficient Team** (06-27): mask = skills covered, iterate people, each = fixed skill-submask. Needed parent[] for reconstruction. Stalled on state design.
  - **LC 1655 Distribute Repeating Integers** (07-03): mask = customers satisfied, iterate distinct-value piles, each pile serves any customer-submask with `need[sub] <= count`. Boolean, no traceback.
*New machinery in 1655 (wasn't in 1723):*
  - **Submask enumeration:** `for (int sub = S; sub; sub = (sub - 1) & S)` — and remember `sub = 0` (assign to nobody) is valid, handle it outside the loop. Getting this loop wrong is a silent bug.
  - **`need[submask]` precompute:** `need[sub] = need[sub & (sub-1)] + quantity[__builtin_ctz(sub)]` — off-by-one between bit position and quantity index is the category-A identifier trap.
  - **Pruning:** only the ≤~50 largest resource counts can matter when m ≤ 10 (can't use more piles than that). Complexity ~ (#distinct) × 3^m.
*Why DP not greedy:* assigning a resource to one target constrains all future assignments non-locally → local optima strand later targets. (Standard greedy red flag.)

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

**Permutation-counting DP (installed 2026-07-06, LC 1866):** trigger = counting arrangements where the property is defined purely by comparisons. Move: relabel to ranks (bijection preserves all comparisons ⇒ every i-set has identical counts ⇒ identities drop from state), then build by inserting elements in rank order and count insertion positions by their effect. 1866 recurrence: `dp[i][j] = dp[i-1][j-1] + (i-1)*dp[i-1][j]` (leftmost slot = visible; widen before `*`, base dp[0][0]=1 only). The compression DIES the moment magnitudes matter (sums/thresholds on values) — 2-second scan for any non-comparison use of values.

**DP state-derivation procedure (named 2026-07-06 — countermeasure for the §2.2 state-design stalls):**
1. **Parameterize the question** — turn the problem's constants into variables; "exactly/at most k of P" puts P's count in the state.
2. **Sufficiency test** — what must the future know about a partial solution? List the leaks.
3. **Eliminate before enlarging** — for each leak try: reorder processing (position→rank/sorted/reverse), canonicalize (relabel to ranks), condition on a special element (min/max/first/last). Only a leak that survives all three becomes a dimension. (Reorder killed the ordering dimension in 630 and the running-max dimension in 1866; 3977's power leak was intrinsic → dimension. Dividing question: *does the future need this intrinsically, or is it an artifact of my processing order?*)
4. **Size it** against constraints; if it doesn't fit, return to 3 with more force.

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
- **LC 2528** (07-03) — Maximize the Minimum Powered City. BSoA correctly self-identified (monotonic feasibility). **Technique learned:** difference array for O(1) range-update during the greedy `feasible(x)` sweep — was the missing piece (didn't know it). Greedy placement = push forced stations as far right as still covers the deficient city, covering `[i, i+2r]`; consume at `leftBound`, cancel at `i+r+1` (window-relative matched pair). **Bugs (Ethan's):** initial consume-index alignment wrong before correcting to `leftBound` [E, draw-the-array countermeasure]; `vector<int> adjustments` truncating long long deltas [G]; `int minimumPower` param truncating long long `mid` at the call boundary [G — narrowing-across-function-boundary]. Type bugs fail on large inputs (k≤1e9, power≤1e10). Countermeasures → §2.3 type-width pass + draw-the-array pass.
- **LC 1655** (07-03, PAUSED — resume tomorrow) — Distribute Repeating Integers. ⚠️ bitmask-DP-over-target-set, Ethan's self-identified weak area. Correctly ruled out greedy (non-local constraint). Reached the state-design question (mask = customers, iterate piles) with a nudge — same stall as LC 1723. Not yet coded: the `dp[mask|sub]` transition, submask enumeration loop, and `need[submask]` precompute. Full paradigm writeup in §2.3 "Bitmask DP over a small target set." **Resume plan: drill set 1655 → 698 (Partition to K Equal Sum Subsets) → 2305 (Fair Distribution of Cookies) — same submask move ×3 until it's automatic. Write the transition + submask loop cold each time.** Recurrence in plain English (from 07-03 session): dp[j][mask] = "using first j piles, can group `mask` be fully served?" = idle (dp[j-1][mask]) OR exists sub⊆mask with need[sub]≤count AND dp[j-1][mask^sub]. `sub` = chunk this pile serves; `mask^sub` = chunk earlier piles served. Cost 3^m via `for(sub=mask;sub;sub=(sub-1)&mask)`.

- **LC 2136 / 1642 / 2454** (07-06) — assigned, reported previously solved (2136/2454 not in log; no session data — freshness unverified).
- **LC 630** (07-06, **revisit** — first seen 06-16 paradigm session) — Course Schedule III. Identification stall ("don't know if there's a greedy") despite prior exposure AND 871's regret-greedy self-identification on 07-01 → retention/transfer failure, logged §2.2. Exchange argument for deadline-sort reached with prompting; correctness proof twice anchored on the wrong invariant quantity (**time instead of count** — the {3,3}+dur-5 counterexample kills the time framing). Full race-argument proof walked. Solved + submitted on LC after walkthrough. Interview sentence locked: slack↔count converts at ≤1-for-1.
- **LC 1937** (07-06, revisit) — solved independently on LC, no bugs reported.
- **LC 2407** (07-06) — LIS with adjacent difference ≤ k. Identification GOOD: correctly diagnosed why the tails-array O(n log n) trick dies (min-tail-per-length no longer dominant — extension window [v−k, v−1] over VALUES means a larger admissible tail can exist while the min is below the window) and reached value-indexed dp + range-max-query reformulation. Implementation (segment tree over value domain, point update / range max) **KIV'd — needs a return date**; feeds §2.WEAK #3.
- **LC 1986** (07-06) — assigned, unattempted (excluded by request — bitmask; overlaps the 1655→698→2305 drill).
- **LC 421 / 1707** (07-06) — assigned, unattempted (excluded by request — trie family; 1707 = 421 + offline sorted queries). Feeds §2.WEAK #4.
- **LC 1866** (07-06) — Number of Sticks with K Visible. Permutation-counting DP — new subfamily, walked (see §2.2 row + §2.3 block). Strong probing on WHY counts suffice (rank-bijection) and on state derivation — the questions were right, the toolkit was missing. Drill set queued: 629 → 920 → 1359.
- **LC 1838** (07-06) — Frequency of the Most Frequent Element. Solved independently on LC in 15 min (on-target for tier), no bugs reported.
- **LC 1478** (07-06) — Allocate Mailboxes (partition DP + median cost). Identification: framed the choice as continuous mailbox position instead of split point; sort→contiguous-blocks exchange-argument reduction needed a nudge [§2.2]. **Bug:** over-broad base-case loop set dp[1][0]=0, infeasible state marked feasible [C]. Sensed the framing was off and asked rather than forcing it — good instinct, log as positive. Complexity note: left O(n) median-cost inline in the transition → O(n³k); should precompute cost(l,r) (O(n²) via extension trick `cost(l,r)=cost(l,r−1)+houses[r]−houses[(l+r)/2]`) for O(n²k) DP. AC'd.
- **LC 2218** (07-06) — Maximum Value of K Coins (grouped knapsack, top-of-pile prefix sums). Category self-identified, intended algorithm reached independently. **Bug:** inner dp dimension allocated `k` not `k+1` → ASan heap-buffer-overflow [C, allocation flavor]. Debugging win: self-directed via ASan report + index enumeration after flow prompt; `.at()` substitution added to toolkit. **Complexity lesson locked:** tight bound is O(k·T) where T = total coins, NOT O(n·k·max-pile) — inner loop amortizes over the *summed* constraint (Σ|pileᵢ|), a named pattern: "constraint bounds a sum, not a max → honest bound is over the sum" (same move as m·n≤1e5 problems, tree-DP knapsack merges). 7th-percentile runtime = constants not paradigm: 2D→1D rolling array (grouped knapsack, iterate j downward) + drop hot-loop assert; queued as a 5-min rep. AC'd.

- **LC 1642** (07-07) — assigned, reported previously solved (no session data — freshness unverified).
- **LC 2542** (07-07) — assigned; no verdict reported (status unknown).
- **LC 2439** (07-07, **revisit** — identification logged 06-29) — solved. Identification held (BSoA + right-to-left backlog feasibility — a valid equivalent of the prefix bound). **Bug:** `accumulate(..., 0)` int accumulator overflow [G]. Also `valid` took the vector **by copy**, unused mutation — O(n) copy per feasibility call [I-minor]. O(n) closed form noted for follow-ups: answer = max over i of ceil(prefix[i]/(i+1)).
- **LC 1976** (07-07) — Number of Ways to Arrive at Destination. 40 min total. Bugs: `vector<int>` distances + int addition overflow [G]; `>=` on the pop-time staleness check — retention miss of 3977 [E]. Identification: augmented-Dijkstra counting was a genuinely new structural pattern (initially conceived as one-pop-per-path); the "seen/finalization" role of the staleness check took the bulk of the time [§2.2]. Post-solve grading initially undercounted the conceptual gap — corrected on Ethan's pushback.
- **LC 1786** (07-07) — assigned as same-day consolidation of the 1976 pattern; reported done, no bugs reported.
- **LC 2831** (07-07) — Find the Longest Equal Subarray. Identified sliding window solo; stalled on the deletions-per-window quantity → occurrence-space reframe walked [§2.2]. Ethan then independently surfaced the stale-max alternative (winLen − maxFreq ≤ k) — retrieval had been keyed on "replacement" not the formula. Both solutions now known; staleness-safety argument walked.
- **LC 1888** (07-07) — Minimum Flips to Make Alternating (KIV → walked in full). DP attempt bugs: free rotation never modeled [F]; loop never read s[i]; degenerate second state dimension. Full reduction walked after frustration-abandon (commute → s+s window → absolute-index anchoring, code provided). **Implementation by Ethan unconfirmed — counts as walked, not cleared. Return date needed.**
- **LC 2919** (07-07) — Minimum Increment Operations. Attempt submitted in-chat (dp[i][0/1] make-self-beautiful vs settled-by-others, i−3 lookback); **grading pending** — verdict never delivered (delivery failure), Ethan moved on. Preliminary: close, state design has a hole. **Grade at next session start.**
- **LC 2547** (07-07) — Minimum Cost to Split an Array. Partition DP identified instantly (drilled paradigm); incremental uniques bookkeeping clean. **Bug:** split-loop direction vs segment-growth direction mismatch — freq map held the wrong element set at every evaluation [E]. Solved after one clue.
- **LC 2560 / 2517 / 2064** (07-07) — BSoA block, all reported solved clean, identification-first on 2064 (binary search + greedy, self-identified). **2517 is a revisit** (06-25 shadowing bug) — clean this time; countermeasure held. Self-reported no-bugs: silent-clean vs actually-clean unverified.
- **LC 2318** (07-07) — Number of Distinct Roll Sequences (2090). Solved, no bugs reported. The one problem of the evening block at band-edge.
- **LC 2266 / 2088** (07-07) — solved, reported easy (1857 / 2105). 2088's height-recurrence insight landed without help — strong signal on grid counting DP.
- **LC 2731** (07-07) — Movement of Robots. Asked the right question ("why don't collisions matter") → pass-through/relabeling reframe explained + contribution-sum step and the G/B landmines flagged. **Solve unconfirmed** — session pivoted to the recognition catalog.
- **Session note (07-07):** heavy volume (~13 problems) but concentrated in strong paradigms (BSoA ×4, partition DP, counting DP); band-edge/recognition-gated work was 2318, 1888, 2731, 1976. **recognition-catalog.md added to repo** — full trigger-heuristic reference (reframes A1–A15, greedy B1–B7, DP C1–C10, DS D1–D7, counting E1–E5, deprioritized F, 60-sec pre-solve scan). Study between sessions; blind drills remain the test.

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

*Last updated: 2026-07-06 (session 2) · session: 1478 (partition-DP reduction via exchange argument [§2.2] + over-broad base case [C]), 2218 (k vs k+1 allocation [C] + amortized-over-sum complexity pattern locked + ASan/.at() debugging toolkit). Category C at 10, 2× today — allocation-vs-loop-bounds pair-check discipline added. New drill family: sort+exchange→partition DP (410, 813, 1959); queued rep: 2218 2D→1D grouped-knapsack rewrite. Earlier same day: 630 revisit, 2407 (segment tree KIV), 1866, 1937/1838 clean. Open/unattempted: 1986, 421, 1707, 2407-impl.*
