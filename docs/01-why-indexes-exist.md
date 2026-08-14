# 01 — Why Indexes Exist

## 1.1 The problem, stated honestly

Say I have 1,000,000 documents, average 2 KB each. Total 2 GB of text.

Someone asks: **"which documents contain the word `postgres`?"**

### Approach A: scan everything

Read all 2 GB, run a substring search over each document.

- Sequential read from NVMe SSD at ~3 GB/s → **~0.7 s** just for I/O.
- From a spinning disk at 150 MB/s → **~13 s**.
- From RAM, if it all fits, at ~10 GB/s with SIMD substring search → **~0.2 s**.

That is _per query_. Ten queries per second means ten machines doing nothing but scanning. And that
is the best case: a single literal word. Now make it `postgres AND replication NOT slony`, ranked by
relevance, with the results highlighted.

### Approach B: precompute

Before any query arrives, walk the corpus **once** and build a map:

```
"postgres"     → [17, 402, 8891, 8892, 15003, ...]
"replication"  → [402, 903, 8891, ...]
"slony"        → [8891]
```

Now the query is: look up two lists, intersect them, subtract a third. If `postgres` appears in
20,000 documents and `replication` in 5,000, I touch ~25,000 integers, maybe 40 KB compressed,
instead of 2 GB. **~30 µs instead of 700 ms. A 20,000× improvement.**

That map is an **inverted index**, and it is the single most important data structure in these notes.

### What it costs

Nothing is free. The bill:

| Cost           | Typical magnitude                                                                       |
| -------------- | --------------------------------------------------------------------------------------- |
| Build time     | One full pass over the corpus, plus sorting. Minutes to hours.                            |
| Disk space     | 10–30% of the raw text for a doc+frequency index; 30–80% with positions.                 |
| Write latency  | Every insert/update/delete now has to update the index too.                               |
| Complexity     | Crash recovery, merges, schema evolution, analyzer changes forcing a reindex.             |
| Staleness      | Most search indexes are _eventually_ consistent with the source of truth.                 |

**That is the bargain.** I want to be able to state it for every index I meet.

---

## 1.2 Forward index vs inverted index

**Forward index**, the natural direction, and what I get for free from a document store:

```
doc 17 → ["postgres", "streaming", "replication", "wal", ...]
doc 18 → ["mysql", "binlog", ...]
```

Answers _"what is in document 17?"_. Fast. Needed for highlighting, snippet generation, and
"more like this". Useless for search.

**Inverted index**, term as the key:

```
"postgres" → [17, 402, 8891, ...]
"mysql"    → [18, 55, ...]
```

Answers _"which documents contain X?"_. That inversion is literally the whole trick. "Inverted index"
is 1950s terminology; it just means the index is keyed by term instead of by document.

Real engines keep **both**: the inverted index for matching, and a forward structure (stored fields,
doc values, a column store) for scoring by non-text signals, sorting, faceting, and rendering
results.

> Design note I hit early: doc values / columnar forward data is how sorting by price or faceting by
> category stays fast. With only an inverted index, "sort 50,000 hits by timestamp" means random
> reads over 50,000 documents. With a column of timestamps laid out by docID, it is one sequential
> scan over 400 KB.

---

## 1.3 Vocabulary I need before anything else

| Term                          | Meaning                                                                                                                            |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **Document**                  | The unit I retrieve. A web page, a row, a log line, a source file, a paragraph chunk.                                               |
| **Field**                     | A named part of a document (`title`, `body`, `author`). Fields can have different analysis and different weights.                   |
| **Token**                     | A raw chunk from the tokenizer, before normalisation. `"Running"`.                                                                  |
| **Term**                      | The normalised, indexed form. `"run"`. **Terms are the keys of the index.**                                                        |
| **Posting**                   | One entry in a term's list: term T occurs in doc D, n times, at positions [...].                                                    |
| **Postings list**             | All postings for one term, sorted by document ID. Also called an _inverted list_.                                                   |
| **Term dictionary / lexicon** | The structure mapping term → location of its postings list.                                                                         |
| **df** (document frequency)   | In how many documents the term appears.                                                                                             |
| **tf** (term frequency)       | How many times the term appears in _one_ document.                                                                                  |
| **cf** (collection frequency) | Total occurrences across the whole corpus.                                                                                          |
| **DocID**                     | A dense internal integer, 0..N-1. Not the application's primary key; I keep a separate mapping. Dense small integers are what makes compression work. |
| **Segment**                   | An immutable, self-contained mini-index. A big index is a set of segments.                                                          |
| **Analyzer**                  | The pipeline text → terms. File 02.                                                                                                 |

---

## 1.4 Why the dictionary does not explode: Heaps' and Zipf's laws

My first fear was that a million documents would produce a million distinct words and a huge
dictionary. It does not work out that way.

**Heaps' law**: vocabulary size `V` grows sublinearly with total tokens `T`.

```
V ≈ k · T^β        with k ≈ 10–100, β ≈ 0.4–0.6
```

For a billion tokens of English, `V` lands around 1–2 million distinct terms, a few tens of MB.
Vocabulary grows roughly with the _square root_ of the corpus. That is why the term dictionary
usually fits in RAM even when the postings do not, and why I design for "dictionary hot, postings
cold".

> The caveat that matters in production: this holds for natural language. Corpora full of IDs,
> hashes, URLs, or log lines with embedded UUIDs have near-linear vocabulary growth and _will_ blow
> up the dictionary. That is a real failure mode ("we indexed our logs and the term dictionary is
> 40 GB"). The fix is analysis (file 02), not compression.

**Zipf's law**: the frequency of the _r_-th most common term is roughly proportional to `1/r`.

What I design around as a result:

- A handful of terms (`the`, `of`, `and`) appear in nearly every document. Their postings lists are
  enormous and nearly worthless for discrimination. That motivates stopwords, and later the smarter
  answer: IDF weighting plus skip-based early termination.
- The _long tail_ dominates the dictionary. Most terms occur once or twice, so I get millions of tiny
  postings lists and per-list overhead matters enormously. A 48-byte header per term costs more than
  the data when the average list is 2 entries. **So: pack tiny lists inline in the dictionary rather
  than pointing at a separate block.**

---

## 1.5 The cost model: what actually makes things slow

Indexing is not really about Big-O. It is about **bytes moved** and **random accesses**. The ladder I
keep in my head, order of magnitude, modern hardware:

| Operation                    | Latency        | Notes                                                |
| ---------------------------- | -------------- | ---------------------------------------------------- |
| L1 cache hit                 | ~1 ns          | ~64 B cache line                                     |
| L2                           | ~4 ns          |                                                      |
| L3                           | ~15 ns         | shared                                               |
| Main memory, random          | ~80–100 ns     | the real in-memory "random access" cost              |
| Main memory, sequential      | ~0.1 ns/byte   | prefetcher does the work; ~10–20 GB/s                |
| NVMe SSD random 4 KB read    | ~80–100 µs     | ~1000× RAM                                           |
| NVMe sequential              | ~3–7 GB/s      |                                                      |
| Network round trip (same DC) | ~0.5 ms        |                                                      |
| Spinning disk seek           | ~10 ms         | ~100,000× RAM                                        |

Four rules fall out of that table, and they explain most of index design:

1. **Sequential beats random by 100–1000×.** So postings lists are stored contiguously and read
   forward. That is why they are sorted by docID and why nobody stores a linked list of postings.
2. **The unit of I/O is a page (4 KB), not a byte.** Reading 8 bytes costs the same as reading 4096.
   So pack related data together. B-tree nodes are page-sized for exactly this reason.
3. **Compression is a speed optimisation, not just a space one.** If decoding costs 1 ns/byte and
   fetching costs 5 ns/byte, halving the bytes makes me faster even though I added CPU work. That
   inverts the usual intuition and is why every serious index compresses postings.
4. **Avoid pointer-chasing.** A `map[string][]uint32` in Go is three levels of indirection and a GC
   nightmare at scale. Real indexes use flat byte arrays plus offsets. File 10 has the details.

---

## 1.6 The query shapes

The question I ask of any index: **what does it answer in sub-linear time?**

| Query shape                  | Right structure                     | Where I have seen it                          |
| ---------------------------- | ----------------------------------- | --------------------------------------------- |
| **Exact key → value**       | Hash index                          | `map[k]v`, memcached, Postgres hash index     |
| **Ordered key → range**     | B+tree, sorted array, LSM           | `WHERE ts BETWEEN a AND b`, primary keys      |
| **Term → set of documents** | Inverted index                      | Full-text search, Lucene, Postgres GIN        |
| **Substring / regex**        | n-gram (trigram) index, suffix array | Code search, `LIKE '%x%'`, `pg_trgm`          |
| **Nearest neighbour in ℝⁿ**  | HNSW, IVF-PQ, LSH                   | Semantic search, RAG, recommendations         |
| **Multi-dimensional range**  | KD-tree, BKD, R-tree, GiST          | Geo queries, numeric ranges in Lucene         |
| **Set membership, probabilistic** | Bloom / cuckoo filter          | LSM read path, "definitely not here"          |

An index that answers one shape is usually _useless_ for another. A B+tree on `name` cannot serve
`LIKE '%son'` because the tree is ordered by prefix. An inverted index over words cannot make `route`
match `rerouted` unless I indexed n-grams. **Mismatched index shape is the number one cause of "why
is my query slow when I have an index?"**

---

## 1.7 Where this gets used

Concretely, the applications that make this worth learning:

**Search engines and site search.** The obvious one. Google, but also the search box on any
e-commerce site. Relevance ranking (file 05) matters more than raw matching there, because the user
sees 10 of 40,000 matches.

**E-commerce and faceting.** "Shoes, size 42, under €100, in stock, sorted by rating." That is an
inverted index for the text, plus bitmap/doc-value intersections for the filters, plus aggregation
for the facet counts (`Brand (412)`, `Colour (89)`). Facet counting is counting set bits over
intersected bitmaps, which is file 03's Roaring section.

**Log and observability platforms** (Elasticsearch, Loki, Splunk, ClickHouse). Terabytes a day,
write-heavy, queried by a handful of engineers during incidents. The design shifts hard toward cheap
writes. Loki deliberately indexes only _labels_, not log content, and brute-force greps the rest, a
conscious "the index is too expensive here" call. Knowing when _not_ to index is as valuable as
knowing how.

**Autocomplete.** Edge n-grams or an FST traversal. Latency budget ~20 ms including network, so it is
usually a separate, tiny, RAM-resident index.

**Code search and IDE navigation.** File 08. Trigram indexes for regex, symbol indexes for go-to-
definition, SCIP/LSIF for precise cross-repo references.

**RAG and AI agents.** Chunk documents, embed, ANN-search, and combine with a lexical index because
embeddings are bad at exact identifiers, error codes, and rare proper nouns. It is also how coding
agents work: ripgrep for exact strings, an embedding index for vague intent, a symbol index to
navigate. File 09.

**Deduplication and near-duplicate detection.** Shingles plus MinHash/SimHash over an inverted index
of hash buckets. Plagiarism detection, crawler dedup, training-data cleaning.

**Security/SIEM and e-discovery.** "Every log line mentioning this IP in the last 90 days", "every
email containing these terms between these dates". Needs a text index with time-partitioned segments
and strict auditability.

**Databases themselves.** Every `CREATE INDEX` I have ever run. File 07 shows Postgres GIN is
structurally the same inverted index I am about to build.

### When I should not build an index

- Data volume genuinely small (< ~50 MB). A linear scan with a good substring algorithm beats index
  maintenance and is far less code. ripgrep searches the Linux kernel source (~1.5 GB) in under a
  second with no index at all.
- Write:read ratio very high and queries rare. Indexes optimise reads at the cost of writes.
- The query shape changes constantly and unpredictably. I cannot precompute for a question I do not
  know yet.
- One-shot analysis. Building an index to run three queries is a loss.

---

## 1.8 The mental model I carry forward

```
                 ┌──────────────┐
   raw docs ───▶ │   ANALYSIS   │ ───▶ terms + positions        (file 02)
                 └──────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │  INVERSION   │  sort by (term, docID)        (file 06)
                 └──────────────┘
                        │
                        ▼
        ┌────────────────────────────────┐
        │  TERM DICT  →  POSTINGS LISTS  │  compressed, on disk  (file 03)
        └────────────────────────────────┘
                        │
      query ──▶ ┌──────────────┐
                │  EXECUTION   │  intersect / union / skip      (file 04)
                └──────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │   RANKING    │  BM25, top-k heap             (file 05)
                 └──────────────┘
                        │
                        ▼
                  ranked results
```

Five boxes. That is the entire library. Everything else is optimisation of one box.

---

## Questions I should be able to answer before moving on

1. My corpus doubles. By roughly how much does the term dictionary grow, and why?
2. Why are postings lists sorted by docID rather than by term frequency? There is a real argument for
   both orderings. Find the docID one first, then look up "impact-ordered indexes" for the
   counter-argument.
3. I need to support `WHERE email LIKE '%@gmail.com'` over 50M rows. Which of the seven query shapes
   in 1.6 is that, and which index serves it?
4. State the cost bargain for a positional index versus a doc-only index, one sentence each.
5. Loki indexes only labels and greps the rest. Under what read/write pattern is that the right call,
   and when does it become the wrong one?
