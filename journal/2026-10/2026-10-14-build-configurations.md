# Step 0.3: two build trees (main WITH_CLAM=OFF, build-clam for Path BMC), no Yices yet

Decision, recorded so later `[EMP]` claims can name their build:

| tree | `WITH_CLAM` | Yices | Bitwuzla | used for |
|---|---|---|---|---|
| `seahorn/build` | OFF (same as upstream CI) | not found | added in plan 1.1 | Phase 1, 2.1–2.3, mono BMC in Phase 3 |
| `seahorn/build-clam` | ON | not found | added after 1.1 | Phase 2.4 (Path BMC), `test/bmc` |

Both trees share: SeaHorn `dev18` @ `839a3d57`, Homebrew Clang 18.1.8,
`RelWithDebInfo`, Homebrew Z3 5.1.0, Boost 1.92.0, and the same
`CMAKE_CXX_FLAGS` (below). `build-clam` adds Clam `dev18` @ `3eaf016` and
crab `dev` @ `49773d0a`. Crab was cloned by hand to `seahorn/clam/crab`,
because `extra` does not clone it. `build-clam` is 3.6 GB and lives in
OneDrive. Rebuild with `bash scratch/build_seahorn_clam.sh`.

## The main build has non-default C++ flags

[FACT] `seahorn/build/CMakeCache.txt` has
`CMAKE_CXX_FLAGS=-nostdinc++ -I/opt/homebrew/opt/llvm@18/include/c++/v1 -D_LIBCPP_ENABLE_CXX17_REMOVED_UNARY_BINARY_FUNCTION`
and `BOOST_ROOT=/opt/homebrew/opt/boost`. These were not written down anywhere before.
[OBS 2026-10-14] Without them, sea-dsa fails at `sea-dsa/include/seadsa/Info.hh:90`:
`std::unary_function` is removed in LLVM 18's libc++ in C++17 mode
(`scratch/build-clam-attempt1-libcxx.log`). Any new build tree, and any
standalone program that includes SeaHorn headers, needs these flags.

[OBS 2026-10-14] `scratch/bw_hello.cpp` compiled with exactly these flags
links and prints `sat` / `x = #b11111111`. The program, `libbitwuzla`, and
`seahorn` all link `/usr/lib/libc++.1.dylib`. So plan 1.1 should not hit a
header or ABI mismatch between SeaHorn and Bitwuzla.

## Baselines on build-clam

[OBS 2026-10-14] opsem: 125 pass + 1 XFAIL. opsem2: 50/50. This holds under
both `z3direct` and `smt-z3`, and per test it matches the main build
(`scratch/baseline-0.2/build-clam/`). Clam does not change mono-BMC answers.

[OBS 2026-10-14] `test/bmc`: 20/20 pass under `default` and under `smt-z3`
(`scratch/baseline-0.3/build-clam/`). It runs 71 `sea bpf` commands: 33 mono and 38 Path BMC.
The Path BMC commands split into 18 with `crab=false` + `muc=assume`, 2 with
`crab=true` + `muc=assume`, and 18 with `crab=true` + the default MUC. Summed
test time is 118.9 s (default) and 95.7 s (smt-z3). The slowest test is
`cdaudio_simpl1_false` (21.5 s / 19.3 s). The flag guard confirmed 71/71
commands carried `--horn-bmc-solver=smt-z3`.

```sh
bash scratch/run_bmc_baseline_0.3.sh                 # default + smt-z3
bash scratch/run_bmc_baseline_0.3.sh bw              # Phase 2.4
BUILD=build-clam bash scratch/run_baseline_0.2.sh    # opsem/opsem2 on this tree
```

[OBS 2026-10-14] As a control, the main build cannot run `test/bmc` at all.
`--horn-bmc-crab` is compiled out, so `sea bpf ... --bmc=path` stops at
option parsing and prints the help text. On `build-clam` the same command prints `sat`
plus `BMC path-based:` statistics.

[FACT] `test/bmc` has no lit.cfg of its own and uses `test/lit.cfg`, which
takes no parameters. The runner copies the suite and maps `%sea bpf` to
`sea bpf <sea_extra>`. Path BMC's solver is chosen by
`--horn-bmc-smt-solver`; mono's is chosen by `--horn-bmc-solver`.

## units_z3 has one pre-existing failure with Z3 5.1.0

[OBS 2026-10-14] `units_z3` test `expr.neg` (`units/units_z3.cpp:66`) fails
in **both** trees. It compares `ZSimplifier` output syntactically. It
expects `x=0 ∨ 2 ≤u x`, and Z3 5.1.0 returns
`x=0 ∨ 2 ≤u extract(1,0,x) ∨ extract(63,2,x) ≠ 0`. The two formulas are
equivalent. The other 5 cases pass. [HYP] CI's Z3 4.8.9 produces the
expected form. Lesson for plan 1.6: `units_bitwuzla` should check answers
and semantics, not the syntax of simplifier output.

## Yices: tried, backed out

[OBS 2026-10-14] Homebrew core has `yices2` 2.7.0, but
`Yices2SolverImpl.cc` does not compile against it (enum rename; the
`[FACT]` is in `durable/adding-a-solver-backend-touches-these-files.md`,
log in `scratch/build-clam-attempt2-yices27.log`). Because `FindYices2`
auto-detects `/opt/homebrew`, leaving 2.7.0 installed would have broken the
main build at its next reconfigure. I uninstalled it. Options are in
`loose-ends/parked.md`.

## Correction to the plan

Plan 2.2 said Path BMC's timeout code does an undefined-behavior cast when
Yices is selected. That was wrong: the cast is behind
`get_kind() == SolverKind::Z3` (`PathBmcUtil.cc:14,30`). The real issue is
that non-Z3 solvers silently get no timeout. The plan is fixed. Why I was
misled: I had recorded "static_cast + Yices TODO" without reading the guard
on the line above. The durable FACT itself was accurate, and it now mentions the guard.

## See also

- 2026-10-14-lit-baselines.md
- ../../durable/adding-a-solver-backend-touches-these-files.md
- ../../loose-ends/parked.md
