---
title: "Architecture"
weight: 20
description: "The planned pipeline: a Go parser and lowering, a runtime in Go, Base run unchanged through an interpreter, and a pre-built image."
---

<!-- From go-scikit/roadmap bib/julia.md, sections 1, 2, 4.1, 5 and 6 (go-scikit/roadmap, main). Change the roadmap first, then here. Counts are from the sources at the revisions the bibliography names. -->

**Status: the parser (M1) is released in v0.2.0, the lowering (M2) and the
type system of `core` in v0.3.0; the rest is planned.** This page is the design
the bibliography recommends. Where the bibliography says a point is unmeasured,
this page says so. The lowering section below records what was built.

## The pipeline

```
source ──JuliaSyntax (Go port)──▶ Expr ──macroexpand──▶ Expr ──JuliaLowering (Go port)──▶ CodeInfo
   ──▶ toplevel eval (toplevel.c semantics) ──▶ method tables ──call──▶ dispatch (gf.c, typemap.c,
       subtype.c) ──▶ interpreter over lowered IR (interpreter.c semantics) ──▶ builtins (63 in
       julia 1.12.6), intrinsics (93 in julia 1.12.6), foreigncall shim table
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

### Parser: usage (v0.2.0)

`syntax.ParseAll(src, filename)` in
[go-embedded-julia/julia](https://github.com/go-embedded-julia/julia) returns the
same `Expr` as julia's `Meta.parseall`, including julia's error `Expr`:
the statements before the first error, then `Expr(:error)` or `Expr(:incomplete)`
with the formatted `ParseError`.

```go
e := syntax.ParseAll([]byte("f(x) = x^2 + 1"), "cell.jl")
```

serialises to

```
(:toplevel {L 1 :"cell.jl"} (:"=" (:call :f :x) (:block {L 1 :"cell.jl"} (:call :"+" (:call :"^" :x 2) 1))))
```

Parsing is bounded (64 MiB, depth 4000) and `ParseAllLimits` returns an error
rather than an `Expr` on failure. Known gap: on a few malformed
inputs JuliaSyntax itself throws and julia falls back to femtolisp, which is not
ported; there `ParseAll` returns `Expr(:error)` and `ParseAllLimits` an error
wrapping `syntax.ErrParserFailed`.

## Lowering: what v0.3.0 built (M2)

The risk below came true in the sense the bibliography allowed: julia 1.12.6
lowers with the femtolisp front end by default and JuliaLowering is not part of
the reference's `Meta.lower`, so M2 is a port of `julia-syntax.scm`, the code
that produces the goldens. The `lower` package ports `julia-syntax.scm`,
`macroexpand.scm`, `ast.scm` and `match.scm`, pass by pass: `expand-forms`,
`resolve-scopes`, `analyze-variables!`, `closure-convert`, `linearize`, then
renumbering into `CodeInfo`. Hygiene (`resolve-expansion-vars`,
`rename-symbolic-labels`) is ported too.

`lower.NewState().Lower(ex)` is `Meta.lower` of a macro-expanded top-level form:
an `Expr(:thunk, CodeInfo)`, an atom, or `Expr(:error)`; one `State` per module,
in file order. Macro expansion runs user code, so it stays with the caller, as in
julia. Outputs are compared with julia's field by field, modulo gensym numbering
(the counters live for a whole julia session, so two sessions differ).

| measured on | forms | identical to julia |
|---|---|---|
| corpus (`share/julia`: base, Compiler, stdlib, test), 1191 files | 53 997 | 53 997 (1191/1191 files) |
| lowering snippets | 124 | 124 |
| macro-made toplevels still in hygienic scopes | 48 | 48 |
| built expressions (error paths, rare heads) | 477 | 477 |
| `deparse` vs `julia --lisp` | 119 | 119 |
| front-end functions vs `julia --lisp` | 151 | 151 |

Statement coverage is 100%, every expectation coming from julia. `Lower` refuses
trees deeper than `lower.MaxDepth` (1000) with `lower.ErrLimit`, and `FuzzLower`
runs on every pull request. **Known gap:** a `Float64` inside a lowering error
message makes `Lower` return `lower.ErrUnsupported`.

## Lowering: the design as planned, JuliaLowering vs `julia-syntax.scm`

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
| `builtins.c` 3 161 | the builtins (63 in julia 1.12.6; 83 on 1.15-DEV, where the line counts were taken), each a Go function |
| `runtime_intrinsics.c` 1 862 | the intrinsics (93 in julia 1.12.6; 94 on 1.15-DEV, where the line counts were taken) |
| `interpreter.c` 1 118, `toplevel.c` 1 020 | the interpreter over `CodeInfo` and top-level eval |
| `module.c` 2 504 | modules, bindings, binding partitions |
| `genericmemory.c` 631, `array.c` 324 | `Memory{T}` and `MemoryRef`; `Array` is defined in Julia on top |
| `task.c`, `partr.c`, `gc-*.c`, `signals-*.c` | a goroutine per Task with a hand-off channel (one runs at a time, as in Julia's single-threaded scheduler); the Go GC; no signals |
| `staticdata.c` 5 103 | the interpreter's own image format; not a port of `sys.ji` |

`subtype.c` is where a reimplementation diverges silently (the diagonal rule,
`Union` distribution, `Vararg`, `TypeVar` bounds). The plan is to port it
statement by statement rather than from the paper.

### Released so far: the `core` package (v0.3.0)

core — Julia's type objects, apply_type, and subtyping (a port of jltypes.c and subtype.c of 1.12.6, without intersection): identical to julia on 20 000 random pairs and 4 902 cases from test/subtype.jl.

`FuzzSubtype` runs on every pull request. Subtyping is 1.49× slower than julia's `jl_subtype` on one core (open defect).

### Released so far: the `interp` package (v0.6.0)

interp — a port of julia 1.12.6's `src/interpreter.c` that runs lowered code over Core. `interp.New().NewSession().Eval(src)` parses, lowers and runs a cell; `interp.Repr` shows the value as julia does. Core has the built-in types, boot.jl's exception types, 16 builtins and the integer and float intrinsics. 55 Core-only programs show the same value in Go and julia 1.12.6; 100% statement coverage; fuzzed (`FuzzEval`). Not yet: Base (`+`, `println`, `a.b`), macro expansion, boot.jl constructors and `where`, structs, arrays, `finally`; the speed against julia's interpreter is not measured yet.

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
