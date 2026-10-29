# CI fix: lit's opsem_opts key, and opsem2 forwarding

The professor asked for the CI bug from 0.2 to be fixed in its own branch.
Branch `fix-ci-opsem-opts` is based on `dev18` @ `839a3d57`. Upstream
`dev18` had no new commits on 2026-10-21. The branch is pushed to the fork
`MidVC/seahorn`. The user opens the PR to `seahorn/seahorn` (base `dev18`).

| commit | change |
|---|---|
| `690e462a` `ci: pass opsem_opts to lit with the correct key` | `-D opsem-opts=` → `-D opsem_opts=` in `.github/workflows/test-seahorn-docker.yml:63,65` (push/PR CI), `.github/workflows/seahorn-docker.yml:72,74` (nightly), `scripts/coverage/docker-run-lcov.sh:13,14`, `.travis.yml:28-29,38-39`. 10 lines in total; `git grep opsem-opts` is now empty. |
| `cc65c4f6` `test(opsem2): forward opsem_opts to %sea and %fpfsea` | `test/opsem2/lit.cfg`: `sea` and `sea_fpf` get `+ [opsem_extra_params]`, the same pattern `seabmc` already uses. |

[FACT] `test/opsem/lit.cfg:26` and `test/opsem2/lit.cfg:26` read
`lit_config.params.get('opsem_opts', "")`. lit stores `-D key=value`
under the exact key and does not warn about unused keys. So the hyphen
spelling is silently ignored.

[FACT] opsem2 tests use `%sea` (47 files) and `%fpfsea` (3). None use
`%seabmc`, the only command that received `opsem_opts` before the fix.
       → seahorn/test/opsem2/lit.cfg:64-80 (dev18)

[OBS 2026-10-21] `bash scratch/verify_ci_opsem_opts.sh` (output in
`scratch/ci-fix-verify/`, summary in `scratch/ci-fix-verify.summary`).
It runs the workflow's `lit` arguments against the local install of the
main build. That binary was built from `bitwuzla-support`; the
opsem/opsem2 suites do not touch its Bitwuzla changes, and 0.2 got the
same results on a pure `dev18` binary.

| run | flag in seahorn commands | result |
|---|---|---|
| opsem, `-D opsem-opts=--horn-bv2-widemem` (old key) | 0 / 237 | 125 + 1 XFAIL |
| opsem, `-D opsem_opts=--horn-bv2-widemem` | 235 / 237 | 125 + 1 XFAIL |
| opsem, `-D opsem_opts=--horn-bv2-extra-widemem` | 235 / 237 | 125 + 1 XFAIL |
| patched opsem2, no `opsem_opts` (what CI runs) | 0 / 55 | 50/50 |
| patched opsem2, `-D opsem_opts=--horn-bmc-solver=smt-z3` | 55 / 55 | 50/50 |

The 2 opsem commands without the flag are `%seasmt`, which never takes
`opsem_opts`. Nothing in CI's own opsem2 run changes.

GitHub Actions were not run. The workflows need the Docker images, and
Actions are off on the fork. The workflow change is the key spelling only.

Not included, though they are still in plan 4.3:
- a `lit_config.fatal` on the hyphen key, to stop the mistake recurring;
- a CI job for the generic `smt-z3` route.

Caution for whoever passes memory flags to opsem2: several opsem2 tests
(e.g. `trackingmem/*`) set their own memory model. A memory-model flag in
`opsem_opts` would now be added to theirs. Solver flags are fine.

## See also

- 2026-10-14-lit-baselines.md (where the bug was found)
- ../../loose-ends/parked.md
