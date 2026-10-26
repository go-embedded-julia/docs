---
title: "Licences"
weight: 40
description: "Julia is MIT and may be translated to Go; GMP and MPFR are replaced, not linked; the GPL parts of SuiteSparse are avoided."
---

<!-- From go-scikit/roadmap bib/julia.md, section 3.5 (pull request 12). The licence headers were read in the clones at the revisions the bibliography names. Change the roadmap first, then here. -->

**Status: planned.** No code has been translated yet. This page records what the
plan allows, and the rules it will hold to.

## What may be translated to Go, with the notice kept

Julia is MIT, and Julia's `THIRDPARTY.md` states that the language "consists of the
compiler (the contents of `src/`), most of the standard library (`base/` and
`stdlib/`)". So these are all MIT and may be translated to Go with the notice kept:

| Source | Licence |
|---|---|
| `src/` (the C runtime) | MIT |
| `base/`, `stdlib/` | MIT |
| `JuliaSyntax/` | MIT |
| `JuliaLowering/` | MIT |
| JuliaInterpreter.jl | MIT |
| LinearAlgebra.jl, Statistics.jl | MIT |
| `src/flisp` (femtolisp) | BSD-3-Clause |
| dSFMT | BSD-3-Clause; may be translated |

## Inside Base, with their own licence

`base/ryu/*` is under the Boost licence, and `base/special/rem_pio2.jl` and
`base/special/hyperbolic.jl` carry an FDLIBM notice. Both run as Julia through the
interpreter, and the notices travel with the image.

## Libraries Base calls: replaced, not linked

| Library | Licence | What the plan does |
|---|---|---|
| **GMP** | LGPL-3.0-or-later or GPL-2.0-or-later | not linked, not translated: replaced by `math/big` and go-bignum, written from the documented semantics, not from its source |
| **MPFR** | LGPL-3.0-or-later | the same |
| PCRE | BSD-3-Clause | replaced by a front end on go-ruby-regexp |
| utf8proc | MIT | replaced by Go's `unicode` and `x/text/unicode/norm` |
| libuv | MIT | replaced by Go's `os`, `net` and `time` |
| OpenBLAS and LAPACK | BSD-3-Clause | replaced by go-ndarray's `linalg` |

## The GPL parts to avoid

SuiteSparse's `libumfpack`, `libspqr` and `librbio` are GPL-2.0-or-later, and
`libcholmod` is LGPL-2.1-or-later and GPL-2.0-or-later. They back `SparseArrays`,
an external standard library, whose `\` on sparse matrices goes to UMFPACK and
CHOLMOD. A Go port of SparseArrays must use the fleet's sparse solvers (go-sparse)
and **must not read SuiteSparse source**. Julia's stock build includes these
libraries (`USE_GPL_LIBS ?= 1` in `Make.inc`), so they are a thing to keep out, not
a thing that is absent.

FFTW is GPL but is not in Julia's tree: FFTW.jl is a registry package, and the plan
serves its API over go-fft instead.
