# Reverting the encoder and adapter; Phase 1 restarts from a smallest working example

On 2026-10-29 the user asked to revert "the commit about encoders" and to
rewrite the plan so that a smallest working example is built first, with
features added after it.

## What was reverted

[FACT] The adapter commit `22c723b5` depends on the encoder commit
`0efbdadc`: it includes `MarshalBitwuzla.hh`, holds a `marshal_bitwuzla`
member, and adds to the `units_bitwuzla` target that `0efbdadc` created.
Reverting the encoder alone would have broken the build, so the user chose
to revert both, with `git revert`, and push.
- `654c3098` `revert(smt): remove the Bitwuzla solver adapter` (reverts
  `22c723b5`);
- `7db2c77b` `revert(smt): remove the Bitwuzla encoder` (reverts
  `0efbdadc`);
- both pushed to `MidVC/seahorn` `bitwuzla-support`
  (`22c723b5..7db2c77b`).

[FACT] The tree at `7db2c77b` is identical to `fa82ad3e`
(`git diff fa82ad3e 7db2c77b` is empty). The branch keeps the CMake
detection (`ce226670`, `fa82ad3e`) and the selectors (`406c728e`). With
Bitwuzla compiled in, `--horn-bmc-solver=smt-bw` again stops with "The
Bitwuzla solver adapter is not implemented yet" and exits 1.
       → seahorn/lib/seahorn/SolverBmc.cc:44-52 (branch bitwuzla-support)

[OBS 2026-10-29] Checked after the revert, on `build/` (Bitwuzla 0.9.1
on, `WITH_CLAM=OFF`):
- `ninja install` succeeds;
- `seahorn --version` prints `15.0.0-rc0-7db2c77b` and `Bitwuzla version 0.9.1`;
- `nm` finds no `bitwuzla_solver_impl` or `marshal_bitwuzla` symbol;
- `scratch/test_selector_1.2.sh` passes 6/6 after its `smt-bw` case was
  changed back to expect the "not implemented yet" error.

`build/units/units_bitwuzla` is a stale binary from before the revert.
Its target no longer exists.

## What is kept from the first attempt

The old code stays in git history (`git show 0efbdadc:<path>`). Its
journals are still the record of what Bitwuzla 0.9.1 does:
`2026-10-22-bitwuzla-encoder.md` (the API probe: `ITE` over lambdas
aborts, no function equality, nested lambdas are merged, unbound
variables are accepted) and `2026-10-22-bitwuzla-solver-adapter.md`
(`add()` failures must stick, and the suite preview). Those facts are
about Bitwuzla and SeaHorn, not about our code, so the revert does not
make them wrong. The new plan repeats each one under "From the first
attempt" in the step where it matters.

## The new Phase 1

The plan's Phase 1 was rewritten (Phases 0, 1.1, 1.2 and 2-4 kept; the old
plan is saved at `URA/scratch/BITWUZLA_SEAHORN_PLAN.before-2026-10-29.md`):

| Step | Feature |
|---|---|
| 1.3 | Smallest working example: Bool and the common BV operators; `add`, `check`, `reset`; the other methods stop with an error |
| 1.4 | All bit-vector operators, including the overflow predicates |
| 1.5 | Bool and BV values in models, so traces print |
| 1.6 | Memory as arrays, and array values in models |
| 1.7 | Lambdas and binders |
| 1.8 | push/pop, assumptions, cores, SMT-LIB output |
| 1.9 | Regression runs (was 1.7) |
| 1.10 | Differential and soundness checks (was 1.8) |

The old 1.6 (unit tests) is spread over the steps: each one adds its own.

[OBS 2026-10-29] What the smallest example needs. The `smt-z3` formulas
(`seahorn ... -o`) of four memory-free examples (`inc.c` → sat,
`inc_safe.c` → unsat, `branch.c` → sat, `test/opsem/test_ult_unsat.ll` →
unsat) use only:
- Bool and BV constants (widths 32 and 64), and numerals;
- `and`, `or`, `not`, `=>`, `=`, `ite`;
- `extract`, `bvsub`, and the unsigned comparisons.

There are no arrays. Built at `7db2c77b` with `sea bpf -m64 --bmc=opsem`.
The three C files are in `URA/scratch/mwe/`.

[FACT] Mono BMC calls only `add` and `check` on a normal run. `get_model`
runs only under `--log=cex` / `--cex`, `to_smt_lib` only with `-o`, and
`reset` from `SolverBmcEngine::reset`. Path BMC's MUC is what uses
`push`, `check_with_assumptions` and `unsat_core`.
       → seahorn/lib/seahorn/SolverBmc.cc:71,108,116,126,132;
         seahorn/lib/seahorn/BmcPass.cc:460-495;
         seahorn/lib/seahorn/PathBmcMuc.cc:25-28

Our reading: the smallest adapter can therefore stub everything except
`add`, `check` and `reset`. The stubs should stop with an error, like the
1.2 selector stubs, because a stub that returned a value would look like a
real answer.
