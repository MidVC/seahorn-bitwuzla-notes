# Step 0.1: Bitwuzla 0.9.1 builds and links on this Mac

[OBS 2026-10-14] Step 0.1 is complete. Homebrew offered stable 0.9.1,
but a source build was chosen to match SeaHorn's Homebrew Clang 18.1.8.
The standalone program returned exit status 0 and printed exactly:

```text
sat
x = #b11111111
```

[OBS 2026-10-14] Versions and configuration for this run:

| Component | Recorded value |
|---|---|
| Host | macOS 26.5.2 (25F84), arm64 |
| SeaHorn | `dev18` @ `839a3d574ada12e2eea0a70a6e6dffa3b52c0183` |
| SeaHorn compiler / standard | `/opt/homebrew/opt/llvm@18/bin/clang++`, 18.1.8, C++17 (`seahorn/build/CMakeCache.txt`) |
| Bitwuzla | tag `0.9.1` @ `8d1eb01093ae54d9b4586456b69c3bf31000a4c2`; clean source checkout |
| Source | `$HOME/opt/src/bitwuzla-0.9.1` |
| Install prefix | `$HOME/opt/bitwuzla-0.9.1` |
| Bitwuzla C / C++ compilers | `/opt/homebrew/opt/llvm@18/bin/clang` / `clang++`, both 18.1.8 |
| Build | Release (`-O3`, `-DNDEBUG`), C17 / C++17, shared libraries |
| SAT backends | CaDiCaL **only**, version 2.1.2; Kissat, CryptoMiniSat, Gimsatul disabled and not found |
| CaDiCaL source | Meson wrap `rel-2.1.2`, archive SHA-256 `292c2bb8d712d6d05fce3d3df63b922b8fa45e03974a79f7bae5bf68c284f131` |
| SymFPU | `40bdec00e99f8ea1b96c3dac0a05eed11c541639` (Meson fallback checkout) |
| Numeric dependencies | Homebrew GMP 6.3.0 and MPFR 4.2.2, dynamically linked |
| Build tools | Meson 1.12.1, Ninja 1.13.2, pkgconf 3.0.8 |

[OBS 2026-10-14] The exact successful configure/build commands were:

```sh
git clone --depth 1 --branch 0.9.1 https://github.com/bitwuzla/bitwuzla.git \
  "$HOME/opt/src/bitwuzla-0.9.1"
cd "$HOME/opt/src/bitwuzla-0.9.1"
CC=/opt/homebrew/opt/llvm@18/bin/clang \
CXX=/opt/homebrew/opt/llvm@18/bin/clang++ \
  ./configure.py release --shared --prefix "$HOME/opt/bitwuzla-0.9.1"
ninja -C build -j 6 install
```

[OBS 2026-10-14] `configure.py` expanded to `meson setup build
-Dbuildtype=release -Dprefix=/Users/hanyanglu/opt/bitwuzla-0.9.1
-Ddefault_library=shared -Dcadical=true -Dfpexp=false`. Other options
kept their defaults, including optional SAT backends off. Saved effective
options and compiled feature macros in `URA/scratch/bw-build-options.json`;
the sole SAT feature macro is `-DBZLA_USE_CADICAL`. Help text lists all
possible SAT backends, so it is not evidence that they were compiled in.

[OBS 2026-10-14] Initial configuration failed because MPFR was absent.
Installed Meson, pkgconf, and MPFR with Homebrew (Ninja and GMP already
existed), then configuration and installation succeeded. Added MPFR to the
plan's dependency list; the requirement is in the pinned source's
`src/meson.build:13-16`. Raw logs: `URA/scratch/bw-configure-initial.log`,
`bw-configure.log`, and `bw-build-install.log`.

[OBS 2026-10-14] Acceptance source is `URA/scratch/bw_hello.cpp`.
It enables model production before solver construction, asserts the
8-bit equation `x + 1 = 0`, requires SAT, prints the model, and checks
that its value is `#b11111111`. Run from `URA/`:

```sh
bash scratch/run_bw_hello.sh
```

[OBS 2026-10-14] The runner compiles with C++17 and
`-Wall -Wextra -Werror`, links `-lbitwuzla` from the install prefix, embeds
that directory as an rpath, and checks the full output. It supports
`BITWUZLA_HOME` and `CXX` overrides. `bw-hello.log` preserves the output.
`otool -L scratch/bw_hello` resolves the installed
`$HOME/opt/bitwuzla-0.9.1/lib/libbitwuzla.0.dylib` and system libc++;
`file` identifies both executable and library as arm64. No loader
environment override is required. The installed CLI reports `0.9.1`.

[OBS 2026-10-14] The installed pkg-config metadata reports version 0.9.1
and exposes the installed include/library paths:

```sh
PKG_CONFIG_PATH="$HOME/opt/bitwuzla-0.9.1/lib/pkgconfig" \
  pkg-config --modversion bitwuzla
PKG_CONFIG_PATH="$HOME/opt/bitwuzla-0.9.1/lib/pkgconfig" \
  pkg-config --cflags --libs bitwuzla
```

[OBS 2026-10-14] This establishes standalone C++17 linking with SeaHorn's
compiler. SeaHorn integration and regression tests remain later steps;
this run makes no claim about backend correctness inside SeaHorn.
