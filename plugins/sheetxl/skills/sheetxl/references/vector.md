---
name: vector
description: "The IRange vector surface — SheetXL's NumPy-shaped array API for computing over many cells without materializing them. Iteration with entries(), building output with sparseBuilder(), dispatch with mask/choose/select, shape and coercion, and the read/write bridges getValuesRange() and setComputedValues(). Load when transforming a column, filtering or sorting values, broadcasting an operation, or writing any formula-function body. Triggers on: IRange, getValuesRange, setComputedValues, entries, sparseBuilder, pushAt, mask, choose, select, tile, repeat, resize, getShape, getValueAt, fillEmpty, Axis, vector, columnar, sparse, broadcast, transform a column, compute without materializing, NumPy."
---

# Vector — `IRange`, the Compute Surface

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.
> For finding and profiling data, `reading.md`; for writing cells, `writing.md`;
> for authoring formula functions, `udf.md` (whose bodies are `IRange` code).

`IRange` is where computation happens. It is an immutable, two-dimensional,
mixed-type **sparse** array with functional vector semantics, shaped for
NumPy/JAX familiarity: every operation returns a new range through structural
sharing, so transforms are zero-copy and reads are SIMD-optimized. Sparse and
mixed types are the normal case here, not an edge case.

**This is the answer to "I need to loop over the values".** You almost never do.

## Getting in and out

| | |
|---|---|
| a cell range → a vector | `range.getValuesRange(): IRange` |
| a vector → back into the sheet | `range.setValues(vector)` |
| both at once | `range.setComputedValues(fn)` |

`setComputedValues(fn)` is exactly `setValues(fn(getValuesRange()))` — the range
goes in as an `IRange`, the transform runs in the engine, and the values never
round-trip through your runtime:

```typescript
wb.getRange('B2:B400').setComputedValues(src => /* IRange in, IRange out */);
```

Inside a formula function there is no cell range at all: the arguments arrive as
`IRange` already, and the return value is one. Same vocabulary, no bridge.

## The five primitives

Everything else composes from these, and they are the ones worth knowing by name:

| | |
|---|---|
| `entries(options?)` | iterate — sparse or dense, by axis, by direction, coercible |
| `sparseBuilder()` | streaming writer for building sparse output |
| `getShape()` | `{ rows, columns }` |
| `resize()` | a new range at a different shape, with a fill value |
| `getType()` | the scalar type used for default coercion |

Optimised alongside them: `getValueAt(row, column)`, `fill()`, `fillEmpty()`,
`clone()`, `getStats()`. The rest of the utility surface — `mask`, `choose`,
`select`, `tile`, `repeat` and friends — composes from the five, and the NumPy
names mean what you expect. **The shipped types are the authority**; roughly 45
operations are there, and this file teaches the ones whose usage is not obvious
from a signature.

## Iteration — `entries()`

Use `entries()` for all traversal. **Column orientation is faster** — it aligns
with the internal columnar storage — so use it whenever row-major order is not
actually required.

```typescript
const iterEntries = array.entries({
  axis: IRange.Axis.Column,
  includeEmpty: false,
});
let iter = iterEntries.next();
while (!iter.done) {
  const { value, coords } = iter.value;
  const { rowIndex, colIndex } = coords;
  iter = iterEntries.next();
}
```

`getValueAt(row, col)` is the **slowest** access pattern. Reserve it for O(1)
spot lookups; never build a loop out of it.

## Building output — `sparseBuilder()`

> `sparseBuilder()` is **additive** — it inherits the existing values of the range
> it is called on. To build fresh output, always call `empty().sparseBuilder()`.
> Calling `sparseBuilder()` directly is correct only when you are selectively
> overwriting values in an otherwise unchanged range.

```typescript
const builder = empty().sparseBuilder();
builder.pushAt(rowIndex, colIndex, value);  // monotonically increasing coordinate order
const result: IRange = builder.build();
```

`pushAt` requires monotonically increasing coordinates. That constraint is what
makes the builder streaming rather than a sort.

## Dispatch primitives

Branching over a whole range is a primitive, not a loop with an `if` in it:

| Primitive | Use for |
|---|---|
| `mask` | Boolean dispatch — `IF` |
| `choose` | Index dispatch; `selectFn` returns a 0-based index into `ranges[]` — `CHOOSE`, `IFS` |
| `select` | Value-equality dispatch, matching an expression against `match[]` — `SWITCH` |

## Typing and coercion

Declare the element type in the generic: `IRange<number>`, `IRange<string>`,
`IRange<Scalar>`. Coercion is handled for you; reach for `ValueOptions` only when
the default coercion is insufficient.

`IRange` is **immutable** — every method returns a new range and none mutates.
This is what makes structural sharing safe, and it is why a "transform" is a
return value rather than an in-place edit.

## Choosing between the vector surface and stats

They answer different questions, and the wrong one is the expensive one:

| Question | Reach for |
|---|---|
| what are these values like — sum, min, distribution | `range.getStats()` (`reading.md`) |
| produce new values from these values | `IRange` |
| I need the values in JavaScript | `getValues()` — the exit door |

`getStats()` reduces in one pass and returns an answer. `IRange` transforms and
returns a range. Only `getValues()` copies the grid into your runtime, and it is
for handing data to something outside the engine — a file, another tool, the
user's eyes.
