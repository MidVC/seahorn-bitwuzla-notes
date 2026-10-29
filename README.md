# Research notes: a Bitwuzla backend for SeaHorn

Working notes from a URA project (Fall 2026). The project adds the
[Bitwuzla](https://bitwuzla.github.io) SMT solver as a backend for
[SeaHorn](https://github.com/seahorn/seahorn)'s bounded model checker,
behind SeaHorn's generic `solver::Solver` interface, and checks it against
the Z3 backend. The code is on the `bitwuzla-support` branch of
[MidVC/seahorn](https://github.com/MidVC/seahorn/tree/bitwuzla-support).

The notes are kept with
[better-than-fish](https://github.com/agurfinkel/better-than-fish): an AI
agent writes and maintains them while the author directs the work, so
that what was learned in one session carries over to the next.

## Where to start

- **`sessions.md`**: one entry per working session, newest at the bottom.
  This is the quickest way to see where the project stands.
- **`journal/YYYY-MM/`**: dated entries, one per step or investigation,
  with the evidence.
- **`durable/`**: facts checked against the source code, with file and
  line references.
- **`loose-ends/parked.md`**: questions set aside, with enough context to
  pick them up again.
- **`CLAUDE.md`**: the conventions the agent follows for this project.

## Markers

Each claim starts with a marker that says how much to trust it:

| Marker | Meaning |
|---|---|
| `[FACT]` | Checked against source code, a paper or the docs; usually cited |
| `[EMP]` | Seen repeatedly; tied to specific versions |
| `[OBS date]` | Seen once, on that date |
| `[HYP]` | A working hypothesis, not yet tested |
| `[OPEN]` | A known unknown, also listed in `loose-ends/` |
| `[SUPERSEDED → note]` | Turned out wrong or out of date; the note says what replaced it |

Wrong claims are marked superseded, never deleted, so the history of what
changed and why stays readable.

## Paths

`seahorn/...` paths refer to the SeaHorn checkout, at the commit named
next to them. `URA/...` and `scratch/...` paths (the project plan, test
scripts and logs) refer to the author's local workspace and are not in
this repository.
