# 10 — The Go You Need for This Project

Not a Go tutorial. This is the subset of Go that this specific project exercises, ordered by when the
roadmap needs it, with the traps that actually bite when you write systems code.

Toolchain here: **Go 1.26**. Use it — iterators, generics, and `slices`/`maps` are all stable.

---

## 10.1 Learning path, mapped to milestones

| Milestone | Go you must know by then |
|-----------|--------------------------|
| M0 warm-up | modules, packages, slices, maps, strings/bytes, `bufio`, error handling, `go test` |
| M1 in-memory index | structs, methods, interfaces, sorting, table-driven tests |
| M2 analyzers | `unicode/utf8`, `x/text`, iterators (`iter.Seq`), composition |
| M3 scoring | `container/heap` or a hand-rolled heap, generics, float care |
| M4 on-disk | `encoding/binary`, `io.ReaderAt`, `unsafe` basics, benchmarks, `pprof` |
| M5 durability | `os` file semantics, `Sync()`, atomic rename, `defer`, fuzzing |
| M6–M7 perf | escape analysis, `sync.Pool`, bounds-check elimination, `benchstat` |
| M9 concurrency | goroutines, channels, `sync/atomic`, `errgroup`, `context` |

**External resources worth the time:** the Tour of Go (a day), *Effective Go*, Go Code Review
Comments, "Learn Go with Tests" (Chiu-Ki Chan / Chris James — free, test-first, ideal here), Dave
Cheney's blog on practical Go, and the official blog posts on slices, strings, and profiling.

---

## 10.2 Slices — the thing you will misuse first

```go
s := make([]uint32, 0, 128)   // len 0, cap 128 — preallocate when you know the size
s = append(s, 5)              // may or may not reallocate
```

Rules that matter for postings buffers:

- `append` **may return a different backing array**. Always `s = append(s, x)`. Never keep a stale
  copy of a slice you appended to elsewhere.
- Growth is amortised (roughly ×2 for small slices, ×1.25 above ~256 elements). Preallocating with
  `make([]T, 0, n)` when `n` is known removes most allocations from a build loop — this is often the
  single biggest indexing speedup available.
- **Sub-slicing shares memory.** `b := a[10:20]` keeps the whole of `a` alive; if `a` is a 4 MB read
  buffer and you retain one 10-byte term, you have leaked 4 MB. This is *the* Go memory leak in text
  processing. Copy when you retain: `term := append([]byte(nil), b...)` or `bytes.Clone(b)`.
- Three-index slicing `a[1:3:3]` caps capacity, so a later `append` cannot stomp on data past the
  end. Use it when handing a sub-slice to someone who might append.

```go
// The pattern you will write a hundred times:
buf = binary.AppendUvarint(buf[:0], value)   // reuse the buffer, keep allocations at zero
```

---

## 10.3 Strings vs []byte

- `string` is immutable, has a pointer + length, and **`[]byte(s)` copies**.
- Terms come from bytes and are used as map keys. `m[string(byteSlice)]` in a *lookup* position is
  optimised by the compiler to **not** allocate. Assigning `k := string(b)` first does allocate.
  This exact distinction is worth measuring once so you believe it:

```go
if v, ok := m[string(b)]; ok { ... }   // no allocation (compiler special case)
k := string(b); v, ok := m[k]           // allocates
```

- For a big dictionary, **do not use `map[string]X` with millions of keys.** Every string key is a
  separate heap object whose pointer the GC must scan on every cycle; 5M keys can add seconds of GC
  pause and hundreds of MB of overhead. The systems answer:

```go
type arena struct {
    buf  []byte              // all term bytes, concatenated
    offs []uint32            // start offset per term id
}
// dictionary: sorted []termID + binary search over arena, or a hash map from
// a 64-bit term hash → termID with the arena for verification.
```

One `[]byte` and one `[]uint32` are **two** GC objects instead of five million. This is the single
most important Go-specific performance lesson in this project.

- `unsafe.String` / `unsafe.Slice` (or `unsafe.StringData`) convert without copying. Correct only if
  you *guarantee* the bytes never change. Use sparingly, comment loudly, and confine to one file.

---

## 10.4 Structs and memory layout

```go
type Posting struct {
    DocID uint32
    TF    uint32
}                        // 8 bytes, no padding — good

type Bad struct {
    A bool     // 1 byte + 7 padding
    B uint64   // 8
    C bool     // 1 + 7 padding
}                        // 24 bytes; reorder to (B, A, C) → 16
```

- Field order affects size. `go vet`'s `fieldalignment` (via `golangci-lint`) flags it.
- **Struct-of-arrays beats array-of-structs** in hot loops: `docIDs []uint32; freqs []uint32` gives
  the prefetcher a clean sequential stream and lets you skip decoding freqs entirely when you do not
  need them. `[]Posting` forces you to touch both.
- Prefer value types in hot paths; a `[]*Posting` is a pointer chase per element plus GC work.

---

## 10.5 Interfaces — and when not to use them

Interfaces are how your library stays pluggable (`Analyzer`, `Codec`, `Directory`, `Scorer`). But a
method call through an interface is an indirect call: no inlining, and it defeats bounds-check
elimination.

Practical policy:

- Interfaces at **module boundaries** (the query iterator, the codec, the directory): yes.
- Interfaces **inside the decode loop that runs 10 million times**: no. Use concrete types or
  generics there, and keep the interface one level up so the cost is amortised over a whole block.
- Go idiom: **accept interfaces, return structs.** Define the interface in the *consuming* package,
  not next to the implementation.
- Keep them small. `io.Reader` is one method and is the most reused interface ever written.

---

## 10.6 Iterators (Go 1.23+) — a great fit here

```go
type Seq[V any] func(yield func(V) bool)

func (t *TermIterator) Postings() iter.Seq2[uint32, uint32] {
    return func(yield func(docID, tf uint32) bool) {
        for _, blk := range t.blocks {
            for i := range blk.n {
                if !yield(blk.docs[i], blk.freqs[i]) { return }
            }
        }
    }
}

for docID, tf := range term.Postings() { ... }
```

Lets you stream tokens and postings without materialising slices, with early termination for free.

**But**: query execution needs `Advance(target)` (file 04), which a pull-free `range` loop cannot
express. So use iterators for **analysis pipelines and bulk enumeration**, and a classic
`Next()/Advance()` interface for the **query executor**. `iter.Pull` bridges the two when needed, at
some cost. Knowing when *not* to use the shiny feature is the lesson.

---

## 10.7 Binary encoding

```go
import "encoding/binary"

buf = binary.AppendUvarint(buf, v)          // varint, allocation-free with a reused buf
v, n := binary.Uvarint(buf)                 // decode; n<=0 means error/truncation — check it
binary.LittleEndian.PutUint32(b[0:4], v)    // fixed width
```

- **Pick little-endian and write it in the format spec.** Every mainstream CPU is LE; do not pretend
  to be portable and then never test on a BE machine.
- `binary.Read`/`binary.Write` use reflection and are slow — fine for headers, never in a loop.
- Always check the `n <= 0` return from `Uvarint` — a truncated/corrupt file otherwise decodes as
  garbage silently.
- For bit-packing, write your own: `binary.LittleEndian.Uint64` over a byte slice, shift and mask.
  Generate the 32 unpack functions (one per bit width) with `go generate` — that is what Lucene does
  and it is a legitimate use of code generation.

---

## 10.8 File I/O and durability

```go
f, err := os.Create(path)          // truncates!
w := bufio.NewWriterSize(f, 1<<16) // always buffer; syscalls are expensive
...
w.Flush()                          // Flush is NOT durability
f.Sync()                           // fsync — this is durability
f.Close()

// atomic replace:
os.Rename(tmp, final)              // atomic within a filesystem
dir, _ := os.Open(filepath.Dir(final))
dir.Sync()                         // ← the step everyone forgets (file 06)
dir.Close()
```

- `io.ReaderAt` (`f.ReadAt`) is **safe for concurrent use** and needs no seeking — the right
  interface for a read-only segment file. Prefer it over `Seek`+`Read`.
- `defer f.Close()` on a *written* file swallows the close error, which can hide a write failure.
  For writers, close explicitly and check the error.
- mmap: `golang.org/x/exp/mmap` gives a safe `ReaderAt`; `syscall.Mmap` gives you the raw
  `[]byte`. With the raw form, a use-after-unmap is a SIGSEGV with no Go stack trace. Guard it with
  refcounting and never hand mmapped slices to callers — copy at the boundary.

---

## 10.9 Testing

```go
func TestIntersect(t *testing.T) {
    tests := []struct{ name string; a, b, want []uint32 }{
        {"disjoint", []uint32{1,3}, []uint32{2,4}, nil},
        {"identical", []uint32{1,2}, []uint32{1,2}, []uint32{1,2}},
        {"empty", nil, []uint32{1}, nil},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            if got := Intersect(tc.a, tc.b); !slices.Equal(got, tc.want) {
                t.Errorf("got %v want %v", got, tc.want)
            }
        })
    }
}
```

Three techniques that are worth more than a hundred hand-written cases in this project:

**1. Differential testing against a brute-force oracle.** You have a linear-scan reference
implementation from M1. Generate random corpora and random queries; assert the fast index returns
exactly what the slow one does. This finds skip-list and codec bugs that no unit test would.

**2. Fuzzing** (`go test -fuzz`) on every decoder. Corrupt bytes must produce an error, never a
panic, an infinite loop, or a 40 GB allocation. This is a security property, not just a robustness
one — a decoder that trusts a length prefix is a denial-of-service bug.

```go
func FuzzDecodePostings(f *testing.F) {
    f.Add([]byte{0x01, 0x02})
    f.Fuzz(func(t *testing.T, data []byte) {
        _, _ = DecodePostings(data)   // must not panic
    })
}
```

**3. Round-trip properties.** `decode(encode(x)) == x` for every codec, over randomly generated
inputs including edge cases: empty list, single element, max uint32, 128-boundary lengths
(127/128/129 are where block codecs break).

Also: `testing/quick` for quick property checks, golden files for the on-disk format (with a
`-update` flag), and `t.TempDir()` for anything touching the filesystem.

---

## 10.10 Benchmarking and profiling — the part that makes this project worth doing

```go
func BenchmarkTermLookup(b *testing.B) {
    idx := setup(b)
    b.ResetTimer()
    b.ReportAllocs()
    for b.Loop() {              // Go 1.24+; keeps the value alive, no need for a sink
        idx.Lookup("postgres")
    }
}
```

```bash
go test -bench=. -benchmem -count=10 > new.txt
benchstat old.txt new.txt          # statistically meaningful comparison, not vibes

go test -bench=BenchmarkQuery -cpuprofile=cpu.out -memprofile=mem.out
go tool pprof -http=:8080 cpu.out

go build -gcflags="-m" ./...       # escape analysis: what escapes to the heap and why
go test -bench=. -benchmem         # allocs/op is the number to drive to zero in hot loops
GODEBUG=gctrace=1 ./yourbuild      # GC frequency and pause during index build
```

Order of attack, from experience:

1. **Allocations first.** `allocs/op` in a hot loop is almost always the top cost in Go. Reuse
   buffers, avoid interface boxing, avoid `[]byte↔string` conversions.
2. **Then the algorithm.** Skip lists, better planning, WAND.
3. **Then the encoding.** Bit-packing, SIMD-friendly layouts.
4. **Only then micro-optimisation.** Bounds-check elimination (`_ = b[7]` hints), manual unrolling.

Never optimise before you have a benchmark that moves. Write the benchmark first; it is the only way
to know whether the clever thing helped.

---

## 10.11 Concurrency (M9)

```go
g, ctx := errgroup.WithContext(ctx)
for _, seg := range segments {
    g.Go(func() error { return searchSegment(ctx, seg, q, results) })
}
err := g.Wait()
```

- Go 1.22+ fixed the loop-variable capture footgun; `seg` above is per-iteration. If you read older
  code with `seg := seg`, that is why.
- **`sync/atomic` pointer swap** for the current segment snapshot → readers never lock.
- **`sync.Pool`** for decode buffers in the query path. Measure — a Pool that is not hit is pure
  overhead.
- **`context.Context`** threaded through search and merge so a slow query or a long merge can be
  cancelled. Check `ctx.Err()` at block boundaries, not per document.
- Run **everything** under `-race` in CI. A data race in an index is silent corruption.
- Bound your parallelism: `runtime.GOMAXPROCS(0)` workers, not one goroutine per document.

---

## 10.12 API design conventions

```go
// Functional options — the Go idiom for optional config that can grow compatibly.
func Open(path string, opts ...Option) (*Index, error)
func WithBufferSize(n int) Option
func WithCodec(c Codec) Option
```

- Zero values should be useful. `var b Buffer` working out of the box is very Go.
- Return `error` as the last value; wrap with `fmt.Errorf("open segment %d: %w", id, err)` so
  `errors.Is`/`errors.As` work through the stack.
- Sentinel errors (`var ErrCorrupt = errors.New("index: corrupt segment")`) for conditions callers
  branch on.
- Package name is part of the API: `index.Open`, not `index.OpenIndex`. Avoid stutter.
- `internal/` for anything you do not want to support forever. **Be aggressive with `internal/`** —
  every exported symbol is a promise.
- Doc comments start with the symbol name. `Example*` functions in tests appear in `go doc` and are
  compiled and run — the best documentation format Go has.

---

## 10.13 Traps checklist

| Trap | Reality |
|------|---------|
| `w.Flush()` means it is on disk | No. `f.Sync()` does. And fsync the directory after rename. |
| `defer f.Close()` on a writer | Swallows write errors. Close explicitly, check the error. |
| Keeping `b[10:20]` of a 4 MB buffer | Retains all 4 MB. `bytes.Clone` when you retain. |
| `map[string][]uint32` with 5M keys | GC death. Arena + offsets. |
| `for i, v := range bigStructSlice` | `v` is a copy. Use the index, or a pointer element type. |
| Comparing floats with `==` | Scores are floats. Use a tolerance in tests. |
| Ignoring `binary.Uvarint`'s `n` | Corrupt files decode as garbage silently. |
| `int` on disk | `int` is 64-bit here, 32-bit elsewhere. Use explicit `uint32`/`uint64` in formats. |
| One goroutine per document | Scheduler thrash. Bound to GOMAXPROCS. |
| Benchmarking with `-count=1` | Noise. Use `-count=10` + `benchstat`. |

---

## 10.14 Go codebases to read (in this order)

1. **`encoding/binary`** in the stdlib — 200 lines, and you will use it constantly.
2. **`github.com/RoaringBitmap/roaring`** — a real compressed-bitmap implementation, well documented.
3. **`github.com/blevesearch/vellum`** — FST construction and traversal. The heart of file 03.
4. **`github.com/blevesearch/bleve`** — a complete Go search library. Read its `index/scorch`
   package: segments, merges, snapshots. This is the closest existing thing to what you are building;
   study its design decisions and then decide which you disagree with.
5. **`github.com/sourcegraph/zoekt`** — trigram code search in Go (file 08).
6. **Tantivy** (Rust) or **SQLite FTS5** (C) — for how a *very* good implementation is structured, if
   you are willing to read another language.
