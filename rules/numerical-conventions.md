# Numerical And Physics Conventions

The physics contracts every wrapper must **state**, **enforce**, and **test**.
An unstated convention is a defect: the next caller will assume the other one.

## Complex Versus Real Expansion Coefficients

**Failure it prevents:** the imaginary part of a physically complex quantity
discarded, giving a wrong answer with no warning.

- The IR expansion coefficients of a general Green's function are **complex**.
  Off-diagonal components of a matrix-valued `G`, and anything with a nonzero
  phase, carry a genuine imaginary part.
- Never apply `real()` unconditionally to a fitted or evaluated result. A real
  return type must come from a real input type, not from a projection.
- If a code path requires real output, it must be selected by the input dtype
  or by an explicit, documented keyword — and it must raise or warn when the
  discarded imaginary part is not negligible:

```python
# Wrong: silently drops physics.
return np.real(coeffs)

# Right: the caller's dtype decides, and a violated assumption is reported.
if np.iscomplexobj(gtau):
    return coeffs                       # complex in, complex out
if np.max(np.abs(coeffs.imag)) > tol:
    raise ValueError("real input produced complex coefficients "
                     f"(max |imag| = {np.max(np.abs(coeffs.imag)):.3e})")
return coeffs.real
```

- Preserve realness rather than manufacture it: a real input through a
  real-preserving transform stays real by construction. Where the transform is
  not real-preserving, the output type must be complex.
- Fitting must not change the caller's notion of type: `fit(evaluate(x))`
  round-trips to the same element kind as `x`.
- Document the element type of every public function's return in its docstring,
  as a function of the input element type.

## `positive_only`

**Failure it prevents:** half the Matsubara axis dropped, and the imaginary part
zeroed, under a keyword whose meaning was never written down.

- `positive_only = true` asserts the symmetry `g(-i omega) = conj(g(i omega))`
  — i.e. that the underlying quantity is real in imaginary time. It is a
  statement about the caller's data, not a display option.
- Its meaning must be stated in the docstring of every function that accepts
  it, in exactly those terms, together with what the returned object contains
  (only the non-negative frequencies) and what shape callers should expect.
- With `positive_only = true`, the wrapper must **either** enforce the symmetry
  or warn when the data violates it. Silently zeroing the imaginary part of the
  coefficients is forbidden:

```python
# Wrong: the keyword becomes an undocumented projection.
if positive_only:
    coeffs = coeffs.real.astype(complex)

# Right: enforce or report.
if positive_only and np.max(np.abs(coeffs.imag)) > tol * scale:
    warnings.warn("positive_only=True assumes g(-iw) = conj(g(iw)); "
                  f"input violates it (max |imag| = ...); ", RuntimeWarning)
```

- The default must be the general case (`positive_only = false`). A symmetry
  assumption is never the default.
- Both wrappers must agree on the keyword's name, default, sampling-point set,
  and returned shape.

## Fermionic Versus Bosonic Statistics

**Failure it prevents:** a closed form valid only for one statistics used for
the other, producing a rank-deficient or wrong basis with no error.

- Statistics is part of the identity of a basis, a sampling object, an
  augmentation, and a set of Matsubara points. Never infer it from context and
  never default it.
- Matsubara index parity follows the statistics: odd for fermions
  (`omega_n = (2n+1) pi / beta`), even for bosons (`omega_n = 2n pi / beta`).
  Validate parity at the boundary — see [`ffi-boundary.md`](ffi-boundary.md).
- **Augmentations are statistics-specific.** A constant-in-`tau` augmentation
  has a closed form that is valid for bosons; it is not valid for fermions. An
  augmentation applied to the wrong statistics must raise at construction,
  naming the augmentation and the statistics, not produce a basis that is
  quietly rank-deficient.
- Every augmentation type must document which statistics it supports, and its
  constructor must reject the others. "It happens to build" is not support.
- After constructing an augmented basis, its rank/conditioning must be
  verifiable: an augmentation that adds no independent direction is an error,
  not a degenerate success.
- Any mixed-statistics combination (a fermionic basis with bosonic sampling
  points, a bosonic `G` fitted against a fermionic basis) raises.

## Imaginary-Time Domain

- `tau` arguments lie in `[0, beta]`. Values outside are an error, not
  something to fold.
- If a wrapper supports negative `tau` via the (anti)periodicity
  `G(tau + beta) = -G(tau)` for fermions and `+G(tau)` for bosons, that
  extension is opt-in, documented per function, and carries the correct sign
  for the statistics. Never apply the fermionic sign to a bosonic object.
- `beta > 0` and `wmax > 0` are validated at construction.
- The endpoint convention (`tau = 0` versus `tau = beta`, and which side of a
  discontinuity a value belongs to) must be stated where it is observable.

## Memory Order At The C Boundary

**Failure it prevents:** a transposed or strided result, or an out-of-bounds
read, from a layout mismatch between host arrays and the C API.

- The C API takes an explicit memory-order flag (`SPIR_ORDER_ROW_MAJOR` /
  `SPIR_ORDER_COLUMN_MAJOR`) on every array-taking entry point. The flag passed
  must match the actual layout of the buffer passed — NumPy arrays are
  row-major by default, Julia arrays are column-major. A mismatch is not an
  error the C side can detect; it produces transposed or garbage results.
- Every binding must state, in a comment at the call site or in the binding's
  docstring, the layout it passes and how it got there. Never leave it to be
  inferred from whether the tests pass.
- Python wrappers convert explicitly and take the pointer from the converted
  object. Do not "fix" a layout mismatch by transposing the result afterwards;
  that hides which side was wrong and breaks for non-square shapes.
- The `axis` / `dim` argument that a wrapper exposes refers to the **host**
  array's axis. Translating it to the C API's dimension index is the wrapper's
  job, and must be correct for every axis, not just the first and the last.
  Document the mapping.
- Multi-dimensional input: state which axis is the sampling/basis axis, and
  make the default explicit. Both wrappers should use the same default axis
  convention or document why they differ.

## Accuracy Expectations

- A basis constructed with accuracy target `eps` supports round-trip and
  reconstruction errors on the order of `eps`, not of machine epsilon. Tests
  and docs must use tolerances tied to `eps`, not to `1e-15`.
- State the accuracy contract of each transform in its docstring: what error is
  expected relative to `eps`, and what degrades it (sampling condition number,
  `beta * wmax` range, augmentation).
- Do not advertise an accuracy the implementation does not reach. If a path is
  known to lose digits — a narrower working precision, an ill-conditioned
  sampling matrix, a truncated basis — say so and quantify it.
- Report residuals in a usable form (max absolute error, relative norm error)
  in tests and error messages; "did not converge" without a number is not
  actionable.
- A near-singular sampling or transformation matrix warns with its condition
  number rather than returning a silently wrong solution.

## Cross-Wrapper Consistency

- For the same `beta`, `wmax`, `eps`, statistics, and inputs, Julia and Python
  must return the same numbers to within the accuracy contract, with the same
  shape convention (up to the documented layout difference) and the same error
  for the same invalid input.
- A convention documented in one wrapper and not the other is a documentation
  defect in the second.
