# 09 — Vector Search and Hybrid Retrieval

The inverted index matches **tokens**. If the document says "automobile" and the user types "car",
it finds nothing. Vector search fixes exactly that, and breaks things the inverted index does well.
This file is about knowing which tool answers which question, and how to combine them.

---

## 9.1 The idea

An embedding model maps text to a dense vector in ℝⁿ (n typically 384–3072) such that semantically
similar texts land near each other. Retrieval becomes **nearest-neighbour search**:

```
score(q, d) = cos(v_q, v_d) = (v_q · v_d) / (‖v_q‖ · ‖v_d‖)
```

If you L2-normalise all vectors at index time (do this), cosine reduces to a plain **dot product**,
and ranking by dot product ≡ ranking by Euclidean distance. One less thing to get wrong, and it is
faster.

**Sparse vs dense — the same maths, different vectors:**

| | Lexical (BM25) | Dense (embeddings) |
|---|---|---|
| Vector | sparse, |V|-dimensional, one dim per term | dense, ~768-dimensional |
| Dimensions mean | a specific word | nothing interpretable |
| Match type | exact term overlap | semantic proximity |
| Handles synonyms | no (unless configured) | yes, natively |
| Handles rare identifiers, IDs, codes | **yes, perfectly** | **badly** |
| Explainable | yes ("matched 'postgres' 4×") | no |
| Needs a model | no | yes, at index *and* query time |
| Cost per doc | µs | ms + GPU/API |

That table is the whole argument for hybrid: the failure modes are **complementary**, not
overlapping.

---

## 9.2 Exact NN is O(N·d) — and sometimes that is fine

100,000 vectors × 768 dims × 4 bytes = 300 MB. A brute-force dot product over that is roughly
75M multiply-adds — **~10–30 ms** single-threaded, less with SIMD and multiple cores.

**Under ~100k vectors, do not build an ANN index.** Brute force is exact, trivially correct, has no
build step, supports arbitrary filters for free, and updates instantly. People reach for a vector
database at 5,000 documents and pay complexity for nothing.

Above ~1M vectors, you need approximate search.

---

## 9.3 ANN algorithms

### HNSW (Hierarchical Navigable Small World) — the default

A multi-layer proximity graph. Upper layers are sparse (long-range "highways"), lower layers dense.
Search greedily descends from an entry point, moving to whichever neighbour is closer to the query,
layer by layer.

| Parameter | Effect |
|-----------|--------|
| `M` (neighbours per node) | higher = better recall, more memory (~`M`×8 bytes/node), slower build |
| `efConstruction` | build-time candidate list; higher = better graph, slower build |
| `efSearch` | query-time candidate list; **the recall/latency dial at query time** |

- Recall 95–99% at ~1 ms for millions of vectors. Excellent.
- **Memory-hungry**: vectors *plus* graph must essentially be in RAM.
- **Deletes are awkward** — the standard approach is a tombstone plus periodic rebuild, because
  removing a node can disconnect the graph.
- Used by Lucene (so Elasticsearch/OpenSearch), Qdrant, Weaviate, pgvector, FAISS.

### IVF (Inverted File) — and here the name is not a coincidence

1. k-means cluster all vectors into `nlist` centroids.
2. Index: centroid → list of vectors in that cell. **That is an inverted index whose "terms" are
   centroids.**
3. Query: find the `nprobe` nearest centroids, search only those cells.

`nprobe` is the recall/speed dial. Cheap to build, cheap to update, lower recall than HNSW at the
same speed.

### PQ (Product Quantization) — compression for vectors

Split a 768-dim vector into 96 sub-vectors of 8 dims; k-means each sub-space into 256 centroids;
store one byte per sub-vector. **768 floats (3072 B) → 96 bytes, a 32× reduction.** Distances are
computed approximately from precomputed lookup tables.

Combined as **IVF-PQ**, this is how billion-scale search fits in RAM. Related:
**Scalar quantization** (float32 → int8, 4×, minimal recall loss — the easiest win, do it first);
**binary quantization** (1 bit/dim, 32×, needs a rescoring pass on full vectors).

### DiskANN / Vamana

Graph index designed so most of it lives on **SSD** with a small in-memory cache. The answer when
vectors do not fit in RAM and you refuse to shard.

### LSH

Locality-sensitive hashing. Historically important, theoretically clean, in practice beaten by graph
methods for most workloads. Still useful for MinHash/Jaccard-style dedup.

---

## 9.4 The hard part: filtered vector search

"Nearest neighbours **where** `tenant_id = 42 AND published = true`."

This is where vector databases get genuinely difficult, and it is worth knowing the three strategies
because every vendor picks one and their benchmarks hide it:

- **Pre-filter**: compute the allowed set first, then brute-force within it. Exact and fast when the
  filter is selective. Degrades to full scan when it is not.
- **Post-filter**: ANN-search for top-k, then drop non-matching. **Silently returns too few results**
  when the filter is selective — search 100, keep 3. The most common source of "my vector DB returns
  nothing" bugs.
- **In-graph filtering**: traverse the HNSW graph but only accept nodes passing the filter. Better,
  but a selective filter fragments the graph's connectivity and recall collapses.

Practical answer: **partition by the high-cardinality filter** (one index per tenant), and use
pre-filtering for the rest. If you build vector support into your library, expose the filter as a
Roaring bitmap computed by the *lexical* side — that composition is exactly the advantage a combined
library has over a bolt-on vector DB.

---

## 9.5 Hybrid retrieval — how to combine

### Reciprocal Rank Fusion (RRF) — use this first

```
RRFscore(d) = Σ_over_rankers  1 / (k + rank_r(d))        k = 60 by convention
```

Uses only **ranks**, never raw scores. This matters enormously: BM25 scores are unbounded and
query-dependent, cosine scores are in [-1,1] — they are not on a comparable scale and normalising
them is fragile. RRF sidesteps the problem entirely, needs no tuning, and is remarkably hard to beat.
It is the default in Elasticsearch and OpenSearch for a reason.

### Score normalisation + weighted sum

```
final = α · norm(bm25) + (1−α) · norm(cosine)
```

Needs min-max or z-score normalisation per query, and α tuning. Can beat RRF when carefully tuned on
your data, and is worse when it is not.

### Two-stage with a reranker (best quality)

```
BM25 top-100  ┐
              ├─▶ union/dedup ─▶ cross-encoder rerank ─▶ top-10
ANN   top-100 ┘
```

The cross-encoder (file 05, §5.6) reads query and document together. This is the strongest
configuration in current practice, at the cost of ~50–200 ms of model inference.

### Sparse-neural: the interesting middle ground

- **SPLADE** — a transformer produces a *sparse* vector over the vocabulary, with learned term
  expansion and weights. **You can store it in an ordinary inverted index**: the "terms" are
  vocabulary items and the "tf" is a learned weight. Semantic benefits, lexical infrastructure.
  Cost: postings lists get much longer (documents now "contain" terms they never mentioned).
- **doc2query / docTTTTTquery** — generate likely queries for each document, append them to the
  document text, index normally. Absurdly simple, works well, requires nothing new in the index.
- **ColBERT** — one vector per *token*, late interaction (MaxSim over token pairs). High quality,
  heavy storage.

**Note what SPLADE and doc2query imply**: the inverted index you are building is not obsoleted by
neural retrieval — it is the *substrate* several neural methods run on. Time spent making it good is
not time wasted.

---

## 9.6 RAG, briefly, because it is why most people arrive here

```
docs → chunk → embed → vector index
query → embed → retrieve top-k → stuff into an LLM prompt → answer
```

Where it actually goes wrong, in order of frequency:

1. **Chunking.** Fixed 512-token windows split tables, code, and arguments in half. Chunk on
   structure (headings, functions, paragraphs), overlap slightly, and keep the parent-document
   reference.
2. **Pure-vector retrieval.** Fails on exact identifiers, error codes, names, version numbers,
   acronyms. **Hybrid fixes most "RAG doesn't work" complaints.**
3. **No reranking.** Top-k by cosine is noisy; a reranker over the top-50 is the cheapest large gain.
4. **k too large.** Stuffing 20 mediocre chunks is worse than 5 good ones — the model gets distracted
   and the context cost is real.
5. **No evaluation.** Same discipline as file 05: a labelled question set and nDCG/recall@k, or you
   are guessing.

---

## 9.7 Where this fits your library

A defensible scope decision, stated in order of preference:

1. **Ship the lexical index well.** It is the harder, more durable, more educational artefact, and
   embedding quality is not something you control anyway.
2. **Add exact brute-force vector search** with a pluggable distance function and, crucially, the
   ability to pre-filter with a Roaring bitmap from the lexical side. ~300 lines, exact, genuinely
   useful up to ~100k vectors, and it makes your library *hybrid-capable*.
3. **Add RRF fusion** as a small, standalone utility. ~40 lines. It works with any two rankers,
   including ones the caller supplies.
4. **HNSW only if you want to.** It is a fun, self-contained algorithm and a fine M11, but it is a
   different project from a text index and does not teach you anything about inverted indexes.

Explicitly **do not** put an embedding model inside the library. Accept `[]float32` from the caller.
Models change every six months; your file format should not.

---

## Check yourself

1. Why does normalising vectors at index time let you replace cosine with a dot product, and why
   does that not change the ranking?
2. You have 40,000 documents and a "semantic search" requirement. Argue for brute force over HNSW,
   with numbers.
3. Explain why post-filtering an HNSW search returns too few results, and give a concrete
   tenant-filtering example.
4. Why does RRF use ranks instead of scores? What specifically goes wrong when you average a BM25
   score with a cosine score?
5. SPLADE produces sparse vectors. Why is that architecturally significant for someone who has just
   built an inverted index?
6. Give three query types where BM25 beats a dense retriever outright.
