# Text Indexing, From First Principles to a Go Library

These are my notes for building a text indexing library in Go from scratch, and my reasoning about
why each piece works the way it does. I want to understand every byte I write to disk, so I am not
copying code from anywhere. Each file here records _the idea_, _why it exists_, _what it costs_, and
_the decisions I have to make when I implement it_.

---

## The one sentence version

> An index is a **redundant, precomputed data structure** that trades **write time, memory, and disk
> space** for the ability to answer **one specific shape of question** without looking at all the data.

Every index I have looked at is that same bargain with different parameters: a B-tree in Postgres, an
inverted index in Lucene, a trigram index in Zoekt, an HNSW graph in a vector DB. Once I hold that
straight, "database indexing" and "search indexing" and "code indexing" stop being three topics and
become one topic with three query shapes.

---

## What I want to be able to answer without hand-waving

- Why searching 10 GB of text for a word can take 30 microseconds instead of 4 seconds.
- Why `WHERE name LIKE '%foo%'` cannot use a normal database index, and what index _can_ serve it.
- Why Lucene/Elasticsearch segments are immutable, and why that is the same idea as an LSM tree.
- What BM25 actually computes and why term frequency saturates.
- Why Google Code Search used trigrams, and how a regex gets translated into a boolean trigram query.
- Why my postings list should store `[5, 3, 12, 1]` instead of `[5, 8, 20, 21]`.
- When a vector index beats an inverted index, when it loses badly, and how to combine them.
- How to lay all this out in Go without drowning in garbage collection.

---

## Reading order

| #   | File                                                                     | What it covers                                                                |
| --- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| 01  | [`01-why-indexes-exist.md`](01-why-indexes-exist.md)                     | First principles, the cost model, real applications, when NOT to index         |
| 02  | [`02-text-analysis.md`](02-text-analysis.md)                             | Raw bytes → tokens → terms. The pipeline that decides what is even searchable |
| 03  | [`03-inverted-index-internals.md`](03-inverted-index-internals.md)       | Postings lists, term dictionaries, FSTs, compression. **The core file.**       |
| 04  | [`04-query-execution.md`](04-query-execution.md)                         | Intersection, phrases, wildcards, fuzzy, top-k, early termination              |
| 05  | [`05-ranking-and-relevance.md`](05-ranking-and-relevance.md)             | TF-IDF, BM25, evaluation metrics, learning-to-rank                             |
| 06  | [`06-storage-segments-durability.md`](06-storage-segments-durability.md) | Building indexes bigger than RAM, segments, merges, crash safety               |
| 07  | [`07-database-indexing.md`](07-database-indexing.md)                     | B+trees, LSM, bitmap, Postgres GIN/GiST/BRIN, query planners                   |
| 08  | [`08-code-indexing.md`](08-code-indexing.md)                             | Trigram indexes, ctags, tree-sitter, LSIF/SCIP, how code search really works   |
| 09  | [`09-vector-and-hybrid-search.md`](09-vector-and-hybrid-search.md)       | Embeddings, HNSW/IVF-PQ, hybrid fusion, RAG                                    |
| 10  | [`10-go-for-index-builders.md`](10-go-for-index-builders.md)             | The Go this project actually needs, mapped to the problems above               |
| 11  | [`11-library-design.md`](11-library-design.md)                           | API surface, package layout, file format, testing strategy                     |
| 12  | [`12-roadmap.md`](12-roadmap.md)                                         | M0 → M11 build plan with acceptance criteria                                  |
| 13  | [`13-glossary-and-resources.md`](13-glossary-and-resources.md)           | Vocabulary + the papers/books/codebases worth my time                          |

I work through 01–05 before touching a keyboard. 06–09 I skim first and come back to when the
roadmap reaches them. 10 stays open in a tab the whole time.

---

## How I keep this from turning into passive reading

The failure mode is reading all thirteen files, feeling informed, and building nothing. My guard
against it:

1. **Read 01 and 02.** Then, before reading 03, write down on paper the data structure I would build
   to answer "which of these 100k documents contain the word `router`". Keep the note.
2. **Read 03.** Compare against that note. The gap is the actual learning.
3. **Start M1 in the roadmap** (in-memory index, ~200 lines of Go) as soon as I finish 04. I do not
   wait until I understand compression. A slow correct index is the reference implementation I test
   the fast one against, so it is not throwaway work, it is my test oracle.
4. **Measure everything.** Every milestone gets a benchmark. "It feels fast" is not a result;
   `12.3 µs/op, 4 allocs/op` is a result.
5. **Break it deliberately.** Kill the process mid-write. Feed it Turkish text, emoji, a 500 MB
   single-line JSON file, a document that is one word repeated a million times.

---

## What I am assuming going in

- I already write code in some language. Go itself I pick up in file 10 alongside the project.
- Rough sense of Big-O and of the memory hierarchy (RAM is ~100× slower than L1, an SSD read is
  ~1000× slower than RAM). File 01 makes that concrete.
- No maths beyond logarithms. BM25 looks scary and is not.

---

## Scope

I am building a **library**, not Elasticsearch:

```
documents in  →  [analysis] → [index] → [query] → ranked document IDs out
```

Out of scope, and I am writing it down so scope creep has to argue its way in: distributed sharding,
replication, a query DSL over HTTP, a cluster membership protocol. Those are _systems_ problems
layered on top of an index. Single-node correctness and speed first. If I get that right, sharding is
a weekend.
