# Bootstrap: generic Z3 BMC baseline reproduces; course-notes claims spot-checked

[OBS 2026-10-02] The installed generic-Z3 BMC route returns `unsat` on
`test/opsem/test_ult_unsat.ll` in about 0.09 s wall time. This matches the
result the course notes reported on 2026-10-01. Environment: SeaHorn
`dev18` @ `839a3d57` (one local modification,
`test/tutorial/ssft_simple_6.c`, which does not matter here), Homebrew
Z3 5.1.0 at `/opt/homebrew/opt/z3`, `WITH_CLAM=OFF`, Yices2 NOTFOUND
according to `build/CMakeCache.txt`.

```sh
cd URA/seahorn
build/run/bin/seahorn --horn-bmc --horn-bv2=true \
  --horn-bmc-solver=smt-z3 --horn-solve test/opsem/test_ult_unsat.ll
```

The test stores 10 into two locations and assumes `a <u b` before the
error call. `10 <u 10` is false, so no error execution exists, and UNSAT is
the correct answer. Our reading: the generic `SolverBmcEngine` path works
end to end in this build. That makes it the reference path for any new
adapter. The direct `z3` engine is a different code path and should not
be used as the reference.

[OBS 2026-10-02] I checked the course notes against source for: both
solver enums, the selection switch, the `WITH_YICES2` guard and exit, the
`SolverResult` ordering, Z3 `m_last_result` never being cleared, Yices
ignoring `complete`, the Yices `to_smt_lib` stub, and the Path BMC timeout
`static_cast`. All matched, and I recorded them with line numbers in the
durable notes. I did not check §5–§8 (BV semantics, memory, Expr
internals, Bitwuzla API), so those remain secondary-source claims.

[HYP] Because Yices2 is absent from this build, `smt-z3` is the only
generic adapter that can be run locally. A differential harness for
Bitwuzla can therefore compare against Z3 only, unless we install Yices2 too.

[OPEN] Can the Bitwuzla 0.9.1 library (Meson build) be built on this ARM64
Mac and found by CMake? See loose-ends/parked.md.

## See also

- adding-a-solver-backend-touches-these-files.md
- generic-solver-interface-contract.md
