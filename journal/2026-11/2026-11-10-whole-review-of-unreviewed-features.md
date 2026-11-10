# A whole review of the features that no review had seen

2026-11-10, after plan step 1.6 (`29b09883`, branch bitwuzla-support), at
the user's request.

## Scope: what no fresh review had seen

[OBS 2026-11-10] The earlier fresh reviews (by the session's agent prompts)
saw:
- 2026-11-09: the code of steps 1.1-1.4 (CMake, selector, encoder,
  adapter);
- 2026-11-10: the verification tools (formula check, `verify_all`,
  `mutate_bitwuzla.py`, `compare_answers.py`, coverage);
- step 1.5 (models, `check_traces.py`, `run_cex_suite.py`);
- step 1.6 (arrays).

Each saw the code before the fixes made after it. Never reviewed:
1. the CI fix (`7a284207`, seahorn PR #614);
2. the scripts behind the `build`, `examples`, `selector`, `detection` and
   `clam` rows (`run_mwe.sh` and the examples, `test_selector_1.2.sh`,
   `test_find_bitwuzla_1.1.sh`, `build_seahorn*.sh`), and this step's script
   changes;
3. the fixes after each review: `615ef6fd..4639f539`, `4639f539..1ab43e7b`,
   `1ab43e7b..8b85c705`, and the post-review part of `29b09883`.

## Method

Three fresh agents, in parallel, each with a frozen copy of its material,
the contracts and goals, and none of my results:
- one for the CI change;
- one for the scripts and examples (static reading only, since a
  `verify_all` run was using the build trees);
- one for the whole adapter as it is now, with the four post-review diffs
  marked.

Each finding was checked before acting, as below.

## CI fix (PR #614)

[OBS 2026-11-10] PR #614 was merged upstream on 2026-11-10 at 18:19 UTC
(`e4ce011c`; checked with `gh`). The reviewer found nothing that breaks CI.
It checked seahorn's own runs before and after the change, from their logs:
the same image digest in both, `BUILDPACK_IMAGE` carried into later steps,
and the quoting right. Its findings:
- The fallback to seahorn's image triggers on any error of
  `docker manifest inspect`, not only "not found", and the notice always
  says "not found". ghcr answers "denied" both for a missing image and for
  a private one, so the two may not be separable.
- `buildpack-deps.yml` (lines 36, 38) still uses the mixed-case owner, so a
  fork like `MidVC` cannot publish its own image with it. The new comment
  points to that workflow. Older workflows (`seahorn-docker.yml`,
  `seahorn-coverage.yml`, and on `main` the dev18 nightly) have the same
  pattern, but do not run on the fork.
- [FACT] Our fork's `dev18` is 72 commits behind seahorn's (checked with
  `gh api .../compare`), so the fix has not run on `MidVC` yet. The fork's
  run on `bitwuzla-support`'s draft PR still fails on the mixed-case name.
- **CI does not test our code.** Neither Docker image installs Bitwuzla,
  and `docker/build_seahorn.sh` does not build `units_bitwuzla`, so a CI run
  on our branch builds with Bitwuzla off. A green CI says nothing about the
  Bitwuzla code; the evidence is `verify_all`.

Not acted on: the PR is merged. A follow-up PR (`buildpack-deps.yml`, the
notice) and syncing the fork's `dev18` are outward-facing, so the user
decides. Parked.

## Scripts and examples

Findings, each checked by reading the script, the trace, or the bitcode.
- **Could report PASS wrongly (fixed):** `run_mwe.sh` gave `OK` to a file
  without a readable `Expected:` line when neither solver answered (all
  three empty). A missing `mwe/` folder gave one `OK` line. Now:
  - "no Expected line" is a failure;
  - a non-zero exit of `sea` is a failure;
  - no examples is a failure;
  - `verify_all`'s examples row requires one verdict per file.
- **The clam row never has Bitwuzla (fixed in the wording):** `build-clam`
  finds no Bitwuzla, so the row checks only the "not compiled in" messages.
  No tree builds Clam and Bitwuzla together; that is plan 2.4. The row and
  `build_seahorn_clam.sh`'s header now say so. The selector script takes
  `EXPECT_BW` and `EXPECT_CLAM`, so a tree that lost Bitwuzla or Clam fails
  instead of passing with fewer cases.
- **Detection count (fixed):** skipped cases dropped out of the count, so
  24/24 would have read PASS. The row now requires the 27 cases to be
  passed or skipped.
- **Examples claim operations that the optimizer removes (comments
  fixed):**
  - [OBS 2026-11-10] In the final bitcode of `divshift.c` and
    `divshift_safe.c` there is no `urem`, `srem` or `shl`.
  - `inc.c` and `inc_safe.c` have no addition left (`x + 1 > x` becomes
    `x != 0xffffffff`).

  The comments now say what reaches the formula. Every operator is checked
  by the unit tests (all 3-bit inputs against Z3) and the fuzzer, not by
  the examples. Better examples are parked.
- **`check_traces.py` (fixed):**
  - a load after `shadow.mem.load` that it could not read (a pointer or
    global operand) was skipped silently; now it is a problem;
  - memories from PHIs and from `shadow.mem.init` were not recognised. The
    trace prints their values; I had written that it did not
    (`BmcImpl.hh:234-236, 263-265`). Now they are.
  - a third negative control changes the stored cell that the load reads.
- **Inherited `BLD` (fixed):** `verify_all` now unsets it, so the build row
  cannot build a different tree from the one the other rows test.
- **Gaps, parked:**
  - `overflow.c` violates only the signed addition;
  - `branch.c` has no `unsat` counterpart;
  - memory is tested only with 4-byte accesses;
  - the single load in `mem.c` tests the word rounding only if the model
    puts `i` at an odd index;
  - FindBitwuzla's "shared library first" order and its "version unknown"
    branch have no case.

[OBS 2026-11-10] With the fixes, `check_traces.py` gives 10/10
counterexamples and 7/7 extra checks, all three new negative controls
included; `run_mwe.sh` gives 9 OK of 9 files.

## The adapter as a whole

[OBS 2026-11-10] The reviewer found no path where the adapter gives a wrong
`sat`/`unsat`, a wrong model value or a non-value. It traced the narrow-array
guard through Bitwuzla's source and found the 2^31 argument sound. It also
found no false `SAME` in the formula check. Its findings, and what I did:
1. **n-ary `xor`, `bvxor`, `bvmul`:** SeaHorn's Z3 encoder keeps only two
   arguments (checked: `ExprToZ.def:41`, and `Z3_mk_bvmul(ctx, zargs[0],
   zargs[1])`), while Bitwuzla's encoding keeps all. So on such a formula
   `smt-z3` is the one that is wrong. The formula check would report
   `DIFFERENT` (and fail `verify_all`), which is noisy but safe. Its probe
   says opsem's Z3 simplifier makes such terms under `--horn-bv2-simplify`.
   On opsem/opsem2 with the test flags: 2953 formulas `SAME`, 0 `DIFFERENT`.
   Added to the parked entry about `bvmul`.
2. **Two constants with the same name and sort:** Bitwuzla keeps them
   apart (by Expr identity). Z3's encoder makes one declaration per name
   and sort, so it merges them (checked, `ExprToZ.def:370-381`). The
   formula check reports UNCHECKED here. Not seen in SeaHorn's formulas.
   Recorded.
3. **CMake does not detect a C++ runtime mismatch:** FindBitwuzla's test
   program only compiles and links (`check_cxx_source_compiles`; checked),
   throws nothing, and runs before `LLVM_CXXFLAGS`/`LLVM_LDFLAGS` are added
   (`CMakeLists.txt:201,256-257`; checked). [HYP] The reviewer's probe:
   linked against Homebrew llvm's libc++, the program passes, yet a
   `bitwuzla::Exception` is not caught as `std::exception`, and the
   process aborts. I have not reproduced the probe. Our build links the
   system libc++ on both sides, and `units_bitwuzla` would show it.
   Parked, with the fix the reviewer proposes.
4. **The check option and a Debug build:** SeaHorn's Z3 encoder asserts
   `in_width < out_width` for extensions (checked, `ExprToZ.def:187,194`),
   while our encoder accepts an extension to the same width. A Debug build
   with the check option, or a Debug `units_bitwuzla`, would abort. Our
   builds define `NDEBUG`. Parked.
5. **The check logs before asserting:** for a non-Bool assertion it logs
   `SAME` and then `NOT-ENCODED` (checked in `add()`). SeaHorn never adds
   one, so the counts are not affected in practice. Parked.
6. **Header from another installation:** already parked (2026-11-08).
7. **Version range:** `find_package(Bitwuzla 0.9)` accepts any 0.9.x or
   later, but the array guards are argued from 0.9.1's source. Parked: pin
   0.9.1, or rerun the fuzzer on a new version.
8. **`-o FILE` with `smt-bw` exits** (`to_smt_lib` is plan 1.8). Known.

Test weaknesses it named, also parked:
- the fuzzer leaves out the arities where the two encoders disagree;
- its constant names are unique;
- nothing runs a Debug build, or a SeaHorn built without exceptions.

## After the fixes

The script changes do not touch SeaHorn's code, which stays at `29b09883`.
`verify_all --clam --coverage --mutation` was rerun with them; its table is
in the step 1.6 journal.
