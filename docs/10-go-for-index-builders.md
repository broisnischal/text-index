# 10 — The Go This Project Needs

Not a Go tutorial. This is the subset of Go this specific project exercises, ordered by when the
roadmap needs it, with the traps that bite in systems code.

Toolchain here: **Go 1.26**, so iterators, generics and `slices`/`maps` are all stable.

---

## 10.1 Learning path, mapped to milestones

| Milestone            | Go I need by then                                                                    |
| -------------------- | ------------------------------------------------------------------------------------ |
| M0 warm-up           | modules, packages, slices, maps, strings/bytes, `bufio`, error handling, `go test`    |
| M1 in-memory index   | structs, methods, interfaces, sorting, table-driven tests                             |
| M2 analyzers         | `unicode/utf8`, `x/text`, iterators (`iter.Seq`), composition                         |
| M3 scoring           | `container/heap` or a hand-rolled heap, generics, float care                          |
| M4 on-disk           | `encoding/binary`, `io.ReaderAt`, `unsafe` basics, benchmarks, `pprof`                |
| M5 durability        | `os` file semantics, `Sync()`, atomic rename, `defer`, fuzzing                        |
| M6–M7 perf           | escape analysis, `sync.Pool`, bounds-check elimination, `benchstat`                   |
| M9 concurrency       | goroutines, channels, `sync/atomic`, `errgroup`, `context`                            |

External resources worth the time: the Tour of Go (a day), _Effective Go_, Go Code Review Comments,
"Learn Go with Tests" (free, test-first, ideal here), Dave Cheney's blog on practical Go, and the
official blog posts on slices, strings and profiling.

---

## 10.2 Slices, the thing I will misuse first

```go
s := make([]uint32, 0, 128)   // len 0, cap 128 — preallocate when the size is known
s = append(s, 5)              // may or may not reallocate
```

Rules that matter for postings buffers:

- `append` **may return a different backing array**. Always `s = append(s, x)`, and never keep a stale
  copy of a slice appended to elsewhere.
- Growth is amortised, roughly ×2 for small slices and ×1.25 above ~256 elements. Preallocating with
  `make([]T, 0, n)` when `n` is known removes most allocations from a build loop, which is often the
  single biggest indexing speedup available.
- **Sub-slicing shares memory.** `b := a[10:20]` keeps the whole of `a` alive, so if `a` is a 4 MB read
  buffer and I retain one 10-byte term, I have leaked 4 MB. That is _the_ Go memory leak in text
  processing. Copy when retaining: `bytes.Clone(b)` or `append([]byte(nil), b...)`.
- Three-index slicing `a[1:3:3]` caps capacity so a later `append` cannot stomp on data past the end.
  Use it when handing a sub-slice to someone who might append.

```go
// The pattern I will write a hundred times:
buf = binary.AppendUvarint(buf[:0], value)   // reuse the buffer, keep allocations at zero
```

---

## 10.3 Strings vs []byte

- `string` is immutable, holds a pointer plus length, and `[]byte(s)` **copies**.
- Terms come from bytes and get used as map keys. `m[string(byteSlice)]` in a _lookup_ position is
  optimised by the compiler to **not** allocate; assigning `k := string(b)` first does allocate. Worth
  measuring once to believe it:

```go
if v, ok := m[string(b)]; ok { ... }   // no allocation (compiler special case)
k := string(b); v, ok := m[k]           // allocates
```

- For a big dictionary, **do not use `map[string]X` with millions of keys.** Every string key is a
  separate heap object whose pointer the GC scans every cycle, and 5M keys can add seconds of GC pause
  plus hundreds of MB of overhead. The systems answer:

```go
type arena struct {
    buf  []byte              // all term bytes, concatenated
    offs []uint32            // start offset per term id
}
// dictionary: sorted []termID + binary search over arena, or a hash map from
// a 64-bit term hash → termID with the arena for verification.
```

One `[]byte` and one `[]uint32` are **two** GC objects instead of five million. The most important
Go-specific performance lesson in this project.

- `unsafe.String` / `unsafe.Slice` (or `unsafe.StringData`) convert without copying, and are correct
  only when I _guarantee_ the bytes never change. Sparingly, commented loudly, confined to one file.

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
- **Struct-of-arrays beats array-of-structs** in hot loops. `docIDs []uint32; freqs []uint32` gives the
  prefetcher a clean sequential stream and lets me skip decoding freqs entirely when I do not need
  them; `[]Posting` forces me to touch both.
- Prefer value types in hot paths. `[]*Posting` is a pointer chase per element plus GC work.

---

## 10.5 Interfaces, and when not to use them

Interfaces are how the library stays pluggable (`Analyzer`, `Codec`, `Directory`, `Scorer`). But a
method call through an interface is an indirect call: no inlining, and it defeats bounds-check
elimination.

My policy:

- Interfaces at **module boundaries** (the query iterator, the codec, the directory): yes.
- Interfaces **inside the decode loop that runs 10 million times**: no. Concrete types or generics
  there, with the interface one level up so the cost amortises over a whole block.
- Go idiom: accept interfaces, return structs. Define the interface in the _consuming_ package, not
  next to the implementation.
- Keep them small. `io.Reader` is one method and the most reused interface ever written.

---

## 10.6 Iterators (Go 1.23+), a good fit here

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

Lets me stream tokens and postings without materialising slices, with early termination for free.

But query execution needs `Advance(target)` (file 04), which a pull-free `range` loop cannot express.
So: iterators for **analysis pipelines and bulk enumeration**, a classic `Next()/Advance()` interface
for the **query executor**. `iter.Pull` bridges the two when needed, at some cost. Knowing when not to
use the shiny feature is the lesson.

---

## 10.7 Binary encoding

```go
import "encoding/binary"

buf = binary.AppendUvarint(buf, v)          // varint, allocation-free with a reused buf
v, n := binary.Uvarint(buf)                 // decode; n<=0 means error/truncation — check it
binary.LittleEndian.PutUint32(b[0:4], v)    // fixed width
```

- **Pick little-endian and write it in the format spec.** Every mainstream CPU is LE, and pretending to
  be portable while never testing on a BE machine is worse than committing.
- `binary.Read`/`binary.Write` use reflection and are slow. Fine for headers, never in a loop.
- Always check the `n <= 0` return from `Uvarint`, or a truncated/corrupt file decodes as garbage
  silently.
- For bit-packing, write my own: `binary.LittleEndian.Uint64` over a byte slice, shift and mask.
  Generate the 32 unpack functions, one per bit width, with `go generate`. That is what Lucene does and
  it is a legitimate use of code generation.

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

- `io.ReaderAt` (`f.ReadAt`) is safe for concurrent use and needs no seeking, so it is the right
  interface for a read-only segment file. Better than `Seek`+`Read`.
- `defer f.Close()` on a _written_ file swallows the close error, which can hide a write failure. For
  writers, close explicitly and check the error.
- mmap: `golang.org/x/exp/mmap` gives a safe `ReaderAt`, `syscall.Mmap` gives the raw `[]byte`. With
  the raw form, a use-after-unmap is a SIGSEGV with no Go stack trace. Guard it with refcounting and
  never hand mmapped slices to callers, copy at the boundary.

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

Three techniques worth more than a hundred hand-written cases here:

**1. Differential testing against a brute-force oracle.** I keep the linear-scan reference
implementation from M1, generate random corpora and random queries, and assert the fast index returns
exactly what the slow one does. That finds skip-list and codec bugs no unit test would.

**2. Fuzzing** (`go test -fuzz`) on every decoder. Corrupt bytes must produce an error, never a panic,
an infinite loop, or a 40 GB allocation. That is a security property, not just robustness: a decoder
that trusts a length prefix is a denial-of-service bug.

```go
func FuzzDecodePostings(f *testing.F) {
    f.Add([]byte{0x01, 0x02})
    f.Fuzz(func(t *testing.T, data []byte) {
        _, _ = DecodePostings(data)   // must not panic
    })
}
```

**3. Round-trip properties.** `decode(encode(x)) == x` for every codec, over randomly generated inputs
including the edge cases: empty list, single element, max uint32, and 128-boundary lengths, since
127/128/129 are where block codecs break.

Also: `testing/quick` for quick property checks, golden files for the on-disk format with a `-update`
flag, and `t.TempDir()` for anything touching the filesystem.

---

## 10.10 Benchmarking and profiling, the part that makes this project worth doing

```go
func BenchmarkTermLookup(b *testing.B) {
    idx := setup(b)
    b.ResetTimer()
    b.ReportAllocs()
    for b.Loop() {              // Go 1.24+; keeps the value alive, no sink needed
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

Order of attack:

1. **Allocations first.** `allocs/op` in a hot loop is almost always the top cost in Go. Reuse buffers,
   avoid interface boxing, avoid `[]byte↔string` conversions.
2. **Then the algorithm.** Skip lists, better planning, WAND.
3. **Then the encoding.** Bit-packing, SIMD-friendly layouts.
4. **Only then micro-optimisation.** Bounds-check elimination (`_ = b[7]` hints), manual unrolling.

Never optimise before a benchmark that moves. Write the benchmark first; it is the only way to know
whether the clever thing helped.

---

## 10.11 Concurrency (M9)

```go
g, ctx := errgroup.WithContext(ctx)
for _, seg := range segments {
    g.Go(func() error { return searchSegment(ctx, seg, q, results) })
}
err := g.Wait()
```

- Go 1.22+ fixed the loop-variable capture footgun, so `seg` above is per-iteration. Older code with
  `seg := seg` is why that idiom exists.
- **`sync/atomic` pointer swap** for the current segment snapshot, so readers never lock.
- **`sync.Pool`** for decode buffers in the query path. Measure it: a Pool that is not hit is pure
  overhead.
- **`context.Context`** threaded through search and merge so a slow query or long merge can be
  cancelled. Check `ctx.Err()` at block boundaries, not per document.
- Run everything under `-race` in CI. A data race in an index is silent corruption.
- Bound parallelism: `runtime.GOMAXPROCS(0)` workers, not one goroutine per document.

---

## 10.12 API design conventions

```go
// Functional options — the Go idiom for optional config that can grow compatibly.
func Open(path string, opts ...Option) (*Index, error)
func WithBufferSize(n int) Option
func WithCodec(c Codec) Option
```

- Zero values should be useful. `var b Buffer` working out of the box is very Go.
- Return `error` last, wrap with `fmt.Errorf("open segment %d: %w", id, err)` so `errors.Is`/`errors.As`
  work through the stack.
- Sentinel errors (`var ErrCorrupt = errors.New("index: corrupt segment")`) for conditions callers
  branch on.
- Package name is part of the API: `index.Open`, not `index.OpenIndex`. Avoid stutter.
- `internal/` for anything I do not want to support forever, and I am aggressive about it, because
  every exported symbol is a promise.
- Doc comments start with the symbol name. `Example*` functions in tests show up in `go doc` and get
  compiled and run, which is the best documentation format Go has.

---

## 10.13 Traps checklist

| Trap                                     | Reality                                                                 |
| ---------------------------------------- | ----------------------------------------------------------------------- |
| `w.Flush()` means it is on disk          | No. `f.Sync()` does. And fsync the directory after rename.              |
| `defer f.Close()` on a writer            | Swallows write errors. Close explicitly, check the error.               |
| Keeping `b[10:20]` of a 4 MB buffer      | Retains all 4 MB. `bytes.Clone` when retaining.                         |
| `map[string][]uint32` with 5M keys       | GC death. Arena + offsets.                                              |
| `for i, v := range bigStructSlice`       | `v` is a copy. Use the index, or a pointer element type.                |
| Comparing floats with `==`               | Scores are floats. Use a tolerance in tests.                            |
| Ignoring `binary.Uvarint`'s `n`          | Corrupt files decode as garbage silently.                               |
| `int` on disk                            | `int` is 64-bit here, 32-bit elsewhere. Explicit `uint32`/`uint64` in formats. |
| One goroutine per document               | Scheduler thrash. Bound to GOMAXPROCS.                                  |
| Benchmarking with `-count=1`             | Noise. `-count=10` plus `benchstat`.                                    |

---

## 10.14 Go codebases to read, in this order

1. **`encoding/binary`** in the stdlib, 200 lines, and I will use it constantly.
2. **`github.com/RoaringBitmap/roaring`**, a real compressed-bitmap implementation, well documented.
3. **`github.com/blevesearch/vellum`**, FST construction and traversal, the heart of file 03.
4. **`github.com/blevesearch/bleve`**, a complete Go search library. Its `index/scorch` package covers
   segments, merges and snapshots, and it is the closest existing thing to what I am building. Study
   the design decisions, then decide which I disagree with.
5. **`github.com/sourcegraph/zoekt`**, trigram code search in Go (file 08).
6. **Tantivy** (Rust) or **SQLite FTS5** (C), for how a very good implementation is structured, if I am
   willing to read another language.
