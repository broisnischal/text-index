# 06 — Building, Storing, and Not Losing the Index

Everything so far assumed the index exists. These notes are about creating it, including when it is
larger than RAM, keeping it fresh under writes, and surviving a power cut.

---

## 6.1 Inversion: the sort at the heart of index construction

Index construction is fundamentally **a giant sort**. I produce a stream of tuples:

```
(term, docID, tf, [positions])
```

in document order, and I need them in **term order, then docID order**. That is it. Every construction
algorithm is a different way to do that sort within a memory budget.

### In-memory (M1)

`map[string]*postingsList`, append as I go, sort by term at the end. Works to maybe a few million
documents. Beyond that Go's GC starts scanning millions of live pointers every cycle and the build
crawls. File 10 has the fix: arena plus offsets.

### BSBI, Blocked Sort-Based Indexing

1. Read documents until a memory budget is hit.
2. Sort that block's `(termID, docID)` pairs.
3. Write the sorted block to a temp file.
4. Repeat.
5. k-way merge all blocks with a heap.

Requires a term → termID mapping that fits in memory. Classic external merge sort.

### SPIMI, Single-Pass In-Memory Indexing (what I implement)

1. Accumulate a dictionary → postings map in RAM, no global term IDs needed.
2. When memory fills, sort the terms and write a complete mini-index (a segment) to disk, then clear
   memory.
3. Repeat.
4. Merge segments later, or never, if I can search across them.

SPIMI is better because each flush produces something _immediately searchable_. That is exactly the
segment architecture from §3.5, and it is why Lucene, RocksDB and my library all converge on the same
shape.

### Parallel construction

Two axes, both trivially parallel with goroutines:

- **Document-partitioned** (by docID): each worker indexes a disjoint document range into its own
  segments, and query time fans out across segments. That is what I want, and it is what sharding is
  at cluster scale.
- **Term-partitioned** (by term): each worker owns a slice of the vocabulary. Better for some
  distributed query patterns, much worse for phrase queries and for adding documents. Rare in
  practice; I want to know it exists and then not use it.

Practical Go pipeline:

```
readers → [analyze workers: N goroutines] → [inverter: per-worker in-memory index]
        → [flusher: writes segments]        → [merger: background goroutine]
```

Analysis is CPU-bound and embarrassingly parallel, and it is usually 60–80% of build time. So it is
the first thing I parallelise, and I measure with `pprof` before optimising anything else.

---

## 6.2 The segment lifecycle

```
  AddDocument()
       │
       ▼
 ┌─────────────┐   memory limit or explicit Flush()
 │  in-memory  │ ─────────────────────────────────▶ segment_N  (immutable, on disk)
 │   buffer    │
 └─────────────┘
       │                                             segment_N ─┐
       │ Delete(docID) → tombstone bitset            segment_N+1├──▶ merge ──▶ segment_M
       ▼                                             segment_N+2┘        (deleted docs dropped)
   .liv file (live docs)
```

Three distinct operations that get confused routinely. I am adopting Elasticsearch's naming:

| Operation   | What it does                                                          | Durable? | Visible to search? |
| ----------- | --------------------------------------------------------------------- | -------- | ------------------ |
| **refresh** | writes the in-memory buffer to a new segment, possibly still in page cache | no       | **yes**            |
| **flush**   | fsyncs segments and truncates the write-ahead log                     | yes      | yes                |
| **commit**  | writes a new segments-list file naming the current segment set, fsynced | yes      | yes                |

That is why Elasticsearch is near-real-time: documents become searchable at _refresh_, default every
1 s, not at _commit_. Understanding that distinction explains most "I wrote it but cannot find it"
questions.

---

## 6.3 Merging

Merging N segments is a k-way merge of their sorted term dictionaries. For each term, concatenate
postings with docIDs renumbered into the new segment's space, skipping deleted docs.

Why merge at all:

- Query cost has a per-segment component, a term lookup per segment per query term. 500 segments means
  500 dictionary lookups for one term.
- Deleted documents only free space on merge.
- Compression improves, because longer postings lists compress better than many short ones.

Why not merge constantly: merging rewrites data. **Write amplification** is total bytes written over
bytes of new data, and aggressive merging can cost 10–30× and saturate the disk.

Merge policies:

- **Tiered** (Lucene default): merge segments of roughly similar size, keeping at most ~N per size
  tier. Balances count against write amplification.
- **Log/levelled** (LevelDB-style): strict size levels, each ~10× the last. Predictable, higher write
  amp, better read amp.
- **Force-merge to 1 segment**: excellent for read-only or archived indexes, disastrous if writes
  continue, because that one huge segment has to be rewritten entirely on the next merge.

Rules I take from other people's scar tissue:

- Merges run in **background goroutines** with concurrency limits. A merge must never block writes.
- A merge produces _new_ files and then atomically swaps the commit point. If it crashes halfway, the
  partial output gets garbage-collected on startup and nothing is lost.
- Old segments must not be deleted while a searcher is reading them, so **reference counting**, plus a
  deletion queue for files whose refcount is still > 0. On Windows an open file cannot be unlinked at
  all, which is why Lucene has a deletion-retry queue. Even on Linux I want the bookkeeping.

---

## 6.4 Deletes and updates

- **Delete**: set a bit in the segment's live-docs bitset, write a new `.liv` file (they are small),
  update the commit point. The postings are untouched.
- **Update**: delete + add. The new version lands in a _different_ segment with a _different_ docID.
  There is no in-place update.
- **Delete by query**: run the query, delete the hits. Not atomic with respect to concurrent writers
  unless I make it so.
- **Deleted docs still cost me**: they occupy postings space, they get decoded and then discarded
  during iteration, and they inflate `df` so IDF drifts, until merged away.

A high delete ratio, say above 30%, is the standard signal to trigger a merge. I track it per segment.

The versioning problem: if doc `X` is updated, an old copy may live in segment 3 while the new one
lives in segment 7. I have to guarantee the old one is marked deleted in the _same commit_ that adds
the new one, or a search returns two copies. Either maintain a primary-key → (segment, docID) map, or
resolve at query time by a version field. This is the hardest correctness detail in a mutable index,
and a large reason many systems only index append-only data.

---

## 6.5 Durability

**The write-ahead log (translog).** Segments are only durable at flush, which is expensive, so:

1. Append the document to a WAL, `fsync` (or batch fsyncs).
2. Add it to the in-memory buffer.
3. On crash, replay the WAL from the last commit point.
4. On flush, fsync the segments, write a new commit point, truncate the WAL.

Design points:

- **`fsync` per write** is safe and slow, ~0.1–1 ms on NVMe, ~10 ms on spinning disk. Group commit,
  batching N writes or 5 ms of them into one fsync, is the standard compromise. I make it a policy the
  caller chooses.
- **The commit point must be written atomically.** Standard trick: write `segments_N+1` to a temp
  file, fsync it, rename over the target (rename is atomic within a filesystem), then fsync the
  _directory_. Forgetting the directory fsync is a classic durability bug, because the file contents
  are on disk but the directory entry is not.
- **Checksum every file** with CRC32C, hardware-accelerated, and verify on open. Disks corrupt
  silently.
- **Never trust a partially written file.** On startup, any file not named by the commit point gets
  deleted.

Recovery test I actually run: a writer in a loop, `kill -9` at random points, reopen, assert every
acknowledged document is present and the index passes a consistency check. It automates, it finds real
bugs, and it is M5's acceptance criterion for a reason.

---

## 6.6 Concurrency model

Segment immutability hands me a very pleasant model:

- **Readers** hold a _snapshot_: a reference-counted list of segment readers plus their live-docs
  bitsets. No locks during search. A searcher opened at time T keeps seeing the world as of T while
  writes and merges continue. MVCC, achieved by never mutating anything.
- **One writer**, a single `IndexWriter`, serialises document additions and commit-point updates.
  Multi-writer is possible but the coordination cost usually is not worth it. Lucene enforces
  single-writer via a lock file and so do I.
- **Merges** are background workers, coordinating with the writer only at the swap.
- In Go: `sync/atomic` pointer swap for the current snapshot, `sync.Mutex` around commit-point
  updates, `errgroup` for merge workers, `context.Context` so a long merge can be cancelled at close,
  `sync.Pool` for decoding buffers in the hot query path.

Deadlock avoidance: define a strict lock order (writer lock → segment-set lock → file-deletion lock)
and put it in a comment at the top of the file. Two locks acquired in two orders is the bug I would
spend a weekend on.

---

## 6.7 The operational dials

The knobs a user of my library will actually want:

| Dial                                  | Effect                                                                                          |
| ------------------------------------- | ----------------------------------------------------------------------------------------------- |
| RAM buffer size before flush          | bigger means fewer larger segments, better compression, more to lose on crash, higher latency spike |
| refresh interval                      | smaller means fresher search, more small segments, more merge pressure                            |
| fsync policy                          | per-write / group-commit / periodic                                                               |
| merge policy + max concurrent merges  | throughput vs query latency                                                                       |
| max segment size                      | caps the cost of any single merge                                                                 |

I document these with their trade-offs. A library that exposes them with sane defaults (64 MB buffer,
1 s refresh, group commit at 5 ms, tiered merge, 5 GB max segment) is a good library.

---

## Questions I should be able to answer

1. Why is SPIMI preferred over BSBI for a real system? Two concrete advantages.
2. Explain refresh vs flush vs commit, and which one makes a document searchable.
3. A merge is halfway done when the process is killed. Walk through what happens on restart, file by
   file.
4. Why must I fsync the _directory_ after renaming the commit file?
5. My index has 40% deleted documents. What are the three separate costs I am paying?
6. Design the lock ordering for writer / merger / searcher and justify why it cannot deadlock.
