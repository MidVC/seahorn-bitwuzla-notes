# Sessions

## 2026-10-02
**Theme:** Install better-than-fish and bootstrap notes for the SeaHorn/Bitwuzla project.
**Key outputs:**
- Skill symlinked: `~/.claude/skills/better-than-fish` → the clone in `URA/better-than-fish`
- `URA/CLAUDE.md` (pointer to the notes) and `research-notes/CLAUDE.md`
- `durable/adding-a-solver-backend-touches-these-files.md`
- `durable/generic-solver-interface-contract.md`
- `journal/2026-10/2026-10-02-bootstrap-and-z3-baseline.md` (smt-z3 baseline reproduced: unsat)
- `loose-ends/parked.md` (3 entries)
**Critical corrections:** none
**Status:** complete
**Next-session pickup candidates:** Build Bitwuzla locally (parked, ~half-day).

## 2026-10-14
**Theme:** Write the Bitwuzla integration plan; finish Phase 0: 0.1 (Bitwuzla build), 0.2 (lit baselines), 0.3 (build configurations); then 1.1 (build integration) on the new branch `bitwuzla-support`.
**Key outputs:**
- `URA/BITWUZLA_SEAHORN_PLAN.md` (phases 0–4; Phase 0 and 1.1 marked done)
- SeaHorn branch `bitwuzla-support` (from `dev18` @ `839a3d57`; commit `ce226670`, pushed 2026-10-15 to the fork `MidVC/seahorn`): `cmake/FindBitwuzla.cmake`, the `BITWUZLA_HOME` block in `CMakeLists.txt`, the `SeaSmt` link, and a `seahorn --version` line. `seahorn/build` now has Bitwuzla on.
- `journal/2026-10/2026-10-14-bitwuzla-build-integration.md` (1.1: opt-in design; acceptance; opsem/opsem2 unchanged from 0.2)
- `URA/scratch/test_find_bitwuzla_1.1.sh` (7 configure-only detection cases); `run_baseline_0.2.sh` now takes `OUT=`
- `journal/2026-10/2026-10-14-build-configurations.md` (0.3: `seahorn/build-clam` with Clam; test/bmc 20/20 under default and smt-z3; Yices 2.7.0 incompatible, deferred)
- `URA/scratch/build_seahorn_clam.sh`, `URA/scratch/run_bmc_baseline_0.3.sh`
- `durable/adding-a-solver-backend-touches-these-files.md`: Yices 2.7 / FindYices2 auto-detect facts; timeout-cast guard
- `journal/2026-10/2026-10-14-bitwuzla-build-and-link.md` (0.1)
- `journal/2026-10/2026-10-14-lit-baselines.md` (0.2: smt-z3 = z3direct on opsem 125+1 XFAIL, opsem2 50/50)
- `URA/scratch/run_baseline_0.2.sh` (reusable for smt-bw; checks that the flag reached the commands)
- `loose-ends/parked.md`: new entry for the upstream lit opsem_opts fix
**Critical corrections:** My first smt-z3 baseline was invalid because zsh did not word-split the `-D` parameter, so the "identical results" were two default runs. Caught by counting the flag in the executed commands. Also, plan 2.2 wrongly claimed an undefined-behavior cast when Yices is selected; the cast is guarded (fixed).
**Status:** complete
**Next-session pickup candidates:** Plan 1.2 (smt-bw selector with a stub adapter). Also still unrecorded: the overflow-predicate and BREDAND-width findings from the plan.

## 2026-10-15
**Theme:** Commit and push 1.1 to a fork; write a progress update for the professor; plan step 1.2 (solver selectors).
**Key outputs:**
- Fork `MidVC/seahorn` (`origin`; `upstream` = `seahorn/seahorn`). Commit `ce226670` (1.1) is pushed on `bitwuzla-support`.
- Progress page for the professor: https://claude.ai/artifact/DeZaDQKYPc5FWHVstG9NAy (private until shared), plus a Slack text version
- 1.2 on `bitwuzla-support` (commit `406c728e`, pushed): `SolverKind::BITWUZLA`, `--horn-bmc-solver=smt-bw`, a `SolverBmc` branch, and Path BMC `--horn-bmc-smt-solver=bw`. Every Bitwuzla route exits non-zero with a clear message until 1.4 and 2.4.
- `journal/2026-10/2026-10-15-smt-bw-selector.md`
- `URA/scratch/test_selector_1.2.sh` (5 checks on `build`, 8 on `build-clam`); `run_bmc_baseline_0.3.sh` now takes `OUT=`
**Critical corrections:** The selector test first skipped Path BMC on `build-clam`: under `pipefail`, `seahorn --help-hidden | grep -q` fails from SIGPIPE. It was caught because the printed `clam=no` was wrong for that tree.
**Status:** complete
**Next-session pickup candidates:** Plan 1.3 (`MarshalBitwuzla` encoder), which is the first real Bitwuzla code. Still unrecorded from the plan: the overflow-predicate and BREDAND-width findings.

## 2026-10-21
**Theme:** Fix the CI `opsem_opts` bug in a separate branch, at the professor's request.
**Key outputs:**
- Branch `fix-ci-opsem-opts` from `dev18` @ `839a3d57` (upstream unchanged), pushed to `MidVC/seahorn`: `690e462a` (the CI key in 4 files) and `cc65c4f6` (opsem2 forwarding). The user opens the PR to `seahorn/seahorn` `dev18`.
- `journal/2026-10/2026-10-21-ci-opsem-opts-fix.md`; `URA/scratch/verify_ci_opsem_opts.sh`
- Plan 4.3: the two fix items are marked ✅. The parked entry is marked resumed and fixed.
- PR #601 (base changed to `dev18` by the user): I wrote its description
  (problem, changes, local verification table, notes). The title is still
  "Fix ci opsem opts", which is not in conventional-commit form.
**Critical corrections:** none
**Status:** complete (waiting on the PR review)
**Next-session pickup candidates:** Plan 1.3 (`MarshalBitwuzla` encoder) on `bitwuzla-support`. Also still unrecorded: the overflow-predicate and BREDAND-width findings.

## 2026-10-22
**Theme:** Explain `FindBitwuzla.cmake`, compare ten smart-detection methods with it, and implement the recommended set (a revision of 1.1). Then plan 1.3, the encoder.
**Key outputs:**
- Smart detection, committed as `fa82ad3e` and pushed to `MidVC/seahorn`:
  - `cmake/FindBitwuzla.cmake`: search order `Bitwuzla_ROOT`, then `CMAKE_PREFIX_PATH`, then pkg-config's libdir, then the system prefixes; header from the library's own prefix; link test; searches again on every configure.
  - `CMakeLists.txt:177-214`: `WITH_BITWUZLA=AUTO|ON|OFF`; `BITWUZLA_HOME` (CMake setting or environment variable) is strict and makes Bitwuzla required.
- `scratch/test_find_bitwuzla_1.1.sh`: 25 cases, all passing (`scratch/acceptance-1.1/find-bitwuzla-smart.log`). `build/` is on and passes the selector checks 5/5; `build-clam/` is off and passes 8/8.
- `journal/2026-10/2026-10-22-bitwuzla-smart-detection.md`. The opt-in claims in the 1.1 journal, the durable note and plan 1.1 are marked superseded.
- Durable: two find-module facts (how `find_package_handle_standard_args` treats a missing version, and how `check_cxx_source_compiles` caches its result).
- Plan: a 1.1 revision note and new Linux checks in 4.3. Parked: test against a real Homebrew Bitwuzla.
**Critical corrections:** none. The opt-in decision was reversed at the user's request, not because it was wrong.
- Plan 1.3 ✅, committed as `0efbdadc` and pushed:
  - `marshal_bitwuzla` (`include/seahorn/Expr/Smt/MarshalBitwuzla.hh`, `lib/seahorn/Smt/MarshalBitwuzla.cc`, built with `-fexceptions`);
  - `units/units_bitwuzla.cpp` (target `test_bitwuzla`): 12/12 test cases, 2139 assertions, including bit-vector operators against Z3 and the overflow predicates against Z3 and exact arithmetic.
  - Three deliberate encoder bugs were each caught by the tests.
- `journal/2026-10/2026-10-22-bitwuzla-encoder.md`, which includes a probe of what Bitwuzla 0.9.1 accepts:
  - `ITE` over lambdas aborts the process;
  - equality over functions gives `unknown`;
  - nested lambdas are merged;
  - unbound variables are silently accepted.
- Plan: 1.3 ✅; notes for 1.4 (catch exceptions) and 1.7 (pointwise equality may be slow); a new risk row. Durable: two facts (`-fexceptions`; the doctest `TRUE` macro).
- Plan 1.4 ✅, committed as `22c723b5` and pushed:
  - `bitwuzla_solver_impl` and a minimal `bitwuzla_model_impl` (`BitwuzlaSolverImpl.{hh,cc}`, `BitwuzlaModelImpl.{hh,cc}`), wired into `SolverBmc`. `smt-bw` now solves with Bitwuzla.
  - Because `SolverBmc` ignores what `add()` returns, a failed `add` makes every later check `unknown`.
  - `units_bitwuzla` 17/17; selector checks 6/6 and 8/8.
  - Preview of the suites: opsem 123 + 1 expected failure + 2 `unknown` (`calloc`, constant-array equality); opsem2 50/50.
- `journal/2026-10/2026-10-22-bitwuzla-solver-adapter.md`. Durable contract note: the fact about `add()`.
**Critical corrections:** none. One error of my own: the first preview run used a relative `OUT`, so lit found no opsem2 tests. It was rerun with an absolute path.
**Status:** complete
**Next-session pickup candidates:** Plan 1.5 (array and function values, the sign convention), or the `calloc` constant-array idea for 1.7.


## 2026-10-23
**Theme:** Explain 1.3/1.4, the next steps, Phase 2 and BMC vs SMT in short answers. Then write a PDF guide to SeaHorn's representations.
**Key outputs:**
- `URA/SEAHORN_REPRESENTATIONS.pdf` (source `.tex` next to it, built with XeLaTeX). It follows three examples (`inc.c`, `mem.c`, `loop.c`) through LLVM IR, the cut-point graph, `Expr`, the BMC formula, memory arrays and lambdas, Horn clauses with Spacer invariants, the solver interface, and traces. Every listing comes from a run on `bitwuzla-support` @ `22c723b5`.
- [OBS 2026-10-23] Things seen while making the examples:
  - The `int` version of `inc.c` is proved safe: `add nsw` lets the optimizer drop the overflow.
  - With `-m64` and the default word size, `mem.c`'s formula is 48 KB; with `-m32 --horn-bv2-word-size=4` it is 1.4 KB.
  - Z3 decodes the memory array in the trace (`store(... const-array ...)`), while the Bitwuzla model prints the symbolic `store` chain (plan 1.5).
  - In the `inc.c` trace, Z3 prints `-1:bv(32)` and Bitwuzla prints `0xffffffff:bv(32)`.
**Critical corrections:** none
**Status:** complete
**Next-session pickup candidates:** Plan 1.5.

## 2026-10-29
**Theme:** Revert the encoder and adapter, rewrite Phase 1 of the plan to start from a smallest working example, then do plan 1.3.
**Key outputs:**
- `bitwuzla-support`: `654c3098` reverts the adapter (`22c723b5`), `7db2c77b` reverts the encoder (`0efbdadc`); pushed. The adapter depends on the encoder, so the user chose to revert both. The tree equals `fa82ad3e`; `smt-bw` again stops with "not implemented yet". Build OK; selector script 6/6 (its `smt-bw` case changed back).
- `URA/BITWUZLA_SEAHORN_PLAN.md`: Phase 1 rewritten as 1.3 smallest working example (Bool + common BV, `add`/`check`/`reset`), then 1.4 all BV ops, 1.5 models and traces, 1.6 arrays, 1.7 lambdas, 1.8 push/pop/assumptions/cores/SMT-LIB, 1.9 regressions, 1.10 differential checks. Lessons from the first attempt sit in each step. Phases 0, 1.1, 1.2 and 2-4 kept, with cross-references updated. Old plan: `URA/scratch/BITWUZLA_SEAHORN_PLAN.before-2026-10-29.md`.
- `URA/scratch/mwe/`: `inc.c` (sat), `inc_safe.c` (unsat), `branch.c` (sat), checked with `smt-z3`.
- `journal/2026-10/2026-10-29-revert-encoder-and-adapter.md`, with [OBS] on the operators the examples need and [FACT] on which `Solver` methods mono BMC calls. The 1.3/1.4 journals and two durable notes now say the code was reverted.
- Plan 1.3 ✅, committed as `959ea729` and pushed: a minimal `marshal_bitwuzla` (Bool + common BV ops) and `bitwuzla_solver_impl` (`add`/`check`/`reset`; the other methods exit with an error), wired into `smt-bw`. The four examples match `smt-z3`; `units_bitwuzla` 9/9; selector 6/6 and 8/8.
- First progress number: opsem 48 + 1 expected failure (77 `unknown`), opsem2 16/50 (34 `unknown`). All 111 failures rerun: every one is `unknown`, no wrong answers. Blockers: `select` 44, then 1.4's bit-vector operators. Results in `URA/scratch/progress-1.3/`; new scripts `scratch/rerun_lit_tests.py`, `scratch/compare_lit_xml.py`.
- [OBS 2026-10-29] Printing a whole Expr in an error message hung two memset tests (tree printing of a shared DAG is exponential). Fixed and unit-tested.
- `journal/2026-10/2026-10-29-step-1.3-smallest-working-example.md`.
- These notes are now a public git repo: https://github.com/MidVC/seahorn-bitwuzla-notes (`main`), with a `README.md` for readers.
**Critical corrections:** none. The revert is a change of approach, not a wrong claim.
**Status:** complete
**Next-session pickup candidates:** plan 1.4 (all bit-vector operators), probably followed closely by 1.6 (arrays).

## 2026-10-30
**Theme:** Plan 1.4: every bit-vector operator, after restoring the main build tree.
**Key outputs:**
- Plan 1.4 ✅, committed as `615ef6fd` and pushed. `marshal_bitwuzla` encodes every `BvOpKind`, including the nine overflow predicates (Z3's one-direction meaning, built from Bitwuzla's two-direction tests plus the sign). It rejects `INT2BV`/`BV2INT` and `BEXT_ROTATE_*` on purpose.
- `units_bitwuzla` 12/12 (2332 assertions): every operator against Z3 on all 3-bit inputs, the overflow predicates against Z3 and exact arithmetic at widths 1-4. Four planted bugs were all caught.
- Examples `scratch/mwe/{divshift,divshift_safe,overflow,overflow_safe}.c` match `smt-z3`, as do the 1.3 ones; `mem.c` added as `Pending: 1.6`. New runner: `scratch/run_mwe.sh`.
- Progress number: opsem 86 + 1 expected failure (was 48 + 1), opsem2 21/50 (was 16). No 1.3 pass lost; all 68 failures answer `unknown`. Blockers now only `select` 64, `const` 2 (1.6) and lambda `fapp` 2 (1.7). Results in `URA/scratch/progress-1.4/`.
- [FACT] `sea` lowers the overflow intrinsics unless given `--with-arith-overflow`; Z3's marshal silently drops a third argument of `bvsub`, `bvmul`, the divisions and the shifts.
- [OBS 2026-10-30] VS Code's CMake Tools had left `seahorn/build` without `CMakeCache.txt`; reconfigured with the original values (command now recorded in the journal). It happened again at 11:36: CMake Tools used upstream's `Debug-jammy` preset, whose build folder is `seahorn/build`. Fixed in `URA/.vscode/settings.json`; IntelliSense now reads `build/compile_commands.json`.
- `journal/2026-10/2026-10-30-step-1.4-bit-vector-operators.md`; plan notes "From 1.4" in 1.6 and 1.7.
**Critical corrections:** none. One error of my own: restoring the encoder with `cp -p` kept its old timestamp, so ninja installed the planted bugs once; caught by the unit tests and rebuilt.
**Status:** complete
**Next-session pickup candidates:** plan 1.5 (models and traces) or 1.6 (arrays, which unblocks 64 tests).

## 2026-11-08
**Theme:** Full verification of plan steps 1.1-1.4, with fixes.
**Key outputs:**
- One commit, `4639f539`, pushed 2026-11-10 (two local commits squashed): FindBitwuzla reads the version from the library without `bitwuzla.pc`; the encoder rejects `=>`, `<=>`, `=`, `distinct` with other than two arguments; Bitwuzla's encoding errors become `UnsupportedExpr`; new tests.
- New tests: Boolean truth tables against Z3, one-argument n-ary operators, and a differential fuzzer whose `sat` answers are certified by Z3. 320000 fuzzed checks: 0 disagreements.
- Mutation test: 67 of 68 planted bugs caught; the survivor is equivalent. `scratch/mutate_bitwuzla.py`.
- opsem/opsem2 answers per test against `smt-z3`: 107 same (106 + 1 where neither answers; see 2026-11-10), 69 `unknown` (`select` 65, `const` 2, `fapp` 2), 0 different. Other checks (examples, selector 6/6 + 8/8, detection 27/27, UBSan, Guard Malloc, other unit tests) pass.
- Independent review by a subagent: no wrong-answer path; its four points were fixed or parked.
- Build trees moved out of OneDrive to `~/builds/seahorn` (`SEAHORN_BUILDS`); new `scratch/build_seahorn.sh`; scripts and VS Code settings updated; same results on the new trees; old trees (5.8 GB) deleted.
- `journal/2026-11/2026-11-08-verification-of-steps-1.1-1.4.md`; two items added to `loose-ends/parked.md`.
**Critical corrections:** none of earlier claims. The 1.4 note that bit-vector n-ary forms are only built by Z3-to-Expr converters stands; opsem's default simplifier is such a converter (parked question about `smt-z3`).
**Status:** complete
**Next-session pickup candidates:** the CI fix (see 2026-11-10); plan 1.5 or 1.6.

## 2026-11-10
**Theme:** Push the verification fixes; fix CI on the fork.
**Key outputs:**
- `b18030e8` and `50ec55b9` squashed into `4639f539` (same tree) and pushed to `bitwuzla-support`; notes updated to cite it and pushed (`a6a5a89`).
- CI on the fork failed because `MidVC` is not lowercase. Fix (lowercase the owner, fall back to seahorn's buildpack image) on its own branch `fix-ci-buildpack-image` (`7a284207`, based on upstream `dev18` @ `2e9eed07`); PR https://github.com/seahorn/seahorn/pull/614.
- `journal/2026-11/2026-11-10-ci-fork-buildpack-image.md`.
- The user chose verification rules (now in `URA/CLAUDE.md`): pass criteria first, differential testing, mutation with bugs written by a separate agent, safe failure, fresh review before pushing, a correctness case per step; plus three tools set up once.
- Formula check (translation validation), commit `1ab43e7b` (pushed): `--horn-bw-check-encoding=FILE`. On opsem/opsem2, 2485 formulas `SAME`, 0 `DIFFERENT`. It caught 11 of 14 encoder bugs planted at lit level; the 3 survivors are explained.
- `scratch/coverage_bitwuzla.sh` (93.3% of lines; every other line has a reason) and `scratch/verify_all.sh` (one table, all checks).
- Bugs planted in the check by a separate agent: all 21 caught or explained (1 equivalent), after 5 tests added for the survivors.
- A fresh review found false-`SAME` cases through constant names, and `verify_all` passing too easily. All fixed and tested.
- `journal/2026-11/2026-11-10-verification-tools.md`.
**Critical corrections:** my first CI draft (lowercase only, plus a repository variable) would still have failed without the variable: the fork has no image of its own. The 2026-11-08 "107 same" counted a test where neither solver answers: 106 + 1 (superseded there).
**Status:** complete
**Next-session pickup candidates:** after #614 merges, sync the fork's `dev18` so draft PR #1 gets CI; plan 1.5 or 1.6. Each step from now on follows the verification rules.
