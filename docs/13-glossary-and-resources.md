# 13 — Glossary and Resources

---

## Glossary

**Analyzer** — the pipeline turning raw text into indexed terms: char filters → tokenizer → token
filters. Baked into the index at write time.

**ANN (Approximate Nearest Neighbour)** — vector search trading exactness for speed. HNSW, IVF, PQ,
LSH.

**avgdl** — average document length in terms, the length-normalisation baseline in BM25.

**BKD tree** — block KD-tree, Lucene's disk-based structure for numeric and multi-dimensional ranges.

**Block-Max WAND (BMW)** — WAND plus per-block maximum scores, so whole postings blocks can be
skipped. State of the art for exact top-k.

**Bloom filter** — probabilistic set membership: "definitely not present" or "probably present". Used
in LSM read paths.

**BM25** — the standard lexical ranking function. TF saturation + IDF + length normalisation.

**BM25F** — BM25 for multi-field documents, combining field term frequencies _before_ saturation.

**BSBI / SPIMI** — blocked sort-based indexing / single-pass in-memory indexing. Two ways to build an
index larger than RAM. SPIMI is what modern systems use.

**cf (collection frequency)** — total occurrences of a term across the whole corpus.

**Codec** — the pluggable component encoding and decoding postings on disk.

**Commit point** — the file naming the current set of live segments, the atomic unit of durability.

**Compaction / merge** — combining small immutable files into larger ones, discarding deleted data.

**Covering index** — an index containing every column a query needs, so the table is never read. The
search-engine equivalent is doc values.

**df (document frequency)** — how many documents contain a term.

**DocID** — dense internal integer identifier, 0..N-1, segment-local.

**Doc values** — columnar forward storage (docID → value) for sorting, faceting and filtering.

**Elias-Fano** — quasi-succinct encoding of monotone sequences with O(1) random access.

**Forward index** — document → terms. The natural direction.

**FST (Finite State Transducer)** — automaton mapping strings to values, sharing prefixes _and_
suffixes. Lucene's term dictionary. Go: `vellum`.

**GIN / GiST / BRIN / SP-GiST** — Postgres index types: inverted, generalised search tree, block
range, space-partitioned.

**Heaps' law** — vocabulary grows as `k·T^β` (β ≈ 0.5), sublinearly with corpus size.

**HNSW** — hierarchical navigable small world, the default ANN graph index.

**IDF (inverse document frequency)** — `ln((N−df+0.5)/(df+0.5)+1)`. Rare terms weigh more.

**Impact-ordered index** — postings sorted by score instead of docID. Great for top-k, useless for
phrases.

**Inverted index** — term → sorted list of documents. The core structure.

**LSIF / SCIP** — precomputed code-intelligence index formats (definitions, references, hover). SCIP
is the modern successor.

**LSM tree** — log-structured merge tree. Memtable → immutable sorted files → compaction.
Structurally identical to Lucene's segment architecture.

**Leapfrog / galloping search** — intersection strategy that seeks forward rather than stepping.

**Levenshtein automaton** — a DFA accepting all strings within edit distance k, intersected with an
FST for fast fuzzy search.

**Live docs** — bitset marking non-deleted documents in a segment.

**nDCG** — normalised discounted cumulative gain, the standard graded ranking metric.

**Norm** — the stored per-document length factor used by BM25, often compressed to one byte.

**PForDelta** — bit-packing with exceptions for outlier values.

**Permuterm index** — index all rotations of `term$` so any single-wildcard query becomes a prefix
query.

**Posting / postings list** — one entry (docID, tf, positions) / the full list for a term.

**Positional index** — postings including token positions, required for phrase queries.

**RRF (Reciprocal Rank Fusion)** — combine ranked lists by `Σ 1/(k+rank)`, k=60. The default hybrid
fusion method.

**Roaring bitmap** — compressed bitmap using array/bitmap/run containers per 2^16 chunk.

**RUM conjecture** — optimise for Read, Update, or Memory: pick two.

**Segment** — an immutable, self-contained mini-index. Indexes are sets of segments.

**Selectivity** — the fraction of rows or documents a predicate keeps. Drives planner decisions.

**Skip list** — pointers embedded in postings enabling `Advance(target)` in O(log n).

**SPLADE** — learned sparse retrieval, neural weights stored in an ordinary inverted index.

**Stemming / lemmatisation** — rule-based suffix stripping / dictionary-based base-form lookup.

**Stopwords** — ubiquitous terms optionally removed. Mostly obsolete practice.

**tf (term frequency)** — occurrences of a term in one document.

**Tombstone** — a deletion marker; the data is removed only at merge or compaction.

**Trigram index** — inverted index over 3-byte substrings, enabling substring and regex search.

**Two-phase iteration** — cheap approximation first, expensive verification only on survivors.

**Varint / VByte / LEB128** — variable-length integer encoding, 7 bits per byte.

**WAL (write-ahead log) / translog** — append-only durability log replayed after a crash.

**WAND (Weak AND)** — top-k algorithm using per-term max scores to skip documents that cannot make the
cut.

**Write amplification** — bytes physically written / bytes of user data.

**Zipf's law** — term frequency ∝ 1/rank. A few terms dominate; most occur once.

---

## Books

| Book                                                                       | Why                                                                                                                          |
| -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Introduction to Information Retrieval** — Manning, Raghavan, Schütze     | The textbook, free online at nlp.stanford.edu/IR-book/. Chapters 1–7 cover files 01–05 of these notes rigorously. Start here. |
| **Search Engines: Information Retrieval in Practice** — Croft, Metzler, Strohman | More systems-oriented, more practical. Free PDF available.                                                             |
| **Managing Gigabytes** — Witten, Moffat, Bell                              | Old (1999) and still the best book on index compression specifically.                                                        |
| **Designing Data-Intensive Applications** — Kleppmann                      | Chapter 3 is the clearest explanation of B-trees vs LSM ever written. Read alongside file 07.                                |
| **Database Internals** — Petrov                                            | Deeper on B-trees, LSM, storage engine mechanics.                                                                            |
| **The Go Programming Language** — Donovan & Kernighan                      | Still the best Go book.                                                                                                      |
| **Learn Go with Tests** — free online                                      | Test-first Go, matching this project's methodology.                                                                          |

---

## Papers and articles

Must read, short and high value:

- Russ Cox, _"Regular Expression Matching with a Trigram Index"_ (swtch.com/~rsc/regexp/regexp4.html),
  how Google Code Search worked. Read before M10.
- Robertson & Zaragoza, _"The Probabilistic Relevance Framework: BM25 and Beyond"_, the definitive
  BM25 reference.
- Broder et al., _"Efficient Query Evaluation using a Two-Level Retrieval Process"_ (2003), WAND.
- Ding & Suel, _"Faster Top-k Document Retrieval Using Block-Max Indexes"_ (2011).
- Michael McCandless, _"Using Finite State Transducers in Lucene"_ (blog.mikemccandless.com), the most
  approachable FST explanation.
- Schulz & Mihov, _"Fast String Correction with Levenshtein Automata"_ (2002), plus Jules Jacobs' and
  Steve Hanov's blog explanations, which are far easier entry points.
- Lemire et al., _"Better bitmap performance with Roaring bitmaps"_.
- Zobel & Moffat, _"Inverted Files for Text Search Engines"_ (ACM Computing Surveys, 2006), the survey
  that ties everything together.

Worth knowing:

- Vigna, _"Quasi-Succinct Indices"_ (Elias-Fano for postings).
- Lin & Trotman, _"Anytime Ranking for Impact-Ordered Indexes"_.
- Malkov & Yashunin, _"Efficient and robust approximate nearest neighbor search using HNSW"_.
- Formal et al., _"SPLADE"_; Nogueira et al., _"docTTTTTquery"_.
- Cormack et al., _"Reciprocal Rank Fusion"_ (2009).
- Athanassoulis et al., _"Designing Access Methods: The RUM Conjecture"_.

---

## Codebases to read

| Project                                | Language | Read it for                                                                     |
| -------------------------------------- | -------- | -------------------------------------------------------------------------------- |
| **SQLite FTS5**                        | C        | The most readable complete inverted index in existence, with docs describing the format precisely |
| **bleve / scorch**                     | Go       | The closest existing analogue to this project                                    |
| **vellum**                             | Go       | FST construction and traversal                                                   |
| **RoaringBitmap/roaring**              | Go       | Compressed bitmaps done well                                                     |
| **zoekt**                              | Go       | Trigram code search in production                                                |
| **tantivy**                            | Rust     | A modern, clean, fast Lucene-alike. Excellent code quality                       |
| **Lucene**                             | Java     | The reference implementation of nearly everything here. Start with `org.apache.lucene.codecs.lucene99` |
| **ripgrep**                            | Rust     | How to be fast without an index at all                                           |
| **Postgres `src/backend/access/gin/`** | C        | GIN internals; see the `README` in that directory                                |
| **RocksDB**                            | C++      | LSM in anger                                                                     |
| **Quickwit**                           | Rust     | Search over object storage, a genuinely different architecture                   |

---

## Datasets for benchmarking

- **My own notes and Markdown** — smallest useful corpus, and I know the right answers.
- **Go standard library source** (~2M lines) — perfect for M10 code search.
- **Enron email corpus** (~500 MB, 500k messages) — the classic IR test set.
- **Wikipedia dump extract** — a few GB of real prose, via `wikiextractor`.
- **MS MARCO** (passage ranking, 8.8M passages) — has relevance judgments, so real nDCG is possible.
- **BEIR** — 18 retrieval datasets, the standard for comparing lexical vs dense.
- **CodeSearchNet** — code plus natural language queries, for M10/M11.

---

## Tools

```bash
go test -bench=. -benchmem -count=10 | tee new.txt
go install golang.org/x/perf/cmd/benchstat@latest && benchstat old.txt new.txt
go tool pprof -http=:8080 cpu.out
go test -fuzz=FuzzDecode -fuzztime=5m
go build -gcflags="-m" ./... 2>&1 | grep escapes
go vet ./... && golangci-lint run
GODEBUG=gctrace=1 ./tttcli index ./corpus
hyperfine 'rg "pattern" ./corpus' './tttcli search "pattern"'   # honest A/B timing
```

---

## How I use all this

These files are a map, not a curriculum I have to finish before starting. The order that works:

**Read 01 → 02 → 03 → 04 → 05, then build M1 through M5.** Come back to 06–09 when the roadmap
reaches them; they read completely differently once I have written the code they describe. Keep 10
open in a tab the whole time.

Everything in these thirteen files is derivable from one idea: precompute a structure that answers the
question I will be asked. Every specific technique, delta encoding, skip lists, FSTs, WAND, trigram
queries, HNSW graphs, is a consequence of that idea plus a cost model. When I feel lost, I go back to
file 01 §1.5 and ask what is actually expensive here. The right answer is usually visible from there.
