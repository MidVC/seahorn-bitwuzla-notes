# Step 1.4: the Bitwuzla solver adapter (`bitwuzla_solver_impl`)

On branch `bitwuzla-support`, on top of `0efbdadc`. Committed 2026-10-22 as
`22c723b5` and pushed to `MidVC/seahorn`.

**Update 2026-10-29:** this code was reverted (`654c3098`), and Phase 1
restarts from a smallest working example. The suite numbers below are now
the target for the new plan's 1.9. See
`2026-10-29-revert-encoder-and-adapter.md`.

Files:
- **New:**
  - `include/seahorn/Expr/Smt/BitwuzlaSolverImpl.hh` and
    `lib/seahorn/Smt/BitwuzlaSolverImpl.cc`;
  - `include/seahorn/Expr/Smt/BitwuzlaModelImpl.hh` and
    `lib/seahorn/Smt/BitwuzlaModelImpl.cc`.
- **Changed:**
  - `include/seahorn/SolverBmc.hh` includes the adapter under
    `WITH_BITWUZLA`.
  - `lib/seahorn/SolverBmc.cc:46-48` creates it, replacing the 1.2 "not
    implemented yet" exit.
  - `lib/seahorn/Smt/CMakeLists.txt`: the two files are added to `SeaSmt`
    with `-fexceptions`.
  - `units/units_bitwuzla.cpp`: 5 new test cases.

With this step, `seahorn --horn-bmc-solver=smt-bw` solves with Bitwuzla.

## What Bitwuzla 0.9.1 does (probe, 2026-10-22)

[OBS 2026-10-22] I ran a probe program (`scratchpad/bzprobe/probe3.cpp`)
against `$HOME/opt/bitwuzla-0.9.1`:
- **`print_formula(os, "smt2")`** writes a complete script: `set-logic`,
  `declare-const`s, `assert`s, `(check-sat)` and `(exit)`.
- **`check_sat({a})` then `get_unsat_assumptions()`** returns exactly
  the assumption term (`a`).
- **After a plain `check_sat()`, `get_unsat_assumptions()` throws:**
  "cannot get unsat assumptions if input formula is not unsat".
- **`get_value` works after a SAT check, but throws once another
  formula has been asserted:** "cannot get value if input formula is not
  sat".
- **`pop(1)` with nothing pushed throws:** "number of context levels to
  pop (1) greater than number of pushed context levels (0)".
- **Exceptions:** `bitwuzla.h` contains no `throw` or `try`. It compiles
  with `-fno-exceptions -fno-rtti`, so `SolverBmc.cc` can include the
  adapter's header in a build without exceptions.

## Something the adapter has to make up for in SeaHorn

[FACT] `SolverBmcEngine::encode` calls `m_new_smt_solver->add(v)` and
ignores the result.
       → seahorn/lib/seahorn/SolverBmc.cc:106-109

So an adapter that only returned `false` from `add()` for an assertion it
cannot encode would then check the formula without that assertion. It
could answer sat or unsat for the wrong formula.

## Design

[FACT] (commit `22c723b5`)
- **Every change invalidates the last result.** `add`, `push`, `pop`,
  `reset`, `check` and `check_with_assumptions` all call `changed()`.
  That bumps a generation counter and clears the last result, the
  assumption map and the "last check had assumptions" flag. This is the
  opposite of the Z3 adapter (see generic-solver-interface-contract.md).
       → BitwuzlaSolverImpl.cc:41-47
- **An assertion that cannot be encoded is remembered.** `add()` returns
  `false` and records the push depth and the reason, with one warning.
  Every check then answers `UNKNOWN` until a `pop()` goes below that depth
  or `reset()` runs. Bitwuzla exceptions during an assertion are handled
  the same way, e.g. "expected Boolean term".
       → BitwuzlaSolverImpl.cc:64-87, 166-183
- **An assumption that cannot be encoded affects only that one check,**
  which answers `UNKNOWN`.
       → BitwuzlaSolverImpl.cc:104-134
- **The core gives back the caller's own Exprs.** Each check with
  assumptions rebuilds a map from Bitwuzla term to the original
  assumption Exprs. Different Exprs can encode to the same term, so all
  of them are returned. `unsat_core` refuses (logs an error and adds
  nothing) unless the last check was an UNSAT check with assumptions and
  nothing has changed since.
       → BitwuzlaSolverImpl.cc:136-154
- **`push` and `pop`** call `push(1)` and `pop(1)` and track the depth. A
  `pop` at depth 0 is reported and ignored.
- **`reset`** creates a new Bitwuzla instance with the same options. The
  `TermManager` and the encoder's cache are kept.
- **Options:** `PRODUCE_MODELS` and `PRODUCE_UNSAT_ASSUMPTIONS`.
  `PRODUCE_UNSAT_CORES` is a constructor flag, off by default (plan 3.3).
- **`to_smt_lib`** uses `print_formula`. After a check with assumptions,
  it replaces the last `(check-sat)` with `(check-sat-assuming (...))`. A
  missing assertion is flagged with a `; WARNING:` line. A constant that
  occurs only in an assumption is not declared; this is noted in the
  code.
- **No exception reaches SeaHorn.** Every method catches
  `std::exception`. The constructor and `reset` call
  `llvm::report_fatal_error` if Bitwuzla cannot be created at all.
- **The model (`bitwuzla_model_impl`)** stores the solver's generation
  when it is created.
  - `eval` and `print` on a stale model log an error, and `eval` returns
    a null Expr.
  - `eval` decodes Bool and bit-vector values. Other sorts, and terms it
    cannot encode, are returned unchanged; arrays and functions are
    step 1.5.
  - `complete` has no effect, because Bitwuzla gives every term a value.
  - `print` prints `name = value` for every declared constant.
       → BitwuzlaModelImpl.cc:16-67

## Acceptance

[OBS 2026-10-22] `units_bitwuzla`: 17/17 test cases, 2208 assertions
(the 12 from 1.3 plus 5 new). The 5 new cases:
- **The contract checkpoint**, all through `Solver&`:
  - `x + 1 = 0` is SAT, and `eval(x)` is `255:bv(8)` with `complete`
    either false or true;
  - after `push` and `x = 0` it is UNSAT, with no model, and the earlier
    model is stale;
  - after `pop` it is SAT again;
  - after `reset`, the old model and the core are rejected and the empty
    solver is SAT.
- **Core identity:**
  - Activation literals are `boolConst(ASM(v))`, as MUC builds them. The
    core contains the two contradictory literals, and every element is
    pointer-equal to an input literal.
  - A satisfiable subset gives a model and no core.
  - A core after a `push`, or after a plain check, is refused.
- **Unsupported input:**
  - Assertions: `add` returns false and checks answer `UNKNOWN` (even
    with contradictory assertions); a `pop` clears this; at depth 0 only
    `reset` clears it.
  - An unsupported assumption affects only its own check.
  - A non-Boolean assertion is rejected the same way.
- **Misuse:** `get_model` or `unsat_core` before any check, `pop` without
  `push`, and a model used after a new `add` are all reported, not
  fatal.
- **SMT-LIB output:** declarations, assertions, `(check-sat)`;
  `(check-sat-assuming (a))` after a check with assumptions; and the
  warning for a missing assertion.

Log: `scratch/units-1.4.log`.

[OBS 2026-10-22] End to end, on the main tree (`build/`, Bitwuzla 0.9.1):
- **Selector checks:** `scratch/test_selector_1.2.sh` passes 6/6. The
  `smt-bw` case now expects `unsat`, and a new case checks for `Info:
  bmc using Bitwuzla` under `--log=bmc`. `build-clam/` (no Bitwuzla)
  still passes 8/8.
- **Single tests:** with every RUN line of `alloca.02.ll` (sat, with and
  without `--horn-bv2-lambdas`), `test_sadd_with_overflow_carry_sat.ll`
  and `test_ule_unsat.ll` (both modes), `smt-bw` gives the expected answer
  and the same answer as `smt-z3`.
- **Counterexample trace:** with `--log=cex` on
  `test_sadd_with_overflow_carry_sat.ll`, `smt-bw` prints the same trace
  as `smt-z3`, block for block. One value differs in how it is written:
  Z3 gives `%add1 (0x-80000000:bv(32))` and Bitwuzla gives
  `0x80000000:bv(32)`, the same bits. Z3's decoder returns some values as
  negative numbers; step 1.5 should decide which convention SeaHorn
  expects.

## Early look at the regression suites (plan 1.7)

[OBS 2026-10-22] I ran `OUT=$PWD/scratch/preview-1.4 bash
scratch/run_baseline_0.2.sh smt-bw` and compared each test with the
`smt-z3` results from 0.2. This is a preview, not step 1.7.
- **opsem:** 123 passed, 1 expected failure (`alloca.09.ll`, the same as
  `smt-z3`), 2 failed.
  - The failures are `calloc.01.ll` and `calloc.02.ll`. `smt-z3` gives a
    definite answer on both; `smt-bw` answers `unknown`, after Bitwuzla
    prints `[bzla] warning: Equality over constant arrays not fully
    supported yet`. `calloc` fills memory with zeros (a constant array),
    and the formula compares it for equality.
  - The answer is `unknown`, not a wrong verdict. That matches the plan's
    1.7 group (b).
  - Commands: 235, of which 233 carry the flag. 0.2 had 237 and 235; the
    two missing are the second RUN lines of the two failing tests, which
    lit skips after the first one fails.
- **opsem2:** 50/50, the same as `smt-z3` for every test, with the flag in
  55/55 commands.
- **Timing:**
  - opsem took 1.67 s (`smt-z3` in 0.2: 3.66 s).
  - In that run, opsem2 took 12.97 s, with several trivial tests at about
    2 s each. Rerun with nothing else on the machine, opsem2 took 1.40 s
    with `smt-bw` and 1.46 s with `smt-z3`, and `nd_unsat.01.c` alone
    took 0.10 s. So the slow run came from other jobs, which were still
    running at the time. Times from a run that overlaps other work are not
    reliable.

My first attempt used a relative `OUT`. The script runs lit from a
temporary folder, so lit found no opsem2 tests and the script stopped
with "lit did not receive opsem_opts". `OUT` must be an absolute path.

## Open

- [HYP] `calloc.0{1,2}` could be fixed for 1.7 by encoding a constant
  array as the lambda `(lambda x v)`. Equality with it would then use
  the pointwise `forall` that already works for lambdas. Not tried.
- Step 1.5: array and function values, sign convention of decoded values.

## See also

- 2026-10-22-bitwuzla-encoder.md (step 1.3)
- ../../durable/generic-solver-interface-contract.md
