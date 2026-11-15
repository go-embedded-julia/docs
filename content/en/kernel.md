---
title: "Running cells: Kernel and the system image"
weight: 25
description: "How a program loads the prebuilt system image by its pinned SHA-256 and runs notebook cells on it: FetchKernel, Eval, cancellation, budgets and the sandbox."
---

**Status: released in v0.14.0 (the image) and v0.15.0 (`Kernel`).** This page
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

- Cells use hard scope (as `include_string`), not the REPL's soft scope: at the
  top level, `x = 0; while …; x += 1; end` makes `x` local
  ([#54](https://github.com/go-embedded-julia/julia/issues/54)).
- `showerror` of a `MethodError` calls `jl_methtable_lookup`, which is not
  ported: the error is a `*JuliaError` of the right type and its message is the
  Go repr
  ([#55](https://github.com/go-embedded-julia/julia/issues/55)).
- A VM without Base (`Boot()` only) shows values and errors as the Go repr and
  cells use `Core`'s names.
- The image is not authenticated by itself
  ([#49](https://github.com/go-embedded-julia/julia/issues/49)): pin the digest.
- Cold start measured on darwin/arm64 (Apple M4 Max): load plus kernel about
  1 s, the first cell under a millisecond; the load alone is about 0.65 s.
