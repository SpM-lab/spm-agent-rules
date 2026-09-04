# Julia Rules (`SparseIR.jl`)

Read [`common.md`](common.md) and [`ffi-boundary.md`](ffi-boundary.md) first.
This file covers only what is specific to Julia and `ccall`.

## Dispatch Must Not Be Wider Than The `ccall`

**Failure it prevents:** a method annotated `T<:Real` or `AbstractArray` that
accepts `Int64` or `Float32` and then passes a `Ptr{Float64}`.

- A method whose body `ccall`s with `Ptr{Float64}` must be constrained to the
  element type it actually passes, or must convert explicitly inside.
- Prefer exact element-type comparison over subtype tests when deciding what a
  buffer is:

```julia
# Wrong: Int64 and Float32 both satisfy T<:Real.
function evaluate(s::Sampling, x::AbstractVector{T}) where {T<:Real}
    ccall(..., (Ptr{Float64},), pointer(x))

# Right: convert, and take the pointer from the conversion.
function evaluate(s::Sampling, x::AbstractVector{<:Real})
    xd = convert(Vector{Float64}, x)
    GC.@preserve xd ccall(..., (Ptr{Float64},), pointer(xd))
end
```

- Do not add an `unsafe_convert` / `cconvert` overload that makes an arbitrary
  array satisfy a `Ptr{Float64}` argument. That is exactly the fallback that
  turns a type error into silent zeros.
- `reinterpret` is not a conversion. Use it only where the bit layouts are
  provably identical and a comment says why.
- Where narrow types are supported, dispatch to the matching C entry point;
  otherwise convert with the precision consequence documented, or `throw`.

## `GC.@preserve`

- Every pointer passed to C must be taken inside a `GC.@preserve` covering the
  whole `ccall`, referencing the object the pointer came from.
- Passing an array directly as a `Ptr` argument works only because `cconvert`
  roots it for the duration of that single `ccall`. The moment a pointer is
  computed in one statement and used in another, `GC.@preserve` is mandatory.
- Preserve the **converted** object, not the caller's original.

## Validate In Inner Constructors

- Type invariants are enforced in the inner constructor so no code path can
  construct an invalid object. Do not validate in an outer helper that a
  keyword-arg path can bypass, and do not validate after a handle has already
  been created.
- Statistics, `beta`, `wmax`, `eps`, and finiteness of stored matrices are
  checked there.

## Throw What You Constructed

**Failure it prevents:** a carefully built diagnostic replaced by a generic
error at the `throw` site.

```julia
# Wrong: err is discarded.
err = ArgumentError("index $i out of range 1:$(n)")
throw(ErrorException("invalid index"))

# Right
throw(ArgumentError("index $i out of range 1:$(n)"))
```

- Use `ArgumentError` for invalid caller input, `DimensionMismatch` for shape
  problems, `DomainError` for out-of-domain values, and a package-local error
  type for C-level failures carrying the status code. Do not use bare
  `error("...")` for conditions that tests should assert on.
- A `SparseIRError`-style type carrying the numeric status and the C-side
  message keeps the diagnosis testable.

## Status Handling

- Store the `ccall` return value and compare it to success. Never `ccall` in
  statement position and drop the status.
- Map known status codes in one place; the fallback branch prints the raw code
  and throws.
- Check returned handles against `C_NULL` before wrapping them.

## Finalizers And Handles

- Wrap each C handle in a mutable struct with exactly one `finalizer` that
  releases it, and guard against double release.
- Do not `try`/`catch` around a release call to make it "safe"; a failed
  release is a bug worth surfacing.

## Style And Hygiene

- No `@warn`/`println` debugging left in the package. Diagnostics belong in the
  thrown error.
- Type-stable wrapper functions: the return element type must be a function of
  the argument types, not of a runtime branch (see the realness rules in
  [`numerical-conventions.md`](numerical-conventions.md)).
- Keep `export` lists exact; every exported name must resolve and be callable
  (see [`testing.md`](testing.md)).
- `@test_throws` must name a concrete exception type.
