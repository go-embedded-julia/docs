---
title: "Plan"
weight: 10
description: "Milestones M0 to M7: contents, size and exit test; what the playground shows at each; how conformance is measured."
---

<!-- From go-scikit/roadmap bib/julia.md, sections 4.2 and 8 (go-scikit/roadmap, main). Change the roadmap first, then here. -->

**Status: in development. M0 (oracle harness), M1 (parser) and M2 (lowering) are
done, released as [v0.3.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.3.0)
(2026-10-28) of
[go-embedded-julia/julia](https://github.com/go-embedded-julia/julia), against
julia 1.12.6.** M3 is in progress (v0.4.0, v0.5.0, v0.6.0, v0.7.0, v0.8.0, v0.9.0, v0.10.0, v0.11.0, v0.12.0); M4 to M7 are planned.

M2 in numbers: the `lower` package ports julia 1.12.6's femtolisp front end
(`julia-syntax.scm`, `macroexpand.scm`, `ast.scm`, `match.scm`).
`lower.NewState().Lower(ex)` is `Meta.lower` of a macro-expanded top-level form,
compared with julia field by field, modulo gensym numbering. It is identical on
**53 997 of 53 997** forms of julia 1.12.6's `share/julia` tree (base, Compiler,
stdlib, test; **1191 of 1191** files), **124 of 124** lowering snippets, **48 of
48** hygienic-scope forms and **477 of 477** built expressions (error paths,
rare heads); `deparse` matches `julia --lisp` on 119 of 119 forms and the
front-end functions on 151 of 151 calls. Statement coverage is 100%. `Lower`
refuses trees deeper than `lower.MaxDepth` (1000) and `FuzzLower` runs on every
pull request. **Known gap:** a `Float64` inside a lowering error message makes
`Lower` return `lower.ErrUnsupported`.

M3 so far: core — Julia's type objects, apply_type, and subtyping (a port of jltypes.c and subtype.c of 1.12.6, without intersection): identical to julia on 20 000 random pairs and 4 902 cases from test/subtype.jl; and method specificity (jl_type_morespecific, v0.4.0), identical to Base.morespecific on the same corpus. core/dispatch (v0.5.0): method tables with world ages, and the method a call with concrete argument types runs, with its static parameters: identical to julia's `which` and dispatch environment on 9 000 calls over 1 500 random method tables (4 675 method choices, 1 968 with static parameters, 87 ambiguities). interp (v0.6.0 to v0.12.0): a port of julia 1.12.6's `src/interpreter.c` that runs lowered code over Core (`interp.New().NewSession().Eval(src)`, `interp.Repr`); v0.7.0 runs `struct` and `abstract type` definitions with julia's default constructors (`jl_ctor_def`) and user-defined outer constructors, and calling a type dispatches on its constructors. v0.8.0 dispatches methods with `where` clauses with static parameters (e.g. `f(x::T) where {T<:Integer} = T`), through the TypeVar and UnionAll constructors of boot.jl:335-338, built in Go until boot.jl runs. v0.9.0 runs structs and abstract types with parameters (`struct Q{T}`, `abstract type A{T}`): `Q(1)` infers `Q{Int64}`, `Q{Integer}(1)` runs the inner constructor, a parameter no field constrains gets no outer constructor (as jl_ctor_def decides), constructor signatures keep the called type first as in julia's one method table, and a type argument dispatches as `Type{X}`. v0.10.0 expands macros, a port of `jl_expand_macros`: a macro call runs the macro's method with `__source__` and `__module__`, hygiene goes through the `hygienic-scope` that `lower` resolves, nested and recursive macro calls are expanded in turn, quotes go through `lower.Backquote`, and a self-expanding macro raises `StackOverflowError`. v0.11.0 defines Core by julia 1.12.6's own `base/boot.jl`, unmodified and embedded: `vm.Boot()` evaluates its 363 forms in Core, then runs `post_boot_hooks` (jltypes.c:3909-3968), so builtins raise boot.jl's exception types and an error in an included file is a `LoadError`. After Boot, `vm.CoreSession().Eval(src)` runs code in Core; Boot is opt-in, and `interp.New().NewSession().Eval` is unchanged. Ported from julia's C sources for it: the types `jl_init_types` creates (with real layouts for the IR nodes, `GenericMemory`, `GenericMemoryRef` and `Array`, plus `AddrSpace` and `Core.CPU`); `module` expressions and relative `using .M` / `import ..M` paths; the builtins `_primitivetype`, `setfield!`, `invokelatest` and `invoke_in_world`; `getfield` of types (DataType, TypeName, TypeVar, UnionAll, Union, Vararg) and of AST nodes; the intrinsics `trunc_int`, `sext_int`, `zext_int`, `bitcast`, `shl_int`, `lshr_int` and `ashr_int`; and `ccall` to Go shims keyed by the C function's name (an unknown name is `ErrUnsupported` when the call runs). Display fixes, checked against julia 1.12.6: Float32 (`1.0f0`, `NaN32`), Char (escapes, malformed bytes, `\u`/`\U`) and nested modules (`Main.L.B`). `include` reads only `VM.Sources` (julia's embedded sources by default), never the host's filesystem, and an include that nests without end raises `StackOverflowError`. v0.12.0 runs julia 1.11+'s arrays: `GenericMemory` and `GenericMemoryRef` values with the builtins `memorynew`, `memoryrefnew`, `memoryrefoffset`, `memoryrefget`, `memoryrefset!` and `memoryref_isassigned` (builtins.c:1711-1856, genericmemory.c), `Array{T,N}(undef, dims…)` as boot.jl builds it, the overflow-checked intrinsics `checked_[su]{add,sub,mul}_int`, arrays and where-clause types shown as julia 1.12.6 shows them, and an Expr's `args` as one `Vector{Any}`; `VM.MaxMemoryElems` bounds one allocation (`interp.ErrMemoryLimit`), the total is not bounded yet ([#24](https://github.com/go-embedded-julia/julia/issues/24)). Not yet: atomic memory, and Base. Speed of arrays: not measured, there is no julia counterpart until programs index arrays through Base. Speed: `Boot` has no julia counterpart to measure against, since julia's Core comes from its system image. Core has the built-in types, the exception and AST types of boot.jl and jltypes.c (with the `QuoteNode`, `LineNumberNode` and `Expr` constructors), 26 builtins (the six type-definition builtins and `fieldtype` are new in v0.7.0, `_typevar` in v0.8.0, `_expr` and `_apply_iterate` in v0.10.0) and the integer and float intrinsics. Checked: 97 Core-only programs show the same value in Go and julia 1.12.6; 100% statement coverage; fuzzed (`FuzzEval`). Not yet: type intersection, and with it the interference sets julia 1.12 uses to order matches; Base (`+`, `println`, `a.b`, and constructor arguments that need `convert`, macros that use `esc` or Base's macros), splatting anything but a tuple or a simple vector (it calls `Base.iterate`), redefining a struct that has parameters (it needs `Base.getproperty`), `const` and atomic fields, arrays, `finally`, and the remaining builtins and intrinsics. Speed: subtyping is 1.37× slower than julia's `jl_subtype` on one core (3.26 ms against 2.39 ms for the 20 000 pairs, best of 7 interleaved rounds, load 4.4–6.6, 2026-11-01), an open defect ([#20](https://github.com/go-embedded-julia/julia/issues/20)); pooling the query environments (v0.9.1) halved the allocations (−51%); the speed of the interpreter against julia's is not measured yet.

M1 in numbers: a Go port of JuliaSyntax `29ec5226f7b1`, the version julia 1.12.6
ships. Tokens, green trees and `Expr` are identical to julia's on **1191 of 1191
files** of julia 1.12.6's `share/julia` tree (base, Compiler, stdlib, test;
18.5 MB) and on **976 of 976** golden snippets. On one core the corpus parses in
0.98 s against 1.39 s for julia's `Meta.parseall` (0.70x). Parsing is bounded
(64 MiB, depth 4000) and fuzzed; coverage is 100%, tested natively on
amd64/arm64/386, under qemu on riscv64/loong64/ppc64le/s390x/arm, compiled for
every GOOS/GOARCH, and run on wasip1 under wazero. **Known gap:** on a few
malformed inputs (`import. ..`, `import. ...x`, `0:0:<`) JuliaSyntax itself
throws and julia falls back to the femtolisp parser, which is not ported: there
`ParseAll` returns `Expr(:error)` and `ParseAllLimits` returns an error wrapping
`syntax.ErrParserFailed`.

Effort is on the roadmap's scale: **S** days, **M** weeks, **L** months, **XL**
several months. M3 and M4 dominate; in total the effort is on the order of the
rbgo effort, plus a parser and a lowering pass that rbgo did not need in this
form.

## Milestones

| # | Milestone | State | Contents | Size | Exit test |
|---|---|---|---|---|---|
| M0 | oracle harness | **done**: [v0.1.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.1.0) (2026-10-26), against julia 1.12.6; `go run ./cmd/jloracle regen` regenerates the goldens | a Go tool that runs the reference `julia` (a pinned release) on a corpus and stores its outputs: parse trees, `Meta.lower`, `Test` summaries, printed values | S | outputs checked in as golden files, regenerated by one command |
| M1 | parser | **done**: [v0.2.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.2.0) (2026-10-26), 1191/1191 files and 976/976 snippets identical | Go port of `JuliaSyntax/src` (11 326 lines) to a Go `Expr`; fuzzed (`go test -fuzz`) with a depth and size bound | L | `Expr`-identical on all 202 `base/` files, `stdlib/`, all 297 `test/` files, and JuliaSyntax's own tests |
| M2 | lowering and macro expansion | **done**: [v0.3.0](https://github.com/go-embedded-julia/julia/releases/tag/v0.3.0) (2026-10-28), 53 997/53 997 corpus forms (1191/1191 files) identical | Go port of julia 1.12.6's femtolisp front end (`julia-syntax.scm`, `macroexpand.scm`, `ast.scm`, `match.scm`) | L | `CodeInfo`-identical (modulo numbering) to `Meta.lower` on Base's top-level forms |
| M3 | Core runtime | in progress: type objects, `apply_type`, subtyping (v0.3.0), specificity (v0.4.0), method tables and dispatch of concrete calls (v0.5.0), interpreter `interp` (v0.6.0), with `struct` and `abstract type` definitions (v0.7.0) methods with `where` clauses (v0.8.0), structs and abstract types with parameters (v0.9.0), macro expansion (v0.10.0), and Core defined by julia's own `boot.jl` (`vm.Boot`, v0.11.0), and julia's arrays (v0.12.0); Base and the remaining builtins and intrinsics next | types, subtyping, method tables, dispatch, the builtins (63 in julia 1.12.6), the intrinsics (93 in julia 1.12.6), the interpreter, `boot.jl` | XL | `test/subtype.jl`, `test/specificity.jl`, `test/intrinsics.jl`, and the parts of `test/core.jl` that need no Base I/O |
| M4 | Base bootstrap and image | planned | `Base_compiler.jl` and `Base.jl` evaluated; the `jl_*`, libc and libuv shim (about 524 + 88 + 78 call sites); the Compiler interpreted for `_return_type`; an image writer and reader | XL | Base loads, and the `test/` files `arrayops`, `strings`, `dict`, `tuple`, `reduce`, `broadcast`, `iterators`, `sorting`, `numbers`, `int`, `math`, `float16`, `ryu`, `show` |
| M5 | numeric boundary | planned | GMP and MPFR on `math/big` and go-bignum, a PCRE2 front end on go-ruby-regexp, utf8proc tables, a dSFMT port, Printf, Dates, Random, Statistics, LinearAlgebra on go-ndarray | L | `test/gmp.jl`, `mpfr.jl`, `regex.jl`, `unicode.jl`, `rational.jl`, `complex.jl`; the standard-library tests of Random (seeded streams byte-identical), Dates, Printf, Statistics and LinearAlgebra (dense) |
| M6 | notebook bridge and go-scikit/julia | planned | the Julia to Go value table, the names a cell defines and uses, errors, worker limits | M | a mixed Go and Julia example notebook transcript in the go-notebook docs |
| M7 | WebAssembly playground | planned | a lazily loaded Julia core, image loading, SVG display through the existing sanitizer | M | first result under the Ruby core's time on the same machine; size recorded |

## What the playground will show

Milestone by milestone, as the plan lists it:

| Milestone | The playground shows |
|---|---|
| M1 parser | a cell's `Expr` tree (`dump`) and parse errors with JuliaSyntax's diagnostics |
| M2 lowering | `Meta.@lower` of a cell: the `CodeInfo` listing |
| M3 Core | arithmetic, `isa`, `typeof`, `<:` queries, user-defined types and methods with dispatch, using Core only |
| M4 Base | `println`, strings, `Vector`/`Dict`/`Set`, comprehensions, broadcasting, `sort`, `sum`, `BigInt` factorials, `r"…"` matches, `@show`, `@time` (wall clock only) |
| M5 numeric stdlib | `LinearAlgebra` (`A \ b`, `eigen`, `svd`), `Statistics` (`mean`, `std`, `quantile`, `cor`), `Random` with seeds identical to julia's output, `Printf`, `Dates` |
| M6 bridge | a mixed Go and Julia notebook; go-scikit bindings (`Stats`, `Special`, `Plot` SVG) |

There is no playground yet.

## Scope of the first version

**In:** all syntax JuliaSyntax parses; functions and multiple dispatch with
parametric methods and `where`; keyword arguments; closures; `struct` and
`mutable struct`; abstract and parametric types; `Union`, `Nothing` and `Missing`;
tuples and named tuples; `try`/`catch`/`finally`; hygienic macros and `@generated`
functions; string interpolation; broadcasting; comprehensions and generators; `do`
blocks; modules; `using` and `import` of in-image modules; single-threaded tasks
and `Channel`. From the standard library: `LinearAlgebra` (dense, over
go-ndarray), `Statistics`, `Random`, `Printf`, `Dates` and `Test`.
`SparseArrays` comes later.

**Out:** `ccall` to arbitrary user libraries (an error naming the symbol),
`llvmcall`, `@cfunction`, parallel `Threads.@threads` (runs serially),
`Distributed`, `Pkg` (packages ship in the image), and precompilation of user
code.

## How conformance is measured

Julia's test suite is written with the `Test` standard library, and
`test/choosetests.jl` lists the test names. The harness:

1. runs each listed file under the reference `julia` and under the interpreter,
   one file per worker process with a timeout, each with a `Test` reporter that
   emits one JSON record per `@test` (file, line, expression, pass, fail, error or
   broken);
2. compares **records, not totals**: a test passing in both, failing only in the
   interpreter (a defect), or failing in both (upstream), so that an equal count
   with different members is not read as parity;
3. keeps a committed allowlist of files out of scope (`precompile`, `loading`,
   `llvmcall*`, `threads`, `ccall` to user libraries, `spawn`, `cmdlineargs`,
   `compiler`, `jit`), each with a reason, and reports the pass rate per
   remaining file;
4. stores every failure found only in the interpreter as a regression test once
   it is fixed.

The differential oracles of each stage (M1 `Expr`, M2 `CodeInfo`, Random streams,
printed `show` output) are stronger than the suite for the front end and are built
first, which is why M0 comes before everything else.

## Where the plan lives

The bibliography is
[`bib/julia.md`](https://github.com/go-scikit/roadmap/blob/main/bib/julia.md) in
[go-scikit/roadmap](https://github.com/go-scikit/roadmap).
