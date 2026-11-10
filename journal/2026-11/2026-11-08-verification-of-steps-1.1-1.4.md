# Verification of everything up to step 1.4

A full check of plan steps 1.1-1.4 (build detection, the `smt-bw`
selector, the encoder and the adapter), asked for on 2026-11-08. The
fixes are one commit on `bitwuzla-support`, pushed 2026-11-10:
`4639f539` fix(smt): reject chained binary operators for Bitwuzla and
fuzz it against z3. It squashes two local commits of 2026-11-08 (the
FindBitwuzla fix, then the encoder fix and tests), and its tree is
identical to the second's (`50ec55b9`, never pushed). So results below
"with `4639f539`" were obtained on that same code.

Logs are in `URA/scratch/verify/`. Unless noted, everything ran on
`build/` (Bitwuzla 0.9.1, `WITH_CLAM=OFF`, no Yices), LLVM 18.1.8 and
Z3 5.1.0, with the code of `4639f539`.

## What was checked, and the results

[OBS 2026-11-08] An independent review by a fresh agent, which saw the
code but not my conclusions, found no path to a wrong `sat`/`unsat`, no
crash and no hang. It raised four lower-severity points, handled below.

[OBS 2026-11-08] `units_bitwuzla`: 14/14 test cases, 3537 assertions.
New since 1.4:
- every Boolean operator on all eight assignments of three truth values,
  against Z3;
- n-ary operators with one argument;
- `bitwuzla.random formulas agree with z3`, a differential fuzzer:
  - random conjunctions of atoms over Bool and bit-vector constants;
    widths 1-128 (multiplication and division only up to 16), shared
    subterms, and numerals that are negative or wider than their sort;
  - each formula and its negation go through the adapter (one adapter,
    `reset()` between formulas) and through Z3;
  - every `sat` answer is certified: Z3 checks the formula under
    Bitwuzla's values of its constants;
  - 300 formulas by default, `BITWUZLA_FUZZ_N` and `BITWUZLA_FUZZ_SEED`
    scale it up.

[OBS 2026-11-08] Fuzzer at scale: 16 seeds x 20000 checks = 320000
(half before the fixes, half with `4639f539`): 249,996 sat, 70,004
unsat, 0 disagreements, 0 rejected models, 0 unknown to Z3.

[OBS 2026-11-08] Mutation test (`python3 -I scratch/mutate_bitwuzla.py`):
68 planted bugs, one at a time (operator mappings, extract/extend indices,
numerals, overflow directions, traversal, adapter error handling). 67
make some unit test fail. The survivor, "no identity extension" (dropping
the `out_width == in_width` shortcut), is equivalent: Bitwuzla 0.9.1
accepts an extension by 0. The first run found one gap, "no one-argument
shortcut", which the one-argument test now closes.

[OBS 2026-11-08] Per-test answers on opsem/opsem2
(`scratch/rerun_lit_tests.py` on each lit log, then
`scratch/compare_answers.py`):

| Suite | Same answer as `smt-z3` | `smt-bw` unknown | Different |
|---|---|---|---|
| opsem (126) | 86 | 40 | 0 |
| opsem2 (50) | 21 | 29 | 0 |

The 69 `unknown`s stop on `select` 65, `const` 2 (1.6) and `fapp` 2
(1.7). Pass counts are those of 1.4: opsem 86 + 1 expected failure,
opsem2 21.

[OBS 2026-11-08] Also passing: `scratch/run_mwe.sh` (all examples, `mem.c`
pending); `scratch/test_selector_1.2.sh` (6/6 on `build/`, 8/8 on
`build-clam/`, which compiles without Bitwuzla);
`scratch/test_find_bitwuzla_1.1.sh` (27/27, one new case); UBSan on the
encoder, adapter and tests (14/14 with 6000 fuzzed formulas, no report);
Guard Malloc (`DYLD_INSERT_LIBRARIES=/usr/lib/libgmalloc.dylib`, 14/14).

[OBS 2026-11-08] SeaHorn's other unit tests are unchanged by the branch:
`units_z3` 5/6 (the known `expr.neg`), `units_type_checker` 31/31,
`units_fmap` 9/9, `units_hex_dump` 8/8, `units_sea_dsa` 13/13.
`units_evaluate` and `units_fmap_chc` do not compile with this toolchain
(ambiguous `mpz_class` constructors, `std::visit`). Their sources and
headers are unchanged since `839a3d57`, and the errors are the same
without `-DWITH_BITWUZLA`, so this predates the branch.

## Fixes

[FACT] `=>`, `<=>`, `=` and `distinct` with other than two arguments are
rejected (`unknown`) [`MarshalBitwuzla.cc`, `mkNode`; branch
bitwuzla-support, `4639f539`]. SeaHorn types them as binary
(`ExprOpBool.hh:80,82`, `ExprOpCompare.hh:64-65`), and for more
arguments three readings exist: Bitwuzla chains them (SMT-LIB), SeaHorn's
evaluator folds left (`Evaluate.hh:50-58`), and Z3's encoder drops all
but two (`ExprToZ.def:45-50,63-65`). `xor` stays n-ary: parity in
Bitwuzla and in the evaluator.

[FACT] Bitwuzla's own errors while encoding a sort, numeral or constant
(width 0, say) now become `UnsupportedExpr` naming the term, as the
header says. Before, `bitwuzla::Exception` escaped `encode_term`. The
adapter caught it anyway, so no answer changes.

[FACT] Without `bitwuzla.pc`, FindBitwuzla runs a program printing
`bitwuzla::version()`. If that fails too, the library is ignored when a
version is required [`cmake/FindBitwuzla.cmake`, `4639f539`]. Before, the
version check was skipped, so any release with the same C++ API passed.

## Noted, not changed

[FACT] `UnsupportedExpr` holds the offending Expr. Destroying it after its
`ExprFactory` is gone crashes, which happened when a planted bug let one
escape a unit test. `add()` catches it inside the method, so `seahorn`
is not affected. The header now says so.

[HYP] Include order: `include_directories(${BITWUZLA_INCLUDE_DIR})`
comes after the GMP/Z3/Yices include directories. If one of those
directories also had a different `bitwuzla/` header, it would be used
with the library found, and a renumbered `Kind` enum would swap
operators. Not the case here (`/opt/homebrew/include` is `-isystem` and
last). `BEFORE` would risk shadowing other headers. Parked.

[HYP] Z3's encoder drops arguments after the second for `bvmul`, `bvxor`,
`bvsub`, the divisions and more (`ExprToZ.def:217-259`). Opsem runs
non-memory terms through Z3's simplifier and back by default
(`--horn-bv2-simplify-nonmem`, `BvOpSem2.cc:144,2917`), and Z3 flattens
`bvmul`. So `smt-z3` may see a 3-argument `bvmul` and drop a factor. Not
reproduced yet. Bitwuzla encodes n-ary `bvmul`/`bvxor` correctly. The
review also showed that Z3's encoder merges two different constants that
print the same; Bitwuzla does not. When the two solvers disagree, Z3 is
not automatically right.

## Environment problems met on the way

[OBS 2026-11-08] OneDrive changed every file in the URA folder to mode
`rwx------`. Git shows all 1025 SeaHorn files as modified (mode only).
Stage with `git -c core.fileMode=false add <files>` so no mode change is
committed.

[OBS 2026-11-08] OneDrive breaks long writes in the URA folder:
- a log written by `lit` was moved into OneDrive's staging area
  (`FileProvider/.../wharf/delete`) while open, so the run's output was
  lost;
- `printf` into a log failed with "Operation timed out";
- ninja failed at least 12 times writing `.ninja_log`/`.ninja_deps` in
  `seahorn/build` (10 mutants did not build in the first run).

Lesson: write long-running logs to `$TMPDIR`, not under URA, and retry a
ninja build that fails with "Operation timed out" (`mutate_bitwuzla.py`
does).

[OBS 2026-11-08] AddressSanitizer from Homebrew LLVM 18 hangs at start-up
on this macOS (spinning in `__asan::InitializeShadowMemory`). UBSan alone
and Guard Malloc work.

## Build trees moved out of OneDrive (later the same day)

[OBS 2026-11-08] The two build trees took 5.8 GB of the 6.0 GB URA
folder (`build` 2.2 GB, `build-clam` 3.6 GB), and OneDrive was nearly
full. They now live in `$SEAHORN_BUILDS` (default `~/builds/seahorn`):
- `scratch/build_seahorn.sh` (new) makes `build/`: the configure command
  from the 1.4 journal plus `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`, then
  install and `units_bitwuzla`. A full build took 1 min 19 s.
- `scratch/build_seahorn_clam.sh` makes `build-clam/` next to it; 2 min 9 s.
- `run_mwe.sh`, `run_baseline_0.2.sh`, `run_bmc_baseline_0.3.sh`,
  `test_selector_1.2.sh`, `verify_ci_opsem_opts.sh` and
  `mutate_bitwuzla.py` look for the trees there.
- `URA/.vscode/settings.json` points IntelliSense at the new
  `compile_commands.json`, and gives CMake Tools its own folder
  (`build-vscode`) outside OneDrive.

The new trees give the same results as the old ones:
- `units_bitwuzla` 14/14;
- `run_mwe.sh` 7 OK + `mem.c` pending;
- selector 6/6 (`build`) and 8/8 (`build-clam`);
- `units_z3` passes except the known `expr.neg`;
- opsem/opsem2 per-test answers: 107 same as `smt-z3`, 69 `unknown`,
  0 different.

The old `URA/seahorn/build` and `URA/seahorn/build-clam` were then
deleted; URA is now 106 MB. Results of this check are in
`~/builds/seahorn/results/2026-11-08-moved/`.
