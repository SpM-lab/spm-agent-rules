# SpM-lab Agent Rules

Shared guidance for AI agents and human contributors working across SpM-lab
repositories.

This repository is intentionally rules-first. Project repositories keep their
own `AGENTS.md` small and point agents here for durable cross-repository rules.
Repository-specific rules stay in each project's own `AGENTS.md` or
`REPOSITORY_RULES.md`.

## Scope

These rules serve the sparse-IR software stack:

| Repository | Role |
| --- | --- |
| [`SpM-lab/sparse-ir-rs`](https://github.com/SpM-lab/sparse-ir-rs) | Rust core plus the `libsparseir` C API surface |
| [`SpM-lab/SparseIR.jl`](https://github.com/SpM-lab/SparseIR.jl) | Julia wrapper over the C API via `ccall` |
| [`SpM-lab/sparse-ir`](https://github.com/SpM-lab/sparse-ir) | Python wrapper over the C API via `ctypes` |

The domain is the sparse intermediate representation (IR) of many-body Green's
functions: imaginary-time (`tau`) and Matsubara-frequency sampling, the
discrete Lehmann representation (DLR), and fermionic/bosonic statistics. Most
defects in this stack are not algorithmic — they live at the FFI boundary, in
unstated physics contracts, and in tests that were quietly disabled. The rules
here are written against those failures.

## Usage From Project Repositories

Do not vendor copies of these rules. Each member repository references this
repository from its `AGENTS.md` entry point:

```markdown
# AGENTS.md

## Shared Rules

Read the SpM-lab shared agent rules before making changes:
<https://github.com/SpM-lab/spm-agent-rules> — start at `rules/index.md` and
load only the rule files the current task needs.

If network access is unavailable, look for a sibling checkout at
`../spm-agent-rules`.

## Repository-Specific Rules

<!-- Only what is genuinely specific to this repository. -->
```

Prefer the latest online version when internet access is available. Read
[`rules/index.md`](rules/index.md) first, then load only the rule files needed
for the task. Never bulk-load the whole repository.

## Layout

```text
rules/
  index.md                   routing table
  common.md                  cross-language repository policy
  ffi-boundary.md            the C boundary: dtype, contiguity, status codes
  numerical-conventions.md   physics contracts every wrapper must state
  testing.md                 what the test suite must actually cover
  rust.md
  julia.md
  python.md
skills/
  sparse-ir-release/         releasing the whole stack: order, gates, verification
    SKILL.md
    backend.md               sparse-ir-rs, crates.io, pylibsparseir
    julia.md                 Yggdrasil, libsparseir_jll, SparseIR.jl
    python.md                sparse-ir
```

`rules/` states what must hold. `skills/` holds procedures that change
together with the workflows they drive. Each skill directory has a `SKILL.md`
with front matter, so agents that support skills can load it directly.

## Precedence

- Repository-local rules override these shared rules when they are more
  specific. Say so in the pull request when the override affects a review.
- Existing code that violates these rules is a migration target, not a pattern
  to copy.

## Proposing Rule Changes

Open a pull request against this repository. See
[`CONTRIBUTING.md`](CONTRIBUTING.md). Every rule must name the failure it
prevents and must be checkable by a reviewer or a test.
