# 11 — Designing the Library

Design decisions, taken deliberately and written down *before* code exists. This document is the
thing you argue with while building; when the code disagrees with it, one of them is wrong and you
decide which.

---

## 11.1 Scope, stated as commitments

**In scope**
- Single-node, embeddable Go library (like SQLite/bleve, not like Elasticsearch).
- Analysis pipeline; inverted index with configurable per-field options.
- Boolean, phrase, prefix, range, fuzzy queries.
- BM25 scoring with top-k, and a pluggable scorer.
- Durable on-disk segments with merges, deletes, crash recovery.
- Concurrent search over an immutable snapshot; a single writer.
- Optional: trigram index for regex/code search; exact vector search + RRF fusion.

**Explicitly out of scope** (say no on purpose, so scope creep has to argue its way in)
- Distribution, sharding, replication, consensus.
- An HTTP server or query DSL over the wire.
- Embedding model inference.
- SQL, joins, transactions across documents.

**Non-functional targets** (numbers so you can fail visibly):
- Index ≥ 20 MB/s of English text per core, single-threaded analysis.
- Term lookup < 10 µs warm; simple 2-term AND top-10 over 1M docs < 5 ms.
- On-disk size ≤ 40% of raw text with positions.
- Zero allocations per document in the query hot loop.
- Crash at any point → reopen succeeds, no acknowledged document lost.

---

## 11.2 Package layout

```
ttt/                            # module: github.com/<you>/ttt
  index.go                      # Open, Writer, Reader — the public surface
  document.go                   # Document, Field, FieldOptions
  query.go                      # Query AST: Term, Bool, Phrase, Prefix, Fuzzy, Range
  search.go                     # Searcher, TopDocs, Collector
  options.go                    # functional options

  analysis/                     # tokenizers, filters, analyzers
    tokenizer/  filter/  lang/

  internal/
    codec/                      # postings encode/decode, block packing, skip lists
    dict/                       # term dictionary: sorted blocks now, FST later
    segment/                    # segment writer/reader, file format
    merge/                      # merge policy + executor
    docvalues/                  # columnar forward store
    store/                      # Directory abstraction: fs, memory, mmap
    bits/                       # bitsets, roaring wrapper

  cmd/tttcli/                   # index a directory, run a query, dump index stats
  testdata/                     # golden files, small corpora
```

Rationale:
- **`internal/` for everything that is not the user-facing API.** You can then change the codec, the
  dictionary, and the file format freely without breaking anyone. This one decision buys you years
  of freedom.
- **`cmd/tttcli`** early. A CLI that indexes a directory and runs a query is how you dogfood, demo,
  and benchmark. It also forces the API to be usable by someone who is not you.

---

## 11.3 The public API sketch

```go
// ---- Schema ----
type FieldOptions struct {
    Indexed    bool
    Stored     bool          // keep the original for result rendering
    IndexOpts  IndexOptions  // Docs | DocsFreqs | DocsFreqsPositions | ...Offsets
    Analyzer   string        // "" = default
    SearchAnalyzer string    // "" = same as Analyzer
    Boost      float64
    DocValues  bool          // columnar, for sort/facet/filter
}

type Schema struct{ Fields map[string]FieldOptions }

// ---- Writing ----
type Writer interface {
    Add(doc *Document) error
    Update(id string, doc *Document) error   // delete-by-id + add, atomically
    Delete(id string) error
    DeleteByQuery(q Query) (int, error)
    Flush() error                            // new segment, searchable
    Commit() error                           // durable
    Close() error
}

// ---- Reading ----
type Reader interface {
    Snapshot() *Snapshot                     // immutable view; Close() to release
    NumDocs() int
}

type Snapshot struct{ /* refcounted segment set */ }
func (s *Snapshot) Search(ctx context.Context, q Query, opts SearchOptions) (*Result, error)
func (s *Snapshot) Close() error

type SearchOptions struct {
    Size      int
    From      int
    Sort      []SortField
    Filter    Query          // unscored, cacheable
    Explain   bool
    Highlight []string
}

type Result struct {
    Total     int64
    MaxScore  float64
    Hits      []Hit          // DocID, ID, Score, Fields, Highlights, Explanation
    Took      time.Duration
}

// ---- Queries (a struct AST, not a string language) ----
func Term(field, term string) Query
func Phrase(field string, terms []string, slop int) Query
func Prefix(field, prefix string) Query
func Fuzzy(field, term string, maxEdits int) Query
func Range(field string, lo, hi any, incLo, incHi bool) Query
func Bool() *BoolQuery              // .Must() .Should() .MustNot() .Filter() .MinShouldMatch(n)
func Boost(q Query, b float64) Query
```

Design notes worth defending:

- **Snapshot is explicit and must be closed.** It is the refcount that keeps segment files alive
  during a merge (file 06). Making it explicit in the API is honest; hiding it produces mysterious
  file-deletion bugs. Document it loudly.
- **`context.Context` on `Search`, not on `Add`.** Searches get cancelled; writes should not be
  half-applied.
- **`Filter` separate from `Must`** — the scoring/caching distinction from file 04, surfaced in the
  API so callers can express it.
- **Struct AST, not a string parser.** A parser can be built on top later, in a separate package,
  and never becomes a security or compatibility problem in the core.
- **`Explain`** — a per-hit score breakdown. This is not a nicety: it is the debugging tool for every
  "why is this ranked here" question, and it is much harder to add later.

---

## 11.4 The internal interfaces that matter

```go
// The heart of query execution (file 04). Everything is one of these.
type PostingsIterator interface {
    DocID() uint32                    // NoMoreDocs when exhausted
    Next() uint32
    Advance(target uint32) uint32     // first doc >= target
    Freq() uint32
    Positions() []uint32              // only if positions indexed
    Cost() int64                      // ≈ df, for planning
}

// Pluggable postings encoding — swap varint for bit-packing without touching anything else.
type Codec interface {
    WritePostings(w io.Writer, p PostingsList) (offset, skipOffset uint64, err error)
    ReadPostings(r io.ReaderAt, offset, skipOffset uint64) (PostingsIterator, error)
    Name() string
    Version() int
}

// Term dictionary — sorted-blocks first, FST later, same interface.
type Dictionary interface {
    Lookup(term []byte) (TermInfo, bool)
    Iterator(lo, hi []byte) TermIterator     // prefix and range scans
    Automaton(a Automaton) TermIterator      // fuzzy/wildcard (file 04)
}

// Where bytes live: filesystem, memory (tests), mmap.
type Directory interface {
    Create(name string) (WriteCloserSyncer, error)
    Open(name string) (io.ReaderAt, int64, error)
    Delete(name string) error
    List() ([]string, error)
    Sync(names ...string) error
}

type Scorer interface {
    Score(docID uint32, freq uint32, norm float32) float64
    MaxScore() float64                        // for WAND (file 04)
}
```

The `Directory` abstraction pays for itself immediately: an in-memory implementation makes every
test fast and hermetic, and it forces you not to sprinkle `os.Open` through the codebase.

`Codec.Version()` plus `Name()` written into the segment header means a future format change is a
compatibility decision, not a data-loss event.

---

## 11.5 The on-disk format (write this down before coding)

```
ttt_index/
  segments_00000007            ← commit point: list of live segments + generation + checksum
  translog-00000003.log        ← WAL since last commit
  seg_0000001a.meta            ← header, docCount, field infos, analyzer fingerprint, codec info
  seg_0000001a.tdict           ← term dictionary
  seg_0000001a.doc             ← docID + freq postings blocks (+ skip data)
  seg_0000001a.pos             ← positions
  seg_0000001a.dv              ← doc values columns
  seg_0000001a.fdt / .fdx      ← stored fields (chunk-compressed) + index into them
  seg_0000001a.liv             ← live-docs bitset (rewritten on delete)
```

Every file:

```
┌────────────────────────────────────────────┐
│ magic "TTT1" (4B) │ formatVersion (u32)    │
│ codecName (varstring) │ codecVersion (u32) │
├────────────────────────────────────────────┤
│ ... payload ...                            │
├────────────────────────────────────────────┤
│ footer: payloadLength (u64) │ CRC32C (u32) │
└────────────────────────────────────────────┘
```

Invariants to state now and enforce in code:

1. Segment files are **write-once**; only `.liv` and the commit point are ever replaced (and only by
   atomic rename).
2. **Nothing is visible until the commit point names it.** Recovery = read the highest valid
   `segments_N`, delete every file it does not reference.
3. **Every file is checksummed** and verified on open (a header/footer check always; a full-file
   verification on an explicit `CheckIndex`).
4. **All integers little-endian**, all offsets absolute within their file, all lengths explicit.
5. The `.meta` records the analyzer fingerprint; a Reader whose analyzer disagrees returns a typed
   error rather than silently wrong results.

---

## 11.6 Concurrency contract (put this in the package doc)

- One `Writer` per index directory, enforced by a lock file containing the PID.
- Any number of concurrent `Snapshot`s; a `Snapshot` sees a fixed set of segments and is unaffected
  by writes or merges.
- Search is safe from multiple goroutines; a `Snapshot` is safe to share.
- Segment files are deleted only when refcount hits zero; a deletion queue retries.
- Lock order (never violate): `writerLock → segmentSetLock → fileDeletionLock`.

---

## 11.7 Error handling policy

```go
var (
    ErrCorrupt      = errors.New("ttt: corrupt index")
    ErrVersion      = errors.New("ttt: unsupported index version")
    ErrLocked       = errors.New("ttt: index locked by another writer")
    ErrClosed       = errors.New("ttt: closed")
    ErrAnalyzerDrift= errors.New("ttt: analyzer does not match segment")
)
```

- Wrap with context: `fmt.Errorf("read postings for %q in seg %d: %w", term, id, ErrCorrupt)`.
- **Never panic on bad input data.** A corrupt file is an `error`. Panics are reserved for programmer
  errors (calling a method on a closed index). Enforce this with fuzz tests (file 10).
- A failed merge is logged and retried; it must never fail a write or a search.

---

## 11.8 Testing strategy

| Layer | Method |
|-------|--------|
| Codec | round-trip properties + fuzz decode + boundary sizes (0,1,127,128,129) |
| Dictionary | differential test vs `map[string]` + sorted slice |
| Query execution | **differential vs the M1 brute-force index** on random corpora/queries |
| Scoring | fixed corpus with hand-computed BM25 values in a golden file |
| Durability | kill-and-recover fuzz: random `kill -9`, then assert all acked docs present |
| Concurrency | `-race` on a mixed read/write/merge workload |
| Format compat | golden index directories committed to `testdata/`, opened by every later version |
| Relevance | nDCG@10 harness over a small labelled set; report it in CI |

The differential test against the brute-force implementation is the highest-value thing in this
table. Keep the naive implementation forever; it is not scaffolding, it is the oracle.

---

## 11.9 Prior art to study (and steal from deliberately)

| Project | Take from it |
|---------|--------------|
| **Lucene** (Java) | codec architecture, `DocIdSetIterator`, two-phase iteration, segment lifecycle |
| **Tantivy** (Rust) | modern, clean version of the same; excellent code |
| **bleve / scorch** (Go) | the Go-specific version of everything you are doing; segment snapshots |
| **SQLite FTS5** (C) | compact, complete, superbly documented format |
| **Zoekt** (Go) | trigram code search |
| **vellum** (Go) | FST |
| **Postgres GIN** (C) | posting lists vs posting trees, pending-list buffering |
| **MeiliSearch / Quickwit** (Rust) | typo tolerance UX; object-storage-backed search |

Read prior art *after* you have made your own attempt at a component. Reading first gives you their
answers without their questions, and the questions are the education.

---

## 11.10 Naming and versioning

- Module `github.com/<you>/ttt` (or a better name — `tindex`, `invert`, `grepdb`).
- **v0.x while the format churns**, and say clearly in the README that the on-disk format is unstable
  before v1. After v1, format changes require a version bump and a read path for the old format.
- Keep a `FORMAT.md` next to the code with the byte layout, updated in the same commit as any change
  to it. If they drift, the format is undocumented.
