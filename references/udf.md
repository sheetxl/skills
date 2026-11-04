---
name: udf
description: "SheetXL formula and UDF authoring using @sheetxl/primitives — writing, registering, debugging, or reviewing any formula function, built-in or user-defined. The only import is @sheetxl/primitives. Triggers on: formula function, UDF, custom function, built-in, IRange, FormulaContext, FormulaError, @sheetxl/primitives, sparseBuilder, mask, choose, ScalarType, JSDoc, coercion, broadcast semantics."
---

# UDF — Formula Function Authoring

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.

**In this file:** The only import · Registering a UDF · Core invariants · The
oracle is observable Excel behavior · Never assume scalar · Parameter types ·
`IRange`: the body of every function · `FormulaContext` · JSDoc annotations ·
Error handling · Return values · Coercion · Contributing built-ins

Formula functions — both built-ins and UDFs — are pure TypeScript functions exported
from a module. The engine reads type signatures and JSDoc to auto-generate coercion,
validation, and UI metadata. Built-ins and UDFs use identical API; there is no
privileged internal surface.

## The Only Import

```typescript
import {
  FormulaError, FormulaContext, ScalarType,
  type Scalar, type IRange, type IReferenceRange
} from '@sheetxl/primitives';
```

`@sheetxl/primitives` is the complete authoring surface — the exclusive import for
formula functions. Any other import, including `@sheetxl/sdk`, is treated as a
third-party dependency bundled outside the formula sandbox. For method-level
detail, read the package's shipped types (`IRange`, `IFormulaContext`,
`FormulaError`, `ScalarType`) — as everywhere in SheetXL, the type definitions
are the authoritative reference.

---

## Registering a UDF

Functions register against a workbook's calculation engine. From `run` code
(either transport), use the function collection — pass the function source, JSDoc
included, as a string:

```typescript
const functions = wb.getCalculation().getFunctions();
functions.add(`
/**
 * Returns the value doubled.
 */
export function DOUBLE(x: number): number { return x * 2; }
`);
wb.getWorksheet().evaluate('=DOUBLE(21)')   // 42
```

In application code, providers can also be passed at construction
(`new Workbook({ functions: [...] })`) — see `embedding.md`. Either way the
authoring rules below are identical.

---

## Core Invariants

- Pure functions. No side effects, no global mutation.
- `IRange` is immutable. All methods return a new range, never mutate.
- `export function` declarations only. One function per export.
- Engine handles auto-boxing, broadcasting, and coercion. Do not reimplement them.

---

## The Oracle is Observable Excel Behavior

Excel compatibility is non-negotiable. Every function must match Excel's output exactly.

The published spec documents Excel's intended behavior but is incomplete by design —
it omits edge cases, implicit broadcast rules, type coercion behavior, and sorting
semantics that Excel implements but never formally specifies. The only authoritative
source is what Excel actually does. When spec and observation diverge, observation
wins. When the spec is silent, observation is the only guide.

Practical rule: test in Excel directly. Trust the empirical result over documentation,
including Microsoft's own.

---

## The Golden Rule: Never Assume Scalar

In modern Excel, functions that appear to take scalar arguments are implicitly lifted
(auto-broadcast) to support range/array inputs with full broadcast semantics.

Known cases that have caused incorrect implementations in other engines:

- `CHOOSE(index, ...)` — index accepts a range
- `SWITCH(expr, ...)` — expr accepts a range
- `IF(condition, ...)` — condition is a range in array context
- `IFS(cond1, val1, ...)` — all conditions and values accept ranges

---

## Parameter Types

### Scalar

```typescript
export function ADD(a: number, b: number): number { return a + b; }
```

The engine auto-coerces and auto-broadcasts. A scalar parameter can receive a range
at the call site; the engine handles the broadcast. Do not implement manual broadcasting.

### IRange<T>

Declare the type in the generic: `IRange<number>`, `IRange<string>`, `IRange<Scalar>`.
Coercion is handled automatically. Use `ValueOptions` only when default coercion
is insufficient. The operations themselves are in [vector](vector.md).

### Optional and default parameters

```typescript
export function ROUND(number: number, numDigits?: number): number { ... }
export function LOG(number: number, base: number = 10): number { ... }
```

### Variadic (1-N ranges)

```typescript
export function MAX(...number: IRange<number>[]): number { ... }
```

### Enum (a closed set of values)

Declare a plain TypeScript `enum` in the SAME file and type the parameter with it. Doc comment
every member on its own line; the doc is the help a user sees when picking the value.

```typescript
/** When payments are due in each period. */
enum PaymentType {
  /**
   * Payments are due at the end of each period.
   * @name At the end of the period
   */
  End = 0,
  /** Payments are due at the beginning of each period. */
  Beginning = 1,
}

export function FV(rate: number, nper: number, pmt: number, pv: number = 0,
  type: PaymentType = PaymentType.End): number { ... }
```

- **It gates.** The body only ever receives a member. A value outside the set (`2`, and also `0.5`:
  no truncation) is `#VALUE!` and the body does not run; under a lift, just that element is.
- **Strings are case-insensitive.** A string enum delivers the DECLARED spelling (`"MERCURY"` → `'mercury'`).
- **`ReportType<E>` opts out of the gate.** The value arrives unchecked and the body decides. Use it
  when Excel's own rule is not "exactly a member": the built-ins that truncate, read any nonzero as
  1, or answer `#NUM!` are all declared this way.
- **`E | Scalar` documents the values of an uncoerced slot.** The body receives the raw argument,
  whatever it is, and reads it itself; the enum only drives the dropdown, the help and the editor's
  warning. Use it when a slot takes more than the set (`WORKDAY.INTL`'s weekend is a code OR a
  `"0000011"` mask).
- **`@name` sets the label** the editor shows for the value, and works on the enum too, just as it
  does on a function. Without it the label is the member's name pretty-printed (`MemAvail` →
  "Mem Avail", `OSVersion` → "OS Version").
- **`@deprecated` on a member keeps it out of the dropdown.** It stays in the set, so a formula that
  names it is not refused and the body decides what it answers. Use it for a value Excel defines that
  the function does not compute (`CELL`'s `"width"`); the tag's text says why.
- **Also accepted:** an `as const` object with a same-name type alias, and a bare literal union
  (`mode: 0 | 1`, which shows the raw values with no labels). Numbers or strings, never mixed; the
  set need not be contiguous (`-1 | 0 | 1`).
- **Composes with the markers:** `Omittable<E, V>` (`V` must be a member or `null`), `AcceptError<E>`,
  `NoLift<E>`, `...values: E[]`. It is REFUSED under `IRange<E>`, `Flatten<E>`, `E[][]` and
  `Lazy<E>`: a range's members are never coerced one argument at a time.
- **`IRange<ReportType<E>>` documents a range's members.** Nothing is checked, and the body reads the
  range itself; the enum drives the dropdown, the help and the warning (`SORT`'s `sort_order`, one
  order per key). It is not `ReportType<IRange>`, which is about how a multi-area reference arrives.
- **Not imported.** An enum from another file is invisible to the compiler (SXL2052).

In the formula editor and the Function Arguments dialog, an empty argument of an enum type offers
its members as a dropdown, each row its label then its value; choosing one inserts the value. Any
other text is still accepted. A literal outside the set is
underlined as a warning in the editor, and never refused there.

### IReferenceRange

Use only when the function returns or requires a live cell reference (`INDIRECT`,
`OFFSET`). This registers a dynamic precedent in the dependency graph. Most functions
that look like they need this do not.

---

## IRange: the body of every function

Formula functions are pure transformations: `IRange -> IRange`. `ICellRange` is the
mutable sheet-level API used by the UI and `run` code — do not use it in formulas.

**The `IRange` surface itself lives in [vector](vector.md)** — iteration with
`entries()`, building output with `sparseBuilder()`, the `mask`/`choose`/`select`
dispatch primitives, shape and coercion. Load it before writing a function body;
everything there applies unchanged here, because a function body *is* vector code.

Two things are specific to formulas rather than to `IRange`:

- Arguments arrive as `IRange` already and the return value is one — there is no
  `getValuesRange()` bridge inside a function, and no `ICellRange` in scope.
- `FormulaContext.createRange()` is how you make a new range to return as a
  dynamic-array result (below).

---

## FormulaContext

`FormulaContext` is the sandboxed gateway from a formula function to the calculation
runtime. Import it from `@sheetxl/primitives` and call its static methods directly —
no instantiation, no passing it around. The engine configures it automatically before
each function call so every method is already scoped to the evaluating function's cell
and sheet.

This is the **only** way a function should access the host runtime. Reaching outside
(e.g. via `@sheetxl/sdk` imports or global state) is not supported and breaks the
sandbox contract.

```typescript
import { FormulaContext } from '@sheetxl/primitives';

export function NOW(): number {
  FormulaContext.markVolatile();
  FormulaContext.formatResults('Short DateTime');
  return FormulaContext.getNow();   // deterministic, session-scoped time
}
```

### Method reference

| Method | Purpose |
|---|---|
| `FormulaContext.markVolatile()` | Recalculate every cycle: `RAND`, `NOW`, `TODAY` |
| `FormulaContext.getNow()` | Current time as a serial date — deterministic and session-scoped. Never call `Date.now()` directly |
| `FormulaContext.getRandom()` | A draw in `[0, 1)` — the runtime's source, which a host can seed (a seeded draw is keyed by cell and pass, so take draws where they are used). Never call `Math.random()` directly |
| `FormulaContext.getEntireCoords()` | Destination cell bounds for array functions (e.g. `SEQUENCE`) |
| `FormulaContext.formatResults(numFmt)` | Hint engine to apply number format to result cell |
| `FormulaContext.getFunction(name)` | Call another registered function by name |
| `FormulaContext.createRange()` | Create a new `IRange` to return as a dynamic array result |
| `FormulaContext.getReference(coords)` | Resolve a dynamic range reference (adds a tracked dependency) |
| `FormulaContext.toSerialDate(date)` / `fromSerialDate(n)` | Convert between JS `Date` and Excel serial numbers |
| `FormulaContext.parseAsDate(v)` / `parseAsSerialDate(v)` | Parse a scalar (string or number) into a date |
| `FormulaContext.parseAsNumber(v)` | Parse a scalar into a number |
| `FormulaContext.filterHidden(range)` | Strip hidden rows/columns (used by `SUBTOTAL`, `AGGREGATE`) |
| `FormulaContext.getRow()` / `getColumn()` | 0-based coords of the cell being evaluated |

---

## JSDoc Annotations (Required)

The comment text above the tags is the function's description: it is shown in the formula dropdown and tooltips. A function without one gets a warning.

| Tag | Notes |
|---|---|
| `@param <n>` | Should match parameter names exactly — mismatches produce a warning. |
| `@name` | Required when the Excel function name contains symbols invalid in TypeScript (e.g. `.` in `CEILING.MATH`, `PERCENTILE.INC`). Maps the export name to the Excel name. |
| `@category` | UI grouping. Defaults to `Custom`. UDFs can use any category name, including the built-in ones. |
| `@description` | Long description for the detail panel. |
| `@hidden` | Hides from UI but remains available to the calc engine. Always include a reason string. |

```typescript
/**
 * Returns a number rounded up to the nearest integer or multiple of significance.
 * @name CEILING.MATH
 */
export function CEILING_MATH(number: number, significance?: number, mode: number = 0): number { ... }
```

---

## Error Handling

Prefer `return` for propagated errors (faster). Use `throw` to abort immediately.

```typescript
if (FormulaError.isError(v)) return v;   // propagate
return FormulaError.BuiltIn.Value;        // #VALUE!
return FormulaError.BuiltIn.NA;           // #N/A
return FormulaError.BuiltIn.Div0;         // #DIV/0!
```

Automatic JS-to-error mappings:

| Return value | Becomes |
|---|---|
| `Number.NaN` | `#NUM!` |
| `Infinity` / `-Infinity` | `#NUM!` |
| `undefined` | `#VALUE!` |

### Write plain arithmetic — but say `#DIV/0!` out loud

The engine normalizes every result the way Excel does: a non-finite becomes `#NUM!`, and a
magnitude below the smallest normal double flushes to `0`. So ordinary JavaScript is already
correct for overflow and underflow, and a function body should not check for them.

```typescript
export function SQUARE(x: number): number { return x * x; }   // 9E+307 -> #NUM!, free
```

`#DIV/0!` is the one error that does **not** fall out of the arithmetic. Excel never derives it
from a float — it raises it deliberately before dividing — and in JavaScript `x / 0` is the same
`Infinity` an overflow produces, which the rule above reports as `#NUM!`. A function that can
divide by zero has to say so:

```typescript
export function RATIO(a: number, b: number): number {
  if (b === 0) return FormulaError.BuiltIn.Div0;   // WITHOUT this, RATIO(1, 0) is #NUM!
  return a / b;
}
```

The same applies to any implicit division — `1 / Math.sin(0)`, `Math.log(x) / Math.log(1)`,
a mean over an empty sample. If Excel answers `#DIV/0!` for the degenerate input, guard it.

Never hand back a raw `Infinity` to mean `#DIV/0!`; it means `#NUM!`.

---

## Return Values

| Return type | Spill behavior |
|---|---|
| `Scalar` | Single cell |
| `Scalar[]` | Spills as a single row |
| `Scalar[][]` | Spills as a 2D block (outer array = rows) |
| `IRange` | Spills; preferred for large or sparse results |

---

## Coercion

| Source | Aggregation fns (SUM, AVERAGE...) | Non-aggregation fns (ROUND, etc.) |
|---|---|---|
| Cell reference | Boolean skipped | Boolean coerced |
| Literal scalar arg | Coerced | Coerced |
| Literal array `{TRUE}` | Skipped | Skipped silently |

Empirically verified — verify any new aggregation against all of these:

- `=AVERAGE(1, TRUE)` -> `0.667` (literal scalar TRUE coerces)
- `=AVERAGE(A1, B1)` (B1=TRUE) -> `1.0` (cell-ref boolean skipped)
- `=AVERAGE(1, {TRUE})` -> `0.5` (array boolean skipped)
- `=AVERAGE(1, "TRUE")` -> `#VALUE!` (string literal errors)

---

## Contributing built-ins (SheetXL repo only)

This section applies when working **inside the SheetXL repository** on the shipped
function set — not when authoring a UDF in your own project.

- Built-ins live in `packages/formulas` (`@sheetxl/formulas`). The signature
  determines the test cases; the implementation can be reviewed for edge cases,
  but Excel behavior — not the SheetXL implementation — is the oracle.
- Every implementation requires a fixture file, manually validated against Excel:

  ```text
  packages/sdk/src/__tests__/functions/<category>/<FUNCTIONNAME>.json
  ```

  (Categories: `mathTrig`, `statistical`, `financial`, `text`, `logical`,
  `lookupReference`, `engineering`, `information`, `database`, `operators`.
  Closely related functions may share one file, e.g. `CEILING-FLOOR.json`.)

**Simple** — all arguments are literals:

```json
[{
  "label": "DISC",
  "simple": {
    "DISC(\"01/04/2023\", \"01/31/2023\", 99.71275, 100, 0)": 0.0383,
    "DISC(\"01/04/2023\", \"01/31/2023\", 99.71275, 100, -1)": "#NUM!"
  }
}]
```

**Detailed** — formula references cell addresses:

```json
[{
  "label": "single criterion",
  "formula": "COUNTIF(A1:A4, \">2\")",
  "inputs": {
    "A1": [
      [1],
      [2],
      [3],
      [4]
    ]
  },
  "expect": 2
}]
```

Every fixture must cover: happy path, each non-default optional argument, error
propagation and error conditions, coercion edge cases (text, string, boolean),
implicit lifting for scalar parameters, broadcast/spill cases, literal **and**
referenced inputs, and at least one non-A1 origin address (detailed fixtures only).
