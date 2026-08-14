# 05 — Ranking and Relevance

Matching tells me _which_ 40,000 documents qualify. Ranking decides which 10 a human sees. For
anything user-facing, ranking is where the product lives.

---

## 5.1 Why boolean matching is not enough

A boolean index answers a set-membership question. People do not want sets, they want the best answer
at position one. The IR field's whole history is about turning "matches" into "ordered by likely
usefulness".

The core intuition, before any formula:

1. A document mentioning the query term **more often** is probably more about it. _(term frequency)_
2. But **not linearly**. The 20th occurrence tells me far less than the 2nd. _(saturation)_
3. A term appearing in **few documents** is more informative than one appearing everywhere.
   _(inverse document frequency)_
4. A **long** document mentioning the term 5 times is less focused than a short one mentioning it 5
   times. _(length normalisation)_

BM25 is exactly those four intuitions written down. Nothing more.

---

## 5.2 TF-IDF and the vector space model

The historical foundation. Weight of term _t_ in document _d_:

```
w(t,d) = tf(t,d) × idf(t)

tf  = raw count, or 1 + log(count)      ← the log is the saturation fix
idf = log(N / df(t))                    ← N = total docs
```

Documents and queries become vectors in |V|-dimensional space, and similarity is the cosine:

```
                 Σ_t  w(t,q) · w(t,d)
cos(q,d)  =  ───────────────────────────
                  ‖q‖ · ‖d‖
```

The `‖d‖` divisor is the length normalisation and it is the crude part, over-penalising long documents
that are genuinely rich. TF-IDF is worth understanding because the vocabulary (_term weight_,
_document vector_, _cosine similarity_) is everywhere, including in the embedding world (file 09),
where "cosine similarity" means the same thing over a dense vector instead of a sparse one.

I am not shipping TF-IDF as my default scorer. BM25 beats it consistently and costs the same.

---

## 5.3 BM25, the one I implement

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

| Symbol    | Meaning                       | Default            |
| --------- | ----------------------------- | ------------------ |
| `tf(t,d)` | occurrences of t in d         | —                  |
| `df(t)`   | docs containing t             | —                  |
| `N`       | total docs                    | —                  |
| `|d|`     | length of d in terms          | —                  |
| `avgdl`   | average doc length            | —                  |
| `k₁`      | tf saturation control         | **1.2** (0.5–2.0)  |
| `b`       | length normalisation strength | **0.75** (0–1)     |

### Reading the formula properly

**Saturation.** Ignoring length for a second, the tf factor is `tf(k₁+1)/(tf+k₁)`. As `tf → ∞` it
approaches `k₁+1`, a hard ceiling. With k₁ = 1.2:

| tf  | factor |
| --- | ------ |
| 1   | 1.00   |
| 2   | 1.38   |
| 5   | 1.77   |
| 10  | 1.96   |
| 100 | 2.17   |
| ∞   | 2.20   |

Going from 1 to 2 occurrences buys 38%; going from 10 to 100 buys 11%. **That is what defeats keyword
stuffing**, and it is the main thing BM25 has over TF-IDF. Lower k₁ saturates faster, good for short
homogeneous docs; higher k₁ is closer to linear tf.

**Length normalisation.** The `(1 − b + b·|d|/avgdl)` term inflates the denominator for
longer-than-average documents. `b = 0` disables it entirely, which is right for fields like `title`
where length is not a proxy for dilution. `b = 1` fully normalises.

**IDF.** For a term in half the corpus, ≈ 0.69. For one in 0.1%, ≈ 6.9. A tenfold weight difference,
which is what makes rare discriminating terms drive ranking.

### Implementation notes that matter

- Precompute per-term `idf` once per query, not per document.
- `|d|` is stored per doc as a **norm**. Lucene famously compresses it to a _single byte_ with a lossy
  encoding, because precision there barely affects ranking and the array has to stay small enough for
  cache. I will start with `float32` and optimise later.
- Precompute the per-term `maxScore`, the score at that term's highest tf and shortest doc, at index
  time to enable WAND (file 04). That is why the codec tracks max impact per block.
- Scores are **not** probabilities, **not** comparable across queries, and **not** in [0,1]. I never
  show them to users and never threshold them with a magic constant like `score > 0.5`.

### BM25F, multiple fields

Wrong way: score each field separately and sum. That double-counts saturation and lets a term matching
in five fields dominate.

Right way: combine the _term frequencies_ across fields with per-field weights **first**, then apply
saturation once:

```
tf̃(t,d) = Σ_f  boost_f · tf(t, d.f) / (1 − b_f + b_f · len_f/avgdl_f)
```

then feed `tf̃` through the BM25 saturation. That is what Elasticsearch's `combined_fields` query does,
and it is measurably better than `best_fields` for title + body setups.

---

## 5.4 Beyond the text score

Real ranking is a blend, and text relevance is one signal:

| Signal                           | Examples                                                              |
| -------------------------------- | --------------------------------------------------------------------- |
| **Query-dependent**              | BM25, phrase-proximity bonus, exact-match bonus, which field matched   |
| **Query-independent (static)**   | PageRank, popularity, star rating, is-verified, stock status           |
| **Freshness**                    | exponential decay on age; critical for news/logs, harmful for reference docs |
| **Personalisation**              | user history, locale, previously-clicked                              |
| **Business**                     | margin, promotion, inventory                                          |

Two combination styles:

- **Multiplicative / functional**: `finalScore = bm25 × freshnessDecay(age) × log(1+popularity)`.
  Simple, explainable, tunable by hand. I start here.
- **Learned**: features into a model (§5.6).

Design implication for the library: the scorer is an **interface**, and non-text signals come from
**doc values**, the columnar forward store, not from the postings. So I bake in
`Scorer { Score(docID, tf, norm) float64 }` plus a way to read a numeric column by docID.

---

## 5.5 Evaluating relevance, the part that gets skipped

I cannot improve what I do not measure, and "it looks better to me" is not measurement.

**Set metrics** (no ranking):

- `precision = relevant retrieved / retrieved`, how much of what I showed was good
- `recall = relevant retrieved / relevant total`, how much of the good stuff I found
- `F1 = harmonic mean`

**Ranked metrics**, which is what I actually want:

- **P@k**: precision in the top k. Blunt but intuitive.
- **MRR**: mean of `1/rank of first relevant result`. Right metric when there is one correct answer,
  like navigational queries or go-to-definition.
- **MAP**: mean average precision, averaging precision at each relevant hit.
- **nDCG@k**: discounted cumulative gain, normalised. Handles **graded** relevance (perfect / good /
  ok / bad) and discounts by `1/log₂(rank+1)`. The standard for web-style search.

```
DCG@k = Σ_{i=1..k}  (2^rel_i − 1) / log₂(i + 1)
nDCG@k = DCG@k / IDCG@k          (IDCG = DCG of the perfect ordering)
```

Where judgments come from: hand-labelled sets (expensive, high quality), click models (cheap, biased
toward whatever already ranks highly, since position bias is severe), or public test collections
(MS MARCO, BEIR, TREC, and CodeSearchNet for code).

Online: A/B tests with interleaving, plus guardrail metrics (zero-result rate, abandonment, time to
first click). Offline and online metrics disagree more often than is comfortable.

For my library: build a tiny evaluation harness early. A JSON file of `{query, [relevant docIDs]}` and
an nDCG@10 report. It converts ranking work from taste into engineering and takes about an hour.

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

**LTR approaches**: pointwise (regression on relevance), pairwise (RankNet, learn "A above B"),
listwise (LambdaMART/LambdaRank, optimise nDCG directly). LambdaMART with gradient-boosted trees is
still the workhorse and beats neural rankers on tabular feature sets surprisingly often.

Typical features: BM25 per field, exact-match flags, query/doc length, proximity, popularity,
freshness, click-through rate, embedding cosine.

**Neural stage 2**: a cross-encoder, encoding query and document _together_ with a transformer, is far
more accurate than any bi-encoder similarity and far too slow to run over the whole corpus. Exactly
why it belongs in stage 2.

My library's job is to be an excellent **stage 1** and to expose the hooks (feature extraction,
pluggable scorer, "return top 1000 with features") that let someone build stage 2 on top.

---

## 5.7 Relevance tuning checklist

When someone says search is bad, I work this list in order:

1. **Is it a matching problem or a ranking problem?** Is the desired doc in the result set at all? If
   not, it is analysis (file 02), not scoring. That distinction resolves most complaints.
2. **Zero-result queries**, the top source of user pain. Fix with fuzzy fallback, OR-relaxation
   (`minimumShouldMatch`), synonyms, or spell correction.
3. **Field weighting.** Title should outweigh body, 3–10×. Cheapest big win available.
4. **Phrase/proximity boost.** Documents where the query words appear _together_ are usually better.
   Add a phrase clause as an optional `SHOULD` boost, not a `MUST`.
5. **Exact-match boost.** If I stem, also index the unstemmed form and boost exact hits.
6. **Tune `b` per field**, `b=0` for titles.
7. **Static quality signal**, popularity or rating, gently: `log`, not linear.
8. Then, and only then, consider embeddings and LTR.

---

## Questions I should be able to answer

1. Compute the BM25 tf-factor (k₁=1.2, b=0.75) for tf=3 in a document of length 200 when avgdl = 100.
   Show every substitution.
2. Why does BM25 use `ln((N−df+0.5)/(df+0.5) + 1)` rather than plain `log(N/df)`? What goes wrong
   without the `+1` for a term in 90% of documents?
3. A term appears 50 times in a 10,000-word document and 3 times in a 100-word document. Which scores
   higher, and which `b` value flips the answer?
4. Why is summing per-field BM25 scores worse than BM25F? Construct a document where it misranks.
5. My nDCG@10 improves offline but click-through drops in the A/B test. Three plausible explanations.
6. Why does a cross-encoder belong in stage 2 and never in stage 1?
