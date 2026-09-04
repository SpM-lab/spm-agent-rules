# FFI Boundary Rules

Every call from Julia or Python into `libsparseir` crosses a boundary where the
host language's type system stops applying. The C API sees a pointer, a length,
and nothing else. These rules exist because the compiler cannot help here.

Load this file whenever a diff touches a `ccall`, a `ctypes` call, an
`extern "C"` signature, an array pointer, a dtype, or a status code.

## The Boundary Contract

Every function that calls into C must, in this order:

1. **Validate** every scalar argument: range, sign, parity, index bounds.
2. **Normalize** every array argument: element type, memory order, contiguity.
3. **Take the pointer from the normalized object**, never from the original.
4. **Keep the normalized object alive** for the whole duration of the call.
5. **Call** C.
6. **Check the returned status exhaustively**, treating anything unrecognized
   as an error.
7. **Only then** read the output buffer.

A function that skips a step is defective even if it currently produces correct
results for the inputs the tests happen to use.

## Type Laundering Is Forbidden

**Failure it prevents:** an array of one element type handed to C as a pointer
to a different element type. The C side reads the wrong number of bytes per
element and returns silent zeros, silent garbage, or reads out of bounds.

- Never cast an array pointer to an element type the array does not have. A
  pointer cast is not a conversion.
- Convert first, with an **explicit** target element type, then take the
  pointer from the result of the conversion.
- Do not rely on a generic conversion fallback to do the right thing. A generic
  path that accepts any array and produces a `Float64` pointer will happily
  reinterpret `Int64` or `Float32` data.

Violation (Python):

```python
# f is float32 or complex64: C reads 8 bytes per element from a 4-byte stride.
_lib.spir_sampling_evaluate(s, f.ctypes.data_as(POINTER(c_double)), n, out)
```

Fix:

```python
f = np.ascontiguousarray(f, dtype=np.float64)   # explicit target dtype
_lib.spir_sampling_evaluate(s, f.ctypes.data_as(POINTER(c_double)), f.size, out)
```

Violation (Julia):

```julia
# x may be Vector{Int64} or Vector{Float32}; the pointer says Float64.
ccall((:spir_basis_u_eval, libsparseir), Cint,
      (Ptr{Cvoid}, Ptr{Float64}, Csize_t), b, x, length(x))
```

Fix:

```julia
xd = convert(Vector{Float64}, x)      # a real conversion, not a reinterpret
GC.@preserve xd ccall((:spir_basis_u_eval, libsparseir), Cint,
      (Ptr{Cvoid}, Ptr{Float64}, Csize_t), b, pointer(xd), length(xd))
```

- Where a narrower type is genuinely supported end to end, dispatch to the C
  entry point that takes that type. Where it is not, either widen explicitly
  (documented, with the precision consequence stated) or raise. Silently
  reinterpreting is never an option.
- Complex arrays: the host and C layouts must agree on interleaving and on the
  real element type. `complex64` is not `complex128`; convert explicitly.

## Pointer Provenance

**Failure it prevents:** a contiguous copy is made and then discarded, while C
receives the pointer of the original non-contiguous or wrong-typed object.

- The pointer passed to C must be derived from **the same object** that the
  normalization produced. Assigning the conversion to a throwaway variable and
  then passing the original is a defect that no test on already-contiguous
  inputs will catch.
- Normalize by rebinding the name that is later used for the pointer:

```python
a = np.ascontiguousarray(a, dtype=np.float64)   # rebound; pointer comes from a
```

not

```python
np.ascontiguousarray(a, dtype=np.float64)       # result thrown away
ptr = a.ctypes.data_as(...)                     # original object's pointer
```

- Views, slices, transposes, and reversed arrays are the inputs that expose
  this. Test with them.
- Output buffers must be allocated by the wrapper with the exact element type
  and length the C signature expects, and must be contiguous. Never pass a view
  or a slice as an output buffer.

## Contiguity And Memory Order

- Every array crossing the boundary must be contiguous. Do not assume the
  caller's array is.
- The C API takes an explicit order flag (`SPIR_ORDER_ROW_MAJOR` /
  `SPIR_ORDER_COLUMN_MAJOR`) on every array-taking entry point. Pass the flag
  that matches the buffer you actually pass; never hard-code one order and
  convert blindly. State the layout and flag in the binding, and see
  [`numerical-conventions.md`](numerical-conventions.md) for the layout
  contract and for what the `axis`/dimension arguments mean on each side.
- A normalization that changes order must also be the object the pointer comes
  from — see Pointer Provenance.

## Validate Before The Call, Not After

**Failure it prevents:** invalid input reaching C, where it becomes a panic, a
segfault, or an out-of-bounds read instead of an exception.

- All argument validation happens before the C call, in the constructor or the
  entry function. Checking a field after an object is already built, or after
  the buffer has been passed, is too late.
- In Julia, put the validation in the inner constructor so no path can build an
  invalid object.
- Validation must cover:
  - **Index range**: basis-function indices, sampling-point indices, sizes.
  - **Parity**: Matsubara indices are odd for fermions and even for bosons.
    Reject a mismatch; do not adjust it.
  - **Domain**: `tau` in `[0, beta]`, `beta > 0`, `wmax > 0`, `eps > 0`.
  - **Shape**: array length against basis size or sampling-point count, on the
    dimension the call actually uses.
  - **Statistics**: the statistics of every operand agree with the basis.

## No Silent Value Coercion

**Failure it prevents:** a numeric argument quietly changed into a different,
valid-looking value, so the computation succeeds on the wrong input.

- Never truncate a float toward zero to obtain an integer argument. A Matsubara
  index or frequency given as `1.9` must raise, not become `1`.
- Convert an incoming float to an integer only when it is exactly integral, and
  raise with the received value otherwise:

```python
if float(n) != int(n):
    raise ValueError(f"Matsubara index must be an integer, got {n!r}")
```

- Never wrap an out-of-range index with a modulo or a clamp. An index outside
  the valid range is an error and must name the requested index and the valid
  range.
- Negative indices are host-language sugar. If a wrapper supports them, resolve
  them explicitly and document it; if it does not, reject them.
- Do not silently reorder, deduplicate, or sort sampling points. If the API
  requires sorted, unique points, validate and raise.

## Exhaustive Status Handling

**Failure it prevents:** a nonzero status that is not one of the codes the
wrapper enumerated, so the wrapper proceeds and returns uninitialized memory.

- Check the status of **every** C call. A status assigned to a variable and
  never read is a defect; so is a call whose return value is discarded.
- Handle the status by success, not by enumerating known failures:

```python
status = _lib.spir_sampling_fit(...)
if status != SPIR_COMPUTATION_SUCCESS:
    raise RuntimeError(f"spir_sampling_fit failed with status {status}: "
                       f"{_status_name(status)}")
```

not

```python
if status == SPIR_GET_IMPL_FAILED or status == SPIR_INVALID_DIMENSION:
    raise RuntimeError(...)
# every other nonzero status falls through
```

- An unrecognized status is an error. Map known codes to specific messages, and
  fall back to a message that prints the raw numeric code. Never fall back to
  success.
- A returned handle or pointer must be checked for null in addition to the
  status, before it is stored or used.
- Where the core layer can abort or panic, the wrapper must not treat that as a
  recoverable non-event. Do not catch it and return a default-constructed
  object; propagate it as an error naming the operation.

## No Uninitialized Output Escapes

**Failure it prevents:** a caller receiving a freshly allocated, never-written
buffer as if it were a result — typically all zeros, which is a physically
plausible value.

- Allocate output buffers, but return them only on the success path. Every
  early return, exception path, and status branch must not hand back the
  buffer.
- Prefer allocation that would fail loudly if unwritten over zero-filled
  allocation where the language offers a choice; where it does not, the status
  check is the only guard, so it must be exhaustive.
- Never return zeros, an empty array, or a default object as an error signal.
  See Error-Report Faithfulness in [`common.md`](common.md).

## Finiteness Checks Where NaN Is Constructible

**Failure it prevents:** a NaN-bearing matrix accepted at construction, whose
downstream failure in the core is swallowed and returns an all-zeros result
that looks like physics.

- Whenever a wrapper constructs or accepts a matrix or vector that the core
  will factorize, decompose, or solve with — sampling matrices, transformation
  matrices, user-supplied Green's function data — check for finiteness at
  construction and raise naming the first offending index.
- Do this at the point where the object becomes reusable, not on every element
  access in a hot loop. Construction-time is the right cost.
- If the check is genuinely too expensive for a given path, the status handling
  of the downstream call must be exhaustive enough to surface the core's own
  failure — and that must be tested with a deliberately NaN-poisoned input.
- Conditioning is a separate concern: a well-formed but near-singular sampling
  matrix should warn with its condition number, not silently produce garbage.

## Resource Lifetime

- Every C-allocated handle has exactly one owner in the host language, and its
  release path must run exactly once.
- Keep host buffers alive across the call explicitly (`GC.@preserve` in Julia; a
  live local reference in Python). A pointer taken from a temporary is a
  use-after-free waiting for a garbage collector.
- Do not wrap a release call in a bare catch-all. If releasing fails, that is
  information.

## Review Checklist

For each C call in the diff:

- [ ] Every scalar argument validated for range, sign, and parity before the call.
- [ ] Every array converted with an explicit element type.
- [ ] Every pointer taken from the converted object, not the original.
- [ ] Contiguity and memory order explicit and correct.
- [ ] No float-to-int truncation, no modulo or clamp on an index.
- [ ] Status checked against success, unknown status treated as error.
- [ ] Returned handles null-checked.
- [ ] Output returned only on the success path.
- [ ] Finiteness checked where NaN input is constructible.
- [ ] Host buffers preserved across the call.
- [ ] A test exists that passes a non-`Float64`, non-contiguous, and
      out-of-range input to this function — see [`testing.md`](testing.md).
