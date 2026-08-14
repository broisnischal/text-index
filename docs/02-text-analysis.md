# 02 — Text Analysis: From Bytes to Terms

The index can only find what analysis produced. If my analyzer turns `"Wi-Fi"` into `["wi", "fi"]`,
then no amount of clever postings compression will ever let someone find it by typing `"WiFi"`.
**Analysis sets the ceiling on search quality.** It is also the part I expect to get wrong first,
because it is unglamorous and full of linguistics.

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

Each surviving token carries:

| Attribute                     | Why it exists                                                                                          |
| ----------------------------- | ------------------------------------------------------------------------------------------------------ |
| `term` (bytes)                | the dictionary key                                                                                       |
| `position` (ordinal)          | token #0, #1, #2… needed for **phrase queries**                                                         |
| `startOffset`, `endOffset`    | maps back into the original text, needed for **highlighting**                                            |
| `positionIncrement`           | usually 1; `0` stacks a synonym at the same position; `>1` leaves a gap where a stopword was removed      |
| `type`                        | `<ALPHANUM>`, `<NUM>`, `<EMAIL>`, so later filters can behave differently                               |

`positionIncrement` is subtle and worth getting right early. Removing the stopword in
`"king of denmark"` has to leave a gap, so the phrase query `"king of denmark"` still matches while
`"king denmark"` as an exact phrase does not accidentally become the same thing. Getting that wrong
produces a class of bug that only shows up in phrase search, months later.

---

## 2.2 The iron rule: index-time and query-time analysis must agree

If I index `"Running"` as `run`, I have to analyse the query `"Running"` to `run` too. Otherwise I
look up `Running`, find nothing, and conclude the index is broken.

Deliberate asymmetries exist, and they are always _query-time-only expansions_:

- **Synonyms** are often applied only at query time so the synonym list can change without a reindex.
  Trade-off: query-time expansion makes queries slower and multi-word synonyms harder.
- **Edge n-grams for autocomplete** are index-time only. I index `["m","mo","mon","mong"]` and at
  query time look up the whole typed prefix `"mong"` once.

So a field in my library declares **two** analyzers, `indexAnalyzer` and `searchAnalyzer`, defaulting
to the same one. Five-minute design decision now, saves a rewrite later.

---

## 2.3 Character filters and Unicode normalisation

Before tokenising:

- **Strip markup**, HTML tags mostly, but preserve offsets so highlighting still points at the right
  place in the original. Keeping offsets correct through a stripping filter is genuinely fiddly and I
  expect to spend time on it.
- **Unicode normalisation.** `é` can be one code point (U+00E9) or two (`e` + U+0301). Identical to
  the eye, unequal to `==`. Normalise to **NFC** (composed) or **NFKC** (compatibility, which also
  folds `ﬁ`→`fi`, `①`→`1`, full-width `Ａ`→`A`). NFKC is usually right for search and wrong for
  storage. Go: `golang.org/x/text/unicode/norm`.
- **Case folding**, not `strings.ToLower`. Unicode defines case folding separately and there are
  traps: German `ß` folds to `ss`; Turkish dotted/dotless `i` (`İ`/`ı`) lowercases differently under
  the Turkish locale; Greek final sigma `ς`/`σ`. `golang.org/x/text/cases.Fold()` is the correct
  tool. For a search index I want **locale-independent** folding, so the same document indexes the
  same way regardless of the server's locale.
- **Diacritic folding**, `café` → `cafe`, so people who cannot type accents still find it. NFD
  decomposition then drop combining marks (`unicode.Mn`). Careful: in some languages that destroys
  meaning. `schön`/`schon` are different German words, and Swedish `å ä ö` are distinct letters, not
  decorated `a`/`o`. Language-dependent decision.

---

## 2.4 Tokenization

### The naive version and why it fails

`strings.Fields`, split on whitespace, breaks immediately:

- `"end."` and `"end"` become different terms.
- `"state-of-the-art"` becomes one term nobody will search for.
- Chinese, Japanese and Thai have **no spaces at all**.

### Standard approach: Unicode word boundaries (UAX #29)

The Unicode standard defines word-break rules, and Lucene's `StandardTokenizer` implements them. They
handle most Latin-script text sensibly, keep `3.14` and `can't` together, and split on punctuation.

In Go, `x/text` does not expose a full UAX#29 word segmenter today; `github.com/rivo/uniseg` does.
Writing a simplified one is a good early exercise: classify runes into Letter/Digit/Mark/MidLetter
and apply the break rules.

### Domain-specific tokenizers

| Domain            | What it needs                                                                                                                                                 |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CJK**           | Either a dictionary-based segmenter (MeCab/Kuromoji for Japanese, jieba for Chinese) or **bigrams**, indexing every adjacent character pair. Bigrams are dumb, language-agnostic, and work surprisingly well. Most engines default to CJK bigrams. |
| **Code**          | Split identifiers: `getUserName` → `[getUserName, get, user, name]`; `snake_case` → parts; keep the original too. Punctuation is meaningful (`->`, `::`, `!=`). File 08. |
| **Logs**          | Preserve IPs, UUIDs and paths as single tokens or the vocabulary explodes.                                                                                      |
| **Emails/URLs**   | Index whole _and_ parts: `nischal@aitc.ai` → `[nischal@aitc.ai, nischal, aitc.ai, aitc, ai]`.                                                                  |
| **Product codes** | A `keyword` field: no tokenisation at all, exact match only.                                                                                                    |

The practical rule I take from this: most schemas need both a `text` field (analysed, for search) and
a `keyword` field (raw, for exact filters, sorting, faceting) over the same source string.
Elasticsearch's `title` + `title.keyword` convention exists for that reason, so I build the
capability in.

---

## 2.5 Token filters

### Stopwords

Drop `the, of, and, is…`. Historically vital: in a 1990s index, `the` might be 7% of all postings.

**Modern view: mostly do not.** Because:

- IDF weighting (file 05) already gives near-zero weight to ubiquitous terms.
- Block-Max WAND (file 04) skips their postings efficiently.
- Removing them breaks real queries. `"to be or not to be"`, `"The Who"`, `"vitamin A"`, and the band
  `"Take That"` all become empty or wrong.

So I keep them, stay aware of the cost, and consider index pruning or **common-grams** (indexing
`the_who` as a single term) if one specific ubiquitous term hurts.

### Stemming vs lemmatisation

**Stemming** chops suffixes with rules. `running → run`, `caresses → caress`, `ponies → poni`.

- **Porter / Snowball**: rule-based, fast, no dictionary, language-specific (Snowball has ~25
  languages). The standard choice.
- **Errors go two ways.** Over-stemming: `university` and `universe` both → `univers`, false matches.
  Under-stemming: `mice` never reaches `mouse`, missed matches.
- **Aggressiveness ladder**: minimal (plural-only) → light → Porter → Lovins. For most products
  _light_ beats aggressive, because users tolerate a missed match more than a nonsense one.

**Lemmatisation** uses a dictionary plus part-of-speech to get the true base form. `better → good`,
`mice → mouse`. Much more accurate, much slower, needs language models. Overkill for my library, but
I want the word.

**KStem/Krovetz** sits between the two: dictionary-checked stemming, conservative, popular in IR
research.

> Alternative worth keeping in mind: do not stem, index both forms, original _and_ stem, at the same
> position. Costs index size, buys precision, because then I can boost exact matches over stemmed
> ones. Several modern engines do this.

### Synonyms

`laptop ⇄ notebook`, `nyc → new york city`.

- **Same-position stacking** (positionIncrement 0) keeps phrase queries working.
- Multi-word synonyms are genuinely hard. Expanding `nyc` into three tokens shifts positions and
  breaks phrases unless handled carefully. A known dragon in Lucene, so I will not be surprised when
  it bites.
- Directional (`→`) vs equivalent (`⇄`) matters.

### N-grams and shingles

- **Character n-grams**: `search` at n=3 → `sea, ear, arc, rch`. Enables **substring** and **fuzzy**
  matching, language-agnostic. Costs: index size roughly ×(word length), and terms lose meaning so
  ranking degrades. This is the mechanism behind `pg_trgm` and code search (file 08).
- **Edge n-grams**: prefixes only, `s, se, sea, sear, searc, search`. The classic autocomplete index.
- **Word shingles**: `["the quick", "quick brown"]`. Turns common phrase queries into single-term
  lookups, and is used for near-duplicate detection.

### Phonetic

Soundex, Metaphone, Double Metaphone, Beider-Morse. `Smith`/`Smyth` collide. Useful for name search
(HR systems, customer lookup), noisy elsewhere. It belongs on a _separate field_ contributing a small
boost, never as the only match path.

### Language detection

For a multilingual corpus, per-document language detection routes to the right stemmer and stopword
list, and I would index into per-language fields (`body_en`, `body_de`). Detection is itself an
n-gram classifier. Getting it wrong silently ruins recall for minority languages.

---

## 2.6 Worked example

Input document field `body`:

```
"The Quick-Running Foxes aren't café-hopping in NYC!"
```

Pipeline: HTML strip → NFKC → UAX#29 tokenizer → lowercase-fold → ASCII-fold → English light stem.

| pos | token     | after fold | after stem | offsets |
| --- | --------- | ---------- | ---------- | ------- |
| 0   | `The`     | `the`      | `the`      | 0–3     |
| 1   | `Quick`   | `quick`    | `quick`    | 4–9     |
| 2   | `Running` | `running`  | `run`      | 10–17   |
| 3   | `Foxes`   | `foxes`    | `fox`      | 18–23   |
| 4   | `aren't`  | `aren't`   | `aren't`   | 24–30   |
| 5   | `café`    | `cafe`     | `cafe`     | 31–35   |
| 6   | `hopping` | `hopping`  | `hop`      | 36–43   |
| 7   | `in`      | `in`       | `in`       | 44–46   |
| 8   | `NYC`     | `nyc`      | `nyc`      | 47–50   |

What that enables and forbids:

- Query `"running fox"` matches, both stemmed identically. ✅
- Query `"quick-running"` matches as a phrase, positions 1 and 2 adjacent. ✅
- Query `"cafe"` matches `café`. ✅
- Query `"new york"` does **not** match `NYC`. Needs a synonym filter. ❌
- Query `"Fox"` with case sensitivity is impossible, case was destroyed. That needs a second field,
  which is why code search indexes case separately (file 08). ❌

---

## 2.7 Failure modes to design against

| Symptom                                                    | Usual cause                                                        |
| ---------------------------------------------------------- | ------------------------------------------------------------------ |
| "Search finds nothing for a word I can see in the document" | Query analyzer ≠ index analyzer                                    |
| "Highlighting underlines the wrong characters"             | Offsets not preserved through a char filter                        |
| "Phrase search matches things it shouldn't"                 | positionIncrement not handled around removed tokens                |
| "Term dictionary is 40 GB"                                  | Indexing UUIDs/hashes/base64 as text. Keyword field, or drop them   |
| "Index size 5× the raw corpus"                              | n-gram filter applied to a large body field                        |
| "Turkish users get wrong results"                           | `ToLower` instead of Unicode case folding                          |
| "Changing the stemmer did nothing"                          | Analyzer change requires a **reindex**; existing terms are frozen  |

That last one is worth emphasising: **analysis is baked into the index at write time.** Changing an
analyzer means rebuilding. So my segment metadata records the analyzer configuration, and a reader
whose analyzer does not match refuses or warns rather than returning nonsense. That is a design
decision most hobby projects miss and I do not want to.

---

## 2.8 What I build (M2 in the roadmap)

Minimum viable analyzer set:

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

Go 1.23+ range-over-func iterators fit this well and avoid materialising token slices (file 10). I
design for _streaming_: a 100 MB document must not require a 100 MB token slice.

---

## Questions I should be able to answer

1. Why must `positionIncrement` be > 1 after a removed stopword? Construct a phrase query that gives
   a wrong answer without it.
2. I index `"C++"` with a standard tokenizer. What terms come out, and how do I make `C++` findable?
   (Famous real problem.)
3. Searching `"apple"` returns documents about `"apples"`, but searching `"apples"` returns nothing.
   What is broken?
4. Why is NFKC usually right for indexing and wrong for storing the original document?
5. I need both `"iPhone"` case-insensitive and a case-_sensitive_ code search over the same corpus.
   How many indexes or fields, and why?
