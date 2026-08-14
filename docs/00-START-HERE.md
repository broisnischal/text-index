# Text Indexing, From First Principles to a Go Library

This is a self-study course. The end product is **a text indexing library written in Go**, built by
you, from scratch, understanding every byte you write to disk.

Nothing here asks you to copy code. Every file explains _the idea_, _why it exists_, _what it costs_,
and _what you will have to decide when you implement it_.

---

## The one sentence version

> An index is a **redundant, precomputed data structure** that trades **write time, memory, and disk
> space** for the ability to answer **one specific shape of question** without looking at all the data.

Every index — a B-tree in Postgres, an inverted index in Lucene, a trigram index in Zoekt, an HNSW
graph in a vector DB — is that same bargain with different parameters. Once you internalise this,
"database indexing" and "search indexing" and "code indexing" stop being three topics and become one
topic with three query shapes.

---

## What you will end up understanding

By the end you should be able to answer, without hand-waving:

- Why searching 10 GB of text for a word can take 30 microseconds instead of 4 seconds.
- Why `WHERE name LIKE '%foo%'` cannot use a normal database index, and what index _can_ serve it.
- Why Lucene/Elasticsearch segments are immutable, and why that is the same idea as an LSM tree.
- What BM25 actually computes and why term frequency saturates.
- Why Google Code Search used trigrams, and how a regex gets translated into a boolean trigram query.
- Why your postings list should store `[5, 3, 12, 1]` instead of `[5, 8, 20, 21]`.
- When a vector index beats an inverted index, when it loses badly, and how to combine them.
- How to lay all this out in Go without drowning in garbage collection.

---

## Reading order

| #   | File                                                                     | What it gives you                                                             |
| --- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| 01  | [`01-why-indexes-exist.md`](01-why-indexes-exist.md)                     | First principles, the cost model, real applications, when NOT to index        |
| 02  | [`02-text-analysis.md`](02-text-analysis.md)                             | Raw bytes → tokens → terms. The pipeline that decides what is even searchable |
| 03  | [`03-inverted-index-internals.md`](03-inverted-index-internals.md)       | Postings lists, term dictionaries, FSTs, compression. **The core file.**      |
| 04  | [`04-query-execution.md`](04-query-execution.md)                         | Intersection, phrases, wildcards, fuzzy, top-k, early termination             |
| 05  | [`05-ranking-and-relevance.md`](05-ranking-and-relevance.md)             | TF-IDF, BM25, evaluation metrics, learning-to-rank                            |
| 06  | [`06-storage-segments-durability.md`](06-storage-segments-durability.md) | Building indexes bigger than RAM, segments, merges, crash safety              |
| 07  | [`07-database-indexing.md`](07-database-indexing.md)                     | B+trees, LSM, bitmap, Postgres GIN/GiST/BRIN, query planners                  |
| 08  | [`08-code-indexing.md`](08-code-indexing.md)                             | Trigram indexes, ctags, tree-sitter, LSIF/SCIP, how code search really works  |
| 09  | [`09-vector-and-hybrid-search.md`](09-vector-and-hybrid-search.md)       | Embeddings, HNSW/IVF-PQ, hybrid fusion, RAG                                   |
| 10  | [`10-go-for-index-builders.md`](10-go-for-index-builders.md)             | The Go you specifically need, mapped to the problems above                    |
| 11  | [`11-library-design.md`](11-library-design.md)                           | API surface, package layout, file format, testing strategy                    |
| 12  | [`12-roadmap.md`](12-roadmap.md)                                         | M0 → M10 build plan with acceptance criteria                                  |
| 13  | [`13-glossary-and-resources.md`](13-glossary-and-resources.md)           | Vocabulary + the papers/books/codebases worth your time                       |

Read 01–05 before touching a keyboard. 06–09 can be skimmed first and revisited when the roadmap
reaches them. 10 is a reference you will keep coming back to.

---

## How to actually learn this (not just read it)

The failure mode is reading all thirteen files, feeling informed, and building nothing. Avoid it:

1. **Read 01 and 02.** Then, before reading 03, write down on paper the data structure _you_ would
   build to answer "which of these 100k documents contain the word `router`". Keep the note.
2. **Read 03.** Compare with your note. The gap is your actual learning.
3. **Start M1 in the roadmap** (in-memory index, ~200 lines of Go) as soon as you finish 04. Do not
   wait until you understand compression. A slow correct index is the reference implementation you
   will test the fast one against — it is not throwaway work, it is your test oracle.
4. **Measure everything.** Every milestone has a benchmark. "It feels fast" is not a result;
   `12.3 µs/op, 4 allocs/op` is a result.
5. **Break it deliberately.** Kill the process mid-write. Feed it Turkish text, emoji, a 500 MB
   single-line JSON file, a document that is one word repeated a million times.

---

## Prerequisites

- Comfortable in _some_ language. Go itself is taught in file 10 alongside the project.
- Rough sense of Big-O and of the memory hierarchy (RAM is ~100× slower than L1, an SSD read is
  ~1000× slower than RAM). File 01 makes this concrete.
- No maths beyond logarithms. BM25 looks scary and is not.

---

## A note on scope

You are building a **library**, not Elasticsearch. The library's job:

```
documents in  →  [analysis] → [index] → [query] → ranked document IDs out
```

Not in scope (and it is important to say so out loud): distributed sharding, replication, a query
DSL over HTTP, a cluster membership protocol. Those are _systems_ problems layered on top of an
index. Single-node correctness and speed first. If you get that right, sharding is a weekend.
