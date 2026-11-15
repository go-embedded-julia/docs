---
title: "go-embedded-julia documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A pure-Go, embeddable Julia interpreter, in development: v0.13.0 completes M0 (oracle harness), M1 (parser), M2 (lowering) and M3 (runtime): julia 1.12.6's real Base and seven stdlibs build and initialize, with tasks, an event loop and a two-mode virtual file system."
---

**go-embedded-julia is in development. [v0.13.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.13.0)
(2026-11-14) completes milestone M3 (the runtime): the interpreter runs julia
1.12.6's real Base and its seven stdlibs.** It follows milestone M0 (the oracle
harness, against julia 1.12.6), milestone M1 (the parser): a Go port of
JuliaSyntax, `Expr`-identical to `Meta.parseall` on 1191 of 1191 files of julia's
own tree, and milestone M2 (lowering): a Go port of julia 1.12.6's femtolisp front
end, identical to `Meta.lower` on 53 997 of 53 997 forms of that tree
([v0.3.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.3.0),
2026-10-28).

What M3 holds, in v0.13.0:

- **Base and seven stdlibs.** `sysimg.jl` builds the whole system image;
  `VM.RunModuleInitializers` runs `Base.__init__` and the `__init__` of FileWatching,
  Libdl, Artifacts, SHA, Sockets, LinearAlgebra and Random. 16 Base-level task
  programs (`@async`, `Channel`, `Condition`, `Timer`, `sleep`, `ReentrantLock`,
  `Threads.@spawn`, `ScopedValue`) give julia's own text/plain output.
- **Tasks and the event loop.** One goroutine per task with a baton, the per-task
  state saved at each switch, a libuv loop written in Go, `@cfunction`, and
  `ErrDeadlock` for a wait that nothing can end, where julia would hang.
- **A VFS in two modes.** The sandbox (julia's own tree, read-only) and host mode
  (the real file system on macOS, Linux and Windows).
- **Dynamic libraries** are the libraries the VM implements in Go.
  libblastrampoline reports julia's configuration. Loading a C library is a stated
  divergence.
- **Portability.** It builds on every Go target and is tested on 386, arm, riscv64,
  s390x, ppc64le and loong64 (floats are bit-exact independently of the FPU), and
  on macOS, Linux and Windows.

It stands on the earlier M3 releases: Julia's type objects, subtyping and method
specificity, and the dispatch of concrete calls (`core`, `core/dispatch`), and the
package `interp`, a port of julia 1.12.6's `src/interpreter.c` (v0.6.0 to v0.12.0:
`struct` and `abstract type` definitions, methods with `where` clauses, macro
expansion, `boot.jl`, arrays). The details are in [Architecture]({{< relref "architecture" >}}).

**Known limits of v0.13.0:**

- `[@async i^2 for i in 1:4]` is correct but takes minutes; this is under
  investigation.
- The stdlibs are still julia's `.jl` sources; rewriting them in Go is
  [#36](https://github.com/go-embedded-julia/julia/issues/36).
- No BLAS kernel computes yet
  ([#33](https://github.com/go-embedded-julia/julia/issues/33)).
- Writing a file in host mode is not ported yet
  ([#32](https://github.com/go-embedded-julia/julia/issues/32)).
- Loading a C library is a stated divergence from julia (see above).

M4 to M7 are planned: the interpreter is therefore not finished, and the pages
that describe them describe a plan. Every statement comes from the
bibliography of the [go-scikit roadmap](https://github.com/go-scikit/roadmap),
[`bib/julia.md`](https://github.com/go-scikit/roadmap/blob/main/bib/julia.md), and
from the release notes of each version.

## What it will be

A Julia interpreter written in pure Go, which can be embedded:

- **Julia's parser and lowering ported to Go.** JuliaSyntax is Julia's parser;
  julia 1.12.6 lowers with its femtolisp front end (`julia-syntax.scm`), which is
  what the `lower` package ports. A Go VM has no femtolisp, so both have to be
  native before any Julia can run.
- **An interpreter over the lowered IR**, which runs **Julia's own Base
  unchanged**. Base is evaluated at build time and the resulting heap is shipped
  as an image.
- **The runtime boundary mapped onto fleet libraries.** Intrinsics and `ccall`
  targets are implemented in Go, with go-bignum, go-ndarray, go-fft and the
  regexp stack standing where Julia calls GMP and MPFR, BLAS and LAPACK, FFTW and
  PCRE2. See [Runtime boundary]({{< relref "runtime-boundary" >}}).

## What it is for

The `julia` cells of [go-notebook](https://go-notebook.github.io/) and of its
WebAssembly playground. It mirrors
[go-embedded-ruby](https://github.com/go-embedded-ruby), whose `rbgo` is the Ruby
VM behind go-notebook's `ruby` cells.

## Honest scope

It is an interpreter. It keeps Julia's semantics, not its native speed; heavy
computation goes through the Go libraries. An interpreter cannot match Julia's JIT
on loops. The comparisons that can hold are start-up and first result, and the
kernels routed to go-ndarray and go-bignum. The plan marks every interpreter speed
statement as unverified, except the parser's (0.98 s against 1.39 s for
`Meta.parseall` on one core, over julia 1.12.6's `share/julia` tree); the ratio
of each later milestone will be stated once it is measured.

## Pages

| Page | What it holds |
| --- | --- |
| [Plan]({{< relref "plan" >}}) | milestones M0 to M7 with their size and exit test, what the playground shows at each, and how conformance is measured |
| [Architecture]({{< relref "architecture" >}}) | the pipeline, the parser (with its usage), the lowering, the runtime in Go, type inference, the Base bootstrap and the image |
| [Kernel]({{< relref "kernel" >}}) | the prebuilt system image pinned by its SHA-256, `FetchKernel`, `Eval` per cell, cancellation, budgets and the sandbox |
| [Runtime boundary]({{< relref "runtime-boundary" >}}) | the counts: intrinsics, builtins, `ccall` sites by target, regex, FFT |
| [Licences]({{< relref "licences" >}}) | what may be translated, what is run or replaced, and the GPL parts avoided |

## Repositories

| Repo | What it is |
| --- | --- |
| [julia](https://github.com/go-embedded-julia/julia) | the interpreter (source) |
| [docs](https://github.com/go-embedded-julia/docs) | this site |
| [go-embedded-julia.github.io](https://github.com/go-embedded-julia/go-embedded-julia.github.io) | the landing page |
| [brand](https://github.com/go-embedded-julia/brand) | the logo, icons and social images |

The interpreter itself is in
[go-embedded-julia/julia](https://github.com/go-embedded-julia/julia).
