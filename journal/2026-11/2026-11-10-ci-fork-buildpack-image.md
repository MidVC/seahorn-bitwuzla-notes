# CI fix: the buildpack image name on forks

The CI job (`test-seahorn-docker.yml`) on our draft PR `MidVC/seahorn#1`
(`bitwuzla-support` → the fork's `dev18`) had failed within about 10 s
on every run since 2026-10-29.

[OBS 2026-11-10] The error, from run 42761548169:
`docker: invalid reference format: repository name
(MidVC/buildpack-deps-seahorn) must be lowercase`.

[FACT] The workflow names the base image
`ghcr.io/${{ github.repository_owner }}/buildpack-deps-seahorn`
(`seahorn/.github/workflows/test-seahorn-docker.yml:50,57`, upstream
`dev18` @ `2e9eed07`). Image names on ghcr.io must be lowercase, and our
owner is `MidVC`. The Dockerfiles' default is already
`ghcr.io/seahorn/buildpack-deps-seahorn` (`docker/seahorn.Dockerfile:8`).

[OBS 2026-11-10] With the ghcr.io registry API (anonymous token):
`seahorn/buildpack-deps-seahorn:jammy-llvm18` gives HTTP 200, and
`midvc/buildpack-deps-seahorn:jammy-llvm18` gives 403 (we never ran
`buildpack-deps.yml`). So lowercasing alone would only swap the error.

Options weighed:
- lowercase plus a repository variable `BUILDPACK_OWNER=seahorn` (the
  first draft): needs a setting on every such fork;
- always use seahorn's image: simplest, but a fork could never use its
  own image (for example one with Bitwuzla installed, plan 2.4);
- lowercase, and fall back to seahorn's image when the owner has none
  (`docker manifest inspect`): no setting, unchanged on
  `seahorn/seahorn`. Chosen.

[FACT] Branch `fix-ci-buildpack-image`, based on upstream `dev18` @
`2e9eed07`, one commit `7a284207` "ci: lowercase the buildpack image
owner and fall back to seahorn's image". PR to `seahorn/seahorn` `dev18`:
https://github.com/seahorn/seahorn/pull/614 (opened 2026-11-10). The
other workflows that build the name the same way (`seahorn-docker.yml`,
`seahorn-coverage.yml`, `buildpack-deps.yml`) only run on a schedule or by
hand, and are unchanged.

[OBS 2026-11-10] The PR's CI passed: clang-format, licensing, lint, and
`test` in 6 min 17 s (run 42762118816). The new step found seahorn's
image with `docker manifest inspect` (no fallback notice in the log), and
the build used `ghcr.io/seahorn/buildpack-deps-seahorn`. So
`docker manifest inspect` works on the runner, and nothing changed for
`seahorn/seahorn`.

[HYP] The fork path (fallback taken) works; it has not run in GitHub
Actions yet. The
PR's own CI runs in `seahorn/seahorn`, where the owner's image exists. Our
draft PR #1 gets the fix once the fork's `dev18` is synced with an
upstream `dev18` that includes it.
