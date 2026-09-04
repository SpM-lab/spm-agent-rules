# Rust Rules (`sparse-ir-rs`, `libsparseir` C API)

Read [`common.md`](common.md) and [`ffi-boundary.md`](ffi-boundary.md) first.
This file covers the Rust core and, in particular, the C API surface that the
Julia and Python wrappers depend on.

## Never Panic Across FFI

**Failure it prevents:** a panic unwinding through an `extern "C"` boundary —
undefined behavior at best, and at worst a wrapper that catches it and returns
plausible all-zeros.

- Every `extern "C"` function is panic-free by construction: no `unwrap`,
  `expect`, indexing panic, slice-range panic, or arithmetic overflow panic on
  any reachable path with C-supplied input.
- Convert internal `Result`/`Option` into a status code at the boundary. Where a
  panic is genuinely unavoidable in a called subsystem, wrap it with
  `catch_unwind` at the boundary and map it to a distinct internal-error status
  — never to success.
- A wrapper must never have to guess whether a zeroed buffer means "success
  with zeros" or "the core aborted". The status code is the only channel.

## Status Codes Are The Contract

- Every C entry point returns a status code. Success is a single value;
  failures are distinct, documented, and stable.
- Status codes are append-only: never renumber, never reuse a retired value.
  Wrappers pin against these numbers.
- Every status code has: a name in the header, a documented meaning, and at
  least one test that provokes it.
- Provide a `spir_status_message`-style accessor so wrappers can print a
  human-readable cause instead of a bare integer.
- Output parameters are written only on the success path. An entry point that
  returns a failure must leave caller buffers untouched, and this must be
  tested.

## Validate At The Entry Point

- Validate at the C boundary, before touching Rust internals: null pointers,
  dimensions and lengths, index ranges, Matsubara parity against statistics,
  `beta > 0`, `wmax > 0`, `eps > 0`, and consistency between handles.
- Check finiteness of incoming data arrays and of matrices the core will
  factorize. A NaN entering a decomposition must produce a specific status, not
  a panic and not a zero-filled result.
- Reject rather than repair: no clamping an index, no truncating a value, no
  adjusting a parity.
- Do not trust a length that the caller supplies alongside a pointer beyond
  using it as the bound; there is nothing else to check it against, so document
  the caller's obligation in the header.

## Layout And Ownership Are Explicit

- Document, in the header, for every array parameter: element type, length,
  memory order (which `order` flag values the entry point accepts, and that
  the flag must match the buffer), and whether the callee reads or writes it.
- Document ownership for every returned handle: who releases it, with which
  function, and whether releasing twice is safe.
- Do not expose Rust internals through FFI to avoid designing the right
  high-level entry point. If the wrappers need a computation, add a C function
  for it rather than exporting the pieces.

## Typed Errors In The Rust Core

- Public Rust APIs return crate-local typed errors, not `String` and not an
  unstructured catch-all. Preserve the `source()` chain.
- Classify by kind — invalid argument, dimension/shape, unsupported dtype or
  option, numerical failure (singular, non-convergent), invalid state,
  internal invariant — and map each kind to a stable C status code in one
  place.
- Error messages must carry the observed values; they become the wrappers'
  diagnostics.

## `unsafe` Boundary

- `unsafe` belongs in the FFI layer and in leaf numerical kernels, not in the
  algorithmic layers.
- Every `unsafe` block carries a `// SAFETY:` comment naming the validation
  site that proves its preconditions — the null check, the length check, the
  owning invariant. Keep the proof next to the block.
- Slices built from C pointers are constructed once, after the null and length
  checks, in a single documented place per entry point.

## Cross-Repository Duties

- A change to the C API surface is a change to three repositories. In the pull
  request, list which wrapper functions must follow, and open the corresponding
  issues before merging.
- Never change the meaning of an existing entry point in place; add a new one
  and deprecate the old with a documented window.
- Header, Rust implementation, and both wrappers' bindings must agree on every
  signature. A generated or checked-in header must have a CI check proving it
  matches the implementation.

## Tests

- Numerical paths need tests over representative values, edge cases, layout
  variants, dtype variants, and error branches.
- Every C entry point has a test called through the C ABI, not only through the
  Rust API — including at least one failure-status case per entry point.
- Decompositions and transforms are checked with reconstruction or residual
  tests tied to `eps`, not with shape assertions.
