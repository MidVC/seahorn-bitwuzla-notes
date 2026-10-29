# Parked

## Build Bitwuzla 0.9.1 locally and make CMake find it
**Status:** parked 2026-10-02
**Progress 2026-10-14:** Build and standalone link completed (plan 0.1);
see `journal/2026-10/2026-10-14-bitwuzla-build-and-link.md`.
Only SeaHorn CMake integration remains (plan 1.1). Context below records
the original parked state.
**Context:** This is the first prerequisite for any adapter code. The
   course notes covered only the 0.9.1 API and build sources. No library
   has been built or linked yet. SeaHorn finds optional solvers with a
   `find_package(...)` call that sets `-DWITH_<X>` (Yices pattern at
   CMakeLists.txt:164-168).
**Why parked:** Bootstrap session; notes setup was the priority.
**To resume:** Try `brew info bitwuzla` first, otherwise build from source
   (Meson + Ninja; needs GMP, CaDiCaL, SymFPU). Write a 10-line C++
   program that checks `x + 1 = 0` on 8-bit `x` and prints `x`. Then
   draft `cmake/FindBitwuzla.cmake`.
**Effort estimate:** ~half-day
**References:** durable/adding-a-solver-backend-touches-these-files.md

## Decide whether to install Yices2 as a second reference adapter
**Status:** parked 2026-10-02
**Context:** Yices2 is NOTFOUND in the current build, so only `smt-z3` can be
   run as a generic adapter. The Yices adapter is the closest structural
   model for Bitwuzla (marshal file, assumption map). Running it would let
   us compare three solvers.
**Why parked:** Not needed until a differential harness exists.
**Progress 2026-10-14 (plan 0.3):** Homebrew core now has `yices2` 2.7.0,
   but SeaHorn's adapter does not compile against it (status enum renamed;
   see durable/adding-a-solver-backend-touches-these-files.md). Installing it
   globally also turns `WITH_YICES2` on in the main build at the next
   reconfigure, so it was uninstalled again. Decision: no Yices in either
   build until plan 1.8.
**To resume:** Pick one. (a) Build Yices 2.6.x (CI uses 2.6.1) into
   `~/opt/yices-2.6.x`, then pass `-DYICES2_HOME=` explicitly; this needs no
   SeaHorn change. (b) A small upstream fix that accepts both enum spellings
   (guard on `__YICES_VERSION_MAJOR`); this needs the professor's OK, like the CI fix.
   Then `YICES2=... bash scratch/build_seahorn_clam.sh` and
   `bash scratch/run_bmc_baseline_0.3.sh y2`.
**Effort estimate:** ~1 hour for (a)
**References:** journal/2026-10/2026-10-02-bootstrap-and-z3-baseline.md

## Verify course notes §5–§8 against source
**Status:** parked 2026-10-02
**Context:** The BV operation map, memory representation, Expr internals,
   and Bitwuzla API claims are still secondary-source. The encoder depends
   on them being right (division-by-zero semantics, extract/extend indices,
   wide numerals).
**Why parked:** The integration-path sections came first.
**To resume:** For each operation in §5.2, find its case in
   `MarshalYices.cc` and the Bitwuzla `Kind`. Record the result in
   `durable/bv-op-map-seahorn-to-bitwuzla.md`.
**Effort estimate:** ~2 hours
**References:** URA/SEAHORN_BITWUZLA_COURSE_NOTES.md §5–§8

## Upstream fix: lit opsem_opts never reaches CI widemem runs or opsem2
**Status:** parked 2026-10-14. **Resumed and fixed 2026-10-21** on branch
   `fix-ci-opsem-opts` (fork `MidVC/seahorn`), because the professor asked
   for the fix in a separate branch. The PR to `seahorn/seahorn` `dev18` is
   being opened by the user. See `journal/2026-10/2026-10-21-ci-opsem-opts-fix.md`.
**Context:** CI passes `-D opsem-opts=` but `lit.cfg` reads `opsem_opts`,
   so the widemem/extra-widemem CI jobs test the default memory model.
   Separately, `opsem2/lit.cfg` appends the params only to `%seabmc`, which
   no opsem2 test uses. Both were confirmed by counting the flag in executed commands.
**Why parked:** Not on the Bitwuzla critical path. The local script works around it.
**To resume:** Change `seahorn-docker.yml:72,74` to `opsem_opts` (or make
   lit.cfg accept both spellings), and append `opsem_extra_params` to `sea`
   and `sea_fpf` in `opsem2/lit.cfg`. (Already checked locally: with the
   flag applied, widemem and extra-widemem pass 125 + 1 XFAIL, so the fix
   uncovers no failures.)
**Effort estimate:** ~1 hour
**References:** journal/2026-10/2026-10-14-lit-baselines.md

## Test AUTO detection against a real system-wide Bitwuzla (Homebrew)
**Status:** parked 2026-10-22
**Context:** Since 2026-10-22, `WITH_BITWUZLA=AUTO` searches the system
prefixes, which include `/opt/homebrew` on this Mac. The test script used
`CMAKE_PREFIX_PATH` as a stand-in for them. It is unknown whether
Homebrew's `bitwuzla` 0.9.1 bottle ships a shared library, and whether it
passes the link test with SeaHorn's Homebrew-LLVM-18 flags
(`-nostdinc++ -I.../llvm@18/include/c++/v1`).
**Why parked:** it needs `brew install bitwuzla`, which changes the
machine, and it would also turn Bitwuzla on in `build-clam/`. That is the
user's call.
**To resume:**
1. `brew install bitwuzla`.
2. Configure a throwaway tree with the test script's base flags and no
   Bitwuzla settings. Expect "Found Bitwuzla library:
   /opt/homebrew/lib/libbitwuzla.dylib" or an "Ignoring" warning.
3. Check that `build/` still uses `BITWUZLA_HOME`.
4. `brew uninstall bitwuzla`, or set `WITH_BITWUZLA=OFF` in `build-clam/`.

The test script skips its "nothing installed" cases while a system
Bitwuzla exists.
**Effort estimate:** ~30 min
**References:** journal/2026-10/2026-10-22-bitwuzla-smart-detection.md

