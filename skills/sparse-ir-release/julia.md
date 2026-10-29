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

## Known JLL Build Failures

Both of these appeared while releasing 0.13.0 and are fixed in the recipe that
`sparse-ir-rs` keeps (PRs #333, #334); they are worth recognising, because
neither reports itself as a recipe problem.

**`strided-kernel` dies with a rustc segmentation fault (exit 139).**

```
aarch64-linux-gnu-rustc-for-cargo: line 5: 1826 Segmentation fault  "$@"
error: could not compile `strided-kernel` (lib)   (exit status: 139)
```

The job compiles for five to eight minutes and then dies in that SIMD-heavy
dependency of `faer`/`tenferro`, because the workspace profile builds with a
single codegen unit, the most memory-hungry way to compile it. The same commit
passes on a retry and fails on a different platform, so it looks like
infrastructure flakiness; it is the builder running out of memory. The recipe
now sets `CARGO_PROFILE_RELEASE_CODEGEN_UNITS=16` and `CARGO_BUILD_JOBS=2`.
Five of the seven platforms failed this way in #15042 until that was fixed, and
then all seven passed.

**The JLL is built without the fused multiply-add.** The recipe sets `RUSTFLAGS`
itself, and an explicit `RUSTFLAGS` *replaces* the flags in `.cargo/config.toml`
rather than adding to them, so `-C target-feature=+fma` has to be repeated in
the recipe. Without it the library builds and passes, just ~3x slower on the
SVE, so nothing flags it. All JLL platforms have the instruction (mandatory on
aarch64, present on x86-64 since 2013, available on powerpc64le).

**The bot cannot push if it has to touch a workflow file.** Either grant
`libsparseir-bot` the `workflows: write` repository permission (done, so this
should not recur) or sync the fork as above; the push is refused with
`refusing to allow a GitHub App to create or update workflow ... without
'workflows' permission`.

**The auto-merge bot may not open the PR.** For 0.13.0 the branch appeared but
no PR did, and it had to be opened by hand (step 2). Expect the branch to need
a manual PR after a hand-pushed branch.

**Latency observed at 0.13.0:** PR opened 2026-10-08, merged 2026-10-11
(3.6 days) even though all seven Buildkite platforms were green the same day it
was opened; the General registration PR then auto-merged in 12 minutes and
`libsparseir_jll` `0.13.0+0` was in `Versions.toml`. Nothing to do but wait.

## 3c. Yggdrasil PR

1. Review the bot branch against upstream. A normal release changes exactly
   two lines in `L/libsparseir/build_tarballs.jl`: `version` and the
   `GitSource` hash. The `script` can also differ while the copy in
   `sparse-ir-rs` is ahead of upstream - see the next section.

   The source is pinned by commit and tag:

   ```julia
   GitSource(
       "https://github.com/SpM-lab/sparse-ir-rs.git",
       "<commit hash>";
       tag = "v$(version)",
   ),
   ```

   The hash must be the **commit** the tag points to, not the annotated tag
   object. BinaryBuilder peels the tag and fails the build unless that commit
   is `hash`. Yggdrasil reviewers ask for `tag=`
   (JuliaPackaging/Yggdrasil#15362). `julia/update_build_tarballs.jl` writes
   this form from SpM-lab/sparse-ir-rs#354 on; releases before 0.14.0 used the
   tag object hash and a `# sparse-ir-rs vX.Y.Z` comment.

   ```bash
   R=https://raw.githubusercontent.com
   diff <(curl -fsS $R/JuliaPackaging/Yggdrasil/master/L/libsparseir/build_tarballs.jl) \
        <(curl -fsS $R/SpM-lab/Yggdrasil/libsparseir-vX.Y.Z/L/libsparseir/build_tarballs.jl)
   git ls-remote https://github.com/SpM-lab/sparse-ir-rs 'refs/tags/vX.Y.Z^{}'   # the commit: the hash in GitSource
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
