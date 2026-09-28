# Julia Release: Yggdrasil, `libsparseir_jll`, `SparseIR.jl`

Part of [`SKILL.md`](SKILL.md). This covers step 3c and step 5.

## Before The Backend Release: Sync The Fork

The release job branches from upstream `JuliaPackaging/Yggdrasil` `master`
and pushes to the `SpM-lab/Yggdrasil` fork. If upstream has changed files
under `.github/workflows/` that the fork lacks, the bot's push is rejected,
because the bot app has no `workflows` permission. Sync first:

```bash
gh repo sync SpM-lab/Yggdrasil --source JuliaPackaging/Yggdrasil --branch master
```

## 3c. Yggdrasil PR

1. Review the bot branch against upstream. Expect exactly three changed lines
   in `L/libsparseir/build_tarballs.jl`: `version`, the
   `# sparse-ir-rs vX.Y.Z` comment, and the `GitSource` hash (the tag object
   hash, as in earlier releases).

   ```bash
   R=https://raw.githubusercontent.com
   diff <(curl -fsS $R/JuliaPackaging/Yggdrasil/master/L/libsparseir/build_tarballs.jl) \
        <(curl -fsS $R/SpM-lab/Yggdrasil/libsparseir-vX.Y.Z/L/libsparseir/build_tarballs.jl)
   git ls-remote https://github.com/SpM-lab/sparse-ir-rs refs/tags/vX.Y.Z   # the hash in GitSource
   ```

2. Open the PR upstream. This needs the maintainer's approval unless it has
   been delegated.

   ```bash
   gh pr create --repo JuliaPackaging/Yggdrasil --base master \
     --head SpM-lab:libsparseir-vX.Y.Z \
     --title "[libsparseir] Update version to vX.Y.Z" --body "…"
   ```

3. A Yggdrasil maintainer merges it once Buildkite is green. That can take
   minutes or days. After the merge, the JLL is registered through an
   automatic `JuliaRegistries/General` PR.

4. Gate for step 5 has **two parts**. The General merge alone is not enough:

   ```bash
   # a) the registry has it
   curl -sS https://raw.githubusercontent.com/JuliaRegistries/General/master/jll/L/libsparseir_jll/Versions.toml | tail -2
   # b) the package server serves it (lags the merge by ~10 min)
   julia -e 'using Pkg; Pkg.Registry.update(); Pkg.activate(temp=true);
             Pkg.add(PackageSpec(name="libsparseir_jll", version="X.Y"))'
   ```

   Polling `Registry.reachable_registries()` can still say "missing" after
   `Pkg.add` already works. Use `Pkg.add` as the test.

## 5. `SparseIR.jl`

### Regenerate The Bindings From The Tag

**Failure it prevents:** bindings with the old arity calling the new library.
The mismatch shows up as garbage or a crash at call time, not at load time.

Generate from the header at the release tag, never from a working tree or a
leftover `deps/C_API.jl`:

```bash
git -C /path/to/sparse-ir-rs worktree add --detach /tmp/sparse-ir-rs-vX.Y.Z vX.Y.Z
cd SparseIR.jl/utils        # the generator resolves its output path relative to this directory
julia generate_C_API.jl --libsparseir-dir /tmp/sparse-ir-rs-vX.Y.Z/sparse-ir-capi
cp ../deps/C_API.jl ../src/C_API.jl   # only src/C_API.jl is tracked
```

Then compare the signatures:

```bash
git diff --stat src/C_API.jl
diff <(git show HEAD:src/C_API.jl | grep '^function ') <(grep '^function ' src/C_API.jl)
```

For every changed signature, update each Julia call site.

### Compat And Version

- If any C signature changed, set `libsparseir_jll = "X.Y"` **alone** in
  `[compat]`. Widening the range to older JLLs lets the resolver pair the new
  bindings with an old library, which is an ABI mismatch. Bump the package's
  minor version.
- If there were only additions, the range may keep older JLLs, provided the
  code does not call the new symbols unconditionally.

### Test Twice

1. Against a local backend build of the tag, with the library placed in `deps/`
   as `REPOSITORY_RULES.md` of `SparseIR.jl` describes: `Pkg.test()`.
2. Against the registered JLL, with the local library and `deps/C_API.jl`
   moved aside: `Pkg.test()`. This is the configuration users get.

Before the JLL is served, (2) cannot resolve. Do not merge on (1) alone.

### PR, Merge, Register

- Open one PR with the bindings, the call-site changes, the compat and the
  version. CI jobs that ran before the package server had the JLL fail at
  resolve. Re-run only those: `gh run rerun <run-id> --failed`.
- After merging, request registration on the **full** merge-commit SHA. A short
  SHA returns 422 "No commit found".

  ```bash
  SHA=$(git rev-parse origin/main)
  gh api repos/SpM-lab/SparseIR.jl/commits/$SHA/comments -f body='@JuliaRegistrator register'
  ```

- Registrator opens a General PR within about a minute. It auto-merges after
  the waiting period (~11 min), and TagBot then creates `vX.Y.Z` and the
  GitHub release (~6 min).

## After The Release: Repo-Local Recipe

`sparse-ir-rs` keeps its own copy of the recipe in `julia/build_tarballs.jl`.
The release job runs `julia/update_build_tarballs.jl vX.Y.Z` only inside its
own checkout, so `main` keeps the old version and `check_version.py` keeps
warning. Update it in a follow-up PR:

```bash
julia julia/update_build_tarballs.jl vX.Y.Z   # needs the tag locally
python3 check_version.py                      # the Julia warning disappears
```

## Downstream Julia Environments

Before raising a notebook or tutorial `[compat]` floor to the new `SparseIR.jl`,
run the notebooks on the old and the new version in separate environments and
compare the outputs. Use a second Jupyter kernel whose `--project` points at
the old environment. Confirm inside each kernel which version it actually
loaded: `pkgversion(SparseIR)`.
