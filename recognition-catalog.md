# Recognition Catalog — Paradigms, Reframes & Trigger Heuristics
*Built 2026-07-07 for HFT OA / interview band (Zerotrac ~1800–2100). Format per entry: **the tell → the move → canonical problems**. OA-likelihood: ★★★ common / ★★ plausible / ★ rare-but-famous.*

---

## Part A — Reframes (identification gates; the problem looks like X, is actually Y)

### A1. Elastic collision → pass-through (ants) ★★
- **Tell:** identical agents, symmetric collision rule (both reverse), answer depends only on final positions/times, not identities.
- **Move:** collisions are relabelings; treat everyone as ghosting through. Each agent's trajectory becomes independent.
- **Problems:** 1503 (1619), 2731 (1923).

### A2. Axis swap: iterate over values / occurrence space, not indices ★★★
- **Tell:** objective is about one value at a time ("all equal", "frequency of X"), or a per-window quantity is cheap in occurrence coordinates (deletions = span − count).
- **Move:** group indices by value; slide/scan over each value's positions list. Total work O(n) since each index belongs to one list.
- **Problems:** 2831 (1976), 2302-adjacent family; your ledger's 2555/2818/828 are the subarrays→elements sibling.

### A3. Contribution counting (sum over parts, not over wholes) ★★★
- **Tell:** "sum of f(subarray) over ALL subarrays/subsequences" where enumerating wholes is O(n²)+.
- **Move:** flip the double sum — for each element (or each pair/gap), count how many wholes it contributes to. Monotonic stack gives per-element min/max spans.
- **Problems:** 907 (1976), 2104, 2262 (2033), 828 (2034), 1498 (subsequences w/ powers of 2), 2681. Pairwise-distance version: sort, each gap crossed by (i+1)(n−i−1) pairs (2731 step 2).

### A4. Prefix-sum ↔ subarray condition ★★★
- **Tell:** any condition on subarray sums — equals k, divisible by k, ≥ k, count of them.
- **Move:** subarray(i,j] ⇔ prefix[j] − prefix[i]. Equality/count → hashmap of seen prefixes. Divisibility → hashmap of prefix mod k (watch negative mods). Shortest with ≥ k and negatives allowed → monotonic deque on prefixes (862). Binary condition on 0/1 counts → transform to ±1 then prefix.
- **Problems:** 560, 974, 523, 1590 (mod of the REMOVED piece: total mod first), 862 (2307), 1546 (1855), 325.

### A5. Coordinate anchoring under motion ★★
- **Tell:** sliding window where the target/criterion seems to shift with window position (alternating patterns, periodic targets).
- **Move:** anchor the criterion to absolute indices; track ALL phases of the target simultaneously (e.g., diffA/diffB vs "0101…"/"1010…"); min over phases is phase-swap invariant.
- **Problems:** 1888 (2006). General periodic-target windows.

### A6. All rotations = one window over s+s ★★
- **Tell:** "rotate for free" / "cyclic" / "circular array" + optimize something per rotation.
- **Move:** rotations of s are the n length-n windows of s+s. Combine with A5 or running counters. Circular subarray problems: also consider "total − min-subarray" (918).
- **Problems:** 1888, 918 (circular Kadane), 213 (circular house robber → run twice excluding one end).

### A7. Work backwards / reverse the operations ★★
- **Tell:** forward branching is huge but the final state is constrained, or operations have cheap inverses (divide vs multiply).
- **Move:** run from target to start; greedy often becomes forced (e.g., "if y is even, halve" beats "double x").
- **Problems:** 991 (Broken Calculator), 780 (Reaching Points — mod-jump backwards), 2139, 1558.

### A8. Complement counting ★★★
- **Tell:** "at least one", "not all", counting objects with a forbidden property.
- **Move:** total − bad, where bad has more structure. For "exactly k": atMost(k) − atMost(k−1) — the single most-used window identity.
- **Problems:** 992 (exactly-k distinct), 1358, 2444-adjacent, 930.

### A9. Invert the objective: delete-min ⇔ keep-max ★★★
- **Tell:** "minimum removals/changes so that property holds."
- **Move:** answer = n − (longest/largest kept structure). Turns deletion problems into LIS/window/subsequence problems.
- **Problems:** 1671 (bitonic, LIS both directions), 646→435 (min removals = n − max non-overlap), 1846, 2occurring.

### A10. Distance/abs-value decomposition ★
- **Tell:** maximize expressions with |a−b| terms, Manhattan distances.
- **Move:** |x| = max(x, −x) → expand into 2^k sign cases, each linear → track max/min of the linear form per case. Manhattan → Chebyshev via (x+y, x−y).
- **Problems:** 1131 (2071), 1637-adjacent geometry, 2613-family.

### A11. Pigeonhole / small-state forcing ★
- **Tell:** huge n but tiny answer bound, or "must exist a repeat" (prefix mods: only k classes).
- **Move:** bound the interesting states; two equal prefix-states ⇒ zero-effect segment between them.
- **Problems:** 523 (two prefixes with same mod ≥2 apart), 1497, 2575 (streaming mod).

### A12. Answer decomposes per bit ★★
- **Tell:** XOR/AND/OR aggregates over pairs/subarrays; sums of bitwise results.
- **Move:** solve each bit independently — count pairs/subarrays where that bit is set, multiply by 2^bit. AND/OR windows are monotone in length → per-bit last-seen indices or 30-counter window.
- **Problems:** 477 (Hamming total), 1521 (closest AND), 2411 (1938), 2419, 3097. XOR-max pairs is the trie gate (421) — your queued item.

### A13. State-space BFS (the graph is implicit) ★★★
- **Tell:** "minimum number of operations/moves" between configurations, small state space, unweighted ops.
- **Move:** states are nodes, ops are edges, BFS. Costs 0/1 → deque 0-1 BFS. Multiple starts → multi-source BFS. If states are strings/masks, encode compactly.
- **Problems:** 752, 1345 (2050, value-teleport edges — build value→indices map, clear after use), 815, 1368 (0-1 BFS, 2069), 542/994/1162 (multi-source).

### A14. Offline processing: sort the queries ★★
- **Tell:** many queries, each depending only on a threshold (values ≤ x, time ≤ t); online order irrelevant.
- **Move:** sort queries, sweep data once, answer with two pointers / DSU / heap as the threshold rises.
- **Problems:** 1697 (DSU by edge limit, 2300-ish — aware-level), 2070 (sorted queries + prefix max), 1847.

### A15. "Simulation" that's secretly a cycle ★
- **Tell:** iterate a deterministic function from a state; k is astronomically large.
- **Move:** states repeat → find cycle (map of seen states or Floyd), jump k mod cycle length.
- **Problems:** 957, 1806, 202-style. Matrix-power counting is the linear-recurrence cousin (aware-level only for your band).

---

## Part B — Greedy families (your #1 gap; the 13 condensed to the recognizable cores)

### B1. Exchange argument (canonical ordering) ★★★
- **Tell:** choose an order/subset; asked to prove/produce optimal arrangement.
- **Move:** assume optimal, swap two adjacent items, show no improvement ⇒ sort by the pairwise comparator. Comparator must be a strict weak ordering (Part 1 §STL trap — `<=` is UB).
- **Problems:** 179 (largest number: a+b vs b+a), 1665 (sort by actual−minimum), 621, 2136 (plant longest-growth last), 1478's reduction (your ledger).

### B2. Regret / heap greedy ★★★ — **your confirmed weak spot (871→630→1642)**
- **Tell:** items consumed in a forced order (deadline/positional), a budget, and the option to RETROACTIVELY undo a past choice.
- **Move:** take greedily; when infeasible, un-take the worst past choice (max-heap of taken costs). "Commit cheap, refund the most expensive regret."
- **Problems:** 871, 630 (deadline sort + heap), 1642 (ladders = free passes on the k largest so far), 2813-adjacent, LC 1705 (eat-by-expiry, min-heap variant).
- **Trigger phrase to burn in:** *sequential + budget + undoable ⇒ heap of regrets.*

### B3. Deadline scheduling ★★
- **Tell:** tasks with deadlines/durations, maximize count/value completed.
- **Move:** sort by deadline, greedily add, regret-heap when overtime (this IS 630). Value-weighted with unit slots → sort by value, place latest-free slot (or DSU for slots).
- **Problems:** 630, 1353 (events + min-heap of expiring), 2589-adjacent.

### B4. Two-pointer pairing after sort ★★★
- **Tell:** pair up items under a sum/ratio constraint, maximize pairs or minimize resource.
- **Move:** sort; match extremes (weakest with strongest that still fits).
- **Problems:** 881 (boats), 2938, 1798-adjacent, 2576 (pair smallest half against largest half).

### B5. Lexicographic greedy with stack ★★★
- **Tell:** "smallest/largest string/number after removing k chars / picking a subsequence."
- **Move:** monotonic stack; pop while you can still afford to (chars remaining ≥ needed). The budget condition is where bugs live.
- **Problems:** 402, 316/1081 (need last-occurrence + in-stack set), 321 (merge two — aware-level).

### B6. Threshold greedy via sorting + prefix ★★
- **Tell:** "maximize count under budget" / "minimum number of X so that condition."
- **Move:** sort by cost, take prefix; or binary search the count (feasibility = cheapest way to achieve count c).
- **Problems:** 2279, 1833, 2557-adjacent.

### B7. Interval scheduling / sweeping ★★★
- **Tell:** intervals; max non-overlap, min rooms, min arrows, merge.
- **Move:** max non-overlap → sort by END, take greedily. Min resources → sweep (+1/−1 events) or min-heap of end times. Min points hitting all → sort by end, shoot at end.
- **Problems:** 435, 452, 253, 2054 (1883 — best two non-overlap: sort by start + suffix max, or end + binsearch), 1235 (weighted → DP + binsearch, the bridge to DP).

---

## Part C — DP families beyond what you've drilled

### C1. Partition DP — **installed** (410/813/1043/1959/2547/1478). Skip.

### C2. "Take with cooldown/adjacency constraint" ★★★
- **Tell:** linear sequence, taking i forbids/penalizes a neighborhood of i.
- **Move:** dp over position with a small "recently taken" state. Window-max/deque when the reach is k (jump DP).
- **Problems:** 198/213, 2140 (1709), 1770, 2919 (2031 — your pending grade: window of last-3), 1425 (deque-optimized, 2032).

### C3. Digit-position / last-value counting DP ★★
- **Tell:** count sequences with a local constraint between consecutive elements.
- **Move:** dp[i][last] with transition matrix over small alphabet; compress if the constraint has structure. Distinctness constraints add a "previous≠current" subtlety (2318's gate: subtract sequences where a value repeats with gap <3 — track (prev, prevprev)).
- **Problems:** 1220 (1730), 2318 (2090), 790, 3290-adjacent.

### C4. Grid DP with transition optimization ★★
- **Tell:** dp[i][j] = max over ALL k of dp[i−1][k] − penalty(|j−k|) — naive O(n·m²).
- **Move:** split |j−k| into two directional sweeps carrying a running best (left pass: best = max(best−1, dp[i−1][j])). This is 1937's whole rating.
- **Problems:** 1937 (2106), 2435-adjacent, 1301 (with counting).

### C5. Bitmask DP — **your queue (1655→698→2305)**, condensed rule:
- Single-bit transitions (place items one at a time): O(2^m · m), loop lowest unset bit.
- Whole-submask transitions (assign a GROUP per step): O(3^m) via `for (sub = mask; sub; sub = (sub−1) & mask)`.
- **Tell:** n ≤ 16–20 on one dimension, or "assign/partition small set."
- **Problems:** 1655 (2071), 698, 2305 (1917), 1986, 1349 (2386 — row-pair masks, above band).

### C6. Topological / DAG DP ★★
- **Tell:** longest/count paths under a partial order (prerequisites, strictly-increasing moves in a grid).
- **Move:** memoized DFS over the DAG (grid: 329) or Kahn + relax. Shortest-path counting on weighted graphs is the augmented-Dijkstra you just installed (1976/1786): secondary quantity propagates only from finalized nodes.
- **Problems:** 329, 1857 (2079 — color counting on DAG), 2050.

### C7. LIS family + patience/binsearch ★★★
- **Tell:** longest chain under an order; pairs/envelopes; "minimum number of strictly increasing piles."
- **Move:** tails array + lower_bound (your Category J watch: comparator/argument order). 2D → sort by one key asc, other DESC, LIS on second.
- **Problems:** 300, 354 (2205 — aware), 1626 (2027 — sort by (age,score), LIS-sum), 646, 2111 (k-strided LIS, 2000-ish).

### C8. Insertion-by-rank permutation DP — **your 07-06 queue (629→920→1359).** Skip here.

### C9. String DP core trio ★★★
- Edit distance skeleton (72/583/712), LCS (1143) and its disguises (1035 = LCS, 1092 = LCS + reconstruction — your Category B entry), palindromic (516 = LCS with reverse, 5/647 expand-around-center first).
- **Tell for disguise:** "two sequences, align/match/delete" ⇒ 2D suffix/prefix DP.

### C10. Probability / expectation DP ★
- **Tell:** "probability that after k steps…" small state count.
- **Move:** forward-propagate mass dp[state] over steps; sum absorbing states.
- **Problems:** 688 (knight stays on board), 837, 1227 (the answer is just 0.5 — recognition joke, worth seeing once).

---

## Part D — Data-structure triggers

### D1. Monotonic stack ★★★ — installed (contribution counting). Cross-ref A3. Spans between EXCLUSIVE fenceposts: `right − left − 1` (your 2334 entry).
### D2. Monotonic deque ★★★ — window max/min in O(1) amortized; jump-DP transitions (1425, 3956, 862). Pop direction = your Category E; re-derive the invariant, don't recall it.
### D3. Two heaps / lazy deletion ★★
- **Tell:** streaming median (295); sliding median (480 — heaps + delayed removal map); "pop max but elements expire."
- **Problems:** 295, 480 (aware), 2349, 2353 (lazy heap: pop until top is valid).
### D4. Trie ★★ — **your queue (421→1707).** Tells: prefix queries at scale; XOR max/min pairs (bit-trie, walk greedy opposite-bit). Skip detail, it's queued.
### D5. DSU ★★★
- **Tell:** incremental connectivity, "merge groups", equivalence classes, offline threshold queries (A14), cycle detection in undirected graphs, "redundant connection."
- **Move:** union by size + path halving; augment with component payload (size, min, count) merged on union.
- **Problems:** 547, 684, 947 (2035 — rows/cols as nodes: the recognition gate), 721, 1202 (sort within components), 2316.
### D6. Fenwick / segment tree — **KIV'd (2407/327/2839), your call when.** The tells so you at least RECOGNIZE while deferring implementation: "count inversions / smaller-after-self" (315), "range max over dp of compatible predecessors" (2407), "count of range sums" (327). All need coordinate compression when values are large. When you pick this up: Fenwick-for-counting first (20 lines), segment-tree-for-max second.
### D7. Ordered set / map as sweep structure ★★
- **Tell:** "nearest existing value ≥/≤ x among inserted so far"; calendar booking; merging adjacent occupied ranges.
- **Move:** `std::map` + `lower_bound`, inspect neighbor both sides (`prev(it)` guard begin()).
- **Problems:** 729/731/732 (booking ladder), 715 (aware), 2276-adjacent, 1account.

---

## Part E — Counting & math gates

### E1. Exactly-k = atMost(k) − atMost(k−1) ★★★ (windows). 992, 1248, 930.
### E2. Sorted + pairwise: count pairs with condition via two pointers or binsearch ★★★. 2563 (both bounds), 1498 (power-of-2 contribution), 611.
### E3. Multiplicative counting: independent choices multiply ★★. Sorted-gap freedom: 1569-adjacent (aware), 2338 (above band). Mostly: recognize "answer = ∏ per-position choices" in constructive counting (1359's rank bijection — queued).
### E4. Inclusion–exclusion, 2-set version ★★. |A∪B| via lcm counting: 878 (binsearch on answer + IE — a great fusion problem, 2051), 1201 (aware).
### E5. Mod discipline ★★★ (execution, not recognition — but it gates counting DP): sub then `(x%M+M)%M`; multiply in long long BEFORE mod; never mod a MIN/MAX dp. Your Category G's counting-flavored twin.

---

## Part F — Deprioritized (recognize-only; do NOT drill — per your §2.WEAK)
digit DP (tell: "count numbers ≤ N with digit property"), convex-hull trick / Li Chao (tell: dp transition = max of linear functions), suffix automata / Z / KMP beyond basics (tell: heavy substring counting), most interval DP (tell: "merge/burst on a range, cost couples ends" — 312/1039/1547; know the dp[i][j]-over-split-k shape exists), matrix exponentiation (tell: linear recurrence, k ~ 1e9), binary lifting (tell: k-th ancestor / jump 1e9 steps).

---

## The 60-second pre-solve scan (recognition checklist — run BEFORE deriving)
1. **Constraints first** (your §2.3): n≤20→bitmask; n≤500→O(n²/n³) DP; n≤1e5→O(n log n); values 1e9→long long + no value-indexed arrays; k~1e9→cycle/matrix/backwards.
2. **The ask:** count → DP/contribution/complement; min-max or max-min → binary search on answer; "minimum ops between states" → BFS; "sum over all subX" → contribution.
3. **Undo-ability:** sequential choices + budget + retroactive undo → regret heap (B2).
4. **One-value objectives** → occurrence space (A2).
5. **Free/cyclic operations** → s+s window or invariant anchoring (A5/A6).
6. **Identical agents + collisions** → pass-through (A1).
7. **Stuck on "need range structure"?** → 30-sec reframe check first (A2/A3/A4); structure second (D6).
