# Adding a solver backend to SeaHorn BMC: the selection path and files to touch

Load this note before wiring up a new `--horn-bmc-solver` value or before
changing CMake to find a new solver. Verified against SeaHorn `dev18` @ `839a3d57`.

[FACT] There are two solver enums. `BmcSolverKind` holds the command-line
choices (`z3`, `smt-z3`, `smt-y2`, default `z3`, hidden option).
`SolverKind` holds the generic adapter identities (`Z3`, `YICES2`).
       → seahorn/lib/seahorn/BmcPass.cc:62
       ```cpp
       enum class BmcSolverKind { Z3, SMT_Z3, SMT_YICES2 };
       ```
       → seahorn/include/seahorn/Expr/Smt/Solver.hh:18
       ```cpp
       enum class SolverKind { Z3 , YICES2};
       ```

[FACT] `z3` goes through the direct, Z3-dependent `BmcEngine`. Every `smt-*`
choice is mapped by a switch to `SolverKind` and builds a `SolverBmcEngine`.
An unmapped kind logs a message and returns false.
       → seahorn/lib/seahorn/BmcPass.cc:252-285

[FACT] The `SolverBmcEngine` constructor creates the concrete adapter by
`SolverKind`. Yices is wrapped in `#ifdef WITH_YICES2`. If Yices is
requested but was not compiled in, SeaHorn prints an error and calls
`std::exit(1)`.
       → seahorn/lib/seahorn/SolverBmc.cc:28-45

[FACT] CMake finds Yices with `find_package(Yices2 2.6.0)`. When it is
found, CMake adds `-DWITH_YICES2`. This is the pattern for an optional
`WITH_BITWUZLA`.
       → seahorn/CMakeLists.txt:164-168

[FACT] With `YICES2_HOME` empty, `FindYices2.cmake` searches the default
prefixes (`/opt/homebrew` on this Mac). `2.6.0` is only a minimum. So
installing any Yices globally turns `WITH_YICES2` on at the next
reconfigure, even in a build that never asked for it. A `FindBitwuzla`
modeled on this inherits the same behavior. Decide whether that is wanted
(plan 1.1).
       → seahorn/cmake/FindYices2.cmake:8-24

[FACT] SeaHorn's Yices adapter does not compile against Yices 2.7.0.
It switches on `STATUS_SAT`, `STATUS_UNSAT`, `STATUS_UNKNOWN`, and
`STATUS_ERROR`, which 2.7.0 renamed to `YICES_STATUS_*`. Only
`STATUS_INTERRUPTED` survives, as a compatibility macro. Configure accepts
2.7.0, and the compile fails. Upstream CI pins Yices 2.6.1.
       → seahorn/lib/seahorn/Smt/Yices2SolverImpl.cc:182-186, 211-215
       → yices 2.7.0 `yices_types.h:77-85`, `yices.h:63`
       → seahorn/docker/build_seahorn.sh (`-DYICES2_HOME=/opt/yices-2.6.1`)

[FACT] `CMakeLists.txt:116` puts `.a` first in
`CMAKE_FIND_LIBRARY_SUFFIXES` before any solver is searched for. A find
module for a solver that ships both kinds of library gets the static one
unless it asks for the shared library by its full file name.
       → seahorn/CMakeLists.txt:116

[SUPERSEDED → journal/2026-10/2026-10-22-bitwuzla-smart-detection.md]
(true of commit `ce226670`, replaced 2026-10-22) Bitwuzla follows a
different pattern from Yices: it is opt-in.
It is searched for only when `BITWUZLA_HOME` is set, and only in that
prefix. Then the search is `REQUIRED`, and only a shared library is
accepted. Why: see `journal/2026-10/2026-10-14-bitwuzla-build-integration.md`.
       → seahorn/cmake/FindBitwuzla.cmake, seahorn/CMakeLists.txt:177-189
         (branch bitwuzla-support)

[FACT] Bitwuzla detection (branch `bitwuzla-support`, commit `fa82ad3e`):
- **`WITH_BITWUZLA=AUTO|ON|OFF`.** The default is `AUTO`.
- **`BITWUZLA_HOME`** (the CMake setting, or the environment variable)
  makes only that prefix searched and makes Bitwuzla required.
- **Otherwise the search order is** `Bitwuzla_ROOT`, then
  `CMAKE_PREFIX_PATH`, then pkg-config's `--variable=libdir`, then the
  system prefixes.
- **A library is rejected** if it is static, if its own prefix has no
  header, or if a link test with SeaHorn's compiler and flags fails.
  Under `AUTO` a rejection means "build without", with a warning.

Why the opt-in decision was reversed: see the 2026-10-22 journal.
       → seahorn/cmake/FindBitwuzla.cmake, seahorn/CMakeLists.txt:177-214

[FACT] Writing a find module for SeaHorn:
- **`find_package_handle_standard_args` and versions.** It skips the
  version check when the `VERSION_VAR` variable is undefined, but fails
  it when the variable is the empty string. So `unset()` the version
  variable when it is unknown; don't set it to `""`.
       → cmake 4.4.3 Modules/FindPackageHandleStandardArgs.cmake:594-607
- **`check_cxx_source_compiles` caches its result** in the variable you
  name. To rerun the check after the inputs change, `unset(... CACHE)`
  that variable.

## Files a new backend touches

1. `include/seahorn/Expr/Smt/Solver.hh`: add a new `SolverKind` value.
2. `lib/seahorn/BmcPass.cc`: add the `BmcSolverKind` value, its `clEnumValN`, and its switch case.
3. `lib/seahorn/SolverBmc.cc`: add a guarded constructor branch.
4. `lib/seahorn/Smt/CMakeLists.txt`: compile the adapter conditionally into `SeaSmt`.
5. Top-level `CMakeLists.txt` and `cmake/Find<Solver>.cmake`: find the
   dependency and add the define.
6. `units/CMakeLists.txt` and `units/units_<solver>.cpp`: add unit tests.
7. `test/opsem*/lit.cfg`: run regressions with the new selector.
8. `lib/seahorn/PathBmc.cc`: add the `--horn-bmc-smt-solver` value and a
   `PathBmcEngine` constructor branch. This file compiles only with
   `HAVE_CLAM`, so check it in `build-clam`; the main build never compiles it.

Adding only the enum value or only the constructor branch does nothing
visible. The whole chain has to be in place.

[FACT] Bitwuzla reports errors with C++ exceptions, while SeaHorn may be
compiled with `-fno-exceptions` (an LLVM built without exception support
adds that flag). So every file that calls Bitwuzla is compiled with
`-fexceptions`, and must catch all exceptions itself. This applies to the
`SeaSmt` sources and to `units_bitwuzla`.
       → seahorn/lib/seahorn/Smt/CMakeLists.txt, units/CMakeLists.txt
         (branch bitwuzla-support: first `0efbdadc`, reverted 2026-10-29,
         then again in plan 1.3, `959ea729`);
         journal/2026-10/2026-10-22-bitwuzla-encoder.md

[FACT] Watch out in a unit-test file that defines
`DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN`. There, doctest includes macOS's
`mach/boolean.h`, which defines `TRUE` as 1 and `FALSE` as 0. Then
`mk<TRUE>` fails to compile ("invalid explicitly-specified argument").
Fix: `#undef` both after including `sea_doctest.hh`, as
`units_bitwuzla.cpp` does (branch bitwuzla-support, `959ea729`).

## Solver-specific code outside the generic interface

[FACT] The generic `Solver` has no timeout method. Path BMC sets timeouts
with `static_cast<solver::z3_solver_impl &>`, and its Yices case is a TODO.
The cast is guarded by `get_kind() == SolverKind::Z3`, so it is safe, but
any other solver (Yices now, Bitwuzla later) silently runs with no
timeout.
       → seahorn/lib/seahorn/PathBmcUtil.cc:12-34 (guards at :14, :30)

[FACT] `--horn-bmc-smt-solver={z3,y2}` selects both Path BMC solvers
(the Boolean abstraction and the path solver) together. Any other
`SolverKind` hits `assertion_failed("Unsupported smt solver")`.
       → seahorn/lib/seahorn/PathBmc.cc:109-116, 1050-1071

[FACT] Without `HAVE_CLAM`, `PathBmc.cc` builds as a stub. The main build
`seahorn/build` has `WITH_CLAM=OFF`, so Path BMC is not exercised there,
and `test/bmc` cannot run there at all. Use `seahorn/build-clam` instead
(plan 0.3). Upstream CI also builds with `WITH_CLAM=OFF` and skips
`test/bmc`, so CI covers no Path BMC.
       → seahorn/lib/seahorn/PathBmc.cc:3, seahorn/build/CMakeCache.txt,
         seahorn/.github/workflows/test-seahorn-docker.yml (commented-out steps)

[FACT] `--horn-large-reduce` (off by default) creates a Z3 solver inside
VCGen. Horn solving (Spacer) is always Z3. So "BMC with Bitwuzla" can
still do Z3 work, and performance records must state these flags.
       → seahorn/lib/seahorn/VCGen.cc:19

## Baseline command

```sh
cd seahorn && build/run/bin/seahorn --horn-bmc --horn-bv2=true \
  --horn-bmc-solver=smt-z3 --horn-solve test/opsem/test_ult_unsat.ll
# expected: unsat
```

## See also

- generic-solver-interface-contract.md
