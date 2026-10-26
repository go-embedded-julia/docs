---
title: "Architecture"
weight: 20
description: "The planned pipeline: a Go parser and lowering, a runtime in Go, Base run unchanged through an interpreter, and a pre-built image."
---

<!-- From go-scikit/roadmap bib/julia.md, sections 1, 2, 4.1, 5 and 6 (go-scikit/roadmap, main). Change the roadmap first, then here. Counts are from the sources at the revisions the bibliography names. -->

**Status: planned, with the parser (M1) in development.** This page is the design
the bibliography recommends. None of it is released, and where the bibliography
says a point is unmeasured, this page says so.

## The pipeline

```
source ──JuliaSyntax (Go port)──▶ Expr ──macroexpand──▶ Expr ──JuliaLowering (Go port)──▶ CodeInfo
   ──▶ toplevel eval (toplevel.c semantics) ──▶ method tables ──call──▶ dispatch (gf.c, typemap.c,
       subtype.c) ──▶ interpreter over lowered IR (interpreter.c semantics) ──▶ builtins (83),
       intrinsics (94), foreigncall shim table
```

This is the pipeline Julia's own developer documentation describes (`eval.md`),
minus the code-generation step. Julia already interprets some top-level code
instead of compiling it, and "the bulk of the work to interpret code is handled by
`eval` in `interpreter.c`". The port makes the interpreted branch the only one.

## What Julia is made of

| Part | Path | Lines | Role for a Go port |
|---|---|---|---|
| C/C++ runtime and codegen | `src/*.{c,cpp,h}` | 169 453 | a specification; the LLVM half is dropped |
| its semantic core | `subtype.c`, `gf.c`, `jltypes.c`, `datatype.c`, `builtins.c`, `module.c`, … (15 files) | 36 477 | what must be re-expressed in Go |
| Base, in Julia | `base/**/*.jl` (202 files) | 140 705 | run as is through the interpreter |
| type inference and optimiser | `Compiler/src` (41 files) | 36 867 | not needed for speed; see below |
| parser, in Julia | `JuliaSyntax/src` (18 files) | 11 326 | port to Go |
| lowering, in Julia | `JuliaLowering/src` (17 files) | 13 940 | port to Go |
| legacy front end, femtolisp | `src/*.scm` (11 files) | 10 622 | oracle and fallback |
| JuliaInterpreter | `src/*.jl` (12 files) | 6 943 | design reference |
| test suite | `test/**/*.jl` (297 files) | 130 413 | oracle |

The lowered IR has **12 node types** and **30 `Expr` heads**: that is the whole
instruction set an interpreter must execute. Julia's own C interpreter over it is
1 118 lines.

## Parser: port JuliaSyntax

- JuliaSyntax is the default parser since Julia 1.10, and Base itself loads it.
- **Bootstrapping forces a native parser.** Until Base loads it, Julia parses
  `base/*.jl` with the femtolisp parser in C. A Go VM has no femtolisp, so it needs
  a Go parser before any Julia runs; running JuliaSyntax through the interpreter is
  not an option for the first files.
- JuliaSyntax is a hand-written recursive-descent parser over a token stream with a
  separate green tree, and a module that builds `Expr`. It ports line by line to Go,
  as rbgo's parser was built.
- **Oracle:** the real `JuliaSyntax.parseall(Expr, …)` on every file of `base/`,
  `stdlib/`, `test/` and the JuliaSyntax test corpus, compared as serialised `Expr`
  trees.

## Lowering: port JuliaLowering, keep `julia-syntax.scm` as the oracle

| | `src/julia-syntax.scm` (+ `macroexpand.scm`, `ast.scm`) | `JuliaLowering/src` |
|---|---|---|
| language | femtolisp Scheme, pattern-matching macros | Julia, plain functions over a `SyntaxGraph` |
| size | 5 779 + 733 + 568 lines | 13 940 lines |
| status | **default** | in the system image but off by default |
| provenance | line numbers only | byte-precise |

The recommendation is to **port JuliaLowering**: it is Julia with explicit data
structures, closer to Go than Scheme pattern rewriting; it is MIT; it carries the
provenance a notebook needs for `file:line` errors; and upstream is moving to it.
The risk is that its coverage is not measured: if it lowers much less than Base,
M2 becomes a port of `julia-syntax.scm`. The oracle in all cases is `Meta.lower` on
the reference binary, compared on `CodeInfo` after renumbering slots and SSA values.

## Runtime in Go

| C file (lines) | Go component |
|---|---|
| `jltypes.c` 4 728, `datatype.c` 3 200 | type objects: `DataType`, `UnionAll`, `Union`, `TypeVar`, `Vararg`, layout, type cache; field layout must match for `sizeof`, `fieldoffset`, `reinterpret` |
| `subtype.c` 7 737 | `issubtype`, `type_intersection`, `morespecific`: the hardest part, with `test/subtype.jl` (3 666 lines) and `test/specificity.jl` as oracles |
| `gf.c` 6 567, `typemap.c` 1 795, `method.c` 1 538 | method tables, dispatch cache, world ages, ambiguity |
| `builtins.c` 3 161 | the 83 builtins, each a Go function |
| `runtime_intrinsics.c` 1 862 | the 94 intrinsics |
| `interpreter.c` 1 118, `toplevel.c` 1 020 | the interpreter over `CodeInfo` and top-level eval |
| `module.c` 2 504 | modules, bindings, binding partitions |
| `genericmemory.c` 631, `array.c` 324 | `Memory{T}` and `MemoryRef`; `Array` is defined in Julia on top |
| `task.c`, `partr.c`, `gc-*.c`, `signals-*.c` | a goroutine per Task with a hand-off channel (one runs at a time, as in Julia's single-threaded scheduler); the Go GC; no signals |
| `staticdata.c` 5 103 | the interpreter's own image format; not a port of `sys.ji` |

`subtype.c` is where a reimplementation diverges silently (the diagonal rule,
`Union` distribution, `Vararg`, `TypeVar` bounds). The plan is to port it
statement by statement rather than from the paper.

### The interpreter

Its shape follows JuliaInterpreter: a `Frame` with locals (slots) and SSA values, a
program counter over `CodeInfo.code`, `GotoNode`, `GotoIfNot` and `ReturnNode` as
control flow, and `:enter` and `:leave` as an exception-handler stack.
JuliaInterpreter is the evidence that lowered IR is directly executable for the
whole of Base. Its limit is also the port's: it hands `foreigncall` and `llvmcall`
to compiled wrappers, which is exactly where the Go port needs its shim table
instead.

## Type inference: only where Base's results depend on it

Base calls inference for **results**, not only for speed: `Base._return_type` is
used by `promote_op`, and through it by `map`, `broadcast` and `collect` on empty
inputs and by `accumulate`. `map(x -> x + 1, Int[])` returning `Vector{Int}` and
not `Vector{Any}` is an observable result. Two options:

1. interpret `Compiler/src` (36 867 lines) itself, lazily, only when
   `_return_type` is called: correct by construction, slow;
2. a small Go abstract interpreter over the same lowered IR: fast, and wrong in the
   corners.

The recommendation is option 1 first, with a per-signature cache that is part of the
image; option 2 only if measurement shows the notebook waiting on it, and then
checked against option 1.

## Base: run it unchanged

Base is bootstrapped by evaluating `base/boot.jl` and then the file list of
`base/Base_compiler.jl` and `base/Base.jl`. The recommendation is to **run Base
unchanged through the interpreter, at build time, and ship the resulting heap as an
image**. Only two things are ported to Go: the builtins, intrinsics and `foreigncall`
targets; and hot leaf methods after profiling, replaced by method overrides
registered in Go (same signature, same results, a test comparing both). Base is
140 705 lines whose semantics are the definition of Julia: a native rewrite would
diverge, and every Base fix upstream would have to be ported twice.

## The image is the start-up

Julia without a system image evaluates `boot.jl` and all of Base at start. Through an
interpreter that is 140 705 lines, plus the 36 867-line Compiler if inference is
interpreted: minutes, not seconds. So the build runs the bootstrap once, natively,
and writes the resulting heap (modules, bindings, types, method tables, lowered
`CodeInfo`, interned symbols, the `_return_type` cache, constant data such as
Unicode tables) to a versioned binary image, embedded with `go:embed` or fetched
next to the WebAssembly module. This is Julia's own design (`sys.ji`) without native
code.

The image size is unknown until M4. The start-up budget to hold is that image
deserialisation stays under the Ruby core's 2.3 s on the same machine (a figure the
plan reports but did not re-measure); per-module images (LinearAlgebra only when
`using LinearAlgebra` runs) are the lever if it is not met.

## The notebook bridge

The Ruby cells of go-notebook define the contract to copy: one shared interpreter
for all cells of the language; only **variables** cross; a cell defines the
top-level names it assigns and uses the names it reads first, taken from the
language's own parser; errors are reported at the notebook line. The plan proposes
a Julia to Go value table (integers at their width, `BigInt` as `*big.Int`,
`BigFloat` as `*big.Float`, `String` as `string`, arrays as `*ndarray.Array`
sharing the same buffer with column-major strides kept) and a `go-scikit/julia`
package modelled on `go-scikit/ruby`. Julia is column-major and go-ndarray is
NumPy-style strided, so go-ndarray must accept Fortran strides without copying: the
plan lists that as unconfirmed.

## Risks the plan names

- **Subtyping and specificity** diverging silently from `subtype.c`.
- **JuliaLowering's coverage** of Base, not yet measured.
- **Interpretive speed:** Base's generic code relies on inlining and specialisation.
  Mitigations are dispatch caches keyed by concrete argument types, fast paths for
  intrinsic-only methods, and Go overrides for measured hot leaves.
- **Pointers and `unsafe_*`:** a pointer must be modelled as a buffer and an offset
  into Go memory, checked on every access, so that a malformed `unsafe_load`
  raises instead of corrupting the VM.
- **Security:** notebook and playground cells are untrusted. The plan fuzzes the
  parser and the image reader, and bounds recursion depth, allocation sizes and
  regex steps.
- **Unicode version** must match utf8proc's, or `test/unicode.jl` diverges.
- **Upstream moves fast:** the sources were read at `1.15.0-DEV`; the plan pins a
  release and tracks the next one deliberately.
