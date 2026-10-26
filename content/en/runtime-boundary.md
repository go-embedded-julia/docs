---
title: "Runtime boundary"
weight: 30
description: "What the interpreter has to provide below Base: 94 intrinsics, 83 builtins, 896 ccall sites by target, regex and FFT. The counts."
---

<!-- From go-scikit/roadmap bib/julia.md, section 3 (go-scikit/roadmap, main). The counts were made by a Go program over the Julia clones at the revisions the bibliography names, not by grep and not estimated. Change the roadmap first, then here. -->

**Status: planned (milestones M3 to M5).** This page counts what Julia's Base
asks of its runtime, and says which fleet library is meant to answer each part.
Nothing here is implemented yet.

## Intrinsics: 94

From `src/intrinsics.h`: 81 `ADD_I`, 12 `ALIAS` and 1 `ADD_HIDDEN`. By group:

| Group | Count |
|---|---|
| wrap (`bitcast`) | 1 |
| arithmetic | 17 |
| fast arithmetic (aliases) | 7 |
| comparisons | 15 |
| bitwise | 11 |
| conversion | 9 |
| checked arithmetic | 10 |
| functions (`abs_float`, `copysign_float`, `sqrt_llvm`, …) | 9 |
| pointer arithmetic | 2 |
| pointer access | 2 |
| pointer atomics | 6 |
| C interface (`cglobal`, `llvmcall`) | 2 |
| CPU tests (`have_fma`, `cpu_supports`) | 2 |
| hidden (`cglobal_auto`) | 1 |

All except `llvmcall` and the pointer group map to Go in a few lines each, with
`math/bits` and `math`. `Int128` needs a two-word type, and Float16 and BFloat16 are
converted in software. `muladd_float` is computed as a fused multiply-add, and that
choice is documented. `pointerref`, `pointerset` and the atomics work on the
Go-side byte storage of `Memory` and `Ref`: there is no raw address space.
`llvmcall` is not supported: Base has **0** `llvmcall(` sites.

## Builtins: 83

From `src/builtin_proto.h`: `getfield`, `setfield!`, `isa`, `===`, `<:`,
`apply_type`, `_apply_iterate`, `invoke`, `memorynew`, the twelve `memoryref*`, the
fourteen `*global*`, `_structtype`, `_typebody!` and the rest. Each is one Go
function.

## `ccall` and `@ccall` in `base/`: 896 call sites in 202 files

856 `ccall` and 40 `@ccall`, plus 24 `cglobal(`. By target:

| Target | Sites | Go implementation |
|---|---|---|
| Julia runtime `jl_*`, `ios_*` | 524 (387 unique names) | **is** the Go runtime: reflection, `IdDict`, strings, files, GC; no-ops or errors for compiler, precompile and CPU-feature calls |
| libc and OS symbols | 88 | Go `os`, `syscall` and `time`; `memcpy` and friends on Go slices; `malloc` and `free` on a Go arena |
| **MPFR** | 81 static, plus 29 generated | go-bignum's `bigfloat` and `math/big.Float`; `mpfr_asprintf` and `mpfr_snprintf` need a Go formatter |
| libuv `uv_*` | 78 | Go `os`, `os/exec`, `net` and `time.Timer`; the event loop becomes goroutines that wake the scheduler |
| **GMP** | 40 named, plus 3 interpolated | `math/big.Int` and `big.Rat`, following GMP's truncating, floor and ceiling rounding for quotients; go-bignum for factorial and binomial |
| other dynamic (pointer, `@static`, interpolated) | 7 | resolved case by case |
| **PCRE2** | 18 | see Regex below |
| utf8proc | 15 | Go `unicode` and `golang.org/x/text/unicode/norm`; width and grapheme tables generated from the Unicode version pinned by utf8proc |
| Windows-only and macOS `libproc` | 13 | not on WebAssembly; `GOOS=windows` later |
| **libm** | **0** | nothing: Base's elementary functions are written in Julia over intrinsics, so results are bit-identical by construction |

The standard libraries read (Random, Dates, Printf, Statistics) have 14 sites: 7 on
`libdSFMT` in Random (the `MersenneTwister` generator; the default `Xoshiro` is pure
Julia), 3 on MPFR (`mpfr_snprintf` in Printf, two `mpfr_sub_*` in Random), 1 on libuv
in Dates, and 1 on the runtime. Statistics has **0**.

### LinearAlgebra: 194 sites

182 call BLAS or LAPACK through `libblastrampoline`, inside loops over element types
(`blas.jl` 38, `lapack.jl` 144: 105 distinct callee expressions), 9 sites over 8
`lbt_*` names, and 3 `sysctlbyname`. The mapping is to implement the Fortran BLAS
and LAPACK interface of each routine family as Go functions over go-ndarray's
`linalg` (LU, QR, Cholesky, SVD, Eig, Eigh and least squares exist there), on the
column-major `Memory{Float64}` buffer a Julia `Matrix` owns, without copying.
Families go-ndarray lacks (generalized Schur, banded and tridiagonal
factorisations, Bunch-Kaufman) are missing kernels to add **to go-ndarray**, not
to the VM.

## Regex

`base/regex.jl` has PCRE2 semantics. The fleet's engine is go-ruby-regexp, a pure-Go
backtracking VM reimplementing Onigmo. PCRE2 and Onigmo differ in syntax corners
(`\K`, `(*VERB)`s, callouts, variable-length lookbehind, `\h`, option letters). The
plan is a PCRE2 syntax front end that compiles to the same VM, with `test/regex.jl`
as the oracle and the 18 `pcre2_*` names implemented on it. If the VM cannot take a
construct, the interpreter raises Julia's `ErrorException("PCRE compilation error:
…")` rather than match differently.

## FFT

Julia's Base and standard library have **no FFT** (no FFTW in `base/`, `stdlib/` or
`deps/`; FFTW.jl is a registry package). A `go-julia-fftw` package presenting the
FFTW.jl and AbstractFFTs.jl API over go-fft is the port, and it is not part of the
VM.

## Naming of the packages around the VM

The plan's naming mirrors go-embedded-ruby: `go-embedded-julia/julia` for the VM,
with `docs`, the landing and `brand`; `go-julia-<package>` organisations for ports of
Julia packages that are not part of the image's standard library (FFTW, SparseArrays,
DataFrames, Distributions, SpecialFunctions); and `go-scikit/julia` for the
go-scikit bindings. A `go-julia-*` package is Julia code run by the VM plus Go
kernels imported from the go-scikit libraries, never a second Go implementation of
the same algorithm.
