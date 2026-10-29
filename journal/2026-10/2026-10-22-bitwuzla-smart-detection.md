# Step 1.1 revised: smart detection of Bitwuzla (WITH_BITWUZLA=AUTO)

Supersedes the opt-in decision in 2026-10-14-bitwuzla-build-integration.md.
The user asked for smart detection on 2026-10-22, after comparing ten
detection methods with the 1.1 module. Implemented in the working tree of
`bitwuzla-support` on top of `406c728e`. Committed 2026-10-22 as
`fa82ad3e` and pushed to `MidVC/seahorn`. Files:
- `cmake/FindBitwuzla.cmake` (rewritten search)
- `CMakeLists.txt` (the block after Yices, now lines 177-214)
- `scratch/test_find_bitwuzla_1.1.sh` (extended from 7 to 25 cases)

## What the build does now

[FACT] Settings (working tree):
- **`WITH_BITWUZLA`** is `AUTO` (the default), `ON` or `OFF`. CMake
  booleans are accepted in any case. Any other value is a fatal error.
- **`BITWUZLA_HOME`** is read from the CMake setting, or else from the
  environment variable. If it is set, only that prefix is searched, and
  Bitwuzla becomes required, even under `AUTO`. `OFF` wins over it.
- **Without `BITWUZLA_HOME`,** the search order is:
  1. `Bitwuzla_ROOT` (CMake variable or environment variable; CMP0074 is
     NEW at `CMakeLists.txt:10`);
  2. `CMAKE_PREFIX_PATH`;
  3. the libdir pkg-config reports, passed as a HINT;
  4. the system prefixes.

       → seahorn/CMakeLists.txt:177-214, cmake/FindBitwuzla.cmake:26-44
         (branch bitwuzla-support, `fa82ad3e`)

[FACT] Checks, in order (working tree):
1. **Static libraries are rejected,** as in 1.1.
2. **The header must come from the library's own prefix.** The prefix
   is the library's directory minus `lib`, `lib64` or
   `lib/<multiarch>`, and the header must be in `<prefix>/include`.
3. **The version is read** from `<libdir>/pkgconfig/bitwuzla.pc`, as in 1.1.
4. **A link test** (`check_cxx_source_compiles`) builds a program that
   creates a `TermManager`, calls `mk_const` with a `std::string`, and runs
   `Bitwuzla::check_sat`. It uses SeaHorn's compiler, `CMAKE_CXX_FLAGS`
   and C++ standard.
   - Its result is cached in `BITWUZLA_LINKS`. It reruns only when the
     library, include directory, compiler, flags, standard or
     executable link flags change.
   - It is skipped when the version is known to be too old, so that the
     version check reports that case.
5. **Messages:** a rejected library prints "Ignoring <lib>: <reason>" and
   counts as not found. Under REQUIRED, configure then fails. Under
   `AUTO`, SeaHorn builds without Bitwuzla and prints a status line.

Both cache entries are cleared at the top of every configure. This
replaces 1.1's reset when `BITWUZLA_HOME` changes.
       → seahorn/cmake/FindBitwuzla.cmake:18-19, 49-54, 60-75, 84-118

Why each choice was made:
- **Why the opt-in decision was reversed.** 1.1 chose opt-in because a
  global install silently changes builds (the FindYices2 lesson from 0.3).
  Three things now cover that risk:
  - the link test means an unusable library found by `AUTO` cannot break
    the build;
  - configure prints the chosen library, and `seahorn --version` prints
    its version;
  - `OFF` disables the search, and `BITWUZLA_HOME` pins a prefix.

  What remains is that a `brew install bitwuzla` turns `AUTO` trees on.
  That is now the intended behaviour.
- **`BITWUZLA_HOME` is strict** rather than the first entry of a search
  order. If you name an installation and the build quietly used another
  one, a typo would go unnoticed.
- **pkg-config is asked only `--variable=libdir`.** [OBS 2026-10-22]
  With only `bitwuzla.pc` on its path, Homebrew's pkg-config (pkgconf
  3.0.8) answers `--variable=libdir`, but `--cflags` fails with "Package
  'gmp', required by 'bitwuzla', not found". CMake's `pkg_check_modules`
  uses the failing kind of query, and it also caches a found result. So
  it is not used.
- **The first library found is the one checked.** If it is unusable, no
  other installation is tried. That is standard `find_library` behaviour,
  and a test case documents it.
- **No version is not a failure.** [FACT] In CMake 4.4.3, FPHSA skips the
  version check when `VERSION_VAR` is undefined, and fails it when it is
  the empty string. So an install without a `.pc` file is accepted, with
  no version.
       → /opt/homebrew/Cellar/cmake/4.4.3/share/cmake/Modules/FindPackageHandleStandardArgs.cmake:594-607
- **Methods not used:**
  - Finding the `bitwuzla` program on `PATH` was not implemented.
    [FACT] Since CMake 3.28, `find_library` no longer derives
    `<prefix>/lib` from `PATH` entries. (That behaviour was Windows-only
    before.) So the program's location gives no automatic hint.
  - Guessing locations and building Bitwuzla automatically were skipped.
- **Only the mixed-case `Bitwuzla_ROOT` works.** [FACT] The upper-case
  `BITWUZLA_ROOT` needs CMP0144 (CMake 3.27), which is unset because
  `cmake_minimum_required` is 3.10.2.

[OBS 2026-10-22] `CMAKE_SYSTEM_PREFIX_PATH` on this Mac (CMake 4.4.3)
contains the SDK's `usr`, `/opt/homebrew`, `/usr/local`, `/usr`, `/`,
`/usr/X11R6`, `/usr/pkg`, `/opt`, `/sw` and `/opt/local`. So a Homebrew
Bitwuzla would be found, and `$HOME/opt/bitwuzla-0.9.1` would not. No
system-wide Bitwuzla is installed here.

## Acceptance

[OBS 2026-10-22] `bash scratch/test_find_bitwuzla_1.1.sh`: 25/25 cases
pass, 0 skipped, about 21 s (configure-only, in a throwaway tree):
- **The 1.1 cases still pass.** With `BITWUZLA_HOME` set, only that prefix
  is used and Bitwuzla is required. A missing prefix fails even when
  `CMAKE_PREFIX_PATH` holds a good install.
- **The environment variable `BITWUZLA_HOME`** works, and the CMake
  setting beats it.
- **`AUTO` finds 0.9.1** through each of `Bitwuzla_ROOT` (variable and
  environment), `CMAKE_PREFIX_PATH` and `PKG_CONFIG_PATH`.
- **Precedence holds:** `BITWUZLA_HOME` beats `Bitwuzla_ROOT`, and
  `Bitwuzla_ROOT` beats pkg-config.
- **Unusable installs give a warning and build without Bitwuzla under
  `AUTO`:**
  - a static-only install;
  - a library without the API ("does not compile and link");
  - a library whose prefix has no header, even with a good install on
    `CMAKE_PREFIX_PATH`.
- **The same "does not compile and link" library fails configure** when it
  comes from `BITWUZLA_HOME`.
- **`WITH_BITWUZLA`:**
  - `ON` with nothing installed fails;
  - lower-case `on` works;
  - `OFF` wins over `BITWUZLA_HOME`;
  - `maybe` fails.

Output: `scratch/acceptance-1.1/find-bitwuzla-smart.log`.

[OBS 2026-10-22] `cmake --build seahorn/build --target install`
(`BITWUZLA_HOME` is set in that tree):
- The link test ran once and succeeded. An unchanged reconfigure
  afterwards did not rerun it.
- `otool -L` shows `libbitwuzla.0.dylib`, and `--version` prints 0.9.1.
- `scratch/test_selector_1.2.sh` passes 5/5.
- The 15 compile/link steps were the files the 2026-10-21 branch switch
  rewrote on disk, plus `GitSHA1.cc`. A flags change would have rebuilt
  all ~200 files.

Log: `scratch/build-smart-main.log`.

[OBS 2026-10-22] `seahorn/build-clam` (no `BITWUZLA_HOME`, so `AUTO`):
- configure prints "Bitwuzla support disabled (none usable found...)";
- `otool -L` shows no libbitwuzla;
- the selector script passes 8/8.

So it is unchanged from before. Log: `scratch/build-smart-clam.log`.

## Not tested

- [OPEN] A real system-wide install found by `AUTO`, such as `brew install
  bitwuzla`. It is unknown whether the Homebrew bottle is a shared
  library, and whether it passes the link test with SeaHorn's
  Homebrew-LLVM-18 flags. `CMAKE_PREFIX_PATH` stood in for the system
  prefixes. Parked in `loose-ends/parked.md`.
- [HYP] On Linux, the link test may fail if `ld` cannot find the
  libraries `libbitwuzla.so` itself needs (`libbitwuzla{ls,bv,bb}`) at
  link time. CMake's build RPATH for a full-path library probably covers
  this. Added to plan 4.3.
- The prefix derivation for `lib/<multiarch>` is untested, because macOS
  has no `CMAKE_LIBRARY_ARCHITECTURE`.
- The link test only links. A problem that appears only at run time
  would still pass.

## See also

- 2026-10-14-bitwuzla-build-integration.md (the opt-in design this replaces)
- 2026-10-14-build-configurations.md (the FindYices2 lesson, the CXX flags)
- ../../durable/adding-a-solver-backend-touches-these-files.md
