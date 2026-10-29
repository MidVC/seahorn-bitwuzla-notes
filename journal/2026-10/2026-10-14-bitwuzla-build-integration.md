# Step 1.1: Bitwuzla build integration (opt-in through BITWUZLA_HOME)

Branch `bitwuzla-support` from `dev18` @ `839a3d57`. Committed 2026-10-15
as `ce226670` and pushed to the fork `MidVC/seahorn`. Files touched:

- `cmake/FindBitwuzla.cmake` (new)
- `CMakeLists.txt` (the block after Yices)
- `lib/seahorn/Smt/CMakeLists.txt` (link line)
- `tools/seahorn/seahorn.cpp` (a `--version` line)

## What the build does now

[SUPERSEDED → 2026-10-22-bitwuzla-smart-detection.md] On 2026-10-22 the
opt-in design below was replaced by `WITH_BITWUZLA=AUTO|ON|OFF` with
smart detection. `BITWUZLA_HOME` still pins one prefix and makes
Bitwuzla required. The opt-in, cache-reset and "pass an empty
`BITWUZLA_HOME` to turn it off" items below describe commit `ce226670`,
not the current tree.

[FACT — as of `ce226670`] Bitwuzla is searched for only when `BITWUZLA_HOME` is set, and only
in that prefix (`NO_DEFAULT_PATH`). When it is set, the search is
`find_package(Bitwuzla 0.9 REQUIRED)`, followed by `add_definitions(-DWITH_BITWUZLA)` and
`include_directories`. `SeaSmt` links the library PRIVATE, and `seahorn`
gets it through that.
       → seahorn/CMakeLists.txt:177-189, lib/seahorn/Smt/CMakeLists.txt:20-22
         (branch bitwuzla-support)

Why each choice was made:

- **Opt-in, unlike Yices.** (Reversed 2026-10-22; see the note at the top.) `FindYices2` auto-detects default prefixes,
  and 0.3 showed that this silently changes builds. Homebrew also has a
  `bitwuzla` 0.9.1 formula (not installed here), so a `brew install`
  would have turned Bitwuzla on in every tree. This settles the decision
  plan 1.1 left open. The plan's pkg-config fallback was dropped because
  it only makes sense with auto-detection. The `.pc` file is still read,
  for the version. For a system-wide install, pass `-DBITWUZLA_HOME=/usr`.
- **REQUIRED.** If `BITWUZLA_HOME` is given but the library cannot be
  used, configure fails. Yices instead falls back silently to a build
  without it.
- **The cache is reset when `BITWUZLA_HOME` changes.** (Now the module
  searches again on every configure.) Otherwise
  `find_library` keeps the old prefix's cached path.
- **Shared only.** [FACT] SeaHorn puts `.a` first in
  `CMAKE_FIND_LIBRARY_SUFFIXES` before any solver search
  (`CMakeLists.txt:116`). Bitwuzla's own default is static
  (`bitwuzla-0.9.1/meson.build:11`), and a static `libbitwuzla.a` also
  needs `libbitwuzla{ls,bv,bb}` (`Libs.private` in `bitwuzla.pc`) plus
  GMP/MPFR. So the module asks for `libbitwuzla.dylib`/`.so` by its full
  name first, and rejects a static library with a "build with --shared"
  warning.
- **The version comes from the `bitwuzla.pc` installed next to the
  library.** [FACT] The headers have no version macro, only the runtime
  `bitwuzla::version()` (`include/bitwuzla/cpp/bitwuzla.h:102`).
- **The library is not installed with SeaHorn.** [FACT] On macOS,
  `libbitwuzla.0.dylib` has an absolute install name, and it refers to
  its sibling libraries by absolute path too. Copying it into `run/lib` would not help.
  [HYP] On Linux, `CMAKE_INSTALL_RPATH_USE_LINK_PATH` (`CMakeLists.txt:118`)
  puts `BITWUZLA_HOME/lib` in the installed RPATH. Untested; Phase 4 CI will
  show it.
- **`seahorn --version` prints `Bitwuzla version 0.9.1`.** This makes the
  acceptance check a real symbol reference, not just a load command.
  It also lets later `[EMP]` records take the solver version from the
  binary itself.
- The three adapter `.cc` files from plan item 3 are not in `SeaSmt` yet,
  because they do not exist yet. They are added in 1.3–1.5.

## Acceptance

[OBS 2026-10-14] Main tree `seahorn/build`, built without
`BITWUZLA_HOME`:
- `otool -L` and `nm -u` show nothing from Bitwuzla.
- `build.ninja` has no `-DWITH_BITWUZLA`.
- `--version` has no Bitwuzla line.

Evidence: `scratch/acceptance-1.1/disabled.txt`.

[OBS 2026-10-14] The same tree reconfigured with
`-DBITWUZLA_HOME=$HOME/opt/bitwuzla-0.9.1`:
- `otool -L` shows
  `/Users/hanyanglu/opt/bitwuzla-0.9.1/lib/libbitwuzla.0.dylib`.
- `nm -u` shows `bitwuzla::version()`.
- `--version` prints `Bitwuzla version 0.9.1`.
- The compiler and linker warning counts are identical to the disabled
  build. Nearly all warnings are pre-existing ld "built for newer macOS"
  warnings from `libLLVMSupport.a`.

Evidence: `scratch/acceptance-1.1/enabled.txt`, plus
`scratch/build-1.1-{disabled,enabled}.log`.

[OBS 2026-10-14] `bash scratch/test_find_bitwuzla_1.1.sh` passes all
7 cases (configure-only, in a throwaway tree, about 7 s):
- no home → off;
- 0.9.1 → on;
- a switch to a missing prefix → fails (this proves the cache reset);
- switching back → on;
- a static-only prefix → fails with the warning;
- a prefix whose `.pc` says 0.8.0 → fails;
- clearing the home → off.

Output: `scratch/acceptance-1.1/find-bitwuzla-tests.txt`.

[OBS 2026-10-14] On the Bitwuzla-enabled main build, opsem (125 +
1 XFAIL) and opsem2 (50/50) match the 0.2 baseline test for test, under
both `z3direct` and `smt-z3`. The flag reached 235/237 and 55/55 commands,
the same counts as 0.2 (`scratch/baseline-1.1/`, summary in
`scratch/baseline-1.1.summary`). The smt-z3 smoke test is still `unsat`.

## The main tree was out of date before today

[OBS 2026-10-14] The first rebuild on this branch, with Bitwuzla
disabled, recompiled all 201 C++ files. That includes `sea-dsa` and
`llvm-seahorn`, which this branch does not touch, and whose sources are
dated 2026-09-30. A plain reconfigure afterwards triggers no rebuild. So
the old objects did not match their current build commands, and my
change did not cause the full rebuild. The cause is unknown. Because the
results above are identical to 0.2, the 0.2 baseline still stands.

A full rebuild of the main tree takes about 70 s wall time.

## State after this step

- `seahorn/build` now has Bitwuzla enabled. To reproduce:
  ```sh
  cmake seahorn/build -DBITWUZLA_HOME=$HOME/opt/bitwuzla-0.9.1
  cmake --build seahorn/build --target install
  ```
  Pass `-DBITWUZLA_HOME=` (empty) to turn it off again. (Since
  2026-10-22: `-DWITH_BITWUZLA=OFF`. An empty `BITWUZLA_HOME` now means
  `AUTO`, which is off here only because nothing else is installed.)
- `seahorn/build-clam` was not touched. It builds from the same checkout,
  so its next build reconfigures with Bitwuzla off until 2.4 adds
  `-DBITWUZLA_HOME` to `scratch/build_seahorn_clam.sh`.
- `scratch/run_baseline_0.2.sh` now takes `OUT=` so a rerun does not
  overwrite the 0.2 results.

## See also

- 2026-10-22-bitwuzla-smart-detection.md (superseded by: smart detection)
- 2026-10-14-build-configurations.md (the FindYices2 lesson, the CXX flags)
- 2026-10-14-bitwuzla-build-and-link.md (how Bitwuzla itself was built)
- ../../durable/adding-a-solver-backend-touches-these-files.md
