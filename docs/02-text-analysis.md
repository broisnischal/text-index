# 02 — Text Analysis: From Bytes to Terms

The index can only find what analysis produced. If your analyzer turns `"Wi-Fi"` into
`["wi", "fi"]`, then no amount of clever postings compression will ever let a user find it by typing
`"WiFi"`. **Analysis decides the ceiling of your search quality.** It is also the part most people
get wrong, because it is unglamorous and full of linguistics.

---

## 2.1 The pipeline

```
raw bytes
   │
   ▼  character filters      strip HTML, map "&amp;"→"&", normalise Unicode
   │
   ▼  tokenizer              split into tokens + record byte offsets & positions
   │
   ▼  token filters (chain)  lowercase, fold accents, remove stopwords, stem, synonyms, n-gram
   │
   ▼
terms  →  index
```

Each token that survives carries:

| Attribute | Why it exists |
|-----------|---------------|
| `term` (bytes) | the dictionary key |
| `position` (ordinal) | token #0, #1, #2… — needed for **phrase queries** |
| `startOffset`, `endOffset` (bytes) | maps back into the original text — needed for **highlighting** |
| `positionIncrement` | usually 1; `0` stacks a synonym at the same position; `>1` leaves a gap where a stopword was removed |
| `type` | `<ALPHANUM>`, `<NUM>`, `<EMAIL>` — lets later filters behave differently |

`positionIncrement` is subtle and worth understanding early. Removing the stopword in
`"king of denmark"` must leave a gap so that the phrase query `"king of denmark"` still matches while
`"king denmark"` (as an exact phrase) does not accidentally become the same thing. Getting this wrong
produces a class of bug that only shows up in phrase search, months later.

---

## 2.2 The iron rule: index-time and query-time analysis must agree

If you index `"Running"` as `run`, you must analyse the query `"Running"` to `run` too. Otherwise
you look up `Running`, find nothing, and conclude your index is broken.

Deliberate asymmetries exist, and they are always *query-time-only expansions*:

- **Synonyms** are often applied only at query time so you can change the synonym list without
  reindexing. Trade-off: query-time expansion makes queries slower and multi-word synonyms harder.
- **Edge n-grams for autocomplete** are index-time only — you index `["m","mo","mon","mong"]` but at
  query time you look up the whole typed prefix `"mong"` once.

Your library should therefore let a field declare **two** analyzers (`indexAnalyzer`,
`searchAnalyzer`) defaulting to the same one. This is a five-minute design decision that saves a
rewrite later.

---

## 2.3 Character filters and Unicode normalisation

Before tokenising:

- **Strip markup** — HTML tags, but preserve offsets so highlighting still points at the right place
  in the original. (Keeping correct offsets through a stripping filter is genuinely fiddly; note it.)
- **Unicode normalisation.** `é` can be one code point (U+00E9) or two (`e` + U+0301). They look
  identical and compare unequal. Normalise to **NFC** (composed) or **NFKC** (compatibility —
  also folds `ﬁ`→`fi`, `①`→`1`, full-width `Ａ`→`A`). NFKC is usually right for search, wrong for
  storage. In Go: `golang.org/x/text/unicode/norm`.
- **Case folding.** Not `strings.ToLower`. Unicode defines *case folding* separately, and there are
  traps: German `ß` folds to `ss`; Turkish dotted/dotless `i` (`İ`/`ı`) lowercases differently under
  the Turkish locale; Greek final sigma `ς`/`σ`. `golang.org/x/text/cases.Fold()` is the correct
  tool. For a search index you almost always want **locale-independent** folding, so that the same
  document indexes the same way regardless of the server's locale.
- **Diacritic folding** — `café` → `cafe`, so users who cannot type accents still find it. Done via
  NFD decomposition then dropping combining marks (`unicode.Mn`). Careful: in some languages this
  destroys meaning (`schön`/`schon` in German are different words; Swedish `å ä ö` are distinct
  letters, not decorated `a`/`o`). **Language-dependent decision.**

---

## 2.4 Tokenization

### The naive version and why it fails

`strings.Fields` (split on whitespace) breaks immediately:

- `"end."` and `"end"` become different terms.
- `"state-of-the-art"` becomes one term nobody will search for.
- Chinese/Japanese/Thai have **no spaces at all**.

### Standard approach: Unicode word boundaries (UAX #29)

The Unicode standard defines word-break rules. Lucene's `StandardTokenizer` implements them. They
handle most Latin-script text sensibly, keep `3.14` and `can't` together, and split on punctuation.

Go: `x/text` does not expose a full UAX#29 word segmenter today; `github.com/rivo/uniseg` does, and
writing a simplified one is a good early exercise (roughly: classify runes into Letter/Digit/Mark/
MidLetter/etc., then apply the break rules).

### Domain-specific tokenizers

| Domain | What you need |
|--------|---------------|
| **CJK** | Either a dictionary-based segmenter (MeCab/Kuromoji for Japanese, jieba for Chinese) or **bigrams** — index every adjacent character pair. Bigrams are dumb, language-agnostic, and work surprisingly well. Most engines default to CJK bigrams. |
| **Code** | Split identifiers: `getUserName` → `[getUserName, get, user, name]`; `snake_case` → parts; keep the original too. Punctuation is meaningful (`->`, `::`, `!=`). See file 08. |
| **Logs** | Preserve IPs, UUIDs, paths as single tokens or you get a term explosion. |
| **Emails/URLs** | Index whole *and* parts: `nischal@aitc.ai` → `[nischal@aitc.ai, nischal, aitc.ai, aitc, ai]`. |
| **Product codes** | Usually a `keyword` field: no tokenisation at all, exact match only. |

**Practical rule:** most schemas need both a `text` field (analysed, for search) and a `keyword`
field (raw, for exact filters, sorting, faceting) over the same source string. Elasticsearch's
`title` + `title.keyword` multi-field convention exists for this reason. Build the capability in.

---

## 2.5 Token filters

### Stopwords

Drop `the, of, and, is…`. Historically vital: in a 1990s index, `the` might be 7% of all postings.

**Modern view: mostly don't.** Reasons:
- IDF weighting (file 05) already gives near-zero weight to ubiquitous terms.
- Block-Max WAND (file 04) skips their postings efficiently.
- Removing them breaks real queries: `"to be or not to be"`, `"The Who"`, `"vitamin A"`, and the
  band `"Take That"` all become empty or wrong.

Keep them, but be aware of the cost, and consider **index pruning** or **common-grams** (index
`the_who` as a single term) if a specific ubiquitous term hurts.

### Stemming vs lemmatisation

**Stemming** — chop suffixes with rules. `running → run`, `caresses → caress`, `ponies → poni`.

- **Porter / Snowball**: rule-based, fast, no dictionary, language-specific (Snowball has ~25
  languages). The standard choice.
- **Errors go two ways.** *Over-stemming*: `university` and `universe` both → `univers` (false
  matches). *Under-stemming*: `mice` never reaches `mouse` (missed matches).
- **Aggressiveness ladder**: minimal (plural-only) → light → Porter → Lovins. For most products,
  *light* stemming beats aggressive: users tolerate a missed match more than a nonsense one.

**Lemmatisation** — dictionary + part-of-speech to get the true base form. `better → good`,
`mice → mouse`. Much more accurate, much slower, needs language models. Usually overkill for a
library like yours, but know the word.

**KStem/Krovetz** sits between: dictionary-checked stemming, conservative, popular in IR research.

> Alternative worth knowing: don't stem, index both forms — original *and* stem — at the same
> position. Costs index size, buys precision (you can boost exact matches over stemmed ones). This
> is what several modern engines do.

### Synonyms

`laptop ⇄ notebook`, `nyc → new york city`.

- **Same-position stacking** (positionIncrement 0) lets phrase queries still work.
- Multi-word synonyms are genuinely hard: expanding `nyc` into three tokens shifts positions and
  breaks phrases unless you handle it carefully. This is a known dragon in Lucene; do not be
  surprised when it bites.
- Directional (`→`) vs equivalent (`⇄`) matters.

### N-grams and shingles

- **Character n-grams**: `search` at n=3 → `sea, ear, arc, rch`. Enables **substring** and **fuzzy**
  matching and is language-agnostic. Costs: index size roughly ×(word length), and terms lose
  meaning so ranking degrades. This is the mechanism behind `pg_trgm` and code search (file 08).
- **Edge n-grams**: only prefixes — `s, se, sea, sear, searc, search`. The classic autocomplete
  index. Index-time expansion, query-time single lookup.
- **Word shingles** (word n-grams): `["the quick", "quick brown"]`. Speeds up common phrase queries
  by turning them into single-term lookups; used for near-duplicate detection too.

### Phonetic

Soundex, Metaphone, Double Metaphone, Beider-Morse. `Smith`/`Smyth` collide. Useful for name search
(HR systems, customer lookup), noisy elsewhere. Usually applied to a *separate field* that
contributes a small score boost, never as the only match path.

### Language detection

If your corpus is multilingual, per-document language detection routes to the right stemmer/stopword
list, and you typically index into per-language fields (`body_en`, `body_de`). Detection is itself a
classifier (n-gram based). Getting this wrong silently ruins recall for minority languages.

---

## 2.6 Worked example

Input document field `body`:

```
"The Quick-Running Foxes aren't café-hopping in NYC!"
```

Pipeline: HTML strip → NFKC → UAX#29 tokenizer → lowercase-fold → ASCII-fold → English light stem.

| pos | token | after fold | after stem | offsets |
|-----|-------|-----------|-----------|---------|
| 0 | `The` | `the` | `the` | 0–3 |
| 1 | `Quick` | `quick` | `quick` | 4–9 |
| 2 | `Running` | `running` | `run` | 10–17 |
| 3 | `Foxes` | `foxes` | `fox` | 18–23 |
| 4 | `aren't` | `aren't` | `aren't` | 24–30 |
| 5 | `café` | `cafe` | `cafe` | 31–35 |
| 6 | `hopping` | `hopping` | `hop` | 36–43 |
| 7 | `in` | `in` | `in` | 44–46 |
| 8 | `NYC` | `nyc` | `nyc` | 47–50 |

Note what this enables and forbids:
- Query `"running fox"` matches (both stemmed identically). ✅
- Query `"quick-running"` matches as a phrase (positions 1,2 adjacent). ✅
- Query `"cafe"` matches `café`. ✅
- Query `"new york"` does **not** match `NYC` — you would need a synonym filter. ❌
- Query `"Fox"` with case sensitivity is impossible — case was destroyed. If you need it, index a
  second field. ❌ (This is why code search indexes case separately; file 08.)

---

## 2.7 Failure modes to design against

| Symptom | Usual cause |
|---------|-------------|
| "Search finds nothing for a word I can see in the document" | Query analyzer ≠ index analyzer |
| "Highlighting underlines the wrong characters" | Offsets not preserved through a char filter |
| "Phrase search matches things it shouldn't" | positionIncrement not handled around removed tokens |
| "Term dictionary is 40 GB" | Indexing UUIDs/hashes/base64 as text. Use a keyword field or drop them |
| "Index size 5× the raw corpus" | n-gram filter applied to a large body field |
| "Turkish users get wrong results" | `ToLower` instead of Unicode case folding |
| "Changing the stemmer did nothing" | Analyzer change requires a **reindex**; existing terms are frozen |

That last one deserves emphasis: **analysis is baked into the index at write time**. Changing an
analyzer means rebuilding. Your library should therefore *record the analyzer configuration in the
segment metadata* and refuse (or warn) when a reader's analyzer does not match what wrote the
segment. This is a genuinely good design decision that most hobby projects miss.

---

## 2.8 What to build (mapping to the roadmap)

Milestone M2 in file 12. Minimum viable analyzer set:

```
Tokenizers:   whitespace, unicode (UAX#29-ish), keyword (no-op), ngram, edgeNgram, regex-split
Filters:      lowercase/fold, asciiFold, stop, porter/snowball stem, length-limit,
              truncate, synonym, shingle
Analyzers:    standard = unicode + fold + stop(none) + lightStem
              keyword  = keyword tokenizer, no filters
              code     = identifier-splitting (file 08)
```

The interface shape matters more than the filter count:

```
Tokenizer:  Tokenize(text []byte) iter.Seq[Token]     // streaming, zero-copy where possible
Filter:     Filter(iter.Seq[Token]) iter.Seq[Token]   // composable chain
Analyzer:   Analyze(field string, text []byte) iter.Seq[Token]
```

Go 1.23+ range-over-func iterators fit this beautifully and avoid materialising token slices — see
file 10. Design for *streaming*: a 100 MB document should not require a 100 MB token slice.

---

## Check yourself

1. Why must `positionIncrement` be > 1 after a removed stopword? Construct a phrase query that gives
   a wrong answer if you get this wrong.
2. You index `"C++"` with a standard tokenizer. What terms come out? How would you make `C++`
   findable? (This is a famous real problem.)
3. Your users complain that searching `"apple"` returns documents about `"apples"` but searching
   `"apples"` returns nothing. What is broken?
4. Why is NFKC usually right for indexing but wrong for storing the original document?
5. You must support both `"iPhone"` (case-insensitive) and a case-*sensitive* code search over the
   same corpus. How many indexes/fields, and why?
