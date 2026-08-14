# 13 — Glossary and Resources

---

## Glossary

**Analyzer** — the pipeline turning raw text into indexed terms: char filters → tokenizer → token
filters. Baked into the index at write time.

**ANN (Approximate Nearest Neighbour)** — vector search that trades exactness for speed. HNSW, IVF,
PQ, LSH.

**avgdl** — average document length in terms; the length-normalisation baseline in BM25.

**BKD tree** — block KD-tree; Lucene's disk-based structure for numeric and multi-dimensional ranges.

**Block-Max WAND (BMW)** — WAND plus per-block maximum scores, so whole postings blocks can be
skipped. State of the art for exact top-k.

**Bloom filter** — probabilistic set membership: "definitely not present" or "probably present".
Used in LSM read paths.

**BM25** — the standard lexical ranking function. TF saturation + IDF + length normalisation.

**BM25F** — BM25 for multi-field documents; combines field term frequencies *before* saturation.

**BSBI / SPIMI** — blocked sort-based indexing / single-pass in-memory indexing. Two ways to build an
index larger than RAM. SPIMI is what modern systems use.

**cf (collection frequency)** — total occurrences of a term across the whole corpus.

**Codec** — the pluggable component encoding/decoding postings on disk.

**Commit point** — the file naming the current set of live segments; the atomic unit of durability.

**Compaction / merge** — combining small immutable files into larger ones, discarding deleted data.

**Covering index** — an index containing every column a query needs, so the table is never read.
Search-engine equivalent: doc values.

**df (document frequency)** — how many documents contain a term.

**DocID** — dense internal integer identifier, 0..N-1, segment-local.

**Doc values** — columnar forward storage (docID → value) for sorting, faceting, and filtering.

**Elias-Fano** — quasi-succinct encoding of monotone sequences with O(1) random access.

**Forward index** — document → terms. The natural direction.

**FST (Finite State Transducer)** — automaton mapping strings to values, sharing prefixes *and*
suffixes. Lucene's term dictionary. Go: `vellum`.

**GIN / GiST / BRIN / SP-GiST** — Postgres index types: inverted, generalised search tree, block
range, space-partitioned.

**Heaps' law** — vocabulary grows as `k·T^β` (β ≈ 0.5): sublinearly with corpus size.

**HNSW** — hierarchical navigable small world; the default ANN graph index.

**IDF (inverse document frequency)** — `ln((N−df+0.5)/(df+0.5)+1)`. Rare terms weigh more.

**Impact-ordered index** — postings sorted by score instead of docID. Great for top-k, useless for
phrases.

**Inverted index** — term → sorted list of documents. The core structure.

**LSIF / SCIP** — precomputed code-intelligence index formats (definitions, references, hover).
SCIP is the modern successor.

**LSM tree** — log-structured merge tree. Memtable → immutable sorted files → compaction. Structurally
identical to Lucene's segment architecture.

**Leapfrog / galloping search** — intersection strategy that seeks forward rather than stepping.

**Levenshtein automaton** — a DFA accepting all strings within edit distance k; intersected with an
FST for fast fuzzy search.

**Live docs** — bitset marking non-deleted documents in a segment.

**nDCG** — normalised discounted cumulative gain; the standard graded ranking metric.

**Norm** — the stored per-document length factor used by BM25, often compressed to one byte.

**PForDelta** — bit-packing with exceptions for outlier values.

**Permuterm index** — index all rotations of `term$` so any single-wildcard query becomes a prefix
query.

**Posting / postings list** — one entry (docID, tf, positions) / the full list for a term.

**Positional index** — postings including token positions; required for phrase queries.

**RRF (Reciprocal Rank Fusion)** — combine ranked lists by `Σ 1/(k+rank)`, k=60. The default hybrid
fusion method.

**Roaring bitmap** — compressed bitmap using array/bitmap/run containers per 2^16 chunk.

**RUM conjecture** — you can optimise for Read, Update, or Memory — pick two.

**Segment** — an immutable, self-contained mini-index. Indexes are sets of segments.

**Selectivity** — the fraction of rows/documents a predicate keeps. Drives planner decisions.

**Skip list** — pointers embedded in postings enabling `Advance(target)` in O(log n).

**SPLADE** — learned sparse retrieval; neural weights stored in an ordinary inverted index.

**Stemming / lemmatisation** — rule-based suffix stripping / dictionary-based base-form lookup.

**Stopwords** — ubiquitous terms optionally removed. Mostly obsolete practice.

**tf (term frequency)** — occurrences of a term in one document.

**Tombstone** — a deletion marker; the data is removed only at merge/compaction.

**Trigram index** — inverted index over 3-byte substrings; enables substring and regex search.

**Two-phase iteration** — cheap approximation first, expensive verification only on survivors.

**Varint / VByte / LEB128** — variable-length integer encoding, 7 bits per byte.

**WAL (write-ahead log) / translog** — append-only durability log replayed after a crash.

**WAND (Weak AND)** — top-k algorithm using per-term max scores to skip documents that cannot make
the cut.

**Write amplification** — bytes physically written / bytes of user data.

**Zipf's law** — term frequency ∝ 1/rank. A few terms dominate; most occur once.

---

## Books

| Book | Why |
|------|-----|
| **Introduction to Information Retrieval** — Manning, Raghavan, Schütze | The textbook. **Free online at nlp.stanford.edu/IR-book/**. Chapters 1–7 cover files 01–05 of this course rigorously. Start here. |
| **Search Engines: Information Retrieval in Practice** — Croft, Metzler, Strohman | More systems-oriented, more practical. Free PDF available. |
| **Managing Gigabytes** — Witten, Moffat, Bell | Old (1999) and still the best book on index compression specifically. |
| **Designing Data-Intensive Applications** — Kleppmann | Chapter 3 is the clearest explanation of B-trees vs LSM ever written. Read it alongside file 07. |
| **Database Internals** — Petrov | Deeper on B-trees, LSM, and storage engine mechanics. |
| **The Go Programming Language** — Donovan & Kernighan | Still the best Go book. |
| **Learn Go with Tests** — free online | Test-first Go; matches this project's methodology exactly. |

---

## Papers and articles

**Must read (short, high value):**
- Russ Cox, *"Regular Expression Matching with a Trigram Index"* (swtch.com/~rsc/regexp/regexp4.html)
  — how Google Code Search worked. Read this before M10.
- Robertson & Zaragoza, *"The Probabilistic Relevance Framework: BM25 and Beyond"* — the definitive
  BM25 reference.
- Broder et al., *"Efficient Query Evaluation using a Two-Level Retrieval Process"* (2003) — WAND.
- Ding & Suel, *"Faster Top-k Document Retrieval Using Block-Max Indexes"* (2011).
- Michael McCandless, *"Using Finite State Transducers in Lucene"* (blog.mikemccandless.com) — the
  most approachable FST explanation.
- Schulz & Mihov, *"Fast String Correction with Levenshtein Automata"* (2002); plus Jules Jacobs'
  and Steve Hanov's blog explanations, which are far easier entry points.
- Lemire et al., *"Better bitmap performance with Roaring bitmaps"*.
- Zobel & Moffat, *"Inverted Files for Text Search Engines"* (ACM Computing Surveys, 2006) — the
  survey that ties everything together.

**Worth knowing:**
- Vigna, *"Quasi-Succinct Indices"* (Elias-Fano for postings).
- Lin & Trotman, *"Anytime Ranking for Impact-Ordered Indexes"*.
- Malkov & Yashunin, *"Efficient and robust approximate nearest neighbor search using HNSW"*.
- Formal et al., *"SPLADE"*; Nogueira et al., *"docTTTTTquery"*.
- Cormack et al., *"Reciprocal Rank Fusion"* (2009).
- Athanassoulis et al., *"Designing Access Methods: The RUM Conjecture"*.

---

## Codebases to read

| Project | Language | Read it for |
|---------|----------|-------------|
| **SQLite FTS5** | C | The most readable complete inverted index in existence. Its docs describe the format precisely. |
| **bleve / scorch** | Go | The closest existing analogue to your project. |
| **vellum** | Go | FST construction and traversal. |
| **RoaringBitmap/roaring** | Go | Compressed bitmaps done well. |
| **zoekt** | Go | Trigram code search in production. |
| **tantivy** | Rust | A modern, clean, fast Lucene-alike. Excellent code quality. |
| **Lucene** | Java | The reference implementation of nearly everything here. Start with `org.apache.lucene.codecs.lucene99`. |
| **ripgrep** | Rust | How to be fast without an index at all. |
| **Postgres `src/backend/access/gin/`** | C | GIN internals; see `README` in that directory. |
| **RocksDB** | C++ | LSM in anger. |
| **Quickwit** | Rust | Search over object storage; a genuinely different architecture. |

---

## Datasets for benchmarking

- **Your own notes / Markdown** — smallest useful corpus, and you know the right answers.
- **Go standard library source** (~2M lines) — perfect for M10 code search.
- **Enron email corpus** (~500 MB, 500k messages) — the classic IR test set.
- **Wikipedia dump extract** — a few GB of real prose; use `wikiextractor`.
- **MS MARCO** (passage ranking, 8.8M passages) — has relevance judgments, so you can compute real
  nDCG.
- **BEIR** — a benchmark suite of 18 retrieval datasets; the standard for comparing lexical vs dense.
- **CodeSearchNet** — code + natural language queries, for M10/M11.

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

## A closing note on how to use all this

The documents are a map, not a curriculum you must complete before starting. The order that actually
works:

**Read 01 → 02 → 03 → 04 → 05. Then build M1 through M5.** Come back to 06–09 when the roadmap
reaches them; they will read completely differently once you have written the code they describe.
Keep 10 open in a tab the whole time.

Everything in these thirteen files is derivable from one idea: *precompute a structure that answers
the question you will be asked*. Every specific technique — delta encoding, skip lists, FSTs, WAND,
trigram queries, HNSW graphs — is a consequence of that idea plus a cost model. If you ever feel
lost, go back to file 01 §1.5 and ask what is actually expensive here, and the right answer is
usually visible from there.
