# 05 — Ranking and Relevance

Matching tells you *which* 40,000 documents qualify. Ranking decides which 10 the human sees. For
anything user-facing, ranking is where the product lives.

---

## 5.1 Why boolean matching is not enough

A boolean index answers a set-membership question. Users do not want sets; they want the *best*
answer at position one. The IR field's whole history is about turning "matches" into "ordered by
likely usefulness".

The core intuition, before any formula:

1. A document mentioning the query term **more often** is probably more about it. *(term frequency)*
2. But **not linearly** — the 20th occurrence tells you far less than the 2nd. *(saturation)*
3. A term appearing in **few documents** is more informative than one appearing everywhere.
   *(inverse document frequency)*
4. A **long** document mentioning the term 5 times is less focused than a short one mentioning it 5
   times. *(length normalisation)*

BM25 is exactly these four intuitions, written down. Nothing more.

---

## 5.2 TF-IDF and the vector space model

The historical foundation. Weight of term *t* in document *d*:

```
w(t,d) = tf(t,d) × idf(t)

tf  = raw count, or 1 + log(count)      ← the log is the saturation fix
idf = log(N / df(t))                    ← N = total docs
```

Documents and queries become vectors in |V|-dimensional space; similarity is the cosine:

```
                 Σ_t  w(t,q) · w(t,d)
cos(q,d)  =  ───────────────────────────
                  ‖q‖ · ‖d‖
```

The `‖d‖` divisor is the length normalisation, and it is the crude part — it over-penalises long
documents that are genuinely rich. TF-IDF is worth understanding because the vocabulary
(*term weight*, *document vector*, *cosine similarity*) is everywhere, including in the embedding
world (file 09), where "cosine similarity" means the same thing over a dense vector instead of a
sparse one.

**Do not ship TF-IDF as your default scorer in 2026.** BM25 beats it consistently and costs the same.

---

## 5.3 BM25 — the one to implement

```
                                    tf(t,d) · (k₁ + 1)
score(q,d) = Σ   idf(t) · ────────────────────────────────────────
            t∈q                                     |d|
                          tf(t,d) + k₁ · ( 1 − b + b · ───── )
                                                       avgdl
```

with

```
              N − df(t) + 0.5
idf(t) = ln( ───────────────── + 1 )        ← the +1 keeps it positive for very common terms
               df(t) + 0.5
```

| Symbol | Meaning | Default |
|--------|---------|---------|
| `tf(t,d)` | occurrences of t in d | — |
| `df(t)` | docs containing t | — |
| `N` | total docs | — |
| `|d|` | length of d in terms | — |
| `avgdl` | average doc length | — |
| `k₁` | tf saturation control | **1.2** (range 0.5–2.0) |
| `b` | length normalisation strength | **0.75** (0 = none, 1 = full) |

### Reading the formula properly

**Saturation.** Ignore length for a second: the tf factor is `tf(k₁+1)/(tf+k₁)`. As `tf → ∞` it
approaches `k₁+1` — a hard ceiling. With k₁ = 1.2:

| tf | factor |
|----|--------|
| 1 | 1.00 |
| 2 | 1.38 |
| 5 | 1.77 |
| 10 | 1.96 |
| 100 | 2.17 |
| ∞ | 2.20 |

Going from 1 to 2 occurrences buys you 38%; going from 10 to 100 buys 11%. **This is what defeats
keyword stuffing**, and it is the main thing BM25 has over TF-IDF. Lower k₁ → saturates faster
(good for short, homogeneous docs); higher k₁ → closer to linear tf.

**Length normalisation.** The `(1 − b + b·|d|/avgdl)` term inflates the denominator for
longer-than-average documents. `b = 0` disables it entirely (right for fields like `title`, where
length is not a proxy for dilution); `b = 1` fully normalises.

**IDF.** For a term in half the corpus, ≈ 0.69. For one in 0.1%, ≈ 6.9. A ten-fold weight
difference — which is what actually makes rare, discriminating terms drive ranking.

### Implementation notes that matter

- Precompute per-term `idf` once per query, not per document.
- `|d|` is stored per doc as a **norm** — Lucene famously compresses it to a *single byte* with a
  lossy encoding, because precision there barely affects ranking and the array must be tiny enough
  to stay in cache. Consider `float32` first, then optimise.
- Precompute the per-term `maxScore` (the score at that term's highest tf, shortest doc) at index
  time to enable WAND (file 04). This is why the codec should track max impact per block.
- Scores are **not** probabilities, are **not** comparable across queries, and are **not** in [0,1].
  Never show them to users; never threshold them with a magic constant like `score > 0.5`.

### BM25F — multiple fields

Wrong way: score each field separately and sum. That double-counts saturation and lets a term
matching in five fields dominate.

Right way: combine the *term frequencies* across fields with per-field weights **first**, then apply
saturation once:

```
tf̃(t,d) = Σ_f  boost_f · tf(t, d.f) / (1 − b_f + b_f · len_f/avgdl_f)
```

then feed `tf̃` through the BM25 saturation. This is what Elasticsearch's `combined_fields` query
does, and it is measurably better than `best_fields` for "title + body" setups.

---

## 5.4 Beyond the text score

Real ranking is a blend. Text relevance is one signal:

| Signal | Examples |
|--------|----------|
| **Query-dependent** | BM25, phrase-proximity bonus, exact-match bonus, field matched |
| **Query-independent (static)** | PageRank, popularity, star rating, "is verified", stock status |
| **Freshness** | exponential decay on age; critical for news/logs, harmful for reference docs |
| **Personalisation** | user history, locale, previously-clicked |
| **Business** | margin, promotion, inventory |

Two combination styles:

- **Multiplicative / functional**: `finalScore = bm25 × freshnessDecay(age) × log(1+popularity)`.
  Simple, explainable, tunable by hand. Start here.
- **Learned**: features into a model (§5.6).

Design implication for your library: the scorer must be an **interface**, and non-text signals come
from **doc values** (the columnar forward store), not from the postings. Bake in
`Scorer { Score(docID, tf, norm) float64 }` and a way to read a numeric column by docID.

---

## 5.5 Evaluating relevance — the part people skip

You cannot improve what you do not measure, and "it looks better to me" is not measurement.

**Set metrics** (no ranking):
- `precision = relevant retrieved / retrieved` — how much of what I showed was good
- `recall = relevant retrieved / relevant total` — how much of the good stuff I found
- `F1 = harmonic mean`

**Ranked metrics** (what you actually want):
- **P@k** — precision in the top k. Blunt but intuitive.
- **MRR** — mean of `1/rank of first relevant result`. Right metric when there is one correct
  answer (navigational queries, "go to definition").
- **MAP** — mean average precision; averages precision at each relevant hit.
- **nDCG@k** — discounted cumulative gain, normalised. Handles **graded** relevance (perfect / good /
  ok / bad), and discounts by `1/log₂(rank+1)`. **The standard for web-style search.**

```
DCG@k = Σ_{i=1..k}  (2^rel_i − 1) / log₂(i + 1)
nDCG@k = DCG@k / IDCG@k          (IDCG = DCG of the perfect ordering)
```

**Where judgments come from**: hand-labelled sets (expensive, high quality), click models (cheap,
biased toward whatever you already rank highly — *position bias* is severe), or public test
collections (MS MARCO, BEIR, TREC, and for code, CodeSearchNet).

**Online**: A/B tests with interleaving; guardrail metrics (zero-result rate, abandonment, time to
first click). Offline and online metrics disagree more often than you would like.

**For your library**: build a tiny evaluation harness early — a JSON file of
`{query, [relevant docIDs]}` and an nDCG@10 report. It converts ranking work from taste into
engineering, and it takes an hour to write.

---

## 5.6 Learning to rank, and the two-stage architecture

Modern relevance is almost always **two-stage**:

```
     millions of docs
           │
           ▼  STAGE 1: RECALL — cheap, high recall, top ~1000
           │   BM25 (+ ANN vector retrieval, file 09)
           ▼  STAGE 2: PRECISION — expensive, top ~10
           │   LTR model / cross-encoder over 1000 candidates
           ▼
       final ranking
```

Stage 2 can afford ~1000× more compute per document because it sees 1000 docs, not 10 million.

**LTR approaches**: pointwise (regression on relevance), pairwise (RankNet — learn "A above B"),
listwise (LambdaMART/LambdaRank — optimise nDCG directly). **LambdaMART (gradient-boosted trees) is
still the workhorse** and beats neural rankers on tabular feature sets surprisingly often.

Typical features: BM25 per field, exact-match flags, query/doc length, proximity, popularity,
freshness, click-through rate, embedding cosine.

**Neural stage 2**: a cross-encoder (query and document encoded *together* by a transformer) is far
more accurate than any bi-encoder similarity, and far too slow to run over the whole corpus —
exactly why it belongs in stage 2.

Your library's job is to be an excellent **stage 1** and to expose the hooks (feature extraction,
pluggable scorer, "return top 1000 with features") that let someone else build stage 2.

---

## 5.7 Practical relevance tuning checklist

When someone says "search is bad", work this list in order:

1. **Is it a matching problem or a ranking problem?** Is the desired doc in the result set at all?
   If not, it is analysis (file 02), not scoring. This distinction resolves most complaints.
2. **Zero-result queries** — top source of user pain. Fix with fuzzy fallback, OR-relaxation
   (`minimumShouldMatch`), synonyms, or spell correction.
3. **Field weighting** — title should outweigh body (3–10×). Cheapest big win available.
4. **Phrase/proximity boost** — documents where the query words appear *together* are usually better.
   Add a phrase clause as an optional `SHOULD` boost, not a `MUST`.
5. **Exact-match boost** — if you stem, also index the unstemmed form and boost exact hits.
6. **Tune `b` per field** — `b=0` for titles.
7. **Static quality signal** — popularity/rating, gently (`log`, not linear).
8. **Then, and only then**, consider embeddings and LTR.

---

## Check yourself

1. Compute the BM25 tf-factor (k₁=1.2, b=0.75) for tf=3 in a document of length 200 when
   avgdl = 100. Show every substitution.
2. Why does BM25 use `ln((N−df+0.5)/(df+0.5) + 1)` rather than plain `log(N/df)`? What goes wrong
   without the `+1` for a term in 90% of documents?
3. A term appears 50 times in a 10,000-word document and 3 times in a 100-word document. Which
   scores higher, and which `b` value would flip the answer?
4. Why is summing per-field BM25 scores worse than BM25F? Construct a document where it misranks.
5. Your nDCG@10 improves offline but click-through drops in the A/B test. Give three plausible
   explanations.
6. Why does a cross-encoder belong in stage 2 and never in stage 1?
