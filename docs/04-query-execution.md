# 04 — Query Execution

The index is a data structure; query execution is the *algorithm* that walks it. This is where a
naive implementation loses 100× to a good one, on identical data.

---

## 4.1 The lifecycle of a query

```
"quick brown fox" +tag:animal -deprecated
        │
        ▼  PARSE            → syntax tree of query nodes
        ▼  ANALYZE          → run each text chunk through the *query* analyzer
        ▼  REWRITE          → wildcard/fuzzy/range expand to concrete terms; constant-fold
        ▼  PLAN             → order clauses by cost (df), choose iterators, pick strategy
        ▼  EXECUTE          → advance iterators, score matching docs
        ▼  COLLECT          → top-k heap, or filter/aggregate collectors
        ▼  FETCH            → load stored fields for the k winners only
        ▼  HIGHLIGHT        → re-analyse or use offsets
```

Two things people get wrong:

- **Fetch only the k winners.** Scoring touches millions of docs; loading document *content* must
  touch 10. Keep them separate; never load stored fields during scoring.
- **Rewrite before planning.** A `foo*` that expands to 40,000 terms is a completely different cost
  than one that expands to 3, and the planner must know.

---

## 4.2 The iterator abstraction

Everything below is built on one interface. Get this right and the rest composes:

```
DocID()          current document, or NO_MORE (= maxint) when exhausted
Next()           advance to the next matching doc
Advance(target)  advance to the first doc >= target      ← the important one
Cost()           estimated number of docs (≈ df), for planning
Score()          score of the current doc (lazily computed)
```

`Advance(target)` — "seek forward" — is what makes intersection sublinear (skip lists, §3.6). Every
query type is an iterator: a term iterator reads postings; a conjunction iterator wraps N children;
a disjunction wraps a heap of children. Queries nest arbitrarily because they are all the same shape.

This is Lucene's `DocIdSetIterator` and Tantivy's `DocSet`. It is a genuinely elegant design; copy it.

---

## 4.3 Boolean operators

### AND (conjunction) — leapfrog

```
lists:  A = [2, 7, 9, 40, 41]
        B = [1, 9, 40, 88]

leapfrog:
  candidate = A.first = 2
  B.Advance(2)  → 9   → mismatch, candidate = 9
  A.Advance(9)  → 9   → match!  emit 9, both Next()
  candidate = 40 ; B.Advance(40) → 40 → match! emit 40
  ...
```

Rules:

- **Start with the rarest list** (lowest `Cost()`), and re-sort children by cost. Intersecting a
  50-doc list with a 10M-doc list should cost ~50 seeks, not 10M steps.
- Use `Advance`, not repeated `Next`. With skip lists this is `O(k log n)`.
- **Galloping/exponential search** inside a block: probe at +1, +2, +4, +8… then binary search the
  bracket. Optimal when one list is far shorter than the other.

### OR (disjunction) — heap merge

Maintain a min-heap of child iterators keyed by current docID. Pop all children sharing the smallest
docID, sum their scores, emit, advance them, re-push. `O(n log k)` for k clauses.

For pure boolean OR over *filters*, converting to Roaring bitmaps and unioning is far faster.

### NOT

Never materialise "all docs except X" — that is the whole corpus. Implement as a filter on the
iterator: advance the positive side, and check membership in the excluded set (bitset lookup, O(1)).
A `NOT` clause alone is not a query; it must be paired with something positive.

### MUST / SHOULD / MUST_NOT / FILTER

The Lucene boolean model, which you should copy:

- `MUST` — required **and scored**
- `FILTER` — required, **not scored** (so it can be cached as a bitmap, and skips the scoring path)
- `SHOULD` — optional, contributes score (with an optional `minimumShouldMatch` count)
- `MUST_NOT` — excluded

The `MUST` / `FILTER` distinction is a big practical win: `status:active` should never influence
ranking, and a filter's result bitmap can be cached across queries — a filter cache is often the
single highest-ROI cache in a search engine.

---

## 4.4 Phrase and proximity queries

`"quick brown fox"` — all three terms, adjacent, in order.

**Two-phase execution:**

1. Cheap: intersect the docID lists (conjunction over the three terms).
2. Expensive: for surviving docs, load positions and verify.

Positional verification: for each candidate doc, get position lists
`P₀, P₁, P₂` and look for `p ∈ P₀` with `p+1 ∈ P₁` and `p+2 ∈ P₂`. Do it as a merge over the
position lists, not nested loops.

**Two-phase iteration is a general pattern**, not a phrase-only trick. Lucene's `TwoPhaseIterator`
separates a cheap `approximation` from an expensive `matches()`. Any expensive predicate (phrase,
geo-distance, script) should use it, so the expensive check runs only on docs that survived
everything cheap.

Variants:

- **Slop / proximity**: `"quick fox"~3` — allow up to 3 position moves between them. Score usually
  decays with distance.
- **Ordered vs unordered** proximity (`span_near` in Lucene).
- **Common-grams / bigram index**: index `"quick_brown"` as a single term so that hot phrases become
  single-term lookups. Also called a *next-word index*. Big win for a known set of frequent phrases;
  costs index size.

**Cost warning:** positions are the expensive part of the index, and phrase queries on very common
terms (`"the who"`) are pathological — both lists are enormous and the intersection is enormous.
Common-grams exist precisely for this.

---

## 4.5 Wildcards, prefixes, ranges

All of these are **term expansion**: rewrite into a disjunction of concrete terms found by walking
the term dictionary.

- **Prefix `auto*`** — trivial with an ordered dictionary or FST: seek to `auto`, enumerate while the
  prefix holds.
- **Suffix `*tion`** — cannot use a prefix-ordered dictionary. Options: (a) a second dictionary of
  **reversed terms**; (b) the **permuterm index** — index every rotation of `term$` (`hello$`,
  `ello$h`, `llo$he`…), turning any single-wildcard pattern into a prefix query; (c) an n-gram index.
- **Infix `*sear*`** — n-gram index (file 08) or brute force over the dictionary.
- **Range `[apple TO banana]`** — enumerate the dictionary between two bounds.
- **Numeric ranges** — do *not* enumerate 1..1,000,000 terms. Two real approaches:
  - **Prefix-encoded numeric terms / trie levels**: index a number at multiple precisions so a range
    becomes a few dozen term lookups (old Lucene `NumericRangeQuery`).
  - **BKD tree / points** (modern Lucene): a disk-based, block-oriented KD-tree, which also handles
    multi-dimensional (geo) ranges. Numeric ranges are *not* text; they get their own structure.

**Expansion blow-up is a real hazard.** `a*` might match 200,000 terms and build a 200,000-clause
disjunction. Defences: a `maxExpansions` cap; rewriting to a `FILTER`-style bitmap instead of a
scored disjunction (Lucene's `CONSTANT_SCORE_REWRITE` builds a bitset — much faster, loses per-term
scoring); or refusing leading wildcards outright, which many production systems do.

---

## 4.6 Fuzzy search

Goal: `recieve` finds `receive` (Levenshtein distance 1).

**Bad:** compute edit distance against every term in the dictionary. O(|V|).

**Good — the Levenshtein automaton trick** (Schulz & Mihov; how Lucene does it):

1. Build a DFA that accepts exactly the strings within edit distance ≤ k of the query term. For
   fixed small k (1 or 2) this automaton is small and constructible in linear time.
2. **Intersect that DFA with the term dictionary FST.** Both are automata; the intersection walks
   both in lockstep and enumerates only matching terms.
3. Cost is proportional to the *output* size, not the dictionary size.

This is the single most beautiful algorithm in the whole field, and it is why `k ≤ 2` is a hard
limit in Elasticsearch — the automaton grows fast with k.

Alternatives: **BK-trees** (metric tree over edit distance, good for a modest dictionary);
**n-gram candidate generation** (terms sharing ≥ m trigrams, then verify) — simpler, approximate,
used by `pg_trgm` and by spell checkers.

Practical notes: require a matching prefix (`prefixLength=1–2`) — it cuts candidates enormously and
users rarely mistype the first letter; and score fuzzy matches lower than exact ones.

---

## 4.7 Top-k retrieval and early termination

The realisation that changes everything: **the user wants 10 results.** You do not need exact scores
for the other 4,999,990 — you only need to be sure none of them belongs in the top 10.

### Baseline: exhaustive + heap

Score every matching doc, keep a min-heap of size k. Correct, simple, and what you build in M3.
Cost ∝ number of matching docs.

### WAND (Weak AND / Weighted AND)

Precompute per term the **maximum contribution** it can ever make (`maxScore`, from its highest tf
and its idf). Keep the current k-th best score as a threshold `θ`.

Sort iterators by current docID. Accumulate max-scores from the top until the running sum exceeds
`θ` — the term at which it does is the **pivot**. Any document not containing the pivot term
cannot reach `θ`, so **advance every earlier iterator straight to the pivot's docID**. Whole
swathes of documents are never scored at all.

Result is **exact top-k** (not an approximation) with typically 2–10× fewer scored documents.

### Block-Max WAND (BMW)

WAND's weakness: one global `maxScore` per term is pessimistic — most blocks are nowhere near it.
Fix: store a **per-block max score** (§3.6). Now you can skip individual *blocks* whose max cannot
beat `θ`. Typically another 2–5× on top of WAND. This is the state of the art in production engines
and the reason modern Lucene is fast.

### Other strategies

- **MaxScore** (Turtle & Flood): partition terms into "essential" and "non-essential" given θ;
  non-essential terms are only checked, never driven. Often beats WAND for pure-OR queries.
- **Impact-ordered postings**: sort postings by impact score instead of docID, read the highest
  impacts first, stop early. Excellent for top-k, but breaks intersection and phrase queries — a
  fundamental trade-off, and the reason docID order is the default.
- **Tiered indexes**: a small "champion list" of the best documents per term, searched first;
  fall back to the full index only if not enough results.
- **Early exit on filters**: apply cheap, highly-selective filters as bitmaps *before* scoring.

**Ordering of work in a good executor:** cheapest and most selective first; expensive predicates
two-phase; scoring last; document loading last of all.

---

## 4.8 Multi-segment execution and result merging

Each segment is searched independently — naturally parallel across goroutines — producing a local
top-k with segment-local docIDs. The coordinator merges N local heaps into a global top-k, mapping
`globalDoc = segmentBase + localDoc`.

Correctness trap: **IDF is a corpus-level statistic**, but each segment only knows its own `df`. If
each segment computes IDF locally, scores are not comparable across segments and merged ranking is
subtly wrong. Real systems either accept the skew (it is small when segments are large and
random) or run a **collection-statistics gathering phase first** (this is exactly what
Elasticsearch's `dfs_query_then_fetch` does, and why it is slower). Know which you chose, and write
it down.

The same issue reappears one level up in sharded/distributed search — it is the identical problem
with a network in the middle.

---

## 4.9 Caches

| Cache | Key | Value | Notes |
|-------|-----|-------|-------|
| Filter/bitset cache | filter clause | Roaring bitmap of matching docs | Huge win; per segment, invalidated on merge |
| Query result cache | full query + filter set | top-k docIDs | Only useful with repeated identical queries |
| Page cache | OS-level | index blocks | Free, and usually the one that matters most |
| Field data / doc values | field | column of values | For sorting/faceting |

Cache **per segment**, never across the whole index: segments are immutable, so their cache entries
are valid until the segment dies. Caching against a mutable global view means invalidation logic,
which means bugs.

---

## 4.10 Query languages

Your library needs a programmatic query API first (`And(Term("a"), Term("b"))`), and *optionally* a
string parser. Do not start with the parser. Notes when you get there:

- Classic Lucene syntax: `+must -mustnot field:value "phrase"~2 term^3 wild* fuz~`
- Simple-query-string variants deliberately never throw parse errors on user input — important when
  the string comes from a public search box.
- Structured (JSON/DSL) query representations are easier to validate, compose, and secure than a
  free-text mini-language. Given a choice, expose a struct API and let callers build a parser.

---

## Check yourself

1. Why does `Advance(target)` matter more than `Next()` for performance? Give the asymptotic
   difference for intersecting lists of size 50 and 10,000,000.
2. Sketch WAND's pivot selection on three terms with maxScores 3.0, 2.0, 1.0 and θ = 4.5. Which term
   is the pivot and what happens next?
3. Why is a phrase query on two very common terms pathologically slow, and what index-time change
   fixes it?
4. What breaks if each segment computes IDF from its own local statistics? Construct a concrete
   two-segment example that mis-ranks.
5. `MUST` vs `FILTER`: give two distinct reasons the distinction earns its keep.
6. Why can impact-ordered postings not serve a phrase query?
