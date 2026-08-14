# 03 — Inverted Index Internals

This is the core file. Everything else in my notes orbits it, and I read it twice.

---

## 3.1 What is actually stored

An inverted index is two structures that reference each other:

```
 TERM DICTIONARY                      POSTINGS FILE (one contiguous byte blob)
 ┌───────────────────────────┐        ┌────────────────────────────────────────────┐
 │ term      df   offset  len│        │ ...                                        │
 ├───────────────────────────┤        │ [len][docID gaps...][tf...][positions...]  │◀─┐
 │ "postgres" 20k  0x4A10  812│───────▶│ ...                                        │  │
 │ "redis"    3k   0x4D3C  190│──────┐ │                                            │  │
 │ "run"      88k  0x5001 3100│      └▶│ [len][docID gaps...][tf...]                │  │
 └───────────────────────────┘        └────────────────────────────────────────────┘  │
        (kept hot, often in RAM)              (mmapped, mostly cold) ──────────────────┘
```

A single posting, fully populated:

```
docID , tf , [pos₀, pos₁, ... pos_{tf-1}] , [(startOff, endOff), ...]
```

Which parts I store is a **per-field configuration decision**, and one of the most important levers I
have:

| Index option                     | Stores        | Enables                                | Rough size     |
| -------------------------------- | ------------- | -------------------------------------- | -------------- |
| `DOCS`                           | docIDs only   | boolean match, filters                 | 1× (baseline)  |
| `DOCS_AND_FREQS`                 | + tf          | BM25 / TF-IDF scoring                  | ~1.3×          |
| `DOCS_AND_FREQS_AND_POSITIONS`   | + positions   | phrase, proximity, span queries        | ~3–4×          |
| `…AND_OFFSETS`                   | + byte offsets | fast highlighting without re-analysis | ~5–6×          |

A filter-only field (`status`, `tenant_id`) should be `DOCS`. Storing positions on every field and
then wondering why the index is 4× larger than necessary is a common mistake, so this is configurable
from day one. Retrofitting it is painful.

---

## 3.2 Why docIDs are sorted, dense integers

Two properties, both non-negotiable:

**Sorted** so I can intersect two lists with a linear merge (and skip, §3.6), and so I can
delta-encode.

**Dense (0..N-1)** so gaps between consecutive docIDs stay small, delta encoding wins big, and docID
works directly as an array offset into doc-value columns and live-docs bitsets.

An application key like `"user_9f2ba7c1"` is _not_ a docID. I keep a separate forward mapping
`docID → external key` in a column file. New documents get the next docID; deleted ones leave holes
until a merge compacts them (file 06).

### DocID assignment order is a real performance lever

If I assign docIDs so similar documents land adjacent, postings gaps shrink. Sorting web documents by
URL before assigning IDs is the classic result, and it can cut index size 10–30% because pages from
the same host share vocabulary and cluster into runs. Sorting by date is common for logs, with the
bonus that "last 24h" becomes a docID range. Free compression that costs only a sort during the
build.

---

## 3.3 Compression: delta encoding + variable-length integers

### Step 1: delta (gap) encoding

```
docIDs:  [3, 7, 12, 13, 40, 41, 42, 100]
gaps:    [3, 4,  5,  1, 27,  1,  1,  58]
```

Same information, since the list is sorted and a prefix-sum reconstructs it, but much smaller
numbers. Small numbers compress, large ones do not. For a term with df = 1M in a corpus of 10M docs,
the average gap is 10, which fits in 4 bits versus 24 bits for the raw ID.

Entropy intuition: a term appearing in a fraction `p` of docs needs about `log₂(1/p)` bits per posting
at best. Common terms are _cheaper per posting_ than rare ones. So my encoder has to be good at small
numbers.

### Step 2: variable-byte (varint / VByte / LEB128)

7 payload bits per byte, top bit is a continuation flag.

```
value 5      → 0000_0101                      (1 byte)
value 300    → 1010_1100  0000_0010           (2 bytes)  // 300 = 0b100101100
```

Go has it in `encoding/binary`: `binary.PutUvarint`, `binary.Uvarint`, `binary.AppendUvarint`. I
start here because it is simple, decent, and correct.

The downside is one unpredictable branch per byte, and on modern CPUs branch misprediction dominates.
Which leads to:

### Step 3: block-based bit-packing (what real engines do)

Process postings in **fixed blocks of 128** (Lucene's number):

1. Take 128 gaps.
2. Find the max, compute `b = bits needed`.
3. Write `b` (1 byte), then pack all 128 values at exactly `b` bits each, branch-free and
   SIMD-friendly.

128 gaps that all fit in 5 bits → 80 bytes + 1 header byte, versus ~128 bytes for varint. Decoding is
a tight unrolled loop with no branches, typically 3–5× faster than varint.

**PForDelta** ("Patched Frame Of Reference") refines it: choose `b` to cover say 90% of values and
store the 10% outliers separately as exceptions, so one large value no longer forces every value in
the block to be wide. Lucene's current codec is a variant of this.

Tail handling: the final partial block (< 128 postings) gets written as plain varints. Every real
format does this and I will not try to be clever there.

### Other encodings worth knowing by name

| Encoding          | Idea                                                                 | When                                                  |
| ----------------- | -------------------------------------------------------------------- | ----------------------------------------------------- |
| **Group Varint**  | 4 values share one control byte of 4×2-bit lengths                    | Faster varint, used by Google                         |
| **Simple-9 / 16** | Pack as many values as fit into one 32-bit word, 4-bit selector       | Older, elegant                                        |
| **Elias-Fano**    | Quasi-succinct; near-optimal space _and_ O(1) random access/`nextGEQ` | Modern research engines, Facebook's folly             |
| **Roaring**       | See below                                                            | Dense sets, filters, deletions                        |
| **Golomb-Rice**   | Optimal for geometrically-distributed gaps                            | Bloom-filter-adjacent uses                            |

### Roaring bitmaps, the other way to represent a docID set

Split the 32-bit docID space into 2^16 chunks. Each chunk holds up to 65536 IDs, stored as whichever
is smaller:

- an **array** of uint16 (sparse chunk, < 4096 values),
- a **bitmap** of 8 KB (dense chunk),
- a **run-length** list (long consecutive runs).

Why it matters: intersecting two dense chunks is `AND` over 1024 words, SIMD, no branches, absurdly
fast. That makes it the right representation for **filters** (`status=active`), **deleted-doc sets**,
and **facet counting** (popcount over an intersection). Go has a mature implementation:
`github.com/RoaringBitmap/roaring`.

My rule of thumb: compressed postings for scored full-text terms, Roaring for boolean filters,
deletions, and anything I intersect repeatedly.

---

## 3.4 The term dictionary

Requirements, in tension with each other:

1. Map term → postings offset (exact lookup).
2. Support **prefix** enumeration (`auto*`), so ordered or trie-like.
3. Support **range** enumeration (numerics-as-terms, wildcard rewriting).
4. Fit in RAM if at all possible, for millions of terms.

### Option A: sorted blocks + a sparse in-memory index

Write all terms sorted, in blocks of ~25–100, with **front coding** inside a block, sharing the
common prefix with the previous term:

```
block: "postgres", "postgresql", "postgresql-14", "postgrey"
front-coded:  0,"postgres"  8,"ql"  10,"-14"  5,"rey"
              ↑ shared prefix length
```

Keep only the **first term of each block** in an in-memory sorted array. Lookup is a binary search
over that array (RAM), then a scan of one block (one page read). Simple, compact, and essentially what
SSTables in LevelDB/RocksDB do. **I start here.** Maybe 150 lines of Go for 90% of the value.

### Option B: FST (Finite State Transducer), the Lucene approach

A minimal deterministic automaton where terms are paths and edges carry **output values** that sum
along the path to give the postings offset.

```
        m       o       n
   ①───────▶②───────▶③───────▶④
   │ +0      │ +0      │+17     │ "mon" → 17
   │         │         └──g───▶⑤  +5   → "mong" → 22
```

Properties that make it worth the complexity:

- Shares **prefixes and suffixes**, unlike a trie which shares only prefixes, so it comes out 5–20×
  smaller than the raw term list. Millions of terms in a few MB, RAM-resident.
- Traversal gives prefix enumeration for free.
- Can be **intersected with another automaton**, which is the trick that makes fuzzy search fast
  (file 04): build a Levenshtein automaton for the query term, intersect with the FST, and enumerate
  exactly the terms within edit distance _k_ without scanning the dictionary.
- Built with the Daciuk et al. incremental minimisation algorithm in one pass, requiring terms in
  sorted order, which I have anyway after inversion.

Go implementation to read: **`github.com/blevesearch/vellum`**, a port of Rust's `fst` crate by Andrew
Gallant, who also wrote ripgrep. Reading vellum is one of the highest-value things in this whole
project.

### Option C: hash map

`map[string]postingsRef`. Fine for the in-memory prototype in M1. Loses ordering, so no prefix or
range, and in Go a map with 5M string keys is brutal on the GC, because every key is a heap object
with a pointer the collector must scan. File 10 has the fix: one big `[]byte` arena plus offsets.

---

## 3.5 Segments: the index is a set of small indexes

Updating a compressed, sorted, mmapped file in place is essentially impossible. The universal
solution:

- Buffer new documents in RAM.
- When the buffer hits a threshold, **flush** it as a new **immutable segment** with its own term
  dict, postings and doc values.
- A search runs against **all** segments and merges the results.
- Background **merges** combine small segments into bigger ones.
- **Deletes** are tombstones, a per-segment bitset of live docs. The posting stays; it is filtered at
  read time. Space comes back only at merge.
- **Updates** are delete + insert. There is no in-place update, ever.

If that sounds exactly like an **LSM tree**, it is (file 07). Lucene and RocksDB are cousins.

What follows from it:

- **Immutability gives lock-free reads.** A searcher holds a reference to a set of segment files;
  writers never mutate them. Snapshot isolation for free, with refcounted file deletion.
- **Segment count is a latency/throughput dial.** More segments means cheaper writes and slower
  queries, since I pay per-segment overhead per term. Merge policy tuning is real operational work.
- **DocIDs are segment-local.** Global docID = segment base + local docID. Merging renumbers
  everything, so anything storing docIDs externally breaks. Classic bug.

---

## 3.6 Skip lists: making intersection sublinear

Intersecting `df=10,000,000` (`the`) with `df=50` (`kubernetes`) by linear merge costs 10M steps for
50 results. Absurd. I want to jump.

Store, every 128 postings (once per block), a **skip entry**:

```
skip level 0:  every 128 docs   → (docID at that point, byte offset, position-file offset)
skip level 1:  every 128²       → ...
skip level 2:  every 128³       → ...
```

A multi-level skip list embedded in the postings. `advance(target)` walks down the levels and decodes
only the final block, so cost becomes `O(log n)` block reads instead of `O(n)`.

The detail that will cost me time: the skip entry must record the offset into _every_ parallel stream
(docs, freqs, positions) so that after jumping I can decode all of them consistently. Getting those
offsets right is the fiddliest part of a real postings codec. I design the file format so skip data
is written _after_ the block it describes is finished, and buffer accordingly.

Modern refinement, **Block-Max**: also store the **maximum impact score** in each block. Then the
query executor can skip blocks that cannot contain a top-k result, not just blocks that lack the
docID. That is Block-Max WAND (file 04), and it is the single biggest win in modern top-k retrieval.

---

## 3.7 Putting a file format on paper

Before writing code, write the format down. Skeleton for my library, separating streams so cold data
stays cold:

```
segment_00042.meta      magic, version, docCount, fieldInfos, analyzerFingerprint, checksums
segment_00042.tdict     FST or sorted blocks: term → (df, cf, postingsOffset, skipOffset)
segment_00042.doc       per term: blocks of 128 delta-gapped docIDs + tf, bit-packed
segment_00042.pos       per term: position deltas, blocked
segment_00042.dv        columnar doc values: docID → numeric/sortable values
segment_00042.fdt       stored fields (compressed in chunks of ~16 KB, LZ4/zstd)
segment_00042.liv       live-docs bitset (rewritten on delete; Roaring)
```

Rules I hold myself to:

- **Magic number + version in every file.** Refuse to open an unknown version rather than
  misinterpreting bytes.
- **Checksum (CRC32C) at the end of every file**, verified on open. Silent corruption is worse than a
  crash.
- **Write forward-only.** Never seek backwards during a write; buffer whatever needs patching.
- **Store the analyzer fingerprint** so a mismatched reader shouts instead of returning nonsense.
- **All offsets absolute within the file.** Relative offsets feel clever and cause pain.

---

## 3.8 Size expectations, so I know if I got it wrong

For English prose, index size as a fraction of raw text:

| Configuration                          | Typical                                          |
| -------------------------------------- | ------------------------------------------------ |
| docs only, compressed                  | 5–10%                                            |
| docs + freqs                           | 10–15%                                           |
| + positions                            | 25–40%                                           |
| + offsets                              | 40–60%                                           |
| + stored fields (compressed originals) | add ~25–35% of raw                               |
| uncompressed postings (uint32 arrays)  | 50–100%, which is what skipping compression costs |

If my first working index comes out at 3× the corpus, I forgot delta encoding or I am storing
positions on fields that do not need them.

---

## 3.9 Memory mapping and the page cache

The usual read path is to mmap the postings file and let the OS page cache handle everything. The
kernel already implements the best cache I would write.

- Hot postings stay resident, cold ones page in on demand at ~80 µs.
- Multiple processes share one copy.
- In Go there is a real caveat. `syscall.Mmap` returns a `[]byte` backed by memory the GC does not
  manage, and any page fault stalls the _thread_, not just the goroutine. It works well (bleve does
  it) but a mmapped slice must never outlive the unmap, or I get a segfault no Go stack trace
  explains. File 10, the unsafe section.
- Simpler alternative for M4: `io.ReaderAt` plus an explicit block cache. Slower, safer, easier to
  reason about, trivially portable. My `Directory` abstraction hides the choice.

---

## 3.10 Worked micro-example

Corpus:

```
doc 0: "the cat sat on the mat"
doc 1: "the dog sat"
doc 2: "cat and dog"
```

After analysis (lowercase, no stopwords, no stemming):

```
term    df   postings (docID, tf, [positions])
─────────────────────────────────────────────────
and      1   (2,1,[1])
cat      2   (0,1,[1]) (2,1,[0])
dog      2   (1,1,[1]) (2,1,[2])
mat      1   (0,1,[5])
on       1   (0,1,[3])
sat      2   (0,1,[2]) (1,1,[2])
the      2   (0,2,[0,4]) (1,1,[0])
```

Query `cat AND dog` → intersect `[0,2]` and `[1,2]` → `[2]`.
Query `"sat on"` → docs with both = `[0]`; positions `sat`@2, `on`@3, adjacent → match.
Query `the` → tf=2 in doc 0, positions `[0,4]`. That tf is what BM25 consumes.

Exercise for myself: encode `cat`'s postings by hand as varint gaps and count the bytes, then do it
for a term with df=100,000. That is what makes §3.3 stop being abstract.

---

## Questions I should be able to answer

1. Why does delta encoding _require_ the postings list to be sorted, and what breaks if a merge
   produces an out-of-order list?
2. A term has df = 5,000,000 in a 10M-doc corpus. Estimate bytes per posting under (a) raw uint32,
   (b) varint gaps, (c) 128-block bit-packing. Show the arithmetic.
3. Why does an FST beat a trie on size? Give the specific structural reason.
4. I delete 40% of my documents and index size on disk does not change. Explain, and say what makes
   it change.
5. Where exactly must skip-list entries store offsets, and why is one offset not enough?
6. When would I choose Roaring bitmaps over delta-encoded postings for the _same_ field?
