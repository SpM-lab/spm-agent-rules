# Testing Rules

A green suite is worthless if it only exercises the inputs the implementation
happens to handle. These rules define the minimum coverage for a wrapper over
the `libsparseir` C API, and the test-suite anti-patterns that are banned.

## Mandatory Coverage: The Dtype Matrix

**Failure it prevents:** a boundary function that is only ever tested with
`Float64` contiguous input, so type laundering at the C boundary is invisible.

- Every function that crosses the C boundary is tested with each of
  `{float32, float64, complex64, complex128}` (Julia: `{Float32, Float64,
  ComplexF32, ComplexF64}`) as input element type.
- The assertion for a narrow type is explicit: it must **either** raise a
  specific, documented error **or** agree with the `float64`/`complex128`
  reference result to the documented tolerance. Producing zeros, garbage, or a
  silently different answer fails the test.
- Integer input arrays are part of the matrix: `Int64`/`int64` input either
  converts correctly or raises.
- Parametrize the matrix; do not hand-write one dtype per test and let the
  others rot.

```python
@pytest.mark.parametrize("dtype", [np.float32, np.float64,
                                   np.complex64, np.complex128])
def test_evaluate_dtype(basis, dtype, reference):
    x = data.astype(dtype)
    if dtype in UNSUPPORTED_DTYPES:          # documented, not inferred
        with pytest.raises((TypeError, ValueError)):
            sampling.evaluate(x)
        return
    got = sampling.evaluate(x)
    assert np.allclose(got, reference, rtol=tol_for(dtype))
```

  `UNSUPPORTED_DTYPES` must mirror the documented support matrix, so silent
  garbage for a "supported" dtype still fails.

- Non-contiguous inputs are part of the matrix: pass a slice, a transpose, and
  a reversed view of an otherwise-valid array to every boundary function. This
  is the only test that catches a discarded contiguous copy.

## Mandatory Coverage: Physics

**Failure it prevents:** a suite whose every case is real-valued, bosonic,
`axis=0`, and `Float64` — the exact intersection in which several classes of
defect are invisible.

- **Complex coefficients**: at least one test per fitting/evaluation path uses
  a genuinely complex Green's function with a nonzero imaginary part, and
  asserts the imaginary part survives the round trip.
- **Both statistics**: every basis, sampling, DLR, and augmentation test runs
  for `Fermionic` and `Bosonic`. An augmentation valid for only one statistics
  has a test asserting the other **raises**.
- **Axis coverage**: multi-dimensional paths are tested for `axis` in
  `{0, 1, -1, ndim-1}` on an array with `ndim >= 3` and unequal dimension
  lengths, so a transposition bug cannot pass.
- **Domain edges**: `tau = 0`, `tau = beta`, the smallest and largest Matsubara
  index in range, and the first index out of range on each side.
- **Parity**: an even Matsubara index against a fermionic basis, and an odd one
  against a bosonic basis, both assert a raise.
- **Non-integral input**: a float Matsubara index such as `1.9` asserts a
  raise, not a result equal to the `1` case.
- **`positive_only`**: both values of the keyword, including data that violates
  the symmetry, asserting the documented warn-or-raise behavior.
- **Round-trip accuracy** is asserted against a tolerance derived from the
  basis `eps`, and the test reports the observed residual on failure.

## Mandatory Coverage: Failure Paths

- Every documented error condition has a test that asserts it is raised. A
  validation branch with no test will be deleted by a future refactor.
- At least one test drives a genuine C-level failure — an unsatisfiable
  request, a NaN-poisoned matrix, a deliberately invalid dimension — and
  asserts an exception, not zeros. A test that merely checks the happy path
  does not verify status handling.
- Where the core can panic, a test asserts the wrapper turns it into a
  host-language error, not into a plausible default value.
- Finiteness: a NaN or infinity in a user-supplied matrix or data array asserts
  a raise naming the offending position.

## Mandatory Coverage: The Exported Surface

**Failure it prevents:** an exported function that always throws, or an export
list naming a symbol that does not exist, shipping unnoticed.

- A smoke test enumerates the package's public surface programmatically
  (`__all__` in Python, `names(Module)` in Julia) and, for every symbol:
  resolves it, and calls it once with minimal valid arguments.
- Any symbol that raises `NotImplementedError`, `MethodError`, `AttributeError`,
  or `UndefVarError` from that call fails the test. Use an explicit,
  maintained allowlist for symbols intentionally left unimplemented, and keep
  the allowlist short and issue-linked.
- The smoke test must be derived from the export list, not from a hand-written
  list of names — a hand-written list drifts and misses new exports.
- Where a public symbol exists in both wrappers, add the smoke test to both.

## Banned Patterns

These make a suite report success it has not earned. Each is a review blocker.

### Commented-out and stringified tests

- A test disabled by comment characters, or trapped inside a triple-quoted
  string or a `#=` block, is dead code that reads as coverage. Delete it, or
  re-enable it and mark it with the language's skip mechanism plus an
  issue-linked reason.
- Watch for a test body that is a docstring literal: the module imports, the
  suite passes, and nothing ran.
- CI should count collected tests and fail on an unexplained drop.

### Except-skip

- Never wrap a test body in a `try`/`except` that skips on failure. It converts
  every real regression into a pass.

```python
# Banned: a genuine failure becomes a skip.
try:
    result = sampling.fit(gtau)
except Exception:
    pytest.skip("fit not supported")
```

- A test that is legitimately conditional skips on a **precondition checked
  before the work** — a missing optional dependency, an unavailable library
  version — never on an exception from the code under test.
- The same applies to Julia: no `try ... catch; @test_skip ... end`.

### Overbroad exception assertions

- `@test_throws Exception f()` and `pytest.raises(Exception)` pass when the
  code throws the *wrong* error, including a `MethodError` from a typo. Name
  the concrete type:

```julia
# Banned
@test_throws Exception spir_basis_u(basis, -1)
# Required
@test_throws ArgumentError spir_basis_u(basis, -1)
```

- Where the message carries the diagnosis, assert on it too (`match=` in
  Python, `@test_throws ArgumentError(...)` or a message check in Julia), so a
  reworded-but-wrong error is caught.

### Weak assertions

- Shape-only, finite-only, non-empty, and "did not throw" assertions are smoke
  checks, not tests, unless that exact property is the behavior under test.
- Prefer known values, algebraic identities, reconstruction residuals, and
  symmetry relations.
- An all-zeros result must never satisfy an assertion in a test of a nonzero
  computation. Assert a nonzero norm explicitly where zeros are a plausible
  failure mode — which, at this boundary, is nearly everywhere.

### Tolerance drift

- Do not widen a tolerance to make a test pass. Tie tolerances to the basis
  `eps` and to the input precision, and justify any deviation in a comment.

## Cross-Wrapper Parity

- A behavior test that exists for one wrapper should exist for the other. When
  fixing a defect, add the test to both suites even if only one was broken —
  the classes in these rules have repeatedly appeared independently in both.
- Where both wrappers can be compared numerically in CI, do so for at least one
  representative case per public transform.
