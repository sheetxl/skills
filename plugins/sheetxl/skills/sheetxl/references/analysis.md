---
name: analysis
description: "Audit and repair a workbook — find what is wrong and prove you fixed it. The anomalies read — what the engine has already flagged as breaking a pattern, ranked, with its evidence; the shape sweep that finds defects before you look at a single value; the catalogue of which convention lives in which feature partition, where a hardcode is a hole and an outlier is a one-cell region; sibling sheets as a control group; formula errors (#REF!, #VALUE!, #NAME?, #DIV/0!, #N/A, circular references) counted, located and traced to their root; the blast radius before you write; fixing at the root; and verifying by recount and re-partition. Load when the task is auditing, reviewing, checking, explaining or repairing a spreadsheet, or when asked whether a model is correct. Triggers on: analyze, audit, review, check, is this right, find bugs, fix my sheet, broken spreadsheet, formula error, #REF, #VALUE, #NAME, #DIV/0, #N/A, circular reference, hardcode, broken link, inconsistent, sanity check, explain this sheet, what is wrong, root cause, precedents, dependents, blast radius, sign convention, unit mismatch, color convention, anomalies, flagged, what breaks the pattern, verify the fix, best practice violations."
---

# Analysis — Audit and Repair

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing,
> `reading.md` for the read surface this builds on, and `writing.md` for making
> the change. Code below is run-context code (`scripting.md`): snippet form — no
> `await`, nothing to import.

**In this file:** Loud — count and locate · Flagged — the anomalies read ·
Silent — the shape sweep · Re-partition on the convention's dimension · Sibling sheets are a control group
· Trace to the root · The blast radius · Fix at the root · Verify ·
Finding nothing · Reading values — the cheapest rung · The order, end to end ·
Attended workbooks

A spreadsheet with no errors is not a spreadsheet with no bugs. `#REF!` and
`#DIV/0!` are the defects that announce themselves; the expensive ones are
silent — a hardcoded number where a formula belongs, a total that stopped
summing, a link that was pasted over as a value. None of them raise an error,
and none of them are visible in a printout of the numbers.

They are all visible as **shape**.

| | Symptom | How you find it |
|---|---|---|
| **Loud** | the cell displays an error | count and locate — cheap, do it first |
| **Flagged** | it breaks the pattern around it, and the engine has already said so | read the `Anomalies` section, highest rank first |
| **Silent** | it computes, and it is wrong, and nothing around it disagrees | partition, and read the small regions |

Command success is never the evidence. For loud defects the evidence is a falling
error count; for flagged and silent ones it is a finding that has gone, or a
partition that has stopped having outliers. Everything from *"what does this
touch"* onward is the same for all three.

## Loud — count and locate

The workbook report already did the counting: its roster carries an error count
per sheet, and its `Errors` and `Calc Failures` sections list the regions,
sheet-qualified, each with the error literal and the formula behind it. `Errors`
also lists chart series whose reference has gone `#REF!` — counted apart
(`3 · 1 in charts`), because no cell holds them and no cell scan finds them. That is
your baseline. In code, when you want the number alone:

```typescript
const counts = sheet.getStats().getTypedCounts();
// counts.error ?? 0 — your "before" (an empty sheet is empty statistics)
```

Two kinds, kept apart on purpose: an **error** is a successfully calculated error
*value* (`#DIV/0!` from `=1/0`), a **calc failure** is the engine failing to
evaluate at all — mostly circular references. A book with zero of one and three
of the other is still badly broken.

To locate, either entrance works. `grep` matches display text, so error tokens
work directly; `getRegions('error')` partitions by error *value*, so a column of
forty `#N/A`s is one row rather than forty matches:

```bash
sheetxl grep "#REF!" book.xlsx
sheetxl grep "#" book.xlsx                                        # broad sweep
sheetxl -e "wb.getWorksheet('Model').getRegions('error')" book.xlsx   # grouped by value
```

| Error | Usual root cause |
|---|---|
| `#REF!` | deleted rows/columns/sheet a reference pointed at |
| `#NAME?` | misspelled function or an undefined name |
| `#VALUE!` | text where a number is expected (often an upstream cell) |
| `#DIV/0!` | empty or zero divisor — often a missing input, not a formula bug |
| `#N/A` | lookup with no match — check the lookup range and key |

Then jump to *Trace to the root*, below. The rest of the finding half is for the
defects that never show a `#`.

## Flagged — the anomalies read

The engine runs the pattern checks before you ask. Every sheet report carries an
`Anomalies` section directly after `Calc Failures`; the workbook roster counts
each sheet's (`anomalies B17:N73  64 cells · 18 regions · formula 15 ·
sumIncludesSubtotal 5 · …`) and `wb` closes with the workbook's top findings; a
cell report lists the findings at that cell. The list alone, on any range, sheet
or the workbook, is `getAnomalies()`, highest rank first; `getCounts('anomalies')`
says whether there is one to read. When the task names the kind of defect, read
`wb.getAnomalies({ focus, limit: 25 })` first, over the whole workbook, the next
window with `offset: 25` (the focus table below maps the task's words to a
`focus`); the ranked sheet lists come after.

```typescript
wb.getWorksheet('LBO').getAnomalies()
```

```text
## Anomalies (56 cells · 24 regions)
  C10      1×1  lookup key is not the label · MATCH key "EBITDA" in row "Entry EBITDA" ($B$10) · here =@INDEX('Ex 10 - …'!C:C,MATCH("EBITDA",…,0))
  Q15:V15  1×6  typed value in a run of formulas · the cells above and below are formulas · here $0.0
  J62      1×1  refers to an empty cell · reads I62 · here =I62
  P13      1×1  font format break · {"weight":700} here, {…} above and below · also changes: fill, alignment · here 'Exit TEV
  C28      1×1  link label mismatch · row "LIBOR" ($B$28) links to "Senior Secured Term Loan B" ('Ex 12 - Debt Schedule'!$B$10) · here ='Ex 12 - Debt Schedule'!F10
  …
```

One line per finding: a run of cells with the same finding reports once, and the
same finding in several places is one line ending `also at …` with the others. Each
line is the address, the check, **the evidence** — the run it breaks, the cell it
reads, the label it disagrees with, with addresses — and `here`, the anchor's own
content. The constants formulas type are not lines here: they are the `Constants`
section after it, each constant with how many places in the workbook type it,
rarest first — a `12` typed in two places where the model types `365` in twenty is
the line to read. **The list is ranked**: read it top down, and stop where the lines turn
into the author's style rather than breaks in it. A line is evidence, never a
verdict; the next call is yours, and it is usually the cell report at that address.

Reading the lines above: `Q15:V15` is a typed row inside a formula block — the
classic overwritten total. `C28` is a row labeled LIBOR linking to a row labeled
Senior Secured Term Loan B — a link to the wrong row. `P13`, by contrast, carries
`also changes: fill, alignment`: a bold cell where the fill and alignment change at
the same edges is a header, and the line says so. That is the one reading rule
worth holding: **a lone break is the candidate; a break the other layers share is
structure.** `one cell each side` marks thin evidence, a cell between two that
happen to agree. For `format is another role's`, the `N of the sheet's M` is the
strength of the convention it breaks.

The checks, as the report names them:

| The line says | It means | Then |
|---|---|---|
| `inconsistent formula` | this region's formula differs from the run on both sides of it | compare with the neighbor the line names; one of them is wrong |
| `typed value in a run of formulas` | a constant where the row or column is otherwise formulas | the overwritten formula; the neighbors say what it was |
| `omits adjacent cells` · `adds a cell outside its block` · `sum includes a subtotal` · `counted twice` | an aggregate's range disagrees with the block beside it, reaches into another block, holds its own subtotal, or names a cell twice | read the block the line names against the range |
| `aggregates nothing` | AVERAGE/MIN/MAX of one cell, or a SUM of typed numbers | the range it should cover is the block beside it |
| `refers to an empty cell` | a reference landing on nothing — often a relative copy that lost its `$` | the cell report: the value usually sits at the anchor's row or column |
| `link label mismatch` | a pure link whose source row label differs from its target's | the right row is the one whose label matches |
| `lookup key is not the label` · `approximate match` · `lookup arrays misaligned` | a MATCH key unlike the row's label; a lookup without exact match; INDEX and MATCH over different bounds | the lookup's inputs, one by one |
| `sign differs from sibling links` | a link negated where the other links into the same row are not | the sign convention of the row it reads |
| `typed number equals an unread input` | a number typed into a formula that equals an input cell nothing reads (`0.21` where Tax rate holds `0.21`) | the reference it replaced: the input the line names |
| `differs from its twin formulas` | formulas of the same shape that divide by `12` here and `365` there, or add a term the others take away | which side the business meant; the line names a twin |
| `off the scale of its format` | a typed percent a hundred times its siblings (`30` for 30%) | the unit it was typed in |
| `constant` (the `Constants` section) | a constant typed into a formula, with its uses of the workbook's | a constant typed once where the model types another everywhere |
| `<layer> format break` | an authored font, fill, number format or alignment thin against its run | with `also changes`, structure; alone, a cell someone touched |
| `<part> format is another role's` | a link, same-sheet reference or calculation colored as the sheet colors another role | the color convention, per sheet: the majority is the rule |
| `number stored as text` · `unlocked formula` · `reads outside the workbook` | Excel's own checks: text that parses as a number; a formula a user can type over; a reference to another workbook or a deleted one | as Excel would |

When the task names the kind of defect, pass it as `focus`: the list is then
ordered by how often findings like each line's have been that kind of defect in
audited models, every line still there. Measured on a debugging benchmark whose
tasks name the defect kind, with each model ranked from the others, this puts
about 60% more planted defects in the first 25 lines than the ranked list alone,
and more than reading a hand-picked set of checks first. Task names vary in
spelling (`Embedded Hardcodes`, `Incorrect SIgn Conventions`): go by the keyword.

| The task says | `focus` | Lines about it |
|---|---|---|
| a hardcode, a typed-over formula | `embeddedHardcode` | `typed number equals an unread input` · `typed value in a run of formulas` · `Constants` · `format is another role's` |
| double counting | `doubleCounting` | `counted twice` · `sum includes a subtotal` · `adds a cell outside its block` |
| an average over the wrong cells | `average` | `omits adjacent cells` · `aggregates nothing` · `adds a cell outside its block` |
| a bad link, the wrong row or sheet | `crossSheetReference` | `link label mismatch` · `refers to an empty cell` |
| a lookup | `lookup` | `lookup key is not the label` · `approximate match` · `lookup arrays misaligned` |
| a copied formula that lost its anchor | `referenceAnchoring` | `refers to an empty cell` · `inconsistent formula` |
| color coding | `colorCoding` | `format is another role's` · `format break` |
| a sign | `signConvention` | `differs from its twin formulas` (a term) · `sign differs from sibling links`; then `getStats()` per row |
| a unit (×12, ×1000, days for months) | `unitMismatch` | `differs from its twin formulas` (a literal) · `off the scale of its format` · `Constants` |
| deleted rows, broken references | `deletedRows` | `reads outside the workbook` and the `Errors` section |

The call, a window at a time, across every sheet; several kinds merge into one
order:

```typescript
wb.getAnomalies({ focus: 'unitMismatch', limit: 25 })
wb.getAnomalies({ focus: ['unitMismatch', 'embeddedHardcode'], limit: 25 })
```

The next window is the same call with `offset: 25`; a window shorter than the
limit is the last. On one sheet, `sheet.getAnomalies({ … })`; on a block,
`range.getAnomalies({ … })`. To read particular checks first instead, name them:
`getAnomalies({ checks: ['hardcodeInRun', 'styleRole'] })`; with `focus` as well,
the named checks still list first.

For a sign or a unit, know first what the line should carry: [financial-models](financial-models.md)
has the standard lines' signs and units per model type.

**What the read sees is a break.** A row that is wrong in every one of its cells
breaks nothing beside itself: the same wrong formula from first period to last
is one consistent region, and the rows above and below are other line items. Those
are the shape sweep's and the sibling diff's, next, and the semantic judgments —
monthly or daily, deduction or addition, which of two literals the business meant
— are yours once the candidates are in front of you.

## Silent — the shape sweep

**Before drilling into any sheet, compare the workbook's own descriptions of
itself against each other.** Every feature is a different view of the same
cells. When two views disagree about the shape of the same block, the
disagreement is the finding.

The workbook report is most of the sweep for free — its roster carries cell,
formula and error counts per sheet. One call, then read across:

| Compare | The signal | What it means |
|---|---|---|
| **formula count vs. cell count**, per column or row of a block | a column with one fewer formula than its neighbors | a formula was typed over with a value |
| **formula count across sibling sheets** | a filled sheet with *fewer* formulas than its blank template | the model lost a link somewhere it originally had one |
| **region geometry vs. block geometry** | a `formula` partition that splits where the `contentful` block does not | a hole — and the hole is the defect |
| **feature count vs. feature regions** | many cells, one region — or many cells, many regions | uniformity, or fragmentation that should not be there |

The last one is the general form of the rule: **a count is presence, a partition
is identity.** Twelve cells carrying a formula is a count. Whether those twelve
are *one* region or *two* is the audit. Each sheet's report states both on every
section heading — `Formula (401 cells · 2 regions)` — so the ratio is already in
front of you: 401 cells in 2 regions is a model, 401 in 300 is noise.

For a sweep across sheets that the report does not shape the way you need, the
loop belongs in the run:

```typescript
for (const sheet of wb.getWorksheets()) {
  const used = sheet.getUsedRange();
  if (!used) continue;                 // an empty sheet has no shape to compare
  log(sheet.getName(), sheet.getStats().getCount(), used.getCounts(['formula']));
}
```

### Why shape beats values

A number cannot tell you it is wrong. `453.5` in a totals row looks exactly like
a correct total, at any precision a human will check — and if someone typed it
from the displayed value, it is correct to one decimal and wrong forever after.

But the *shape* of a totals row is unambiguous: every cell in it should carry the
same authored formula. That is a claim the engine can check across the whole row
in one call, and it does not depend on knowing what the right answer is.

**Audit the invariant, not the output.**

## Re-partition on the convention's dimension

`getRegions()` with no argument gives `contentful` — geometry, *where the data
is*. That view is for orientation and it will tell you a broken model looks fine.

Every other feature partitions on **payload identity**: cells belong to the same
region when they agree on the feature itself. Pass the feature the convention
lives in, and the convention becomes checkable.

**This is the catalogue.** One table, and it is the whole method — pick the row
whose left column is the claim you are testing. The anomalies read has already
run the first four rows for you (`inconsistent formula`, `typed value in a run of
formulas`, `omits adjacent cells`, `format is another role's`); re-partition
yourself when the convention is one it does not name, or when you want the whole
partition rather than its breaks:

| The convention, or the defect | Partition on | What the break looks like |
|---|---|---|
| this row/block should all compute the same way | `formula` | a **one-cell region** in a run of hundreds |
| this block should be formulas, not typed-in numbers | `formula` | a **hole** — a typed number carries no formula, so it splits the region rather than joining it |
| an aggregate should cover the block it summarizes | `formula` + region bounds | the `SUM`/`AVERAGE` range disagrees with the `contentful` block beside it |
| inputs blue, formulas black, cross-sheet links green | `formula` × `font` | a formula class paired with the wrong color |
| these should all be currency / all be thousands | `numberFormat` | `General` cells inside a formatted run; thousands beside units in one row |
| this heading should be one merged run | `merge` | the run splits where someone unmerged |
| the sign convention holds down a column | `numberFormat`, or `getStats()` per column | a min/max sign that disagrees with its neighbors |
| inputs unlocked, formulas locked (a protected form) | `formula` × `unlock` | a formula inside an editable region — a user can type over the model; or an input cell that is not in the `unlock` partition and cannot be filled in |
| nothing here should be an error | `error` | the partition exists at all — it groups by error value |

Two shapes, and both are readable at a glance: **a hole** (the feature is absent
where it should be) and **a one-cell region** (the feature is present but
different). Everything in the table is one of the two.

```typescript
wb.getWorksheet('Profit and Loss (Monthly)').getRegions(['formula'])
```

```text
# Regions — formula (4)
  N14:N25  12×1  =SUM(N2:N13)
  B26:G26  1×6   =SUM(B14:B25)
  I26:N26  1×6   =SUM(I14:I25)
  B27:N27  1×13  =B26-B21
```

Row 26 is `Total Expenses`. Eleven months carry `=SUM(x14:x25)`; July — `H26` —
carries a typed number, so it is not in the partition at all and the row comes
back split in two. The partition did not need to be told what a total is, what
the right value was, or which month to suspect. **The gap is the bug report.**

Cross two features when the *relationship* is the question — `formula` × `font`
pairs what each cell computes with the color a human sees, so a cross-sheet link
that is not green is one row of output. `reading.md` shows the rendered shape
and the rule for how many features to cross.

**A small region is a candidate, not a verdict.** A one-cell region means *this
cell is different*, not *this cell is wrong*. Sometimes the outlier is correct and
the run around it is the bug — a subtotal row genuinely differs from the rows it
totals. And if a model consistently does something unusual, that is its
convention, not a defect: fix what breaks the pattern, do not impose a pattern the
model never had. Read the formula and decide before you edit.

**A protected sheet that hides its formulas cannot be audited this way.** Its
report says `formulas hidden`, and there a formula reads as a value: it is absent
from the `formula` partition, so every hidden formula looks like a hole. Say so
rather than reporting the holes — or have the user pause the protection first
(`writing.md` §"Protected sheets").

**Partition, do not scan.** The moment you find yourself reading cells to compare
them, stop — that is the failure mode this exists to replace.

## Sibling sheets are a control group

Real workbooks ship their own reference implementation, and it is the cheapest
evidence available. Template pairs (`Monthly` / `Monthly BLANK`), repeated
quarters, one sheet per region, last year's tab next to this year's — each twin
is the same structure with different data.

Run the identical partition on both and diff the shape:

```text
'Profit and Loss (Monthly)'         B26:G26  I26:N26      ← split
'Profit and Loss (Monthly BLANK)'   B26:N26               ← whole
```

That upgrades *"H26 looks odd"* to *"H26 was overwritten"* — same file, no
outside knowledge, one extra call. When a workbook has sibling sheets, comparing
them is not an optional deepening of the audit; it is the audit.

## Trace to the root

Errors cascade: one bad input turns a whole dependency chain into `#VALUE!`. Fixing
downstream symptoms wastes edits and hides the cause. **The cell is the diagnostic
lens**; the formula object is the second one, and it answers a different question:

```bash
sheetxl -e "wb.getRange('D7').getCell()" book.xlsx                          # value, formula text, the graph
sheetxl -e "wb.getRange('D7').getFormula()" book.xlsx                       # the expression analyzed
sheetxl -e "wb.getRange('D7').createFormula('=<formula>')" book.xlsx  # a what-if at D7, unwritten
```

The **cell** report shows the value, the formula text and the resolved precedents —
follow the chain upstream until you reach a cell whose error is *not* inherited.
That cell is the root.

The **formula** report answers the other half: the value, then the expression, its
calculation status, its resolved symbols (a defined name that no longer resolves, a
function that does not exist — the two silent `#NAME?` causes), its indexed
references and its tokens. Reach for it when the *text* is the suspect and the
value alone did not say why. What it does not carry is the graph — precedents and
dependents live on the cell and the range, which is why the cell comes first.

## Before you edit — the blast radius

Tracing upstream tells you *what is wrong*. It does not tell you *what else your fix
moves*, and a correct fix to the wrong cell is still a broken model. So read the
graph the other way before you write — and the report you already have has probably
answered it: every cell, range and name report carries `## Precedents` and
`## Dependents`, so the `getCell()` you ran during the trace was also the blast
radius call.

```bash
sheetxl -e "wb.getRange('H26')" book.xlsx   # values, formulas, and both trees
```

For one direction on its own:

| | reads | is read by |
|---|---|---|
| one hop | `getDirectPrecedents()` | `getDirectDependents()` |
| whole chain | `getPrecedents()` | `getDependents()` |

All four exist on `ICell`, `ICellRange` **and defined names** (`INamedFormula`), are
promises (await one before chaining on it), and
each renders as a flat range listing — one sheet named in the heading, every row
qualified once the set crosses sheets. The graph is indexed by range,
so a range is one call rather than a fan-out, and a whole column of dependents is
one entry rather than four hundred.

**What comes back is an `ICellRanges` — a collection of areas, not one range.**
Its `getCount()` is the number of *areas* (`at(i)` indexes them); the cells are
`getCounts()`, and `entries()` walks them. Nine areas is not nine precedent
cells.

**The report's tree is the shaped view, and its depth is the blast radius.** The
top level is one hop. Anything nested is where the change keeps travelling — cells
you would not have thought to check. A flat tree means the radius is exactly what
you see.

```text
## Dependents

  D2:D4  3×1  =B2*C2
    D6   6308.8  =SUM(D2:D4)
  B6     400  =SUM(B2:B4)
  D6     ↑ shown above
```

Editing this block moves `B6` and `D2:D4` directly, and `D6` — the sheet total — one
hop further on. The tree's grammar — the `↑ shown above` / `↺ circular` marks,
defined names as labeled nodes, the stated depth and node bounds — is `reading.md`
§"The dependency graph"; the one mark to act on here is `↺ circular`, a genuine
cycle. A branch cut at the depth bound is answered by `getPrecedents()` /
`getDependents()`: the whole transitive reach, flat.

Read the radius for its *shape*, not its size. A hundred dependents inside the
column you are already editing is contained. Three that cross to another sheet or
land on a reported total is a change to something you were not asked to change — say
so before you make it, or make it deliberately. And dependents on a range are the
union over its cells, **not** filtered against the range itself: a block holding
both formulas and their inputs lists both — "what does this block feed" and "what
does it feed outside itself" are different questions.

**Charts are in the radius.** Nothing recalculates a chart, so the graph does not
hold one — but a chart plots cells, and the `Dependents` heading counts the charts
that read the range or anything downstream of it (`· 2 charts`), listed by anchor on
the `charts` line. Inserting or deleting rows under a chart's data moves its series
references with the cells, exactly as it moves a formula's; deleting the data leaves
the series at `#REF!`, which the workbook report then lists under `Errors`.

**A formula you have not written yet has a radius too.** Anchor it where it would
land:

```typescript
wb.getRange('H26').createFormula('=SUM(H14:H25)')
```

That computes against the live workbook and commits nothing — the report is the
value with its references and symbols resolved *at that cell* (`wb.evaluate` for
the bare number), so what the new formula would **read** is answered before
anything is written. What it would **feed** is not a property of the formula at
all: a temporary formula is outside the graph, so the dependents of `H26` — what
reads that location today — are the downstream half, unchanged by your edit. To
see downstream *values* move, work on a copy: fork the session (or simply omit
`--save`), apply the change, and read the dependent set back — the full what-if
loop is `sessions.md` §"Experiments and what-ifs".

## Fix at the root

Make the fix as code (see `writing.md`), batch related fixes into one undo unit, and
prefer one root fix over N symptom fixes. If the root is a **missing input** rather
than a wrong formula — common with `#DIV/0!` and `#N/A` — say so instead of
"fixing" the formula to hide it. That is the user's decision, not yours.

## Verify

Loud defects verify by **recount**:

```typescript
const after = sheet.getStats().getTypedCounts().error ?? 0;
```

Report `resolved = before − after`, and list what remains with why. If `after` is
not lower than `before`, your fix did not work — do not report success.

Silent defects verify by **re-partition**: run the same partition that found the
outlier and confirm the small region has merged into the run around it. A partition
that still has the outlier means the fix did not land, whatever the write reported.

Flagged defects verify the same way at lower cost: `getRegions('anomalies')` over
the range you changed. The finding at that address is gone, and no new one has
appeared at its neighbors — a fix that turns the cell beside it into the odd one
out has moved the defect, not removed it.

## Finding nothing

**A clean result is the one to distrust.** Audit studies consistently put the share
of real-world spreadsheets containing at least one error above 90%, so on any
workbook that has been used for actual work, "I found no defects" is a less likely
explanation than "I partitioned by the wrong dimension."

Before you report clean, check the method rather than the file:

- Did you partition by the dimension the convention actually lives in? A model whose
  `Formula` partition is uniform can still have every input miscolored — and the
  sheet report's section headings already named which features are even present.
  A feature with no section cannot hold a defect; one with many regions is an
  invitation.
- Did you check more than one dimension? One clean partition is one answered
  question, not a verdict.
- Did you read the `Anomalies` section of every sheet the roster counts them on,
  not only the sheet the task pointed at? The roster line is the index.
- Did you look past the obvious sheet? Errors cluster in supporting schedules and
  the sheets nobody opens.

Then **report what you checked, not just what you found.** *"No outliers in
`Formula`, `NumberFormat` or `Font` across Operating Model and DCF"* is a
result you can support. *"The model is clean"* is a claim you cannot.

## Reading values — take the cheapest rung

Most analysis needs no values at all: the shape sweep, the partition and the
region profile answer the question before anything is materialized. When you do
need them, the three rungs an audit actually lands on, cheapest first:

1. **The region profile** (`sheet.getRegions()`) already carries per-column type,
   bounds, `Σ`, `x̄`, distinct counts, and the authored formula of a uniform
   formula column. Most "what is in this block" questions end here.
2. **`range.getStats()`** when you are reducing to an answer — aggregates, order
   statistics, in one pass with nothing materialized.
3. **Return the range** when you want to see the cells themselves — an
   A1-addressed table with the `'` text prefix intact.

`getValues()` on the same range returns a bare `Scalar[][]` — the same characters
with the addresses stripped off, so a row has to be located by counting. It is the
exit door, for when the grid is leaving. And a `for` loop over its result
producing a sum, a count, or a rewritten column is the tell that `getStats()` or
`getValuesRange()` was the call. The full contract is `reading.md`.

## The order, end to end

1. **`wb`** — a summary per sheet (content, formula, anomaly and style counts,
   twins) and the error sections. Read the counts against each other — and each
   sheet against its twin — before reading anything else.
2. **The anomalies read** — when the task names the defect kind,
   `wb.getAnomalies({ focus, limit: 25 })` over the workbook, a window at a time;
   then each sheet the roster counts them on, its `Anomalies` section top down.
   Every line is a lead with its evidence attached.
3. **The shape sweep** — formulas vs. values vs. regions, across columns and
   across sibling sheets. Anything asymmetric is a lead.
4. **`getRegions(feature)`** on the dimension the convention lives in. Holes and
   one-cell regions are the findings.
5. **The sibling diff** — same partition, twin sheet.
6. **`getCell()`** on the suspects only. This is where a lead becomes a diagnosis:
   the missing formula, the dependents that inherit the error.
7. **`getFormula()`** to quantify — its value, and the symbols and references
   behind it.
8. **The blast radius**, then the fix at the root, then the proof.

Steps 1–5 are shape and cost almost nothing. By the time you materialize a single
value you should already know which cell you are materializing and why.

## Attended workbooks

If the workbook is open in an attended session, do all of the above through the session
(`run`), and `highlight` the cells as you walk the user through what you found — see
`interactive.md`.
