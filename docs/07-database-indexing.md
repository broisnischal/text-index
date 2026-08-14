# 07 — How Database Indexing Works (and how it relates to yours)

You asked how database search uses indexing. Short answer: **a database index is the same bargain as
a search index, tuned for a different query shape.** Postgres' GIN is literally an inverted index.
This file makes the connection precise, because understanding both together is what turns you from
"someone who uses `CREATE INDEX`" into "someone who knows what it costs".

---

## 7.1 The B+tree — the default index of every relational database

### Structure

```
                       ┌──────────────┐
   internal (routing)  │ 42 │ 91 │ 137│          keys only, no row data
                       └──┬───┴──┬───┴──┬─┘
              ┌───────────┘      │      └───────────┐
        ┌─────▼─────┐      ┌─────▼─────┐      ┌─────▼─────┐
 leaves │ 12│19│33  │◀────▶│ 45│60│88  │◀────▶│ 99│120│135│   keys + row pointers
        └───────────┘      └───────────┘      └───────────┘
                    ← doubly linked leaf chain →
```

- Every node is exactly one **page** (8 KB in Postgres, 16 KB in InnoDB). This is the memory-hierarchy
  rule from file 01 made structural.
- **Only leaves hold data pointers**; internal nodes are pure routing. (That is the "+" in B+tree.)
- **Leaves are linked**, so a range scan is sequential after one descent.
- Kept balanced by splitting full nodes on insert and merging sparse ones on delete.

### Why it is so shallow — do this arithmetic once

Page 8 KB, key ~16 bytes + 8 byte pointer = 24 bytes → **~340 entries per node**.

| Height | Max rows |
|--------|----------|
| 1 | 340 |
| 2 | 115,600 |
| 3 | 39,000,000 |
| 4 | 13,000,000,000 |

**A 4-level B+tree indexes 13 billion rows.** And the root and level-1 are always in cache, so a
point lookup costs ~1–2 actual disk reads. *This* is why B+trees won: the fanout makes tree height
almost constant across every practical data size.

### Clustered vs secondary

- **Clustered index** (InnoDB primary key; SQL Server clustered): the table *is* the leaf level of
  the tree. Rows are physically ordered by the key. Primary-key lookups need no extra hop; range
  scans on the PK are sequential.
  - Consequence: in InnoDB, every **secondary** index stores the *primary key* as its pointer, so a
    secondary lookup is two tree descents. And a wide primary key inflates every secondary index.
- **Heap + secondary indexes** (Postgres): the table is an unordered heap; every index — including
  the primary key — points at a physical tuple id (`ctid`). All indexes are equal citizens.
  - Consequence: an index scan is a random heap access per row. Postgres has the **visibility map**
    and *index-only scans* to avoid it when possible, and `CLUSTER`/`pg_repack` to physically reorder.

### Covering indexes and index-only scans

If the index contains every column the query needs, the database never touches the table:

```sql
CREATE INDEX ON orders (customer_id, created_at) INCLUDE (total);
SELECT total FROM orders WHERE customer_id = 7 ORDER BY created_at DESC LIMIT 10;
```

All three columns are in the index → one descent, ten sequential leaf entries, done. This is the same
idea as **doc values** in a search index: keep a redundant copy of the fields you need for
filtering/sorting so the hot path never dereferences the full document.

### Composite indexes and the leftmost-prefix rule

An index on `(a, b, c)` is sorted by `a`, then `b`, then `c`. It serves:

- `WHERE a = ?` ✅
- `WHERE a = ? AND b = ?` ✅
- `WHERE a = ? AND b = ? AND c = ?` ✅
- `WHERE a = ? ORDER BY b` ✅ (sort comes free)
- `WHERE b = ?` ❌ — cannot skip the leading column
- `WHERE a > ? AND b = ?` ⚠️ — after a *range* on `a`, `b` is no longer usefully ordered

**Rule: equality columns first, then the range column, then the sort column.** This single rule
resolves a large fraction of real-world "why isn't my index used" incidents.

---

## 7.2 Hash indexes

Bucket = `hash(key)`. O(1) equality, and **nothing else** — no ranges, no ordering, no prefix, no
sorting. Postgres has them (WAL-logged and crash-safe since v10) and they are rarely worth it: a
B+tree gives you equality *and* everything else at nearly the same cost. Hash indexes matter more
inside engines (hash joins, in-memory maps) than as user-created indexes.

---

## 7.3 LSM trees — the write-optimised alternative

Used by RocksDB, LevelDB, Cassandra, ScyllaDB, HBase, and (via RocksDB) MyRocks, TiKV, CockroachDB.

```
writes ──▶ [ memtable ]  (sorted in-memory, e.g. skiplist)
                │ full
                ▼
           [ SSTable L0 ]  immutable, sorted, flushed
                │ compaction
                ▼
           [ SSTable L1 ]  ~10× larger
                ▼
           [ SSTable L2 ]  ~100× larger …
```

- **Writes** are sequential appends → very fast, no random I/O.
- **Reads** may consult many levels → each SSTable has a **Bloom filter** ("key definitely not here")
  so a read touches only the levels that might contain it.
- **Compaction** merges SSTables, discards overwritten/deleted keys.

**The three amplifications** — the vocabulary for reasoning about *any* storage engine:

| | Meaning | B+tree | LSM |
|---|---|---|---|
| **Write amplification** | bytes written to disk / bytes of user data | moderate (page writes, WAL, full-page images) | high (recompacted repeatedly) |
| **Read amplification** | pages read / pages of useful data | low (~tree height) | higher (multiple levels) |
| **Space amplification** | disk used / logical data | ~1.3× (page fragmentation) | varies; stale versions until compaction |

You cannot optimise all three — this is the **RUM conjecture** (Read, Update, Memory: pick two).
Every storage engine on earth is a point in that trade-off space.

**Now look back at file 06.** Memtable = in-memory buffer. SSTable = segment. Compaction = merge.
Tombstone = deleted-docs bitset. **Lucene's segment architecture and an LSM tree are the same
design**, arrived at independently because both faced "immutable sorted files + updates". Once you
see this, both stop being separate topics you have to memorise.

---

## 7.4 The other index types, and what each is for

| Type | Structure | Best for | Weakness |
|------|-----------|----------|----------|
| **B+tree** | balanced tree | equality, range, sort, prefix | wide/low-cardinality columns |
| **Hash** | buckets | equality only | no ranges |
| **Bitmap** | one bitmap per distinct value | low-cardinality columns, multi-predicate AND/OR in analytics | expensive under updates; Postgres builds them in-memory only |
| **GIN** | **inverted index** | multi-valued: arrays, `jsonb`, full-text `tsvector`, trigrams | slower updates (mitigated by a pending list) |
| **GiST** | generalised search tree | geometry, ranges, kNN, custom types | lossy, needs re-check |
| **SP-GiST** | space-partitioned (quadtree, radix) | non-balanced structures, IP prefixes, text radix | narrower use |
| **BRIN** | min/max per block range | huge, naturally-ordered tables (append-only logs by time) | useless if data is unordered |
| **Zone maps / skip indexes** | min/max/bloom per granule | columnar OLAP (ClickHouse) | approximate pruning only |
| **Inverted (Lucene)** | term → postings | full-text, relevance ranking | no ordering by value |
| **Vector (HNSW/IVF)** | graph / clusters | nearest neighbour | approximate; file 09 |

**BRIN is worth a second look** because it is so cheap: for a 1 TB append-only table ordered by
timestamp, a BRIN index is a few MB and prunes 99% of blocks for a time-range query. It is the
minimum-viable index — literally just min/max per 128 pages — and it is a great illustration that
"index" does not have to mean "tree".

---

## 7.5 Postgres full-text search, in detail

The one you are most likely to meet in practice, and the closest thing to what you are building.

```sql
-- analysis (file 02!) happens here: parser → dictionaries → stemming → lexemes
SELECT to_tsvector('english', 'The quick brown foxes are running');
-- → 'brown':3 'fox':4 'quick':2 'run':6

CREATE INDEX docs_fts ON docs USING GIN (to_tsvector('english', body));

SELECT id, ts_rank(to_tsvector('english', body), q) AS rank
FROM docs, to_tsquery('english', 'quick & fox') q
WHERE to_tsvector('english', body) @@ q
ORDER BY rank DESC LIMIT 10;
```

Map every piece onto what you now know:

| Postgres | Your library |
|----------|--------------|
| `tsvector` | analysed term list with positions |
| text search *configuration* | analyzer |
| *dictionaries* (simple, snowball, synonym, thesaurus, unaccent) | token filters |
| `tsquery` | parsed query tree |
| `@@` | boolean match |
| **GIN index** | **term dictionary + postings lists** |
| `ts_rank` / `ts_rank_cd` | scorer (a weak TF-IDF-ish; **not** BM25) |

**GIN internals** (this is the payoff for reading file 03): GIN is a B-tree over *keys* (lexemes),
where each key's value is either a **posting list** (small: stored inline) or a **posting tree**
(large: its own B-tree of TIDs). Delta-compressed varbyte TIDs. That is precisely §3.1 and §3.3 with
different names.

**GIN's `fastupdate`** — because inserting one row touches every lexeme's posting list (expensive),
GIN buffers new entries in an unsorted **pending list** and merges them in bulk at vacuum or when
`gin_pending_list_limit` is exceeded. Cost: queries must scan the pending list linearly, so a big
pending list makes queries mysteriously slow. That is exactly the "in-memory buffer + flush" design
from file 06, and the same operational trade-off.

**Where Postgres FTS stops being enough** — worth knowing before you argue for a dedicated engine:
`ts_rank` is not BM25 and has no length normalisation worth the name; there is no built-in fuzzy or
did-you-mean; ranking requires reading the `tsvector` from the heap (no positions in the GIN index),
so `ORDER BY rank` on a large result set is slow; no faceting; no built-in highlighting at scale
(`ts_headline` re-parses documents). It is excellent up to a few million documents with simple
relevance needs — genuinely, use it before reaching for a cluster.

**`pg_trgm`** — trigram index (`GIN`/`GiST`) that makes `LIKE '%foo%'` and `similarity()` indexable.
Same trigram idea as code search (file 08).

### Other databases' full text, briefly

- **MySQL/InnoDB `FULLTEXT`** — inverted index in internal tables, BM25-ish ranking since 5.7,
  natural-language and boolean modes.
- **SQLite FTS5** — a *virtual table* storing the inverted index in ordinary SQLite b-tree tables
  (`%_data` holds the postings as blobs). Supports BM25 out of the box, prefix indexes, and
  `contentless` mode. **The single best codebase to read if you want a compact, complete, real
  implementation** — a few thousand lines of very readable C, and its documentation explains the
  format.
- **MongoDB** — text indexes (inverted, but limited) plus Atlas Search, which is Lucene underneath.
- **ClickHouse** — skip indexes (min/max, bloom, token bloom, ngram bloom) rather than true inverted
  indexes; brute-force scan is fast enough with columnar storage plus pruning. A different point in
  the design space, and a good reminder that "scan fast" is a valid strategy.

---

## 7.6 The query planner: why your index is being ignored

The planner picks a plan by **estimated cost**, using **statistics** (`ANALYZE`): row counts,
histograms, most-common-values lists, per-column distinctness, correlation.

Key concept — **selectivity**: what fraction of rows a predicate keeps. If `status = 'active'`
matches 80% of the table, an index scan means 80% of the table read in random order, which is
**slower than a sequential scan**. The planner is right to refuse the index. Random I/O is ~4–100×
more expensive per page, so the crossover is typically somewhere around 5–20% selectivity.

Common reasons an index is not used, roughly in order of frequency:

1. **Low selectivity** — the index would not help.
2. **Function applied to the column** — `WHERE lower(email) = 'x'` cannot use an index on `email`;
   it needs an *expression index* on `lower(email)`.
3. **Type mismatch / implicit cast** — `WHERE int_col = '5'::text`.
4. **Leading column not constrained** — leftmost-prefix rule (§7.1).
5. **Stale statistics** — planner thinks the table has 100 rows; it has 10 million.
6. **`OR` across different columns** — often needs two indexes and a BitmapOr, or a rewrite to
   `UNION`.
7. **`LIKE '%x'`** — wrong index shape entirely (§1.6). Needs trigram.
8. **Small table** — a sequential scan of 3 pages beats any index.

Tooling: `EXPLAIN (ANALYZE, BUFFERS)`. Read it as a tree, bottom-up. Compare **estimated vs actual
rows** — a large discrepancy is the root cause of most bad plans, and it points at statistics or at
correlated predicates the planner assumes are independent (`WHERE city='Paris' AND country='France'`
— the planner multiplies the selectivities and gets a wildly wrong estimate; extended statistics
exist for this).

---

## 7.7 The costs, stated plainly

Every index you add:

- **Slows writes.** Each `INSERT` updates every index. Ten indexes = eleven trees touched per insert.
- **Consumes space**, often 10–30% of the table each; heavily-indexed tables are more index than data.
- **Must be maintained** — bloat, vacuum, rebuilds, fragmentation.
- **Can be redundant** — an index on `(a)` is subsumed by `(a, b)`. Drop the narrower one.
- **Can be unused** — check `pg_stat_user_indexes.idx_scan`. Unused indexes are pure cost. Auditing
  and dropping them is one of the highest-value, lowest-risk database tasks there is.

---

## 7.8 The unifying table

| Concept | Search index (yours) | Relational DB |
|---------|----------------------|---------------|
| Key | term | column value |
| Value | postings list of docIDs | row pointer(s) |
| Ordered structure | term dictionary (FST/sorted blocks) | B+tree |
| Write buffering | in-memory buffer → segment | memtable → SSTable, or GIN pending list |
| Immutable files | segments | SSTables |
| Background compaction | segment merge | LSM compaction / VACUUM |
| Deletion | live-docs bitset | tombstone / dead tuple |
| Durability | translog + commit point | WAL + checkpoint |
| Statistics for planning | df / term cost | histograms / n_distinct |
| Cost-based plan | clause reordering by df | query planner |
| Redundant forward copy | doc values / stored fields | covering index / INCLUDE |
| Probabilistic filter | (rarely used) | Bloom filter in LSM levels |

Learning both at once is not extra work — it is the *same* work seen twice, and the second view is
what makes the first one stick.

---

## Check yourself

1. Compute the height of a B+tree over 500M rows with 8 KB pages and 40-byte entries. How many disk
   reads for a point lookup with a warm cache?
2. Why does an index on `(a, b)` not help `WHERE b = 5`? Draw the leaf ordering to prove it.
3. You have `WHERE status = 'active' AND created_at > now() - '7 days'` where 90% of rows are active.
   What is the right index, and in what column order?
4. Explain GIN's `fastupdate` in terms of file 06's flush/refresh model. What is the failure mode?
5. Why is `LIKE '%son'` unindexable by a B+tree, and what two different index types can serve it?
6. Give a query where a hash index beats a B+tree, and explain why databases still rarely use them.
7. State the RUM conjecture and place B+tree, LSM, and your segment-based text index in that space.
