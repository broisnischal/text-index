# ttt

I am building a text indexing and search library in Go from scratch, and learning Go properly while I
do it. `docs/` is my study material and build plan.

**Start at [`docs/00-START-HERE.md`](docs/00-START-HERE.md).**

```
docs/
  00-START-HERE.md                  the map, my reading order, how I work through it
  01-why-indexes-exist.md           first principles, cost model, where indexes get used
  02-text-analysis.md               bytes → tokens → terms
  03-inverted-index-internals.md    postings, dictionaries, FSTs, compression   ← the core one
  04-query-execution.md             intersection, phrases, fuzzy, top-k, WAND
  05-ranking-and-relevance.md       BM25, evaluation, learning-to-rank
  06-storage-segments-durability.md segments, merges, WAL, crash recovery
  07-database-indexing.md           B+trees, LSM, Postgres GIN, query planners
  08-code-indexing.md               trigrams, tree-sitter, SCIP, code search
  09-vector-and-hybrid-search.md    embeddings, HNSW, RRF, RAG
  10-go-for-index-builders.md       the Go this project actually needs
  11-library-design.md              API, packages, file format, testing
  12-roadmap.md                     M0 → M11 with acceptance criteria
  13-glossary-and-resources.md      vocabulary, papers, books, codebases
```

Files 01–05 plus milestones M1–M5 get me a working, durable, correct search library I understand line
by line. Everything after that is depth.

No code yet, on purpose. I want the ideas straight before I start writing bytes to disk.
