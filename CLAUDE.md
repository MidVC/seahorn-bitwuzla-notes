# Project: Bitwuzla backend for SeaHorn (URA, Fall 2026)

These notes live at `URA/research-notes/`. The better-than-fish skill
governs how to add and maintain them; re-read its SKILL.md for markers,
triggers, and the supersede pattern.

The folder is a git repo, public at
https://github.com/MidVC/seahorn-bitwuzla-notes (branch `main`, since
2026-10-29). Anyone can read it, so keep secrets and private details
out. Commit and push note changes only when the user asks.

**Goal:** add an optional Bitwuzla backend for monolithic bit-vector BMC
behind SeaHorn's generic `solver::Solver` interface, then validate it
(correct answers, models, cores, traces) against the Z3 adapter.

## Code locations to anchor citations

- SeaHorn checkout: `URA/seahorn/` on branch `bitwuzla-support` (our work,
  branched from `dev18` @ `839a3d57`, LLVM 18). Cite upstream code as
  `seahorn/<path>:<line>` plus the commit when it matters. Mark citations
  of code that exists only on our branch with "(branch bitwuzla-support)".
  Remotes: `origin` = our fork `MidVC/seahorn` (push here), `upstream` =
  `seahorn/seahorn` (read-only for us; pull `dev18` from here).
- Solver interface: `include/seahorn/Expr/Smt/Solver.hh`, `Model.hh`
- Existing adapters: `include/seahorn/Expr/Smt/Z3SolverImpl.hh`,
  `lib/seahorn/Smt/Yices2SolverImpl.cc`, `MarshalYices.cc`
- Our Bitwuzla adapter (branch bitwuzla-support, built up step by step
  from plan 1.3): `lib/seahorn/Smt/MarshalBitwuzla.cc`,
  `BitwuzlaSolverImpl.cc`, `units/units_bitwuzla.cpp`
- Engine + selection: `lib/seahorn/SolverBmc.cc`, `lib/seahorn/BmcPass.cc`
- Build: top-level `CMakeLists.txt`, `lib/seahorn/Smt/CMakeLists.txt`,
  `units/CMakeLists.txt`. Build trees: `seahorn/build/` (main, `WITH_CLAM=OFF`)
  and `seahorn/build-clam/` (`WITH_CLAM=ON`, for Path BMC / `test/bmc`;
  `scratch/build_seahorn_clam.sh`). Both need the main build's
  `CMAKE_CXX_FLAGS`; see `journal/2026-10/2026-10-14-build-configurations.md`.
  Neither tree has Yices. Bitwuzla: `WITH_BITWUZLA=AUTO|ON|OFF` (default
  AUTO, since 2026-10-22); `-DBITWUZLA_HOME` pins a prefix and makes it
  required. `build/` has it on (BITWUZLA_HOME set); `build-clam/` finds
  none, so it is off until plan 2.4.
  `seahorn --version` prints the linked Bitwuzla version.
- Bitwuzla: 0.9.1 built with Clang 18.1.8; shared library installed at
  `$HOME/opt/bitwuzla-0.9.1`. Build/link evidence and exact configuration:
  `journal/2026-10/2026-10-14-bitwuzla-build-and-link.md`.

## Source corpus

- `URA/SEAHORN_BITWUZLA_COURSE_NOTES.md` (and `.pdf`): a long study guide
  prepared 2026-10-01 against SeaHorn `839a3d57`. It is a secondary
  source. Before recording a claim from it as `[FACT]`, check it against
  the code.
- `papers/`: primary references (SeaHorn, Bitwuzla, SMT-LIB) as they are added.

## Project-specific conventions

- Any `[EMP]` claim about solver behavior or performance must name the
  SeaHorn commit, the solver versions, and the build options it depends on
  (`WITH_CLAM`, Yices found or not, `--horn-bv2`, and so on).
- Keep three things apart: the code's current behavior (`[FACT]`, cited),
  proposed adapter design (`[HYP]` or a recorded design pattern), and
  things we have actually run (`[OBS]`).

## Project-specific triggers

- "baseline" — run the smt-z3 BMC baseline (command in
  `durable/adding-a-solver-backend-touches-these-files.md`) and record
  the result as an `[OBS]`.

## Active investigations

Listed in `loose-ends/parked.md`.

## Last weekly digest

None yet (bootstrapped 2026-10-02).
