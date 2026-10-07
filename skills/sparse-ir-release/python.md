# Python Wrapper Release: `sparse-ir`

Part of [`SKILL.md`](SKILL.md). This is step 4. It starts only after
`pylibsparseir` X.Y.Z is visible on **both** PyPI and the `spm-lab` conda
channel (steps 3a and 3b).

The default branch of `SpM-lab/sparse-ir` is `mainline`, not `main`.

## 4a. Compatibility PR

The wrapper calls `pylibsparseir` through `ctypes`, and the declared range is
what users' resolvers act on.

- Change the `pylibsparseir` range in **both** `pyproject.toml` and
  `.conda/meta.yaml` (the build and run requirements). The two must be
  identical. CI enforces this:

  ```bash
  python3 check_libsparseir_version_consistency.py
  ```

- Choose the range from the C API diff (see [`SKILL.md`](SKILL.md)):
  - Only additions: raise the upper bound, for example
    `>=0.8.3,<0.11.0`, so existing environments keep working.
  - Changed signatures: either require the new backend alone, or keep the old
    range and branch on the backend at the `ctypes` call site. Every branch
    needs a test that runs on each supported backend.
- To tell additions from changes, diff the header between the two backend
  tags with comments stripped; a line starting with `<` is a removed or
  changed declaration:

  ```bash
  H=sparse-ir-capi/include/sparseir/sparseir.h
  strip() { git show "$1:$H" | grep -v '^ *\*\|^ */\*\*\|^ *\*/' | grep -v '^\s*$'; }
  diff <(strip vOLD) <(strip vNEW) | grep '^<'   # empty: additions only
  ```

- Run the suite against the oldest and the newest backend in the range:

  ```bash
  uv run --with 'pylibsparseir==<oldest>' pytest tests/ -q
  uv run --with 'pylibsparseir==X.Y.Z'   pytest tests/ -q
  ```

Merge after CI is green. CI installs from PyPI, so it cannot pass before 3a.

## 4b. Version Bump

Bump `[project].version` in `pyproject.toml`. A patch bump fits when no public
API changed. State in the PR body which backend range the release supports.
When 4a is only a range change, put the bump in the same PR as 4a (one PR per
repository is the maintainer's preference; 2.1.6 was released this way).

## 4c. Tag

Push an annotated tag on the merge commit of the version PR. The tag starts
`wheel.yml` (PyPI) and `conda.yml` (conda). Because the tag is pushed with a
personal token, both workflows start without a manual dispatch.

```bash
git fetch origin mainline
SHA=$(git rev-parse origin/mainline)
git show --stat "$SHA"   # must be the version-bump merge
git tag -a vW.W.W "$SHA" -m "sparse-ir W.W.W"
git push origin vW.W.W
```

Verify both channels (see the final table in [`SKILL.md`](SKILL.md)). If one
of the workflows did not start, dispatch it with `--ref vW.W.W`.
