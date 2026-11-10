# Verification tools: formula check, coverage, verify_all

On 2026-11-10 the user picked, from a list of ways to show the code is
correct without reading it:
- **every step:**
  - differential testing against Z3;
  - mutation testing;
  - a fresh review agent;
  - a one-page correctness case;
  - safe failure;
  - pass criteria written first;
- **set up once:**
  - checking that Bitwuzla gets the same formula as Z3 (translation
    validation);
  - a coverage report;
  - one command that reruns everything.

The rules are in `URA/CLAUDE.md`. This entry records the set-up of the
three tools.

## Pass criteria (written before building them)

Formula check (translation validation):
- F1. For every formula that SeaHorn gives Bitwuzla in the opsem and
  opsem2 suites, Z3 is asked whether Bitwuzla's term (as Bitwuzla prints
  it) equals Z3's own encoding of the same Expr. Pass: 0 `DIFFERENT`.
  Every `UNDECIDED` (Z3 time limit) and `UNCHECKED` (could not compare) is
  counted and explained.
- F2. It catches bugs: a unit test hands it a wrong term and gets
  `DIFFERENT`. With a bug planted in the encoder, the opsem run reports
  `DIFFERENT`.
- F3. The fuzzer in `units_bitwuzla` also checks every random formula:
  0 `DIFFERENT`.
- F4. Off by default. Without the option, the lit answers are unchanged:
  107 same as `smt-z3`, 69 `unknown`, 0 different.
- F5. None of my code sits between the two encodings: Bitwuzla prints,
  Z3 parses, Z3 decides.

Coverage:
- C1. Line coverage of the Bitwuzla files from `units_bitwuzla` plus the
  `smt-bw` lit runs. Every line not run is listed with a reason, or gets a
  test.

One command:
- V1. `scratch/verify_all.sh` rebuilds and reruns every check, writes the
  results with the commit and tool versions, prints one table, and exits
  non-zero if any check fails.
- V2. On the current code it reproduces the numbers already recorded
  (units 14/14 or more, examples OK, selector 6/6, detection 27/27, lit
  107/69/0).

## What was built

[FACT] The formula check (branch bitwuzla-support):
- `lib/seahorn/Smt/EncodingCheck.cc`, the Z3 side. It parses an SMT-LIB
  term with the given constants, and asks Z3 whether `term ≠ ExprToZ(e)`
  is satisfiable, within a time limit:
  - unsat gives `SAME`;
  - sat gives `DIFFERENT`, with the differing values;
  - unknown gives `UNDECIDED`;
  - anything it cannot set up gives `UNCHECKED`.
- `lib/seahorn/Smt/BitwuzlaEncodingCheck.cc`, the Bitwuzla side. It
  collects the term's constants and prints the term with `Term::str()`.
  Bitwuzla's printer uses `let` for shared subterms
  (`src/printer/smt2_printer.cpp:645`), so printing stays linear.
- `--horn-bw-check-encoding=FILE` (hidden, in `BitwuzlaSolverImpl.cc`)
  checks every formula `add()` gives Bitwuzla, and appends one line per
  formula: `SAME`/`DIFFERENT`/`UNDECIDED`/`UNCHECKED`, or `NOT-ENCODED`
  when the encoder rejected it. `--horn-bw-check-encoding-timeout`
  (default 10000 ms) limits each check.

[FACT] Why the check can be trusted more than the encoder (F5):
- No adapter code stands between the two encodings: Bitwuzla prints, Z3
  parses, and Z3 compares with SeaHorn's existing Z3 encoder
  (`ExprToZ.def`). A mapping mistake in `MarshalBitwuzla.cc` therefore
  shows up as a different formula; nothing translates it back.
- Bitwuzla does not rewrite terms when it builds them, so the printed
  term is exactly what the encoder produced. (Found by the reviewer:
  `mk_term`, `bitwuzla.cpp:2249`.)
- The Z3 side compiles like the rest of SeaHorn and uses only Z3's C API,
  checking the error code right after each call. Every Z3 call resets
  that code, including the `Z3_inc_ref` of a z3++ wrapper.

[FACT] Cases the check refuses (`UNCHECKED`) instead of risking a wrong
match:
- two constants with the same name;
- names that SMT-LIB cannot carry safely: names containing `|` or `\`,
  and names starting with `_let`, which is how Bitwuzla names its `let`
  bindings;
- constants that are not Bool or bit-vector, such as arrays (to extend in
  plan 1.6);
- bound variables;
- repeat and rotate, which SeaHorn's Z3 encoder cannot encode. It fails
  on them without a clean error.

The fresh constant that the term is assigned to avoids the names of both
sides, because Z3 identifies constants by name and sort.

[FACT] Scripts:
- `scratch/coverage_bitwuzla.sh`: build tree `build-cov`, units plus the
  lit runs with the check on;
- `scratch/verify_all.sh`: every check, one table, exit code;
- `scratch/mutate_bitwuzla.py`:
  - `MUTATE_MODE=lit` rebuilds seahorn for each planted bug and runs
    opsem with the check;
  - its sources are restored on SIGTERM/SIGHUP;
  - a backup blocks the next run if a run is killed;
- `run_baseline_0.2.sh`: `EXTRA_OPTS`/`TAG_SUFFIX`;
- `compare_answers.py`: `NO-ANSWER`.

## Results against the criteria

[OBS 2026-11-10] F1: opsem and opsem2 with `smt-bw` and the check
(`~/builds/seahorn/results/2026-11-10-encoding-check/`):
- 2485 `SAME`, 0 `DIFFERENT`, 0 `UNDECIDED`, 0 `UNCHECKED`;
- 136 `NOT-ENCODED` (memory, plan 1.6);
- all 247 `seahorn` runs logged;
- the lit pass counts are unchanged;
- the checks took 4.2 s in all, at most 14 ms each.

[OBS 2026-11-10] F2 and F3 in `units_bitwuzla`:
- every operator's encoding on 8 bits is `SAME`;
- wrong terms are `DIFFERENT`:
  - `bvult` against `bvslt`;
  - swapped `bvsub` arguments;
  - a wider sort, a narrower sort, and Bool against bit-vector;
  - a missing constant;
  - the reviewer's false-`SAME` case;
- all 600 fuzzed formulas are `SAME`;
- the 64-bit identity `x = (x udiv y)·y + (x urem y)` with a 1 ms limit
  is `UNDECIDED`.

[OBS 2026-11-10] F2, bugs planted in the encoder, at lit level
(`MUTATE_MODE=lit`, opsem;
`~/builds/seahorn/results/2026-11-10-mutation-lit.log`): 11 of 14 caught,
with 8 to 464 `DIFFERENT` formulas each. The three survivors:
- "extract indices swapped" makes Bitwuzla reject the term (218
  `NOT-ENCODED` instead of 67). That is a safe failure: `unknown`, never
  a wrong formula.
- The sign-extend ↔ zero-extend swaps leave every formula `SAME`. A probe
  that rejects extensions shows they occur in 2 encodable formulas (69
  `NOT-ENCODED` instead of 67), extending `0:bv(1)`, `1:bv(1)` and the low
  bits of an `extract`. `SAME` is a proof, so Z3 proved the swapped
  formulas equal to the correct ones there: the two extensions agree in
  those contexts. The unit tests catch both swaps.

[OBS 2026-11-10] F4: unchanged with the option off. `verify_all` reruns
the lit answers on the new code; see the table below.

[OBS 2026-11-10] Bugs planted in the check itself, written by a separate
agent that had not seen the tests:
- round 1, 14 bugs: 10 caught at first. Three survivors were gaps in my
  tests, now closed:
  - only the wider-sort case was tested;
  - nothing tested the option's log (two survivors).
  The fourth, "duplicate names allowed", is equivalent: Bitwuzla prints
  both constants as `x`, and Z3's parser rejects the same name declared
  twice ("already declared"), so the result is `UNCHECKED` either way.
- round 2, 7 bugs in the code changed after the review: all 7 caught,
  after 2 more tests (rotate-right, repeat below the root, and a constant
  only in the last argument).

## Coverage (C1)

[OBS 2026-11-10] Line coverage of the four files, from `units_bitwuzla`
plus the opsem/opsem2 `smt-bw` runs with the formula check on, at commit
`1ab43e7b` (branch bitwuzla-support): 655 of 702 lines (93.3%). Every line
not run, with its reason:
- `MarshalBitwuzla.cc` (10 lines):
  - 102-105, 218-221: `default:` of the Bool and bit-vector operator
    switches. Every `BvOpKind` has a case. The only `BoolOpKind`s without
    one, TRUE and FALSE, are leaves and never reach `kindOf`.
  - 294-295: `declare` of a function with arguments. A constant (an FAPP
    without arguments) of such a declaration would be ill-typed, and an
    FAPP with arguments is not a leaf: `kindOf` rejects it first.
  - 325-326: a `bitwuzla::Exception` while encoding a numeral or constant.
    Sort errors already arrive as `UnsupportedExpr`, and the numeral is
    reduced into range. Kept so the header's promise holds.
- `BitwuzlaSolverImpl.cc` (27 lines):
  - 59-62, 132-150: the methods not implemented yet (models 1.5, cores,
    push/pop, SMT-LIB output), which exit. No test in the suites calls
    them.
  - 69-71: Bitwuzla refusing its fixed options.
  - 115-116: Bitwuzla answering unknown; no time or resource limit is set.
  - 119-121: `check_sat` throwing; only on misuse of its API.
  - 55: the error printed for a `DIFFERENT` formula. It never happens on
    correct code. The lit-level planted bugs above exercise it, but in
    `build/`, not the coverage tree.
- `BitwuzlaEncodingCheck.cc` (2 lines): 78-79, Bitwuzla throwing while the
  term is walked or printed.
- `EncodingCheck.cc` (8 lines):
  - 31: the fallback of `verdictName`; every value has a case.
  - 120: `ExprToZ` returning null. It exits on operators it lacks;
    `notForZ3` keeps the known ones away.
  - 156, 166, 169: Z3 failing to declare a Bool or bit-vector constant.
  - 189, 194, 199: Z3 failing to build `rhs`, `=` or `not` on terms whose
    sorts were checked to match.

The first coverage run, before the tests added that day, was at 84.8%.
The new tests cover these paths:
- extension to a narrower sort;
- a non-numeral repeat index;
- long error messages;
- the check's own refusals, time-out and log.

## Fresh review (rule 5)

[OBS 2026-11-10] A new agent reviewed a frozen snapshot of the code and
the spec (not my results). It found no way for the check to say `SAME`
wrongly in normal use. With the option off, seahorn is unchanged. Its
findings, double-checked and fixed:
- **False `SAME` through names.** A constant named like the helper
  constant, and `|x|` or `_let0` names. Fixed by avoiding both sides'
  names and refusing unsafe names.
- **Error checks that could not fire.** They read the error code after a
  z3++ wrapper's `Z3_inc_ref`. Fixed.
- **`verify_all` reported PASS too easily:**
  - `UNDECIDED`/`UNCHECKED` formulas only gave WARN;
  - a suite that did not run went unnoticed;
  - a crashed mutation run counted as caught;
  - the sources were not compared after mutation.

  All fixed: the formulas row also requires the checked run to give the
  same lit results as the plain run.
- **Planted bugs could stay in the sources** if `mutate_bitwuzla.py` was
  killed. Fixed with signal handling and a backup guard.
- **`compare_answers.py` counted "no answer from either" as `SAME`.**
  This corrects the 2026-11-08 count (superseded there): 106 same plus 1
  with no answer (`nham.1.ll`), not 107.

## verify_all on the committed code (V1, V2)

[OBS 2026-11-10] `bash scratch/verify_all.sh --clam --coverage --mutation`
on `1ab43e7b` (branch bitwuzla-support, pushed 2026-11-10), Bitwuzla 0.9.1, Z3 5.1.0,
clang 18.1.8, macOS 26.5.2. It took about 23 minutes and printed
"VERIFY ALL: PASS". Results in
`~/builds/seahorn/results/verify-2026-11-10-0142-1ab43e7b/`.

| Check | Result | Details |
|---|---|---|
| build | PASS |  Bitwuzla version 0.9.1 |
| units | PASS | test cases: 17 / 17 passed / 0 failed / 0 skipped; assertions: 9694 / 9694 passed / 0 failed /; fuzzer (seed 1): 4000 formulas: 3135 sat, 865 unsat, 0 unknown to z3; encoding check: 4000 same |
| examples | PASS | 7 OK, 1 pending, 0 wrong |
| selector | PASS | build: 6/6 |
| detection | PASS | 27/27 cases |
| answers | PASS | 106 same as smt-z3, 69 smt-bw unknown, 1 no answer from either, 0 different; lit smt-bw: Passed : 86 (68.25%); Expectedly Failed: 1 (0.79%); Failed : 39 (30.95%); Passed: 21 (42.00%); Failed: 29 (58.00%);  |
| formulas | PASS | 2485 same, 0 different, 0 undecided, 0 unchecked, 136 not encoded; from 247 of 247 seahorn runs |
| clam | PASS | build-clam selector: 8/8 |
| coverage | INFO | line coverage 93.30%; TOTAL 47 lines not run (each needs a reason: coverage/uncovered.txt) |
| mutation | PASS | 87/89 caught; 0 unexplained survivors, 0 did not build |

V2 holds: units 17/17, examples 7 OK and 1 pending, selector 6/6 and 8/8,
detection 27/27. The lit answers are 106/69/0 plus 1 with no answer, the
corrected form of 107/69/0. The two mutation survivors are the known
equivalent ones: "no identity extension" (2026-11-08) and "check:
duplicate names allowed" (above). An earlier run that day failed on two
planted bugs whose text no longer matched the reviewed code. They were
retargeted, and both are caught.

## The check depends on

Bitwuzla's SMT-LIB printer, Z3's SMT-LIB parser and solver, and SeaHorn's
`ExprToZ` as the reference.

A difference is always a disagreement with `ExprToZ`, which is not
always right: it keeps only two arguments of n-ary `bvmul` and others.
The unit test `bitwuzla.encoding check` records this. See
`loose-ends/parked.md`.

## Not checked

- Formulas Bitwuzla cannot encode yet (`NOT-ENCODED`: memory). The check
  says nothing about them, and the adapter answers `unknown`.
- Two different SeaHorn constants that print the same: both encoders
  then name them alike. Z3 merges them; Bitwuzla does not.
- Whether `ExprToZ` itself is right. Disagreements are investigated by
  hand.
- Linux and other Bitwuzla or Z3 versions: everything ran on macOS with
  Bitwuzla 0.9.1 and Z3 5.1.0.
