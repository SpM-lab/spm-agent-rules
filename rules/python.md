# Python Rules (`sparse-ir`)

Read [`common.md`](common.md) and [`ffi-boundary.md`](ffi-boundary.md) first.
This file covers only what is specific to Python and `ctypes`.

## Normalize With An Explicit Dtype

**Failure it prevents:** `.ctypes.data_as(POINTER(c_double))` on a `float32` or
`complex64` array, reading 8 bytes per 4-byte element.

- Every array crossing the boundary goes through `np.ascontiguousarray` (or
  `np.asfortranarray` when passing `SPIR_ORDER_COLUMN_MAJOR`) with an **explicit**
  `dtype=`, and the pointer is taken from the rebound result:

```python
a = np.ascontiguousarray(a, dtype=np.float64)
ptr = a.ctypes.data_as(POINTER(c_double))
```

- `np.asarray(a)` without `dtype=` is not normalization. Neither is
  `a.astype(np.float64, copy=False)` if the result is not the object the
  pointer comes from.
- Never call `np.ascontiguousarray(...)` as a bare statement and then take the
  original array's pointer.
- Keep the normalized array referenced by a local for the duration of the call;
  a pointer into a temporary is a use-after-free.
- For complex data, convert to `np.complex128` explicitly and pass the matching
  `ctypes` complex layout. Do not pass a `complex64` buffer through a
  `c_double`-based pointer for any reason.

## `argtypes` And `restype` On Every Binding

**Failure it prevents:** `ctypes` silently coercing a Python object to an
`int`-sized argument, so a float truncates or a pointer is passed as 32 bits.

- Every function pulled off `_lib` has `argtypes` and `restype` declared in one
  place, before first use. An undeclared function is a defect even if it
  currently works.
- Declare pointer arguments as the concrete `POINTER(...)` type, not
  `c_void_p`, so a mismatched buffer is caught earlier.
- Declare integer sizes to match the C header (`c_int`, `c_int32`, `c_size_t`);
  do not rely on the default.
- A CI check should assert that every attribute accessed on `_lib` appears in
  the declaration table.

## No Silent Coercion

- `int(x)` on a user-supplied Matsubara index or frequency truncates. Validate
  integrality and raise with the received value — see
  [`ffi-boundary.md`](ffi-boundary.md).
- `i % n` on a basis-function index hides an out-of-range request. Raise,
  naming the index and the valid range.
- Do not accept a Python negative index as a wrap-around into a C array unless
  the wrapper resolves it explicitly and documents it.

## Exception Discipline

- Never use a bare `except:` or `except Exception:` around a release, close, or
  `del` path. A failure there is information, not noise.
- Raise the specific type: `TypeError` for a wrong element type, `ValueError`
  for an out-of-domain value, `IndexError` for an out-of-range index,
  `RuntimeError` (or a package-local `SparseIRError` carrying the numeric
  status) for a C-level failure.
- Do not catch an exception from the core layer and return a default array.
- Chain causes with `raise ... from err` so the C-side diagnosis survives.

## Status And Handles

- Compare every status against success; an unrecognized code raises with the
  raw numeric value included.
- Check every returned handle/pointer for null before storing it.
- Allocate output buffers with the exact dtype and size the C signature
  expects, and return them only after the status check passes.

## Hygiene

- No `print` calls in the package. Diagnostics go into the raised exception; a
  genuine advisory uses `warnings.warn` with a specific category.
- `__all__` must list only names that exist and are callable; the smoke test in
  [`testing.md`](testing.md) enforces this.
- Type-annotate public functions, including the returned dtype relationship
  where it depends on the input (see
  [`numerical-conventions.md`](numerical-conventions.md)).
- Keep `numpy` as the only array assumption at the boundary; if array-API
  objects are accepted, convert them to `numpy` explicitly first.
