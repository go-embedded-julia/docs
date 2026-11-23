---
title: "Running cells: Kernel and the system image"
weight: 25
description: "How a program loads the prebuilt system image by its pinned SHA-256 and runs notebook cells on it: FetchKernel, Eval, soft scope, cancellation, budgets, the sandbox, Get, Set, Has and TypeOf of Go values, and Reset."
---

**Status: released in v0.14.0 (the image) and v0.15.0 (`Kernel`); current in
v0.26.0, with soft scope, julia's `MethodError` text, `Get`, `Set`, `Has`,
`TypeOf`, a `Reset` that stops the tasks and timers cells started, an empty
`catch_backtrace()` (v0.23.0) and a Base image that carries the inference of a
`Float64` matrix display (v0.24.0), and LU and Cholesky through go-blas
(v0.25.0), and the bodies of `@generated` functions (v0.26.0).** This page
is the usage of the package `interp` for a notebook: how a VM starts in about a
second instead of building Base for 13 minutes, and how a cell runs on it. The
README of the repository holds the same flow in brief.

## The image, and why a digest is pinned

Each release of julia publishes its system image as the asset
`julia-image-vX.Y.Z.bin`, with `julia-image-vX.Y.Z.sha256`. The release workflow
builds it from the tag, and the bytes are the same on linux/amd64, linux/arm64
and darwin/arm64. The digest of a version cannot be written in that version's
own source, since the image depends on the source; so, as `go.sum` does, the
consumer pins it. `images.sum` at the root of the repository lists the digest of
each release.

An image is not authenticated by its key or its trailer, and loading one writes
state the whole process shares (issue 49 of the repository). Nothing reaches
`LoadImage` that does not have the pinned digest:

```go
digest, err := interp.ParseDigest("a4568120…")   // from images.sum or the release notes
k, err := interp.FetchKernel(ctx, "v0.15.0", digest,
	interp.FetchOptions{}, interp.LoadOptions{}, interp.KernelOptions{})
if err != nil { … }
defer k.Close()
```

`FetchKernel` is `FetchImage`, then `LoadImage`, then `NewKernel`. `FetchImage`
downloads over https within a size bound (256 MiB by default), refuses any other
digest with `ErrImageDigest` before the bytes are decoded, keeps the verified
file in the user cache directory and hashes it again at every use. A program
that ships the image itself uses `LoadKernel(reader, …)` or
`NewKernel(vm, …)` on a VM it loaded.

**`LoadOptions.Digest`.** A program that holds the image bytes pins the digest at
the load: `LoadOptions{Digest: &digest}` makes `LoadImage` and `LoadKernel` hash
the whole input first and refuse any other bytes with `ErrImageDigest`, before the
key, the framing or any object is looked at. The field is a pointer, so that
unset (the image is not pinned) cannot be a pinned all-zero digest by accident.
Load an image from an untrusted source with its digest. A load is atomic with
respect to what the process shares: the types and hints an image adds are
committed in one step after every conflict has been looked for, and a refused
image (`ErrImageConflict`) changes none of them.

## Eval

```go
r := k.Eval(ctx, `f(x) = x + 1; f(41)`)
// r.Value  "42"   julia's text/plain display of the last value
// r.Stdout, r.Stderr   what this cell printed, apart from every other cell's
// r.Err    nil, a *JuliaError (type and showerror's text), or a typed error
```

A cell that throws is a value, and the next cell runs. A cell defines names in
`Main`; functions, structs and redefined methods persist from cell to cell.

The text of an error is julia's own `showerror`, `MethodError` included (the
"Closest candidates" list, the ambiguity and its proposed fix, the keyword and
world-age hints), compared character for character with julia 1.12.6 on the
cases the test suite lists.

**Soft scope.** Cells run as the REPL and IJulia run them (`REPL.softscope`): a
loop or a block at the top of a cell assigns to a global that exists instead of
making a local, so `s = 0` in one cell and `for i in 1:10; s += i; end` in the
next gives `s == 55`. `CellOptions{HardScope: true}` gives the scope of a file
(`include_string`) instead.

## Go values in and out: Get and Set

`k.Get(name)` reads a global of `Main` as a Go value and `k.Set(name, v)`
assigns one, as a cell `name = v` would, also over an existing binding. Both
copy, never alias, and are refused with `ErrKernelBusy` while a cell runs.

| julia | Go |
| --- | --- |
| `Int64` | `int64` (`Set` also takes an `int`) |
| `Float64` | `float64`, every bit kept: NaN payloads, `-0.0` |
| `Bool`, `String` | `bool`, `string` |
| `Vector{Float64}`, `Vector{Int64}` | `[]float64`, `[]int64` |
| `Matrix{Float64}`, `Matrix{Int64}` | `interp.Matrix[float64]`, `interp.Matrix[int64]`: `Rows`, `Cols`, `Data` column-major, `At(i, j)` |

```go
k.Set("v", []float64{1, 2, 3.5})
k.Eval(ctx, "w = v .* 2")
w, ok, err := k.Get("w") // []float64{2, 4, 7}
```

`Get` answers `(value, ok, err)`: `ok` is false for a name that is not bound; a
value of any other type is a `*ConversionError` (`errors.Is(err, ErrKernelType)`)
that names the julia type, and `Set` of a Go type outside the table names the Go
type. A name that is no identifier is `ErrKernelName`, and an array over
`VM.MaxMemoryElems` is `ErrMemoryLimit`. `Set` checks a slice's length (and a
`Matrix`'s `Rows*Cols`, overflow-checked, against `len(Data)`) against
`VM.MaxMemoryElems` before it converts anything, so a refused slice costs a few
KiB. A matrix with a zero dimension reads back with its shape, whichever
dimension is the zero.

`k.Has(name)` answers whether a global of `Main` is bound, and `k.TypeOf(name)`
julia's type of it as text (`Matrix{Float64}`), also for a type the table does not
convert; neither copies the value. Both return an error, not a bare `false`,
while a cell runs: `(false, ErrKernelBusy)` is not the answer "no".

## Reset

```go
info, err := k.Reset()
// info.Reloaded  the image was loaded again (the log was past its bound)
// info.Undone    the writes replayed;  info.Tables  the method tables reverted
```

`Reset` puts the kernel back in the state `NewKernel` returned it in, for the
next run of a notebook: what the cells defined, assigned, imported, overloaded or
mutated is gone, the generator behind `rand` included, and the next cell runs as
the first of a fresh kernel. It is refused with `ErrKernelBusy` while a cell runs.

It costs what the cells changed, not the size of the image. Every write to an
object the image holds (`setfield!`, `swapfield!`, `modifyfield!`,
`replacefield!`, `setfieldonce!`, memory stores and copies) is logged with the
value it replaces, and `Reset` replays the log in reverse; the methods of the
functions a cell defined into or deleted from are reverted to the world of the
start, and the bindings, usings and options of every module are put back. An
object a cell made carries no log. Measured on a corpus of 18 cells (definitions
and redefinitions, a struct, a `show` overload on it, an overload of a Base
method, `using LinearAlgebra`, seeded `rand`, a `push!` to `Base.LOAD_PATH`, a
`delete_method` of a Base method, a module, errors), which wrote 3790 times to
objects of the image: `Reset` took 0.6 ms, loading the image again about 0.85 s.
The same corpus runs on a kernel after a `Reset` as on a fresh one, cell for cell
(`TestKernelResetBase`).

**The bound.** The log holds `DefaultMaxUndo` (262144) entries, about 18 MiB.
Past it, `Reset` loads the image again into a new VM and says so
(`ResetInfo.Reloaded`; `k.VM()` is another VM then, with `Stdout`, `Stderr`,
`MaxSteps`, `MaxDepth` and `MaxMemoryElems` carried over). A kernel made by
`NewKernel` from a VM, not from an image (`LoadKernel` and `FetchKernel` keep the
image's bytes for the reload), has nothing to load: `Reset` answers
`ErrResetOverflow` and changes nothing.

**Tasks and timers (v0.20.0).** Before it rolls back, `Reset` stops the tasks a
cell started that have not finished and closes the libuv timers it opened
(`ResetInfo.Tasks`, `ResetInfo.Timers`; `k.Tasks()` counts them for an
embedder), so that the replay restores the lists such a task waited in without
it. A stopped task never runs again, and **none of its `finally` or `catch`
blocks runs**: user code during a `Reset` could print, block or write the state
the `Reset` is restoring. The task's object is marked done, so `istaskdone` and
`wait` answer as for a task that ended. A timer is dropped as `close(t)` would
be, without its close callback. Base's own tasks are left alone. A kernel also
yields once when it is made, so that the tasks Base makes in its initializers
are parked before the state `Reset` returns to.

**What `Reset` does not undo**, by design:

- Go-side state a cell reaches through a `ccall`: open files and sockets, the
  environment, the working directory, what was already written to `Stdout` and
  `Stderr`, the clock, libraries opened with `dlopen`, the state of libgmp and
  libmpfr.
- The caches. A type a cell made stays in the weak type caches of the image's
  types until the collector drops it, and method specializations and compiled
  bodies stay. A limit an embedder set through `k.VM()` is the one in force.
- The world counter and the object-id counters, which only move forward:
  `Base.get_world_counter()` and `objectid` of a new object differ from a fresh
  kernel's.
- A write from Go code to a field of the image that is the interpreter's own
  bookkeeping (method-instance and code-instance caches, task state). A cell's
  own writes are all logged.

An unseeded `rand()` differs from one fresh kernel to the next, since the
generator is seeded when the kernel starts. After a `Reset` the generator is in
the state it started in, so the first draws are the ones the kernel would have
made first (`TestKernelResetRNG`).

## Cancellation and budgets

Cancelling `ctx` raises `InterruptException` in the running cell (`Eval`
answers `ErrCellCanceled`, wrapping `ctx.Err()`). The interrupt is delivered
once at first, so that a `finally` or a wait list cleans up, and again until the
cell ends, so that a `catch` cannot swallow it. A cell that waits (`sleep`) is
woken. Per-cell budgets, on the kernel or per `EvalWith`:

| Field | Bound | Error |
| --- | --- | --- |
| `Timeout` | wall time of the cell, its display included (30 s by default) | `ErrCellTimeout` |
| `MaxSteps` | statements executed | `ErrStepLimit` |
| `MaxAlloc` | elements the cell's allocations take in total (2^25 by default) | `ErrMemoryLimit` |
| `MaxOutput` | bytes of each of stdout and stderr (1 MiB); the rest is dropped | `Result.Truncated` |

A zero field is the kernel's value, a negative one means no bound.

## Sandbox

The sandbox is the default: the file system is julia's own tree and the
environment is empty. Host mode is `KernelOptions{Host: true}`, never a default,
and a VM that is already in host mode is refused without it
(`ErrKernelHost`). There is one kernel per VM and one running cell per kernel
(`ErrKernelExists`, `ErrKernelBusy`). Two kernels in one process share core's
type tables, which are guarded, and the hints an image sets on core's shared
types; an image sets them once and another image may not change them, so
kernels over images of one release are safe and a second release is refused
at its load. Since v0.20.0 a load writes nothing to the boot types its VM put in
those shared caches (a refused load leaves them as they were), and since v0.21.0
`New` may be called from several goroutines at once.

## Known limits

- Backtraces are off, as in a julia run with them off (v0.23.0):
  `catch_backtrace()` and the backtrace of a failed task are empty, and
  `jl_get_excstack` answers an empty vector, instead of failing as not ported.
- The first `Float64` matrix product and its display on a fresh kernel take 0.84
  million interpreter statements, where they took 69.6 million (v0.24.0): the
  display calls `Base._return_type`, which runs julia's own compiler in the
  interpreter, and the image builder now runs the display of a `Float64` matrix
  before it saves, so the inferred code is in the image. The first display of
  another element type may still pay that cost.
- BLAS computes `gemm`, `trsm`, `trmm`, `syrk` and `symm`, `gemv` and the
  level-1 routines for `Float64` and `Float32` (`A * B`, `A * x`, `x' * y`,
  `A' * A`, `Symmetric(A) * B`, `UpperTriangular(A) * B`, `B / UpperTriangular(A)`),
  through go-blas (v0.3.0 for level 3), within its stated error bound of the
  exact value and not bit for bit with OpenBLAS. Since v0.25.0 LAPACK's `getrf`,
  `getrs`, `potrf`, `potrs` and `lacpy` are forwarded too (go-blas v0.4.0), so
  `lu`, `A \ b`, `factorize`, `cholesky` and `cholesky(A) \ b` work for `Float64`
  and `Float32`, and `SingularException` and `PosDefException` are raised. The
  complex types, `inv`, `qr`, `eigen` and `svd` fail until go-blas has them, and
  so does `UpperTriangular(A) \ B`, which julia sends to LAPACK's `trtrs`, not to
  `trsm`
  ([#67](https://github.com/go-embedded-julia/julia/issues/67)).
- A `Memory` of `Float64` is stored as a `[]float64` (v0.21.0): the
  double-precision routines read and write it in place, at 1.02 to 1.16 times the
  cost of calling go-blas directly (3 to 22 times when the elements were boxed),
  and `Kernel.Get` and `Set` of a `[]float64` or a `Matrix[float64]` box nothing.
  The price is on the read: an interpreted `Float64` loop boxes one more
  allocation per element (`s += v[i]` over 10^6 elements: +2.70% allocations,
  `sum(v)` +1.13%, `v .+ 1.0` +0.52%). A `Memory` of `Float32` is still boxed,
  so a single-precision call copies the elements it reads and writes.
- `rand(m, n)` reaches an `llvmcall` that is not ported
  ([#120](https://github.com/go-embedded-julia/julia/issues/120)).
- The first matrix display of an element type other than `Float64` may still pay
  the cost of inference that the `Float64` one no longer does (see above).
- A VM without Base (`Boot()` only) shows values and errors as the Go repr and
  cells use `Core`'s names.
- The image is not authenticated by itself
  ([#49](https://github.com/go-embedded-julia/julia/issues/49)): pin the digest.
- Cold start measured on darwin/arm64 (Apple M4 Max): load plus kernel about
  1 s, the first cell under a millisecond; the load alone is about 0.65 s.
