# Common Rules

Cross-language policy for every SpM-lab repository.

## Source Of Truth

- The Rust core and the `libsparseir` C API are the source of truth for
  numerical behavior. Wrappers translate; they do not reimplement. If a wrapper
  needs behavior the C API does not offer, add it to the C API rather than
  recomputing it in Julia or Python.
- When wrapper and core disagree, the wrapper is wrong until proven otherwise.
  Fix the wrapper, or fix the core and update every wrapper in the same
  logical change.
- Keep `AGENTS.md` short in every member repository: orient, point here, list
  overrides.

## Public Surface

- Public APIs are contracts. Before adding or keeping a public item, ask
  whether a downstream user should rely on it.
- Keep the exported surface deliberate and small. Do not export a symbol
  because an internal module happens to need it.
- Existing public items that violate these rules are migration targets, not
  patterns to copy.

### No Dead Exported API

**Failure it prevents:** an exported function that always throws, or an export
list naming a symbol that does not exist. Users discover it only at call time;
agents discover it only after writing code against it.

- Every exported symbol must be callable and must do what its name says. A
  public function whose body raises unconditionally, or whose implementation
  calls a method that does not exist, is a defect — not a stub.
- If a feature is not implemented, do not export a placeholder. Remove the
  export, or export it and raise a typed `NotImplementedError`-equivalent whose
  message says so explicitly, documents the state in the docstring, and is
  referenced from an open issue.
- Export lists (`__all__`, `export`, `pub use`, re-exports) are checked
  mechanically against real definitions. A name in an export list that does not
  resolve fails CI.
- A public symbol that is broken in one wrapper is very likely broken in the
  other. When fixing a dead export, check the sibling wrapper for the same
  symbol in the same change.

### Public-Surface Drift

- The three repositories expose the same physics. When a public API is added,
  renamed, or given new semantics in one, either mirror it in the others or
  record in the pull request why it is intentionally single-language.
- Keyword and argument names, default values, and error conditions should match
  across wrappers unless an idiom of the host language requires otherwise.
- A wrapper must not silently accept an input the C API rejects, and must not
  reject an input the C API accepts, without documenting the narrowing.

## Error-Report Faithfulness

**Failure it prevents:** a diagnostic that reports success, a wrong cause, or a
plausible-looking result for an operation that failed.

- Never return a value that could be mistaken for a result when the underlying
  operation failed. An all-zeros array, an empty array, and a default-
  constructed object are all indistinguishable from valid output at the call
  site.
- Error messages must name the actual failing condition and the observed
  values: the dtype received, the index requested, the valid range, the status
  code. "Invalid input" is not a diagnosis.
- Construct and raise the same error object. Constructing an error and then
  raising a different, generic one loses the diagnosis.
- Do not catch an exception and continue with a fallback unless the fallback is
  a documented behavior with the same contract. Swallowing a panic or an
  exception from the core layer is a defect, not robustness.
- Preserve the cause chain across the FFI boundary as far as the host language
  allows: status code, core error string, wrapper context.

## Documentation Must Match Implementation

**Failure it prevents:** a README or docstring that advertises a function that
throws, a keyword that is ignored, or an accuracy that is not achieved.

- Every claim in a README, docstring, or tutorial about behavior must be true
  of the current implementation. A documented example must run.
- Every keyword argument that changes numerical semantics must document that
  semantics, including what it does to the imaginary part, the statistics, or
  the sampling domain. A keyword whose effect is undocumented is treated as
  unimplemented.
- When behavior changes, update the docstring, the README, and any tutorial in
  the same change. A docs-only follow-up is not acceptable for a semantic
  change.
- Prefer executable documentation: doctests, tested example scripts, or
  snippets extracted from tested sources. A hand-copied snippet needs a sync
  check or it will drift.
- Do not document aspirational behavior in the present tense. Planned work goes
  in an issue.

## Naming

- The organization is **SpM-lab** — capital S, capital M, lowercase `lab`,
  hyphen.
- The Python package and its concept are spelled **sparse-ir** in prose and
  `sparse_ir` as the import name. The Julia package is **SparseIR.jl**; the Rust
  repository is **sparse-ir-rs** and its crates are `sparse-ir` (core) and
  `sparse-ir-capi` (the C API crate, library name `sparse_ir_capi`). The
  distributed C library is known as **libsparseir** (e.g. `libsparseir_jll`
  for Julia).
- Write **IR** for the intermediate representation, **DLR** for the discrete
  Lehmann representation, and expand the acronym once per document at first use.
- Use `tau` for imaginary time, `beta` for inverse temperature, `wmax` for the
  frequency cutoff, and `eps` for the basis accuracy target, consistently across
  languages and docs.
- Statistics are named `Fermionic` and `Bosonic` (or `"F"` / `"B"` where a
  string is required). Do not introduce a third spelling.

## Change Discipline

- Do not relax a tolerance, comment out a check, or narrow a test's input range
  in order to make a suite pass. If a test now fails, either the code regressed
  or the test was wrong; say which.
- A change that fixes a boundary defect must add the test that would have
  caught it, in the same change.
- When an audit or review finds one instance of a defect class in this
  document, search for the rest of the class in both wrappers before closing
  the work.
