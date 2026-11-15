---
title: "Running cells: Kernel and the system image"
weight: 25
description: "How a program loads the prebuilt system image by its pinned SHA-256 and runs notebook cells on it: FetchKernel, Eval, soft scope, cancellation, budgets, the sandbox, and Get and Set of Go values."
---

**Status: released in v0.14.0 (the image) and v0.15.0 (`Kernel`); current in
v0.17.0, with soft scope, julia's `MethodError` text, and `Get` and `Set`.** This page
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
`VM.MaxMemoryElems` is `ErrMemoryLimit`.

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
at its load.

## Known limits

- The candidates of a `MethodError` whose candidates are all equally close
  (`sin()`) list in the order of the method table, which in julia depends on
  hashes that mix in each module's random build id: that order is one system
  image's and is not reproduced.
- `uppercase`, `lowercase` and the functions that call `utf8proc` are not ported
  ([#65](https://github.com/go-embedded-julia/julia/issues/65)); a BLAS kernel
  such as `A * x` does not compute yet
  ([#33](https://github.com/go-embedded-julia/julia/issues/33)).
- A VM without Base (`Boot()` only) shows values and errors as the Go repr and
  cells use `Core`'s names.
- The image is not authenticated by itself
  ([#49](https://github.com/go-embedded-julia/julia/issues/49)): pin the digest.
- Cold start measured on darwin/arm64 (Apple M4 Max): load plus kernel about
  1 s, the first cell under a millisecond; the load alone is about 0.65 s.
