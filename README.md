# ttt — learning text indexing by building one in Go

Study material and build plan for writing a text indexing / search library from scratch, while
learning Go.

**Start at [`docs/00-START-HERE.md`](docs/00-START-HERE.md).**

```
docs/
  00-START-HERE.md                  map, learning path, how to use this
  01-why-indexes-exist.md           first principles, cost model, applications
  02-text-analysis.md               bytes → tokens → terms
  03-inverted-index-internals.md    postings, dictionaries, FSTs, compression   ← core
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

Reading 01–05 then building milestones M1–M5 gets you a working, durable, correct search library
you understand line by line. Everything after that is depth.

No code yet — by design.
