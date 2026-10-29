# What a SeaHorn solver adapter must implement, and where existing adapters fall short

Load this note when implementing or reviewing a `solver::Solver`/`Model`
adapter. Verified against SeaHorn `dev18` @ `839a3d57`.

[FACT] The abstract interface has 10 pure virtual methods: `get_kind`,
`add`, `check`, `check_with_assumptions`, `unsat_core`, `push`, `pop`,
`get_model` (returns `model_ref`), `reset`, and `to_smt_lib(raw_ostream&)`.
       → seahorn/include/seahorn/Expr/Smt/Solver.hh:43-70

[FACT] `SolverResult` is declared as `{ SAT, UNSAT, UNKNOWN, ERROR }`, so
`SAT == 0`. Never test a result as a bool. Use a switch or `==`.
       → seahorn/include/seahorn/Expr/Smt/Solver.hh:21-30

[FACT] When `add()` returns true, the assertion was accepted. It says nothing
about satisfiability.
       → seahorn/include/seahorn/Expr/Smt/Solver.hh:46

[FACT] The Z3 adapter sets `m_last_result` only in `check` and
`check_with_assumptions`. It does not clear it in `add`/`push`/`pop`/`reset`.
`get_model` only `assert`s that the result is SAT. A stale SAT therefore
lets you get a model for a state that has since changed. Do not copy this.
       → seahorn/include/seahorn/Expr/Smt/Z3SolverImpl.hh:15, 36-88

[FACT] Yices ignores `Model::eval`'s `complete` flag, and its
`to_smt_lib` only prints a warning. So the existing backends do not behave
the same way, and neither is a complete reference.
       → seahorn/include/seahorn/Expr/Smt/Yices2ModelImpl.hh:33
       → seahorn/lib/seahorn/Smt/Yices2SolverImpl.cc:265-266

[FACT] Callers of `unsat_core` need the *original* assumption `Expr`
objects back. Path BMC's MUC code puts the original constraint inside the
activation literal's structured name and unwraps it from the core. If an
adapter returns rebuilt or renamed expressions, MUC breaks. Yices
rebuilds a native→original map on every assumption check.
       → seahorn/lib/seahorn/PathBmcMuc.cc:12-43
       → seahorn/lib/seahorn/Smt/Yices2SolverImpl.cc

[FACT] `SolverBmcEngine::encode` ignores what `add()` returns. An adapter
that rejects an assertion therefore has to make its later checks
`unknown` itself. Otherwise the formula is solved without that assertion,
and the answer may be wrong. The first Bitwuzla adapter (`22c723b5`,
reverted 2026-10-29) did this, and the new plan's 1.3 does it from the start.
       → seahorn/lib/seahorn/SolverBmc.cc:106-109
       → journal/2026-10/2026-10-22-bitwuzla-solver-adapter.md

## Recorded design pattern (implemented in plan 1.3-1.4, 2026-10-22; reverted 2026-10-29, now built up over plan 1.3-1.8)

[FACT — design pattern] Proposed split for Bitwuzla, mirroring Yices:
`MarshalBitwuzla` (Expr↔Term encode/decode, cache tied to the
TermManager), `BitwuzlaSolverImpl` (assertions, scopes, the per-check
assumption map, explicit result validity), and `BitwuzlaModelImpl`
(a live view bound to a solver "generation" counter, invalidated on any
mutation or re-check). Bitwuzla's `get_value` has no completion flag, so
the adapter must document its `complete` policy.
       → URA/SEAHORN_BITWUZLA_COURSE_NOTES.md §9.4, §9.7, §10.3

## Acceptance checkpoint

These steps use an 8-bit `x` with `x + 1 = 0`:
1. Checking gives SAT, and the model gives `x = 255` as an 8-bit value.
2. After `push` and `x = 0`, checking gives UNSAT.
3. After `pop`, checking gives SAT again.
4. After `reset`, the assertions are empty and the old model/core are rejected.

## See also

- adding-a-solver-backend-touches-these-files.md
