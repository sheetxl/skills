---
name: writing
description: "SheetXL sheet interaction: reading and writing cell values, navigating ranges, applying styles, working with the formula engine — and verifying the change. Load when changing or reading a sheet: setting values, writing formulas, the cursor pattern, styling ranges. Triggers on: ICellRange, ICell, getRange, getRangeAt, getValue, setValue, getText, getCell, setValues, setComputedValues, getStyle, setNumberFormat, number format, getColumnAt, getRowAfter, getCalculation, ready, doBatch, fill a formula across, cursor pattern, navigation."
---

# Writing — Sheet Interaction

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. Code below
> is run-context code (`scripting.md`): snippet form — no `await`, and nothing to
> import (the context is already in scope); every statement auto-awaits. Files
> open via the workbook argument and save via `--save` (CLI) or sessions
> (MCP); application code goes through `node-app.md` or `react-app.md`. To understand structure
> first, see `reading.md`.

**In this file:** Coordinates: two notations · Values: two entry modes (write
and read via `ICellRange`) · Formulas and the calculation engine · Navigation
(suffix convention, methods, idioms) · The cursor pattern · Styling · Protected
sheets · After writing: verify

This reference covers interacting with a sheet — reading and writing cell
values, navigating ranges, and applying styles. In an attended session,
changes appear in the user's UI immediately — follow the rules in
`interactive.md`.

---

## Coordinates: Two Notations

All index parameters are 0-based. A1 = row 0, column 0. Both notations address the
same cells — use whichever makes the intent clearest.

```typescript
// A1 string notation — natural for known addresses and formula strings
sheet.getRange('B2:D10');
sheet.getRange('D1').setValue('=SUM(A1:C1)');

// 0-based index notation — natural for programmatic and loop-driven navigation
sheet.getRangeAt(1, 1);         // B2 (row, col)
sheet.getRangeAt(1, 1, 9, 3);   // B2:D10 (rowStart, colStart, rowEnd, colEnd)
```

---

## Values: Two Entry Modes

These are not interchangeable — the display mode goes through the SheetXL text
parser, which matches Excel input conventions: a literal (`42%`, `1/31/2026`)
gives a General cell its matching number format, and a cell that already has one
keeps it. A formula keeps the cell's format either way.
`setValue('42%')` and `setValue(0.42)` produce different results: the first stores
`0.42` with a percent format applied; the second stores `0.42` with no format.

The full encoding contract — `'` text prefix, formulas, errors, round-trip — is
the **Value encoding** section of the root `SKILL.md`. It is the same rule in
both directions and is not repeated here.

### Writing — ICellRange is the write surface

```typescript
const range = sheet.getRange('A1');     // ICellRange

range.setValue('42%');                  // display mode — parser applies
range.setValue(0.42);                   // typed mode — bypasses parser
range.setValue('=SUM(A1:C1)');          // '=' prefix triggers formula engine
range.setValue("'123");                 // leading apostrophe forces text storage
range.setValues([[1, 2], [3, 4]]);      // 2D array, rows x columns
```

`setValue` writes to the **entire range**, and a formula written that way is a
fill, not a copy: it is read as the top-left cell's formula and every other cell
gets it with relative references shifted and `$` references pinned, exactly as
typing it and filling across would. A 60-month row is one call —
`sheet.getRange('C5:BJ5').setValue('=B5*(1+$B$2)')` — never a loop that builds
an A1 string per cell.

**Transforming what is already there? Do not read it out and write it back.**
`setComputedValues(fn)` runs the transform inside the engine and the values never
round-trip through your runtime — it and the whole `IRange` surface are in
[vector](vector.md). Load it before writing the transform. Reach for
`getValues()` + a loop + `setValues()` only when you genuinely need JavaScript
values in hand, which for a transform you do not: the loop moves the work out of
the engine and copies the grid to do it.

### Reading — ICellRange is the read surface, ICell for per-cell detail

`ICellRange` reads with `getValue()` (the top-left scalar) and `getValues()`; to *see*
a range, return it and its report renders. **`getText()` is on `ICell`, not on the
range** — `getCell()` gives per-cell detail when you need it (display text, type,
formula text, interop type) and defaults to the top-left cell:

```typescript
const range = sheet.getRange('A1');  // ICellRange — write, navigate, read
const cell  = range.getCell();       // ICell — per-cell detail
```

`ICell` only reads: there is no `setValue` or `setStyle` on it. Write through the range.

`range.getFormulas()` reads a whole range's formula text (`'=SUM(A1:C1)'`, `''` where a cell has no formula), and `setFormulas()` writes it back.

`getValue()`, `getText()`, and `getType()` are three lenses on the same stored value.
`getType()` reflects what the engine actually stored — not what you wrote or what it
looks like:

```typescript
// Percentage — format inferred from input
range.setValue('42%');
cell.getValue()  // => 0.42
cell.getText()   // => '42%'
cell.getType()   // => ScalarType.Number

// Date — stored as Excel serial number, formatted back through the cell's format
range.setValue('2024-01-01');
cell.getValue()  // => 45292             (Excel date serial)
cell.getText()   // => '1/1/2024'        (the inferred format, not the input spelling)
cell.getType()   // => ScalarType.Number  — dates are numbers

// Formula — type reflects the computed result, not the expression
range.setValue('=SUM(A1:C1)');
cell.getValue()       // => 60
cell.getText()        // => '60'
cell.getType()        // => ScalarType.Number
cell.getFormulaText() // => '=SUM(A1:C1)'  (null for non-formula cells)

// Force text — leading apostrophe, same convention as Excel
range.setValue("'123");
cell.getValue()  // => '123'
cell.getText()   // => "'123"            (apostrophe restored for round-trip safety)
cell.getType()   // => ScalarType.String

// Error — error values are a first-class value type
range.setValue('#DIV/0!');  // or '=1/0'
cell.getValue()  // => FormulaError.Div0
cell.getText()   // => '#DIV/0!'
cell.getType()   // => ScalarType.Error
```

`getText()` round-trips through `setValue()` — a value read with `getText()` can
always be written back and interpreted identically. That is the round trip, not
the input spelling: `'2024-01-01'` comes back as `'1/1/2024'`, and both name the
same serial.

---

## Formulas and the Calculation Engine

`ready()` is only needed when you write a formula and read its result in the same
run. Not needed for read-only or write-only workflows.

```typescript
sheet.getRange('D1').setValue('=SUM(A1:C1)');
wb.getCalculation().ready();
sheet.getRange('D1').getValue()
```

`ready()` returns a promise. A snippet awaits it for you, like every top-level
statement; a script (default export, `scripting.md`) must `await` it, or the read
that follows sees the value from before the recalculation. That is the whole
what-if loop — set an input, wait, read the output:

```typescript
export default async function main({ wb }: SheetXL.IScript.ScriptContext): Promise<number[]> {
  const sheet = wb.getWorksheet(), results: number[] = [];
  for (const rate of [0.03, 0.04, 0.05]) {
    sheet.getRange('B2').setValue(rate);
    await wb.getCalculation().ready();
    results.push(sheet.getRange('B20').getValue() as number);
  }
  return results;
}
```

Wrap related writes in `doBatch` — on the range — to make them one operation
(one recalculation, one undo step):

```typescript
sheet.getRange('A1:D1').doBatch(() => {
  sheet.getRange('A1:C1').setValues([[10, 20, 30]]);
  sheet.getRange('D1').setValue('=SUM(A1:C1)');
});
wb.getCalculation().ready();
```

(On MCP, every `run` is already a single commit — `doBatch` matters there for
grouping recalculation, not undo.)

### Defining a name

`addFormula(name, expression, replaceExisting?)` — positional, not an options
bag. Reached through a sheet it is scoped to that sheet; through the workbook
it is global. Redefine an existing one with `setExpression`, or pass
`replaceExisting`; without it a collision throws rather than silently dropping
the new expression.

```typescript
wb.getNames().addFormula('TaxRate', '=0.21');              // global
sheet.getNames().addFormula('Rate', '=0.19');              // scoped to that sheet
wb.getNames().getByName('TaxRate').setExpression('=0.22'); // redefine, dependents keep resolving
```

Scope is half a name's identity, so there is no `setScope` — to move one,
delete it and define it again. Reading a name (`getValue`, `getRanges`,
`evaluate`) is in `reading.md`.

---

## Navigation

`ICellRange` is a live reference to sheet cells — not a snapshot. Navigation methods
return a new `ICellRange`; the original is unchanged.

### Suffix convention

| Suffix | Mode | Behavior |
|---|---|---|
| `At(n)` | Absolute 0-based | Goes to that exact row/column regardless of current position |
| `By(n)` | Relative delta from start | Moves n steps from `rowStart`/`colStart` |
| `After` | One past the end | Row/column immediately after `rowEnd`/`colEnd` |
| `Before` | One before the start | Row/column immediately before `rowStart`/`colStart` |
| `To(end)` | Absolute end position | Moves the bottom-right corner to an absolute coordinate |

> **`At` is absolute, not relative.** `cursor.getColumnAt(1)` is always column B
> even if `cursor` is in column E.

> **`By` vs `After`:** `getRowBy(0)` is the first row of the range (identity for a
> 1-row cursor). To advance past the end use `getRowAfter()`, not `getRowBy(height)`.

### Methods

```typescript
// Row navigation — result is 1 row tall, same column span
range.getRowAt(5)       // row index 5 (0-based, absolute)
range.getRowBy(2)       // 2 rows past rowStart
range.getRowAfter()     // row immediately after rowEnd
range.getRowBefore()    // row immediately before rowStart

// Column navigation — result is 1 column wide, same row span
range.getColumnAt(2)    // column C (index 2, absolute)
range.getColumnBy(1)    // 1 column past colStart
range.getColumnAfter()
range.getColumnBefore()

// Offset — reposition top-left, size unchanged
range.getOffsetBy(rowDelta, colDelta)
range.getOffsetTo(rowStart, colStart)

// Resize — move bottom-right corner
range.getResizeBy(rowDelta, colDelta)  // additive: A1.getResizeBy(0,2) -> A1:C1
range.getResizeTo(rowEnd, colEnd)      // fixed end regardless of current size

// Other
range.getTopLeft()
range.getBottomRight()
range.getRangeAt(rowStart, colStart, rowEnd?, colEnd?)
range.getAddress()      // A1-style string, safe to embed in formula strings
```

`getEntireRange()` is on the **sheet**, not the range — `sheet.getEntireRange()`
is the whole grid.

### Common idioms

| Task | Form |
|---|---|
| Column B on current cursor row | `cursor.getColumnAt(1)` |
| First free row below data | `(sheet.getUsedRange()?.getRowAfter()) ?? sheet.getEntireRange().getTopLeft()` |
| Range address in a formula | `` `=SUM(${range.getAddress()})` `` |

---

## The Cursor Pattern

A cursor is an `ICellRange` that advances through a sheet by chaining return values.
Every range operation — `setValue`, `setValues`, navigation — returns the range it
affected. The cursor pattern makes this the primary idiom: always update `currentRange`
from the return value, so the cursor always reflects where the last operation landed.

`getColumnAt(n)` addresses columns by absolute index, independent of the cursor's
current column — this is what makes it safe to reach across columns without losing
your position.

```typescript
let currentRange = sheet.getRangeAt(0, 0);  // start at A1

for (const item of data) {
  currentRange = currentRange.getColumnAt(0).setValue(`'${item.label}`);  // col A
  currentRange = currentRange.getColumnAt(1).setValue(item.value);        // col B
  currentRange = currentRange.getColumnAt(2).setValues(item.matrix);      // col C, may be multi-row

  currentRange = currentRange.getRowAfter();  // next row
}
```

The cursor naturally handles variable-height writes: when `setValues` writes a
multi-row matrix, `currentRange` reflects the full written extent, so `getRowAfter()`
always skips past all of it.

---

## Styling

`getStyle()` applies to the entire range it is called on. Style the full target range
in one call — do not loop over individual cells.

(Reading is the other half of the Excel-ribbon contract: a range's `getStyle()`
*reads* as the anchor cell's style, so it is not a uniformity check — that is
`getRegions(<style features>)` returning a single region; see `reading.md`.)

```typescript
sheet.getRange('A1:E1').getStyle().getFont().setWeight(700);
sheet.getRange('A1:E1').getStyle().getFill().setColor('#EEEEEE');
range.getStyle().getBorders().setOutline({ style: 'double', color: 'blue' });
range.getStyle().getFont().setColor('#FF0000');  // note: setColor not setFill
range.getStyle().setNumberFormat('0.00%');       // any Excel format code: '#,##0', 'd-mmm-yyyy', …
```

The number format is on the style, not the range or the cell. Display-mode input
infers one for a literal in a General cell (`setValue('42%')`); a formula's result is never formatted
for you, so set it on any formula cell whose result is a date, time or percent,
and on typed values.

---

## Protected sheets

A protected sheet refuses writes, and a cell that will refuse one looks exactly
like a cell that will take it. The reports say so **only when protection is in
force** — an unprotected sheet says nothing — and at the altitude you are about to
write from:

| You returned | What it says |
|---|---|
| `wb` | `(protected)` on the sheet's heading, and a `protected` line: where a write is still accepted, how many cells, whether a password is set, whether formulas are hidden. A protected *workbook* adds `Protection: structure locked` under the title — sheets cannot be added, deleted, renamed or moved |
| the sheet | a `Protection` section: the operations still **allowed** (`sort`, `format cells`, `insert rows` … — anything not listed is refused), and the editable regions, each with why it is editable |
| a range | `Protection: sheet locked — 3 of 12 cells editable (B2:B4)` as its first line |
| a cell | `Editable: true` / `false` |

```text
## Protection (locked · password)
  allowed   sort, format cells
  editable  25 cells · 2 regions
  B4:B22  19×1  always unlocked
  D4:F5   2×3   range Inputs · password
```

- **`always unlocked`** is a cell format and asks for nothing. **`range Name`** is
  an unlocked range (`sheet.getProtectedRanges()`); with `· password` it takes
  writes only after `item.unlock(password)`.
- The editable regions are the `unlock` partition —
  `sheet.getRegions('unlock')` returns them to loop over.
- A refused write fails with the sheet's own message, and the run report points
  at the two lookups above. **Write where the sheet is editable** — that is what
  the author left open, and on a form it is the whole task.
- When the task really does need a locked cell: `sheet.getProtection().pause()`
  suspends enforcement for the session and `resume()` restores it exactly;
  `unlock()` is a document change — the file saves unprotected. Both need the
  password when one is set, and that is the user's to supply.
- **Hidden formulas** (`formulas hidden`) read as values everywhere — `getFormula()`
  is `null`, and the cell is absent from the `formula` partition. A formula census
  on such a sheet is short by however many are hidden.

---

## After writing: verify

Command success is not correctness evidence. Read back the ranges you changed and
confirm they hold what you intended — values, formulas, and (after
`getCalculation().ready()`) computed results — before reporting the change done.
Fixing errors? The recount discipline lives in `analysis.md`.
