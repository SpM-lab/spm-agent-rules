# Backend Release: `sparse-ir-rs`, crates.io, `pylibsparseir`

Part of [`SKILL.md`](SKILL.md). This covers steps 1–3b: the version bump, the
crates, the tag, and `pylibsparseir` on PyPI and conda.

## 1. Version-Bump PR

Use one branch (for example `release/vX.Y.Z`) and one PR, containing only the
bump:

- `Cargo.toml`: `[workspace.package].version` and the `version` of every
  internal crate in `[workspace.dependencies]` (`sparse-ir`, `sparse-ir-core`,
  `sparse-ir-dlr`, `sparse-ir-minipole`, `sparse-ir-basis`).
- `python/pyproject.toml`: `[project].version`. It must equal the workspace
  version. Do not edit `python/pylibsparseir/__init__.py` (it reads package
  metadata) or `python/conda-recipe/meta.yaml` (it derives the version from
  `Cargo.toml`).
- Install snippets. `check_version.py` checks all of them and fails on a
  mismatch:
  - `sparse-ir/README.md` (the crates.io page);
  - the root `README.md` quick start;
  - `docs/book/src/getting-started/installation.md` (the user guide).
- Prose that names the release the docs are written against ("the 0.N
  release"): the root `README.md`, `sparse-ir/README.md`, the four sub-crate
  READMEs and `installation.md`. `check_version.py` does not see these, so
  find them with `grep -rn "0\.N" --include=*.md`.
- Lock files. Every one must be regenerated, or `--locked` CI jobs fail:
  - `Cargo.lock`: `cargo update -w`.
  - `python/uv.lock`: `(cd python && uv lock)`.
  - `docs/tutorial-code/Cargo.lock`: `cargo update -p sparse-ir` there. Check
    with `git diff` that only the five workspace crates moved; if unrelated
    crates were upgraded too, revert them and edit the entries by hand.

Choosing the number: before 1.0, a minor bump (`0.N` → `0.N+1`) is the
breaking slot for Cargo. Use it for any behaviour change of the public
surface, including a changed default value, even when no signature changed.

Gate:

```bash
uv run --python 3.12 python check_version.py   # needs Python >= 3.10
```

`check_version.py` reports an error for a Python version mismatch. A mismatch
in `julia/build_tarballs.jl` is only a warning, because that file follows
after the tag (see [`julia.md`](julia.md)).

Merge after CI is green. Stacked PRs retargeted to `main` can show "no checks
reported", because some workflows run only on PRs whose base is `main`. Close
and reopen the PR to trigger them.

## 2. Crates And Tag

```bash
gh workflow run manual-release.yml --repo SpM-lab/sparse-ir-rs \
  -f release_ref=main -f expected_version=X.Y.Z -f confirm_publish=true
```

It publishes the library crates `sparse-ir-core`, `sparse-ir-dlr`,
`sparse-ir-minipole`, `sparse-ir-basis` and `sparse-ir` in that order, waits
until `sparse-ir` at that exact version is visible on crates.io, then
publishes `sparse-ir-capi`, pushes the annotated tag `vX.Y.Z`,
and finally (job `publish-libsparseir`) pushes branch `libsparseir-vX.Y.Z` to
the `SpM-lab/Yggdrasil` fork. It does **not** open the upstream Yggdrasil PR.

Check the result:

```bash
git ls-remote --tags origin vX.Y.Z
for c in sparse-ir-core sparse-ir-dlr sparse-ir-minipole sparse-ir-basis sparse-ir sparse-ir-capi; do
  echo "$c $(curl -sS -A release-check https://crates.io/api/v1/crates/$c | jq -r .crate.max_version)"
done
```

If only the Yggdrasil job fails, the crates and the tag are already public and
must not be released again. Fix the cause (a branch-already-exists error means
deleting `libsparseir-vX.Y.Z` on the fork), then retry just that part:
`gh run rerun <run-id> --failed`, or dispatch `publish-libsparseir.yml`, which
uses the newest `v*` tag.

## 3a. `pylibsparseir` On PyPI

The tag push does not start this workflow (see [`SKILL.md`](SKILL.md)).
Dispatch it against the tag:

```bash
gh workflow run PublishPyPI.yml --repo SpM-lab/sparse-ir-rs --ref vX.Y.Z
```

This step was missed for 0.11.0, which therefore never reached PyPI (2026).

Gate for step 4: the version is on PyPI and has wheels for every supported
CPython tag (currently cp310–cp314) on Linux x86_64 and macOS arm64:

```bash
curl -fsS https://pypi.org/pypi/pylibsparseir/X.Y.Z/json \
  | jq -r '.urls[].filename' | sort
```

## 3b. `pylibsparseir` On conda

This workflow also has to be dispatched by hand. Forgetting it leaves conda on
the previous version with no error anywhere.

```bash
gh workflow run publish_conda.yml --repo SpM-lab/sparse-ir-rs --ref vX.Y.Z
```

The workflow runs the recipe tests on every package before it uploads.
Check the result by counting the files for the version on the channel; do not
use `latest_version`:

```bash
curl -sS https://api.anaconda.org/package/spm-lab/pylibsparseir \
  | jq -r --arg v X.Y.Z '.files[] | select(.version == $v) | .basename'
```

Expect Linux x86_64 and macOS arm64 builds for each supported Python.

## CPU Compatibility

Before announcing the release, import the Linux wheel and the conda `.so` on a
machine without AVX-512, and run a basis construction and a sampling
round-trip. A build that assumes the build host's instruction set fails with
`Illegal instruction` only on older CPUs. If no such machine is available, say
in the release report that this check was skipped.
