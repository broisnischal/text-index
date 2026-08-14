# 08 — Code Indexing

Code is text, so a text index works on it. But code has properties that make a naive text index feel
useless, and the fixes are interesting enough that Google wrote a search engine specifically for it.

---

## 8.1 Why code is not prose

| Property | Consequence for indexing |
|----------|--------------------------|
| **Punctuation is semantic** | `->`, `::`, `!=`, `<<=` are tokens, not noise. A prose tokenizer deletes them. |
| **Identifiers are compounds** | `getUserByID`, `MAX_RETRY_COUNT`, `http_client_v2`. One token, several concepts. |
| **Case is meaningful** | `User` (type) vs `user` (variable). Lowercasing loses real information. |
| **Substring search is normal** | Developers search for `ErrNotFound` and also for `NotFound`. Word-boundary matching is not enough. |
| **Regex is the expected UI** | `func \w+Handler\(` is a normal query for a developer, and no word index can serve it. |
| **Vocabulary is unbounded** | Every project invents identifiers. Heaps' law barely applies; the dictionary keeps growing. |
| **Structure exists and matters** | Definition vs reference vs comment vs string literal are different things, and users care which. |
| **Duplication is massive** | vendored deps, generated files, forks. Dedup by content hash or drown. |
| **No natural relevance signal** | No inbound links, little text length variation. BM25 alone ranks poorly. |

So code search stacks **four different index types**, each answering a different question.

---

## 8.2 Level 0 — no index at all (and why it is competitive)

`ripgrep` / `git grep` scan the files. With a fast literal search (memchr/SIMD, Boyer-Moore, Teddy)
and a proper DFA-based regex engine, this hits 1–5 GB/s per core.

The Linux kernel is ~1.5 GB of source. `rg` searches it in well under a second, cold cache aside.
**For a single repository, brute force is often the correct engineering answer** — zero index to
build, zero staleness, full regex support, no memory overhead.

Why it stops working: multiply by 10,000 repositories (Sourcegraph, GitHub) or by "1000 queries per
second", and per-query full scan is untenable. That is the only reason to build an index.

Worth knowing what makes brute force fast, because it also makes your *verification* phase fast:
literal prefilter first (find candidate lines with `memchr`), full regex only on candidates; skip
binary files by checking for NUL in the first 8 KB; `mmap` for large files, buffered reads for small
ones; parallelism across files.

---

## 8.3 Level 1 — the trigram index (the key idea)

From Russ Cox's *"Regular Expression Matching with a Trigram Index"* (2012), describing how Google
Code Search worked. This is the paper to read; it is short and it will genuinely change how you
think about indexes.

### Index side

Index every **3-byte substring** of every file, as a term:

```
"func main" →  "fun", "unc", "nc ", "c m", " ma", "mai", "ain"
```

Postings: trigram → sorted list of fileIDs. Exactly the inverted index of file 03, with a different
tokenizer. Note there is no analysis, no stemming, no case folding — the raw bytes.

### Query side — the actual trick

You cannot look up a regex. But you *can* compute, from the regex, a **boolean query over trigrams
that every matching file must satisfy**. Then you run the regex only on the files that pass.

Cox defines a recursive analysis producing, per regex node, an `(emptyable, exact-set, prefix-set,
suffix-set, match-query)` tuple:

| Regex | Trigram query |
|-------|---------------|
| `hello` | `"hel" AND "ell" AND "llo"` |
| `hello|world` | `("hel" AND "ell" AND "llo") OR ("wor" AND "orl" AND "rld")` |
| `hel+o` | `"hel"` (the `+` makes the rest uncertain) |
| `abc.*xyz` | `"abc" AND "xyz"` |
| `[a-z]+` | ALL (no trigram is implied — must scan everything) |
| `(foo|foa)bar` | `("foo" OR "foa") AND ("oob" OR "oab") AND "bar"` |

Then: **run the trigram query → get candidate files → run the real regex on those files only.**

Two properties that make it sound:
- The trigram query must be a **necessary** condition, never a sufficient one. False positives are
  fine (verification removes them); false negatives are a correctness bug.
- Queries that imply no trigrams (`.` , `[a-z]+`, a 2-character literal) degrade to a full scan.
  That is acceptable and expected — always keep the brute-force path as the fallback.

### Cost

Trigram postings are large: nearly every 3-byte window in the corpus produces a posting. Index size
is commonly **1–3× the source size**. The saving is not space, it is that a query touches a few
thousand files instead of ten million.

**This is your M10 milestone**, and it is by far the most fun part of the whole project.

### Zoekt: the production refinement

Zoekt (used by Sourcegraph, written in Go — *read this codebase*) improves on plain trigrams:

- **Positional n-grams**: postings store `(fileID, offset)` so candidate *positions* are known, not
  just candidate files. Verification then checks a few bytes rather than re-scanning a file.
- **Case handling**: content is folded for the index, with a separate bit-vector recording original
  case, so case-sensitive queries are still exact without doubling the index.
- **Rarest-ngram-first**: pick the least frequent n-gram in the query to drive the scan.
- **Runes vs bytes**: careful UTF-8 handling so multi-byte characters do not break n-gram alignment.
- **Shards** with an index per shard, memory-mapped, searched in parallel.

Other names: `Hound` (simple Go trigram search), `livegrep` (Google, suffix-array based — a
different, elegant approach: a suffix array supports arbitrary substring search in `O(m log n)` but
costs 4–8× the corpus in RAM and is painful to update incrementally).

---

## 8.4 Level 2 — the symbol index

"Where is `parseConfig` defined?" is a different question from "where does the string `parseConfig`
appear", and it deserves its own index.

### ctags / universal-ctags

Regex-based, per-language pattern files. Emits `(symbol, file, line, kind)`. Fast, ancient,
language-agnostic, imprecise. Still the backbone of a lot of editor tooling.

### tree-sitter (the modern answer)

An incremental parsing library producing a concrete syntax tree.

- **Incremental**: re-parse after an edit costs time proportional to the *edit*, not the file. Made
  for editors, and equally good for an indexer watching a filesystem.
- **Error-tolerant**: gives you a usable tree for code that does not compile — vital, since most code
  you index at any instant is mid-edit or missing dependencies.
- **Query language**: S-expression patterns over the tree, e.g.

```scheme
(function_declaration name: (identifier) @definition.function)
(type_declaration (type_spec name: (type_identifier) @definition.type))
```

Run the query, emit `(symbol, kind, file, byte-range)`, index it as a normal inverted index with the
symbol as the term. Suddenly "go to definition" is a term lookup. Grammars exist for ~200 languages.
Go bindings: `github.com/smacker/go-tree-sitter` (and newer forks).

**This is the pragmatic sweet spot**: 90% of navigation value, no compiler, no build system, works
on any snapshot of any repo.

---

## 8.5 Level 3 — precise, compiler-derived indexes

Symbol *names* are ambiguous: three types called `Config`, methods with the same name on different
receivers, `x.Do()` where `x`'s type depends on inference. Only a compiler knows the truth.

- **LSIF** (Language Server Index Format) — precompute the answers an LSP server would give
  (definition, references, hover, implementations) into a graph, so the IDE features work without a
  running language server. Emitted in CI per commit. Graph-shaped, verbose.
- **SCIP** (Sourcegraph's successor to LSIF) — protobuf, simpler document-centric model, much smaller
  and faster to produce. Uses stable, globally-unique **symbol identifiers**
  (`scip-go gomod github.com/x/y v1.2.3 pkg/Type#Method().`) so references can be resolved *across
  repositories and versions* — which is the entire point.
- **Kythe** (Google) — the most general/ambitious: a language-agnostic graph schema of nodes and
  edges over all code.
- **SemanticDB** (Scala), **rust-analyzer**'s internal index, `gopls`' packages graph — language-
  specific equivalents.

Storage: the interesting shape here is a **graph index**, not an inverted index —
`symbol → definition`, `symbol → [references]`, `file → [occurrences with ranges]`. Two key-value
maps and a per-file occurrence list get you most of it.

**Cost**: requires building the code (dependencies, toolchains, per-language CI). This is why
precise indexing is expensive and why hybrid systems fall back to tree-sitter or plain text when
precise data is missing. Design for graceful degradation:

```
precise (SCIP) → syntactic (tree-sitter) → literal (trigram) → brute force (scan)
```

Every real code-search product implements exactly that ladder.

---

## 8.6 Level 4 — semantic/embedding search over code

"Where do we handle rate limiting?" matches no literal. Embed code chunks, ANN search (file 09).

Practical notes specific to code:
- **Chunk by structure** (function/class), not by fixed token windows. A chunk split mid-function is
  useless. Tree-sitter gives you the boundaries — the levels compose.
- Include context in the embedded text: file path, enclosing type, docstring, signature.
- Code-specific embedding models exist and beat general ones on this task.
- **Embeddings are bad at exact identifiers.** `ErrNoRows` is a *lexical* query. Always hybrid.

---

## 8.7 Ranking code results

BM25 barely applies — code documents have weird length distributions and no natural link graph.
Signals that actually work, roughly in order:

1. **Match kind**: symbol definition ≫ symbol reference ≫ file path ≫ content ≫ comment ≫ test file
2. **Path signals**: `/vendor/`, `/node_modules/`, `/third_party/`, `.min.js`, generated-file markers
   → heavily demoted. This one change improves perceived quality more than any scoring formula.
3. **Whole-word / exact-case match** > substring match
4. **Repository importance**: stars, internal ownership, is-it-the-monorepo
5. **Recency of modification**, file churn
6. **Proximity to the user**: same repo, same package, same directory as the file they are in
7. **Shorter files/paths** slightly favoured (a match in a 20-line file is more likely relevant than
   one in a 50,000-line generated file)

---

## 8.8 Incremental indexing

Code changes constantly; a full reindex per commit does not scale.

- **Content-addressed caching**: key everything by `blob hash` (git already gives you this). An
  unchanged file's postings are reused verbatim across commits and across branches. Git's object
  model is a gift here — take it.
- **Watch the filesystem** (`fsnotify`) for local/editor indexes; debounce, batch, and re-index the
  changed files only.
- **Per-file segments** or small segments per commit, merged in the background (file 06).
- **Branch handling** is the hard part: indexing every branch of every repo explodes. Most systems
  index the default branch fully and only recently-touched other branches.

---

## 8.9 What a coding agent actually does (including this one)

Worth noting because it is the same stack under a different UI:

- **Literal / regex search** (ripgrep) for exact identifiers, error strings, TODOs — no index,
  because the working tree is one repo and brute force wins.
- **Path/glob matching** for structural navigation.
- **Symbol lookup** (LSP/tree-sitter) for definitions and references.
- **Embedding search** when the query is intent-shaped rather than literal-shaped.
- **Ranking + truncation**, because the real constraint is a context window, not a screen. Which
  makes the top-k problem from file 04 *more* important, not less.

The lesson: the retrieval stack for AI tools is the same retrieval stack, with a token budget instead
of a page of results.

---

## 8.10 If you build a code index (M10)

Minimum viable, in order:

1. Walk a directory, skip binaries/`.git`/gitignored paths, dedup by content hash.
2. Trigram index: `trigram → sorted []fileID`, delta+varint encoded — you already have the codec.
3. Regex → trigram query translation. Start with literals and alternation; `.*` and char classes
   simply yield "no constraint".
4. Candidate verification with Go's `regexp` (note: RE2, so no backreferences — and it is linear
   time, which is exactly what you want for untrusted patterns).
5. Fall back to full scan when the query implies no trigrams.
6. Then: positional n-grams, case bits, tree-sitter symbols, path ranking.

Benchmark target: index the Go standard library (~2M lines), then a large repo, and compare
end-to-end query latency against `rg` on the same queries. Beating `rg` on a single repo is *hard*
and that is a genuinely useful thing to learn firsthand.

---

## Check yourself

1. Derive the trigram query for `func\s+(New|Make)[A-Z]\w*\(`. Which parts contribute constraints and
   which do not?
2. Why must the trigram query be a *necessary* condition? What user-visible bug results if it is
   accidentally sufficient-but-not-necessary?
3. Why does Zoekt store case separately instead of indexing both cases?
4. A 2-character query like `if` implies no trigram. What do you do?
5. Why is tree-sitter's error tolerance a *requirement* rather than a nicety for an indexer?
6. You index 10,000 repos with heavy forking. What is the single most important space optimisation?
