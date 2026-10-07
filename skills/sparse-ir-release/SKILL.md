---
name: sparse-ir-release
description: Use when releasing any part of the sparse-IR stack — a new sparse-ir-rs / libsparseir backend version, pylibsparseir, the sparse-ir Python wrapper, libsparseir_jll via Yggdrasil, or SparseIR.jl. Gives the cross-repository order, the publish commands, the gates between steps, and the verification that every channel actually received the version.
---

# Releasing The sparse-IR Stack

A backend release is not one publish. It fans out to seven channels in four
repositories, and several of them are triggered by hand even though they look
automatic. A release is done only when **every channel in the final table
shows the new version**. A green workflow run does not prove that.

Per-component detail:

- [`backend.md`](backend.md) — `sparse-ir-rs`: version bump, crates.io, tag,
  `pylibsparseir` on PyPI and conda.
- [`julia.md`](julia.md) — Yggdrasil, `libsparseir_jll`, `SparseIR.jl`.
- [`python.md`](python.md) — the `sparse-ir` Python wrapper.

## Before You Start

- **Get explicit approval to publish.** crates.io and PyPI never accept the
  same version twice. A bad upload can only be yanked and followed by a new
  version. A pushed tag and a registry entry are public from the moment they
  exist. Unless the maintainer has delegated publishing in writing for this
  release, stop before the first publish and hand over the exact commands.
- **Decide the version bump from the C API diff**, not from the Rust diff.
  Compare the header at the previous tag with the release candidate.
  - C signature or struct layout changed, or a status code renumbered:
    the backend is ABI-breaking. Wrappers that regenerate bindings must require
    the new backend alone, and they get a minor bump (not a patch).
  - Only additions: wrappers may keep accepting older backends.
- **Check the preconditions:**
  - All PRs meant for the release are merged and `main` CI is green.
  - The version is not on crates.io, PyPI or conda yet, and the tag does not
    exist yet.
  - The `SpM-lab/Yggdrasil` fork's `master` is in sync with upstream (see
    [`julia.md`](julia.md)).

## Order

Each arrow is a hard gate. Do not start a step until the previous channel is
**visible from the outside** (index API, registry file), not just when its
workflow is green.

```text
1. sparse-ir-rs: version-bump PR merged
2. manual-release.yml ──► crates.io (sparse-ir, sparse-ir-capi) + tag vX.Y.Z
   │                       + bot branch libsparseir-vX.Y.Z on the Yggdrasil fork
   ├─3a. PublishPyPI.yml    --ref vX.Y.Z ──► pylibsparseir on PyPI   (dispatch by hand)
   ├─3b. publish_conda.yml  --ref vX.Y.Z ──► pylibsparseir on conda  (dispatch by hand)
   └─3c. Yggdrasil PR from the bot branch ──► merged ──► General: libsparseir_jll
4. sparse-ir (Python): wrapper PR, then version PR, then tag ──► PyPI + conda   [after 3a, 3b]
5. SparseIR.jl: bindings + compat PR, then @JuliaRegistrator ──► General + TagBot [after 3c]
6. Downstream (tutorials, notebooks): raise the version floors, re-run, compare  [after 4, 5]
```

3a, 3b and 3c are independent, so start all three at once. Steps 4 and 5 are
independent of each other.

## The Tag Does Not Trigger The Tag Workflows

**Failure it prevents:** a release that looks complete while a channel silently
stays on the previous version. This has happened to conda.

`manual-release.yml` pushes `vX.Y.Z` with the workflow's own `GITHUB_TOKEN`.
GitHub does not start other workflows for pushes made with that token.
`PublishPyPI.yml` and `publish_conda.yml` are declared `on: push: tags` but
**never run on their own after a release**. Dispatch both explicitly against
the tag:

```bash
gh workflow run PublishPyPI.yml   --repo SpM-lab/sparse-ir-rs --ref vX.Y.Z
gh workflow run publish_conda.yml --repo SpM-lab/sparse-ir-rs --ref vX.Y.Z
```

The `sparse-ir` Python tag is different: a maintainer or agent pushes it with a
personal token, so its `wheel.yml` and `conda.yml` do start on the tag.

## Waiting

- Poll with bounded loops (`for i in $(seq 1 N); do …; sleep 60; done`) and
  match every terminal state, not only success. A wait loop that greps for
  `pending` exits early on "no checks reported".
- Typical durations (2026 measurements):

| Step | Takes |
| --- | --- |
| `manual-release.yml` | ~2 min |
| `PublishPyPI.yml` (cp310–cp314, Linux + macOS) | ~4 min |
| `publish_conda.yml` (20 packages) | ~40 min |
| Yggdrasil PR, open → merged and built | minutes to days (a maintainer merges it) |
| General registry PR (JLL or package) | ~11 min |
| Julia package server after the General merge | a further ~10 min |
| TagBot after the General merge | ~6 min |
| `sparse-ir` `wheel.yml` + `conda.yml` | ~4 min |

## Final Verification

Run all of these at the end. Every row must show the new version. Record the
table in the release report.

```bash
V=X.Y.Z   # backend version; W = sparse-ir, J = SparseIR.jl
curl -sS -A release-check https://crates.io/api/v1/crates/sparse-ir      | jq -r .crate.max_version
curl -sS -A release-check https://crates.io/api/v1/crates/sparse-ir-capi | jq -r .crate.max_version
git ls-remote --tags https://github.com/SpM-lab/sparse-ir-rs "v$V"
curl -sS https://pypi.org/pypi/pylibsparseir/json | jq -r .info.version
curl -sS https://api.anaconda.org/package/spm-lab/pylibsparseir \
  | jq -r --arg v "$V" '[.files[] | select(.version == $v)] | length'   # > 0
curl -sS https://raw.githubusercontent.com/JuliaRegistries/General/master/jll/L/libsparseir_jll/Versions.toml | tail -2
curl -sS https://pypi.org/pypi/sparse-ir/json | jq -r .info.version
curl -sS https://api.anaconda.org/package/spm-lab/sparse-ir \
  | jq -r --arg v "$W" '[.files[] | select(.version == $v)] | length'   # > 0
curl -sS https://raw.githubusercontent.com/JuliaRegistries/General/master/S/SparseIR/Versions.toml | tail -2
gh release view "v$J" --repo SpM-lab/SparseIR.jl --json tagName,publishedAt
```

- Do not trust `latest_version` from the anaconda API. It sorts versions as
  strings and has reported `1.85` for this package. Filter `.files` by version,
  as above.
- crates.io rejects requests without a `User-Agent` header.
- JLLs live under `jll/L/…` in General, not `L/…`.

## After The Release

- Bring the repo-local `julia/build_tarballs.jl` in `sparse-ir-rs` up to the
  released version (see [`julia.md`](julia.md)). The release job updates only
  its own checkout and the Yggdrasil fork. It does not update `main`.
- Update downstream environments: tutorial `pyproject.toml` floors and Julia
  `[compat]`. Before raising a floor, re-run the notebooks on the old and new
  versions and compare the outputs (see [`julia.md`](julia.md)). Changes to
  notebook numbers need the maintainer's merge.
- Remove temporary worktrees and branches created for the release.
