# Step 1.3: the Bitwuzla encoder (`marshal_bitwuzla`)

On branch `bitwuzla-support`, on top of `fa82ad3e`. Committed 2026-10-22 as
`0efbdadc` and pushed to `MidVC/seahorn`.

**Update 2026-10-29:** this code was reverted (`7db2c77b`), and Phase 1
restarts from a smallest working example. The observations below about
Bitwuzla 0.9.1 still hold. See
`2026-10-29-revert-encoder-and-adapter.md`.

Files:
- **New:**
  - `include/seahorn/Expr/Smt/MarshalBitwuzla.hh`
  - `lib/seahorn/Smt/MarshalBitwuzla.cc` (641 lines)
  - `units/units_bitwuzla.cpp`
- **Changed:**
  - `lib/seahorn/Smt/CMakeLists.txt`: the source is added to `SeaSmt`
    with `-fexceptions`.
  - `units/CMakeLists.txt`: a `units_bitwuzla` / `test_bitwuzla` target,
    only when Bitwuzla is found.

The encoder is not used by `seahorn` yet. Wiring it into `SolverBmc` is
step 1.4.

## What Bitwuzla 0.9.1 accepts (probe, 2026-10-22)

[OBS 2026-10-22] I ran a probe program against `$HOME/opt/bitwuzla-0.9.1`.
Sources: `scratchpad/bzprobe/probe{,2}.cpp` (session scratch, not kept).
Results:
- **A lambda has a function sort,** not an array sort.
  - `EQUAL(array, lambda)` throws: "terms with mismatching sort".
  - `ARRAY_SELECT(lambda, i)` and `APPLY(array, i)` both throw.
- **Equality between functions is not supported.**
  - Bitwuzla prints "[bzla] warning: Equalities over functions not yet
    supported" and returns `unknown`. This happens for lambda = lambda,
    for `DISTINCT` of two UFs, and for UF = lambda under an implication
    or asserted twice.
  - A single top-level UF = lambda gives `sat`, apparently because
    Bitwuzla treats it as a definition.
- **`ITE(c, lambda1, lambda2)` crashes.** Building the term works, but
  `check_sat` then aborts the process with an uncaught internal
  `bzla::ComputeValueException`. `ITE` of two UF constants is fine.
- **Nested lambdas are merged.** `(lambda x (lambda y b))` gets the sort
  `bv8 bv8 -> bv8`, a single function of two arguments.
- **An unbound variable is silently accepted.** Asserting a formula with
  an unbound variable gives `sat`, with no error.
- **`mk_bv_value` limits:**
  - with base 10, it accepts `-1` (stored as two's complement);
  - it rejects a value of 2^w or more, e.g. "256 does not fit into a
    bit-vector of size 8";
  - it handles a 128-bit value (2^100 − 1) correctly.
- **Arity of `mk_term`:**
  - n-ary, folded left: `BV_ADD`, `BV_MUL`, `BV_AND`, `BV_OR`, `BV_XOR`,
    `BV_CONCAT`, `XOR`;
  - `IMPLIES` folds right; n-ary `EQUAL` becomes a chain;
  - exactly 2 arguments: `BV_SUB`, `BV_NAND`, `BV_UDIV`.
- **Other kinds:** `BV_REDAND` gives `bv1`. `BV_ROLI` accepts an amount
  larger than the width.
- **Quantified extensional equality works on a small case.**
  `c => (forall y. b[y] = lam(y))` together with `b[p] != v` gives
  `sat`, and adding `c` gives `unsat`. Each check took under 2 ms.

## How SeaHorn's lambdas reach the solver

[FACT] In `--horn-bv2-lambdas` mode:
- **Memory values are one-variable lambdas** whose bodies use `bvar 0`.
  Reads are `fapp(mem, addr)`, and an array constant is wrapped as
  `lambda a. select(A, a)`.
       → seahorn/lib/seahorn/BvOpSem2MemRepr.cc:113-179, 699-707
- **The default PHI encoding asserts an equality with a lambda.**
  `defPHINodesEq` is the default, because `--horn-vcgen-use-ite` is
  false. It asserts `edge_j => (newPhi = val_j)` with a fresh `newPhi`.
  For memory, `val_j` is a lambda, so `EQ(memory, lambda)` reaches the
  solver.
       → seahorn/lib/seahorn/VCGen.cc:23-25, 287-305, 381-384
- **Equalities between two lambdas are expected.** `addDef` warns that
  they are outside the decidable fragment (seahorn issue #467).
       → seahorn/include/seahorn/OperationalSemantics.hh:127-136

[FACT] SeaHorn's de Bruijn indices follow Z3's convention: index 0 is the
last variable of the binder.
- **One difference from Z3:** SeaHorn's `abs` and `sub` shift by 1
  when entering a nested binder, whatever that binder's arity. Z3 shifts
  by the nested binder's number of variables.
- **The two agree whenever the nested binders have one variable,** which
  is all that opsem builds.
- **The encoder follows Z3,** because Z3 is what SeaHorn's formulas have
  been checked against so far.

       → seahorn/include/seahorn/Expr/ExprOpBinder.hh:393 (index n-1 to the first
         variable), 426 and 485 (`cachedVisitor(m_offset + 1)` in AbsCst and
         SubBnd); Z3: ExprToZ.def:481-485 (`Z3_mk_bound(bvarId)`)

[FACT] `bind::abs<LAMBDA>(x, body)` returns `body` unchanged when `x`
does not occur in it. So a constant lambda needs `mk<LAMBDA>(decl, v)`,
which is how `FilledMemory` builds one.
       → seahorn/include/seahorn/Expr/ExprOpBinder.hh:214-216;
         BvOpSem2MemRepr.cc:768-775

## Design

[FACT] The `marshal_bitwuzla` class (commit `0efbdadc`):
- **It is an object, not a set of static functions as in Yices.** It
  holds a `TermManager&`, a cache of closed terms, a stack of binder
  scopes, and the list of declarations it has created.
       → seahorn/include/seahorn/Expr/Smt/MarshalBitwuzla.hh
- **It walks the Expr DAG with an explicit stack.** A 100 000-deep `bvadd`
  chain is encoded and checked in the unit tests.
       → seahorn/lib/seahorn/Smt/MarshalBitwuzla.cc:561-641
- **It tracks free bound variables for each term.** For each term it
  records `fv`: one more than the largest de Bruijn index that is free in
  it. Terms with `fv = 0` go into the global cache. Open terms go only
  into the innermost binder's scope, and only that scope is searched for
  them. The same open Expr means a different term under another binder,
  and Bitwuzla would not catch a leaked variable.
       → MarshalBitwuzla.cc:103-129, 189-217
- **It chooses select or apply by sort** (`mkApp`). `SELECT` and `FAPP`
  become `ARRAY_SELECT` on an array and `APPLY` on a function, because
  SeaHorn uses both on both. `STORE` on a function becomes
  `lambda x. ite(x = i, v, f(x))`.
       → MarshalBitwuzla.cc:259-316
- **Equality with a function is stated pointwise:**
  `forall xs. a(xs) = b(xs)`. This is exact (extensional) and covers
  `EQ(memory, lambda)` from the PHI encoding. It is logged under
  `--log=bitwuzla`.
       → MarshalBitwuzla.cc:276-291
- **An ITE over functions is pushed inside a lambda,** the same way
  `bind::lite` does it, which avoids the crash.
       → MarshalBitwuzla.cc:293-304
- **Overflow predicates use the encoding from the plan.** Z3 splits signed
  overflow by direction, while Bitwuzla's `*_OVERFLOW` kinds cover both
  directions.
       → MarshalBitwuzla.cc:434-466
- **Numerals are reduced modulo 2^w first,** then passed as decimal
  strings.
- **Every rejection is an `UnsupportedExpr`** carrying the offending
  sub-expression, including any `bitwuzla::Exception` raised while a term
  is built. A constant whose sort is unsupported is reported as the
  constant, not as the sort. The scope stack is cleared on any exception,
  so the encoder stays usable.
- **Rejected inputs:**
  - integers and reals (sorts, numerals, arithmetic and comparisons);
  - `INT2BV` and `BV2INT`;
  - `BEXT_ROTATE_*`: nothing produces them, and the argument order is
    unclear;
  - `ARRAY_DEFAULT`, `ARRAY_MAP`, `AS_ARRAY`;
  - structs and finite maps;
  - a lambda whose body is a function (Bitwuzla would merge them);
  - a free bound variable.
- **`BREPEAT` and `BROTATE_*` are mapped,** although nothing in SeaHorn
  produces them.

[FACT] Exceptions and the build:
- Bitwuzla reports errors with C++ exceptions.
- Homebrew's LLVM 18 has `LLVM_ENABLE_EH ON`. An LLVM built without it
  makes `add_llvm_library` add `-fno-exceptions`; SeaHorn already
  handles that case for Boost (`CMakeLists.txt:295-296`).
- So `MarshalBitwuzla.cc` and `units_bitwuzla` get `-fexceptions`. The
  compile line shows it last, after all other flags.
- No exception may leave these files. The 1.4 adapter must catch
  `UnsupportedExpr` and `bitwuzla::Exception` at the `Solver` API, and
  `BitwuzlaSolverImpl.cc` needs the same flag.

       → seahorn/lib/seahorn/Smt/CMakeLists.txt, units/CMakeLists.txt

[OBS 2026-10-22] `mach/boolean.h` (macOS SDK) defines `TRUE` as 1 and
`FALSE` as 0. doctest includes it only in the file that defines
`DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN`, which then breaks `mk<TRUE>`. Other
SeaHorn unit files are not affected, because their `main` is in
`units_z3.cpp`. `units_bitwuzla.cpp` undefines both macros after the
doctest include.

## Acceptance

[OBS 2026-10-22] `seahorn/build/units/units_bitwuzla`: 12/12 test cases
and 2139 assertions pass, in about 1.1 s:
- **Every bit-vector operator against Z3.** All 3-bit argument pairs for
  28 binary operators (including `=`, `distinct` and `concat`) and the
  4 unary ones. The test asks Z3, through the existing `EZ3` path, for
  the value, then asserts `t != value` in Bitwuzla and expects `unsat`.
  Division and remainder by zero are included.
- **Overflow predicates.** All 9, with every argument pair at widths
  1-4: 3060 cases. Bitwuzla and Z3 both match exact integer arithmetic,
  with 0 disagreements. This settles the plan's question about Z3's
  `usub` and `smul` semantics: both are the exact "result in range"
  predicate.
- **Numerals and bit-vector structure:**
  - 128-bit values; negative and too-large values;
  - extract, sign and zero extension, concat order (most significant
    first), repeat, rotate;
  - the n-ary `bvadd` and `bvand`.
- **Core operators:** Boolean operators, arrays (including
  extensionality and constant arrays), and uninterpreted functions of
  two arguments.
- **Lambdas:**
  - application, and `select` on a lambda;
  - `store` into a lambda;
  - an ITE between lambdas, and between an array and a lambda;
  - array = lambda and lambda = lambda under an implication, with both
    sat and unsat results;
  - one `bvar 0` shared by two lambdas;
  - a lambda of two variables (checks the index order);
  - `forall` inside a lambda, and top-level `forall`/`exists`.
- **Rejections** carry the right culprit, and the encoder works again
  after a rejection inside a binder.

Log: `scratch/units-1.3.log`.

[OBS 2026-10-22] The tests catch deliberate bugs. I changed the encoder
three ways, rebuilt, and ran the tests each time; then I restored the
file (`cmp` identical) and reran (12/12). Each change made exactly one
test case fail:
- the direction of `sadd` overflow flipped;
- bound variables taken from the front instead of the back;
- open terms cached globally.

[OBS 2026-10-22] Both trees build and install:
- `build/`: `MarshalBitwuzla.cc` is compiled with `-fexceptions`.
- `build-clam/`: Bitwuzla is off, so the file compiles to nothing and
  there is no `units_bitwuzla` target.
- `scratch/test_selector_1.2.sh` passes 5/5 and 8/8.

`seahorn` behaviour is unchanged, because nothing calls the encoder yet.

## Open for later steps

- [HYP] `EQ(memory, lambda)` from the default PHI encoding now becomes a
  quantifier for each memory PHI. Bitwuzla's quantifier procedure may be
  slow, or give `unknown`, on real opsem formulas. Step 1.7 will show it.
  If it does, a fallback is `--horn-vcgen-use-ite`, which builds no
  equalities.
- The model side (decoding Bitwuzla values back to Exprs) is step 1.5.
  `decls()` keeps the list of declared constants for `print`.

## See also

- ../../durable/generic-solver-interface-contract.md
- ../../durable/adding-a-solver-backend-touches-these-files.md
- 2026-10-22-bitwuzla-smart-detection.md (the build side)
