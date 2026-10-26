---
title: "go-embedded-julia documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A pure-Go, embeddable Julia interpreter, in development: milestones M0 (oracle harness) and M1 (parser)."
---

**go-embedded-julia is in development, at milestone M0 (the oracle harness) and
milestone M1 (the parser).** Nothing is released: no tag exists, and these pages
describe a plan, not a finished interpreter. Every statement comes from the
bibliography of the [go-scikit roadmap](https://github.com/go-scikit/roadmap),
[`bib/julia.md`](https://github.com/go-scikit/roadmap/blob/main/bib/julia.md).

## What it will be

A Julia interpreter written in pure Go, which can be embedded:

- **JuliaSyntax and JuliaLowering ported to Go.** JuliaSyntax is Julia's parser
  and JuliaLowering its lowering pass. A Go VM has no femtolisp, so the parser has
  to be native before any Julia can run.
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
statement as unverified: nothing has been run yet, and the ratio will be stated
per milestone once it is measured.

## Pages

| Page | What it holds |
| --- | --- |
| [Plan]({{< relref "plan" >}}) | milestones M0 to M7 with their size and exit test, what the playground shows at each, and how conformance is measured |
| [Architecture]({{< relref "architecture" >}}) | the pipeline, the parser, the lowering, the runtime in Go, type inference, the Base bootstrap and the image |
| [Runtime boundary]({{< relref "runtime-boundary" >}}) | the counts: intrinsics, builtins, `ccall` sites by target, regex, FFT |
| [Licences]({{< relref "licences" >}}) | what may be translated, what is run or replaced, and the GPL parts avoided |

## Repositories

| Repo | What it is |
| --- | --- |
| [docs](https://github.com/go-embedded-julia/docs) | this site |
| [go-embedded-julia.github.io](https://github.com/go-embedded-julia/go-embedded-julia.github.io) | the landing page |
| [brand](https://github.com/go-embedded-julia/brand) | the logo, icons and social images |

The interpreter's own repository is created separately and is not described here
until it exists.
