# Step 1.2: the smt-bw and bw solver selectors

Branch `bitwuzla-support`, commit `406c728e` on top of `ce226670` (1.1).
Pushed to `MidVC/seahorn` on 2026-10-15.

## What changed

[FACT] The selector chain now carries a Bitwuzla value at every link:
- `SolverKind::BITWUZLA`.
- `--horn-bmc-solver=smt-bw`, which maps to `BmcSolverKind::SMT_BITWUZLA`
  and then to `SolverKind::BITWUZLA`.
- A `SolverBmcEngine` branch.
- Path BMC's `--horn-bmc-smt-solver=bw`, plus a branch in the
  `PathBmcEngine` constructor.

There is no adapter yet, so every Bitwuzla branch stops with a message:

| Route | Built with Bitwuzla | Built without Bitwuzla |
|---|---|---|
| mono `smt-bw` (`SolverBmc.cc`) | `ERR` "The Bitwuzla solver adapter is not implemented yet", `exit(1)` | `ERR` "No Bitwuzla found. Compile SeaHorn with BITWUZLA_HOME option!", `exit(1)` |
| Path BMC `bw` (`PathBmc.cc`) | `assertion_failed` "Path bmc does not support Bitwuzla yet" | `assertion_failed` "Compile with BITWUZLA_HOME option" |

Each route copies its neighbouring Yices branch: `ERR` + `exit` in
`SolverBmc.cc`, and `assertion_failed` (which aborts) in `PathBmc.cc`.
       → seahorn/include/seahorn/Expr/Smt/Solver.hh:18,
         lib/seahorn/BmcPass.cc:62,71-72,277-278,
         lib/seahorn/SolverBmc.cc:44-52,
         lib/seahorn/PathBmc.cc:114-115,1071-1077 (branch bitwuzla-support)

Why the not-implemented exits, not a stub adapter: a stub whose `check()`
returns `ERROR` would make `runSolverBmcEngine` print `unknown` and exit 0
(`BmcPass.cc`, `runSolverBmcEngine`). That looks like a real solver
answer. 1.4 replaces `SolverBmc.cc:47-48` with the real
`make_unique<bitwuzla_solver_impl>`. 2.4 replaces `PathBmc.cc:1073-1074`.

Not changed:
- The `default:` message in the `BmcPass.cc` switch ("use either Z3 or
  Yices2") is now unreachable, because every enum value has a case. It is
  left as it was.
- `PathBmcUtil.cc` timeouts are left alone: non-Z3 solvers get no
  timeout, and that is plan 2.2.

The new lines follow `git clang-format` (LLVM 18 clang-format, repo
`.clang-format`). It also reflowed the two option lists that gained a value.

## Acceptance

[OBS 2026-10-15] `bash scratch/test_selector_1.2.sh`, saved in
`scratch/acceptance-1.2/selector-<tree>.txt`:
- **`seahorn/build`** (Bitwuzla on, no Clam): all 5 checks pass.
  - `--help-hidden` lists `=smt-bw - Use Bitwuzla interface`.
  - The default solver and `smt-z3` both still give `unsat` on
    `test_ult_unsat.ll`.
  - `smt-y2` still exits 1 with "No yices2".
  - `smt-bw` exits 1 with "...adapter is not implemented yet".
- **`seahorn/build-clam`** (no Bitwuzla, Clam on): all 8 checks pass.
  - The same 5 checks pass, except that `smt-bw` exits 1 with
    "No Bitwuzla found...".
  - `--help-hidden` lists `=bw - Bitwuzla SMT solver` under
    `--horn-bmc-smt-solver`.
  - Path BMC with `z3` gives `sat` on `test/bmc/test-bmc-2.false.c`.
  - Path BMC with `bw` aborts with "Compile with BITWUZLA_HOME option".
    `sea` reports exit 250.

One combination is not tested: Path BMC `bw` with Bitwuzla compiled in.
No tree has both Clam and Bitwuzla until plan 2.4. It is a single
`assertion_failed` line.

[OBS 2026-10-15] On the main build, opsem (125 + 1 XFAIL) and opsem2
(50/50) match 0.2 test for test, under both `z3direct` and `smt-z3`. The
flag counts are 235/237 and 55/55 (`scratch/baseline-1.2/`).

[OBS 2026-10-15] On `build-clam`, `test/bmc` passes 20/20 under `default`
and under `smt-z3`, matching 0.3 test for test. The flag reached all 71
commands (`scratch/baseline-1.2/build-clam/`).

[OBS 2026-10-15] Rebuild cost: 9 files on the main tree (10 s), and 10 on
`build-clam`. `units_z3` on `build-clam` passes 5, with `expr.neg`
excluded as before.

## Script pitfall

[OBS 2026-10-15] The first version of the test script detected
`clam=no` on `build-clam`. The script runs with `set -o pipefail`, and
`seahorn --help-hidden | grep -q X` fails there. `grep -q` exits at the
first match, `seahorn` is killed by SIGPIPE while still writing its
help, and pipefail reports the pipeline as failed. The fix was to
capture the output into a variable first. The same trap applies to any
`set -o pipefail` script that pipes a long output into `grep -q`.

## See also

- 2026-10-14-bitwuzla-build-integration.md (1.1)
- ../../durable/adding-a-solver-backend-touches-these-files.md
