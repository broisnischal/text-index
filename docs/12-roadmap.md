# 12 — Build Roadmap

Eleven milestones. Each one is **independently useful, independently testable, and small enough to
finish**. Every milestone lists what you build, the Go you learn, how you prove it works, and the
concepts it makes concrete.

Do not skip M1 to get to the "real" index. M1 is your test oracle for everything after it.

---

## M0 — Go warm-up (½–1 day)

**Build:** a CLI that walks a directory, reads text files, counts word frequencies, prints the top
20. Then extend it: `-min-length`, `-lower`, read from stdin, handle UTF-8 correctly.

**Learn:** modules and `go.mod`, packages, `bufio.Scanner` (and its 64 KB line limit — hit it
deliberately), `strings`/`bytes`, maps, sorting with `slices.SortFunc`, `flag`, `os`, error wrapping,
`go test`.

**Done when:** it processes 100 MB in a few seconds, handles a file with no trailing newline, a file
with CRLF, a UTF-8 file with a BOM, and an empty file, without special-casing any of them at the
call site.

---

## M1 — In-memory inverted index + boolean queries (1–2 days)

**Build:**
```
Index:  map[string][]uint32          // term → sorted docIDs
        docs []Document
Ops:    Add(doc), Search(AND/OR/NOT of terms)
Plus:   a BruteForceIndex that scans every document — the reference implementation
```

**Learn:** structs and methods, interfaces (`Query` with one `Matches`/`Iterate` method), sorting,
table-driven tests, subtests.

**Concepts:** files 01 and 03 §3.1–3.2, file 04 §4.3. Implement leapfrog intersection by hand and
watch it beat the naive nested loop.

**Done when:** a randomised differential test — random 1000-doc corpus, 1000 random boolean queries —
shows the fast index and the brute-force index agree exactly. **Keep both implementations forever.**

---

## M2 — Analysis pipeline (2–3 days)

**Build:** `Tokenizer` and `TokenFilter` interfaces; whitespace + Unicode tokenizers; lowercase-fold,
ASCII-fold, stopword, and a Porter stemmer (port it; it is ~300 lines of rules and porting it teaches
you a lot); an `Analyzer` composing them. Positions and offsets on every token.

**Learn:** `unicode`, `unicode/utf8`, `golang.org/x/text` (`norm`, `cases`, `runes`), iterators
(`iter.Seq`), interface composition.

**Concepts:** all of file 02.

**Done when:** the worked example in file 02 §2.6 produces exactly the documented terms, positions
and offsets; a fuzz test confirms the tokenizer never panics and never loses/overruns byte offsets on
arbitrary input (including invalid UTF-8).

---

## M3 — BM25 scoring and top-k (1–2 days)

**Build:** store `tf` per posting and doc length per document; implement BM25 with configurable
`k₁`/`b`; a size-k min-heap collector; `Explain()` producing a score breakdown.

**Learn:** `container/heap` (or hand-roll one with generics), float semantics, benchmark basics.

**Concepts:** file 05 §5.3, file 04 §4.7 baseline.

**Done when:** a hand-computed BM25 score for a 5-document toy corpus matches your implementation to
1e-9, and `Explain` prints each term's idf, tf-factor, and contribution. Add a tiny nDCG@10 harness
now, while it is cheap.

---

## M4 — On-disk segments (4–7 days) ← the big one

**Build:** serialise one segment: term dictionary (sorted blocks + front coding + a sparse in-memory
index) and postings (delta gaps + varint). Read it back with `io.ReaderAt`. Then swap in a
block-of-128 bit-packed codec behind the `Codec` interface and compare.

**Learn:** `encoding/binary`, `io.ReaderAt`, `bufio`, file layout discipline, benchmarking with
`benchstat`, `pprof`, escape analysis.

**Concepts:** file 03 §3.3–3.4 and §3.7, file 11 §11.5.

**Done when:**
- The same differential test passes against the on-disk index.
- Index size on a 100 MB corpus is < 25% of the raw text (docs+freqs, no positions).
- Fuzz test: every corrupted byte sequence yields an error, never a panic or an OOM.
- A benchmark table compares varint vs bit-packed: size and decode ns/op. **Write down the numbers.**

---

## M5 — Durability, deletes, merges (3–5 days)

**Build:** WAL, commit points (`segments_N` + atomic rename + directory fsync), crash recovery,
delete-by-id with a live-docs bitset, a tiered merge policy running in a background goroutine with
refcounted file deletion.

**Learn:** `os` file semantics, `Sync`, atomic rename, `defer` discipline, goroutine lifecycle,
`sync.Mutex`, refcounting.

**Concepts:** all of file 06.

**Done when:** the kill-test passes — a writer loop killed with `kill -9` at 100 random points,
reopened each time, and every acknowledged document is present with no corruption. Automate it.

---

## M6 — Positions, phrases, highlighting (2–3 days)

**Build:** a positions file; phrase queries with two-phase iteration; slop/proximity; a highlighter
using stored offsets (and a fallback that re-analyses stored text).

**Learn:** parallel stream decoding, two-phase iterator design, offset bookkeeping.

**Concepts:** file 04 §4.4, file 02 on offsets.

**Done when:** `"quick brown"` matches only documents with those words adjacent and in order; slop=2
behaves as specified; highlighted snippets point at the correct bytes in documents containing
multi-byte UTF-8 before the match (this is where offset bugs surface).

---

## M7 — Skip lists and Block-Max WAND (3–5 days)

**Build:** multi-level skip data with per-stream offsets; `Advance()` that uses it; per-block max
impact scores; WAND, then Block-Max WAND.

**Learn:** the performance-tuning loop for real — profile, hypothesise, change, `benchstat`, repeat.

**Concepts:** file 03 §3.6, file 04 §4.7.

**Done when:** a benchmark shows (a) intersecting a 50-doc list with a 1M-doc list costs ~50 seeks,
not 1M steps; (b) BMW scores at least 3× fewer documents than exhaustive scoring on a realistic
multi-term query; (c) **top-10 results are byte-identical to the exhaustive scorer** — this is exact
top-k, and any difference is a bug.

---

## M8 — FST dictionary, prefix, wildcard, fuzzy (4–6 days)

**Build:** an FST-backed term dictionary (build your own with the incremental minimisation algorithm,
then compare against `vellum`); prefix and range enumeration; a Levenshtein automaton for k ≤ 2
intersected with the FST; term-expansion caps and constant-score rewriting.

**Learn:** automata, recursion over shared structures, memory-conscious data structures.

**Concepts:** file 03 §3.4, file 04 §4.5–4.6.

**Done when:** the dictionary for 1M terms fits in a few MB and beats the sorted-block version on
lookup; fuzzy search for a term with `maxEdits=1` returns exactly the terms a brute-force edit-distance
scan returns, in a fraction of the time.

---

## M9 — Concurrency and mmap (2–4 days)

**Build:** parallel per-segment search with `errgroup`; atomic snapshot pointer swap; `sync.Pool` for
decode buffers; `context` cancellation; optional mmap directory; near-real-time refresh.

**Learn:** `sync/atomic`, `errgroup`, `context`, `sync.Pool`, `-race`, mmap lifetime hazards.

**Concepts:** file 06 §6.6, file 10 §10.11.

**Done when:** a mixed workload (writers + searchers + merges) runs clean under `-race` for ten
minutes; query throughput scales roughly linearly to 4 cores; a document is searchable within one
refresh interval of being added.

---

## M10 — Trigram / code search (3–5 days)

**Build:** a trigram index over files; regex → trigram boolean query translation; candidate
verification with `regexp`; brute-force fallback; path- and symbol-aware ranking. Optionally
tree-sitter symbol extraction.

**Learn:** `regexp/syntax` (parse a regex into an AST you can analyse — this is the fun part),
recursion over the regex AST, file walking, content-hash dedup.

**Concepts:** all of file 08.

**Done when:** `func \w+Handler\(` over the Go standard library returns exactly what `rg` returns,
and you can state your latency ratio against `rg` honestly, in both directions (index build time
included).

---

## M11 (optional) — Vectors and hybrid

**Build:** exact brute-force kNN with pre-filtering by a Roaring bitmap from the lexical side; RRF
fusion of two ranked lists; optionally HNSW.

**Concepts:** file 09.

**Done when:** a hybrid query beats either retriever alone on your nDCG harness — and if it does not,
you can explain why with data.

---

## Cross-cutting, do continuously

- **Benchmark every milestone** and commit the numbers to a `BENCHMARKS.md`. The graph of your own
  improvement is the most motivating thing in the project.
- **Keep `FORMAT.md` in sync** with the on-disk layout, in the same commit.
- **Keep the brute-force index** and run the differential test in CI.
- **Write the doc comment before the function.** If it is hard to describe, the design is wrong.
- **Use a real corpus** from M4 onward: a Wikipedia dump extract, the Go standard library, the Enron
  email set, or your own Markdown/notes. Toy corpora hide every interesting bug.

---

## Realistic schedule

| Pace | Timeline to M7 (a genuinely good index) |
|------|------------------------------------------|
| Full time | 4–6 weeks |
| ~2 h/day | 3–4 months |
| Weekends | 5–7 months |

**M1–M5 is the 80%.** If you stop after M5 you will have a working, durable, correct search library
and you will understand every line of it. M6–M11 are depth, and each is optional and independent.

---

## The one rule

**Do not optimise before M4's benchmarks exist.** Every performance decision from M4 onward should be
justified by a `benchstat` output you can point at. That habit — measure, change, measure — is worth
more than any single algorithm in these documents.
