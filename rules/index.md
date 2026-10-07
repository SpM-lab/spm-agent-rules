# SpM-lab Agent Rules Index

Read only the files relevant to the current task. Start with `common.md`, then
load the topic and language files the task touches.

## Files

- [`common.md`](common.md): public-surface drift, dead exported API,
  error-report faithfulness, docs-match-implementation, naming.
- [`ffi-boundary.md`](ffi-boundary.md): the C boundary. Dtype normalization,
  contiguity, range and parity validation before the call, pointer provenance,
  exhaustive status handling, uninitialized output, finiteness checks.
- [`numerical-conventions.md`](numerical-conventions.md): the physics
  contracts. Complex vs real expansion coefficients, `positive_only`,
  fermionic vs bosonic statistics, `tau` domain, memory order, accuracy vs
  basis `eps`.
- [`testing.md`](testing.md): dtype matrix, complex and fermionic coverage,
  axis coverage, exported-symbol smoke tests, failure-status paths, banned
  test-disabling patterns.
- [`rust.md`](rust.md): `sparse-ir-rs` and the `libsparseir` C API surface.
- [`julia.md`](julia.md): `SparseIR.jl` and `ccall`.
- [`python.md`](python.md): `sparse-ir` and `ctypes`.

## Skills

Skills are step-by-step procedures, kept under [`../skills/`](../skills/).
Unlike rules, they name concrete workflows, files and commands.

- [`sparse-ir-release`](../skills/sparse-ir-release/SKILL.md): releasing the
  backend, `pylibsparseir`, `sparse-ir`, `libsparseir_jll` and `SparseIR.jl`
  — order, gates, publish commands, final verification.

## Routing Table

| Task | Load |
| --- | --- |
| Any change to a wrapper function that calls C | `common.md`, `ffi-boundary.md`, the language file, `testing.md` |
| Adding or changing a public wrapper API | `common.md`, `numerical-conventions.md`, the language file, `testing.md` |
| Changing the C API surface in `sparse-ir-rs` | `common.md`, `ffi-boundary.md`, `rust.md`, plus the language file of every wrapper that must follow |
| Sampling, fitting, evaluation, DLR, augmented bases | `numerical-conventions.md`, `ffi-boundary.md` |
| Statistics, `positive_only`, augmentation, real/complex handling | `numerical-conventions.md` |
| Writing, fixing, or reviewing tests | `testing.md`, plus the topic file for the behavior under test |
| Re-enabling or deleting a skipped test | `testing.md` |
| Docs, README, docstrings, error messages | `common.md`, plus `numerical-conventions.md` if a physics claim is made |
| Cross-repository audit or release readiness | all of `common.md`, `ffi-boundary.md`, `numerical-conventions.md`, `testing.md` |
| Cutting a release, bumping a version, publishing to crates.io / PyPI / conda / General, or a Yggdrasil PR | the `sparse-ir-release` skill |

## Loading Policy

- Do not bulk-load the entire repository by default.
- Load `common.md` for any implementation work.
- Load `ffi-boundary.md` whenever a diff touches a `ccall`, a `ctypes` call, an
  `extern "C"` function, an array pointer, a dtype, or a status code — even if
  the change looks cosmetic.
- Load `numerical-conventions.md` whenever a diff touches `real`, `imag`,
  `conj`, statistics, `beta`, `tau`, Matsubara indices, or a docstring that
  states a physics contract.
- When a rule file and a repository-local rule conflict, follow the more
  specific repository-local rule and document the reason in the pull request.
