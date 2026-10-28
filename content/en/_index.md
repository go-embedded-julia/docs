---
title: "go-embedded-julia documentation"
linkTitle: "Home"
type: docs
cascade:
  type: docs
description: "A pure-Go, embeddable Julia interpreter, in development: v0.3.0 completes M0 (oracle harness), M1 (parser) and M2 (lowering); M3 (Core runtime) is in progress."
---

**go-embedded-julia is in development. [v0.3.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.3.0)
(2026-10-28) completes milestone M0 (the oracle harness, against julia 1.12.6),
milestone M1 (the parser): a Go port of JuliaSyntax, `Expr`-identical to
`Meta.parseall` on 1191 of 1191 files of julia's own tree, and milestone M2
(lowering): a Go port of julia 1.12.6's femtolisp front end, identical to
`Meta.lower` on 53 997 of 53 997 forms of that tree.** Milestone M3 (the Core
runtime) is in progress. The interpreter itself does not exist yet: apart from
the parser, the lowering and the type system of the `core` package, these pages
describe a plan, not a finished interpreter. Every statement comes from the
bibliography of the [go-scikit roadmap](https://github.com/go-scikit/roadmap),
[`bib/julia.md`](https://github.com/go-scikit/roadmap/blob/main/bib/julia.md).

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
