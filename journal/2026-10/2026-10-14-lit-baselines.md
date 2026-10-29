# Step 0.2: lit baselines show smt-z3 = direct z3 on opsem/opsem2; two test-harness traps

[OBS 2026-10-14] With SeaHorn `dev18` @ `839a3d57` (install tree
`seahorn/build/run`), Homebrew Z3 5.1.0, and lit 23.1.3 (venv
`~/opt/venvs/lit`), the direct `z3` engine and the generic `smt-z3`
adapter give identical per-test results:

| suite | tests | z3direct | smt-z3 | summed test time z3direct / smt-z3 |
|---|---|---|---|---|
| opsem | 126 | 125 pass + 1 XFAIL (`alloca.09.ll`) | same | 16.9 s / 20.3 s |
| opsem2 (patched copy, see below) | 50 | 50 pass | same | 14.9 s / 12.2 s |

```sh
bash scratch/run_baseline_0.2.sh          # z3direct + smt-z3
bash scratch/run_baseline_0.2.sh smt-bw   # later, for Phase 1.7
```

Outputs are `scratch/baseline-0.2/{opsem,opsem2}-{z3direct,smt-z3}.{log,xml}`.
For each run, the script checks that lit printed `Passing extra opsem
command line args` and counts how many executed `seahorn`/`sea` commands
contain the flag: opsem 235/237 (the other 2 are `%seasmt` in `nham.1.ll`,
which never takes extra options), opsem2 55/55.

Our reading: the generic `SolverBmcEngine` path has no functional gap
against the direct engine on these suites. So any `smt-bw` failure in
Phase 1.7 is a Bitwuzla adapter issue, not an inherited limitation of the
generic path. The `smt-z3` lists are the comparison baseline.

[OBS 2026-10-14] `memset.symbolic.unaligned.unsat.ll` takes 12.5 s
(smt-z3) and 9.9 s (z3direct). lit's limit is 30 s (`opsem/lit.cfg:20`).
It is the only opsem test above 1 s. [HYP] This is the first place a
slower Bitwuzla configuration would show up as a timeout rather than a
wrong answer.

## Trap 1: CI's `-D opsem-opts=` is a no-op (confirms the plan's suspicion)

[FACT] lit does not normalize parameter keys. `opsem/lit.cfg:26` reads
`opsem_opts`. Running `test_ult_unsat.ll` with `-D opsem-opts=--horn-bv2-widemem`
puts the flag in 0 executed commands and prints no "Passing extra" line.
With `opsem_opts` it appears in all of them.
       → seahorn/.github/workflows/seahorn-docker.yml:72,74
[OBS 2026-10-14] Consequence: the CI jobs labelled widemem and
extra-widemem run test/opsem with the default memory model. Candidate
small upstream fix. Parked.

[OBS 2026-10-14] I ran them locally with the flag actually applied
(235/237 commands flagged, direct z3 engine). Both `--horn-bv2-widemem` and
`--horn-bv2-extra-widemem` give 125 pass + 1 XFAIL, the same as the default
model. So the CI mistake has not been hiding failures, at this commit. These
are also the struct-sorted memory models plan 2.1 must support, and now
they have a known-good baseline (`opsem-z3direct-{widemem,extra-widemem}.*`).

## Trap 2: opsem2 never receives `opsem_opts`

[FACT] `opsem2/lit.cfg:66` appends `opsem_extra_params` only to
`%seabmc`. `%sea` and `%fpfsea` (lines 71-76) do not get it, and all 50
opsem2 tests use `%sea`/`%fpfsea` (52 + 3 RUN lines, 0 `%seabmc`). So
`-D opsem_opts=--horn-bmc-solver=...` silently does nothing on opsem2.
Workaround, without touching the repo: the script copies the suite to
`scratch/baseline-0.2/opsem2-patched/` and appends the params to those two
substitutions. `sea bpf` forwards `--horn-bmc-solver` to the native binary
(checked with `--dry`, and `--log=bmc` prints `bmc using Z3` from SolverBmc.cc).

[FACT] The opsem2 `%sea` substitution includes `--horn-bv2-vacuity-check=all`.
So every opsem2 run also exercises opsem's built-in Z3 vacuity solver
(`BvOpSem2.cc:2836`), even under `smt-bw`. That is relevant to plan 2.3,
and to any timing comparison.

## Why I was misled (my first attempt)

My first `smt-z3` runs reported identical pass counts, but lit had
received no parameter. In zsh, an unquoted `$P="-D opsem_opts=..."` is
not word-split, so lit got one argument and the key gained a leading
space. Identical results between two configurations are exactly what a
silently ignored flag looks like. Always check that the flag reached the
executed command (the script now enforces this). The invalid logs were
deleted.

## Housekeeping

lit's exec root is the current directory (`lit.cfg:17`), so runs started
inside the install tree left `Output/ ownsem/ trackingmem/ widemem/` in
`seahorn/build/run/share/seahorn/`. Those were removed. The script now
runs lit from a temp dir.

## See also

- adding-a-solver-backend-touches-these-files.md
- ../../loose-ends/parked.md
