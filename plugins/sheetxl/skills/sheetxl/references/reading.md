---
name: reading
description: "SheetXL read surface — the one map from subject to report: return the subject and it renders. Orientation (workbook, sheet, range, cell, formula, names, stats, counts), region detection (where the data blocks, formulas, styles and merges are, each region carrying the payload it was partitioned on), the contentful profile that describes each data block without a follow-up call (header detection, per-column type, bounds, distinct counts), working many blocks with a loop or with sub-agents, crossing features so one call pairs two dimensions, the feature vocabulary and its categories (including `anomalies`, the derived partition of what breaks a pattern), range statistics, style as authored layers, and the text-prefix read contract. Load when the task is understanding a sheet: structure, data regions, stats, profiling, content summary — or whenever you need to know what a report actually contains. Triggers on: getRegions, getCounts, getStats, getValues, getValuesRange, getCell, getFormula, getNames, getAnomalies, features, regions, contentful, anomalies, header detection, column types, unique values, outlier, conditional formatting, direct style, data profiling, range statistics, understand a sheet, sheet structure, explore spreadsheet, many regions, sub-agents, fan out."
---

# Reading — Understand Before Acting

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. Code below
> is run-context code (`scripting.md`): snippet form — no `await`, and nothing to
> import (the context is already in scope).
> To change what you find, see `writing.md`; to audit or repair a workbook,
> `analysis.md`.

**In this file:** Orientation — return the subject and it renders · Structure —
`getRegions()` (crossing features, the contentful profile, many blocks: loop or
fan out) · Defined names — the symbol table · Features — the vocabulary · Style
— authored layers · Profiling — `getStats()` · Reading values — the contract
(`IRange`, `getValues()` materializes) · The dependency graph — the tree and the
flat list · Common patterns

The first step is always **understand before acting** — and the cheap paths come
first. Every read is one line of `run`: **name the subject and its report
renders.** The only read that is not a subject is `grep`.

**`grep` when you have a string; regions when you have an attribute.** That is
the whole split. Searching for `"deferred revenue"` is grep; finding the cells
that are formatted as currency, or that hold an error, is `getRegions`.

## Orientation — return the subject and it renders

**Every run result renders** — *the thing you return is the thing you get
described.* Naming the subject in code is how you ask, and it reaches subjects an
address cannot name: the third region of a partition, a formula nobody has
written yet.

This is the whole read surface. Nothing below adds a call; it explains these.

| Subject | The line | What the report carries |
|---|---|---|
| workbook | `wb` | a short summary of **every** sheet, one line per non-empty layer — `content` (extent · cells · blocks · kinds), `formula` (extent · cells · regions · distinct), `errors` / `calc failures` / `circular` when non-zero, `anomalies` (extent · cells · regions · a count per check) when any fire, `tables` (extent · cells · names), `pivots`, a line per floating type — `charts` (count · names), `shapes` (count · how many carry text), `pictures`, `slicers` — `style` (used range · distinct fonts, formats, fills, borders, alignments), `twin of` (identical formula partition). Under a `shapes` line, the text written on that sheet's shapes, quoted and located: notes the workbook's author left for its reader, so use them to understand the model, as you would a label. Then `Errors` and `Calc Failures` as sheet-qualified region listings (`Errors` includes chart series whose reference is `#REF!`), `Anomalies` — the workbook's highest-ranked findings, sheet-qualified, with the call that lists the rest — the defined-name roster and code. Each sheet's structure is its own report |
| sheet | `wb.getWorksheet('Sales')` · `wb.getSelectedWorksheet()` (the active sheet — the first unless something selected another) | used range, selection, the scalar-kind census, `Errors` and `Calc Failures`, then `Anomalies` — what breaks a pattern, one line per region, highest rank first, each with its evidence (`analysis.md` §Flagged) — then one section per feature partition: data blocks one line each (shape and header, largest first), formula regions with their formulas, style features as per-attribute histograms (region lists via `getRegions(feature)`). Closes with `Tables` and `Pivot Tables`, then the floating layer — the only content that is not in cells — one section per type: `Charts` (labeled lines under each chart's row say what it plots: `series`, `names`, `categories`), `Shapes` (the text written on a shape rides its row, and those rows lead), `Pictures`, `Slicers` |
| regions | `sheet.getRegions(features?)` · also on any range | the uniform blocks carrying the feature, each with the payload it was partitioned on; the default `contentful` partition profiles each block instead (below). The same listing the sheet report embeds |
| cell range | `wb.getRange('Sales!A1:D9')` | **the values**, as an A1-labelled table, plus the kind census, the formula blocks, spills, intersecting tables, the charts, shapes and pictures floating over it (with the text written on those shapes), the conditional-format and data-validation rules over it, and the dependency graph, whose `Dependents` count the charts that plot the range — this is how you read cells |
| cell | `range.getCell()` · `wb.getCell('WACC!H2')` | one cell, every facet: type, display text, underlying value when it differs, formula, merge and spill state, the **rendered** style, the `Formula Row`/`Column` consistency run, the `Anomalies` that fire at it, precedents and dependents |
| anomalies | `getAnomalies({ checks, limit, offset })` on a range, sheet or `wb` · `range.getRegions('anomalies')` · `cell.getAnomalies()` | the findings: regions of cells carrying the same check and evidence; `getAnomalies` lists them in reading order (the named checks first, then by rank, a window at a time, sheet-qualified on `wb`), `getRegions` as a partition; at a cell, `{ check, evidence, rank }[]`. Crosses with any feature (`['anomalies', 'formula']`), counts with `getCounts('anomalies')` |
| formula | `range.getFormula()` · `range.createFormula('=…')` (a what-if anchored at the range) · `wb.createFormula('=NPV(0.08,B2:B9)')` | **its value** first, then status, flags, resolved symbols, indexed references and tokens |
| just the value | `wb.evaluate('=…')` · `formula.evaluate()` | the computed scalar alone, no diagnostics. Returns a **Promise**: a top-level snippet line awaits it for you; inside a loop or callback write `await` |
| stats | `range.getStats()` · `sheet.getStats()` · `wb.getStats()` | value analysis in one pass — counts, aggregates, kind breakdown |
| counts | `range.getCounts(features?)` | per-feature cell occupancy — presence, not location |
| defined names | `wb.getNames()` · `sheet.getNames()` for one sheet's view · `.getByName('TaxRate')` for one | the roster: target, dependents, validity — broken first, then load-bearing. One name renders as a formula report |
| table | `sheet.getTables().getByName('Sales')` · `range.getTables()` for the ones a range touches | one table — its range, header, style and columns. The **collection** does not render; the sheet report's `Tables` section is the listing |
| conditional format | `range.getConditionalFormats()` | one rule — its anchor, evaluation order, predicate (or color scale / data bar / icon set) and the format it paints |
| sparklines | `range.getSparklineGroups()` · `cell.getSparkline()` · `range.getRegions('sparklines')` | a small chart drawn over a cell, not its value — its cell is usually **empty**. A group shares one type and set of marks; `group.getRanges()` are its cells, `group.getDataRanges()` the cells each range draws. The cell report states `Sparkline: line of A3:E3 · group F2:F4 · high, low`; the sheet report has a `Sparkline` section. That section, `Conditionals` and `DataValidations` list over the whole sheet: a layer over empty cells is outside the used range, and is still reported |
| data validation | `range.getDataValidations()` | one rule — its anchor, predicate, **the values a list admits**, whether blanks pass, and whether a bad entry is refused or merely flagged |
| protection | `sheet.getProtection()` · `wb.getProtection()` · `sheet.getProtectedRanges()` for the unlocked ranges | whether it is locked (and behind a password), and the operations it still **allows** — anything not listed is refused. Reported only where it is in force: a protected sheet is marked `(protected)` and carries a `protected` line on the workbook report, a `Protection` section on its own (the editable regions, each with why), a `Protection:` first line on a range report and an `Editable:` fact on a cell. An unprotected sheet says nothing (`writing.md` §"Protected sheets") |
| movable | `sheet.getMovables().getItems()` · `range.getMovables()` for the ones over a range · `{ type: 'chart' }` to filter | the floating layer — pictures, charts, shapes, slicers. One movable renders its name, type, preset, text, alt text, anchor and bounds — and for a chart its type, title, series and axes: the ranges it plots (`chart.getSeries()` states each as a formula), and the axis titles, which are often the only place units are written down. The listing is the sheet report's per-type sections — `Charts`, `Shapes`, `Pictures`, `Slicers` |
| style | `range.getStyle()` · `cell.getStyle()` | the style itself — authored at range altitude, rendered at cell altitude (below) |
| vector values | `range.getValuesRange()` | `IRange` — compute without materializing (`vector.md`) |
| vector write | `range.setComputedValues(fn)` | `IRange` in, `IRange` out — the transform never leaves the engine |
| raw values | `range.getValues()` | `Scalar[][]` in your runtime — the exit door |
| dependencies | `range.getPrecedents()` / `getDependents()` · `getDirect*` for one hop · also on cells and names | the transitive reach as a **flat** range listing, qualified the way the tree is — one sheet named in the heading, every row qualified once the set crosses sheets; the shaped **tree** is the `## Precedents` / `## Dependents` sections of the cell, range and name reports (below) |
| hidden rows/cols | the sheet report's `Hidden Rows` / `Hidden Columns` sections | the data a plain read would report as absent |
| the *third* uniform region | `[...sheet.getRegions()][2]` | whatever that region is — subjects need no address |

```bash
sheetxl -e "wb" book.xlsx   # the whole orientation report, one expression
```

**Start with the workbook.** Its roster is the fastest orientation signal on an
unfamiliar model — one row per sheet with used range and cell, formula and error
counts, and a totals row under them:

```text
# Workbook — Q3 Model.xlsx
Locale: en-US

## Sheets (3)
Sheet                         Used    Cells  Formulas  Errors
Assumptions                   A1:D22     88         4       0
'Operating Model' (selected)  A1:N45     82        22      12
'Cover Page'                  A1:C6      18         0       0
Total                                   188        26      12

## Errors (12)
  'Operating Model'!H26:H31  6×1  #DIV/0!  =H25/H24
  'Operating Model'!N40:N45  6×1  #REF!  =#REF!*N39
```

Read the counts against each other before reading anything else — a sheet that is
mostly formulas is a model, one that is mostly values is data; reading the count
asymmetries as an audit is `analysis.md`'s shape sweep. Names are quoted
exactly when a formula would need the quotes, so every cell of the roster is
directly the next call's argument. `Errors` and `Calc Failures` are separate
sections and neither hides the other: an error is a successfully calculated error
*value* (`#DIV/0!`), a calc failure is the engine failing to evaluate at all
(mostly circular references).

**A formula report leads with its value**, then the analysis behind it — status,
resolved symbols, indexed references, tokens. So `createFormula` answers
*what does this come to* and *why* in one call; `wb.evaluate('=…')` is the same
number with none of the diagnostics, for when the number is all you want.

For a **committed** formula the cell is still the richer lens — `getCell()` adds
display text, number format, style and the precedents/dependents graph. Take the
formula object when the *expression* is the suspect. Its text is
`getExpression()`, without the leading `=`; the cell's `getFormulaText()` has it.

## Structure — `getRegions()`

**`getRegions` answers "where are the X?"** — it returns the rectangles that
*carry* a feature, so it is how you map a sheet you have never seen: where the
data blocks are, where the formulas are, which areas are merged.

`getRegions(features?)` returns the **maximal rectangles whose cells agree on
every requested feature**. It is on `IWorksheet` and on `ICellRange` — the sheet form
is how you start, the range form is how you *constrain*:

```typescript
wb.getWorksheet('Operating Model').getRegions('formula')
```

```text
# Regions — formula (2)
  E4:E7   4×1  =C4*D4
  E9:E11  3×1  =C9*D9
```

- **Every row carries its payload** — the anchor's read for the feature that
  produced it, which is honest for the whole block because the block is uniform
  in exactly that. A formula region shows its authored formula, a fill region its
  color, an error region the error literal, a merge what the merged cell reads, a
  comment its note, a `conditionals` or `dataValidations` region the **rule**
  that covers it (several, joined by `|`, where rules overlap). `contentful` is
  the exception by design: it is profiled instead (below). An `unlock` region
  says why it is editable — `always unlocked` (the cell format) or `range Inputs`
  (an unlocked range, `· password` when it has one). One partition, `autoTable`,
  lists addresses alone.
- The default feature, `Contentful`, gives the contiguous blocks of data — found
  without knowing their locations upfront. Other features (`Formula`, `Merge`,
  styling features …) map that feature instead; the typed `ICellRange.Feature`
  enum is the authority.
- **`Contentful` is the one feature whose regions are a locator, not a class.**
  Every other feature partitions on payload identity — a `Fill` region is one
  color, a `Formula` region one authored R1C1 — so its anchor read describes the
  whole block. Occupancy has no such identity, so a contentful region is pure
  geometry. That is why it is the one partition that comes back **profiled**
  (below) rather than with a payload read off its anchor.
- **A contentful region is a maximal *solid* rectangle**, so it never contains a
  blank. This has a consequence worth expecting before it surprises you: one hole
  fragments a block. A table whose last column is sparse comes back as a header
  row, a solid body, and the stragglers — three regions, not one. The profile
  handles this (it looks *above* a block for a header that the partition left in
  a different region), but the addresses will not always match the table you
  see on screen.
- **Constraining the range clips the regions**, and the report says so. Called on
  a sheet you get whole blocks; called on a range, a block that straddles the
  boundary comes back **cut to your range** — so its address, its size and its
  profile all describe a fragment. The listing marks those
  `✂ clipped at the query edge — the block continues`, and `isClipped()` is the
  same fact in code. Take the marker seriously: `A1:D50` on a table that runs to
  row 200 will otherwise read as a fifty-row table. **Map with
  `sheet.getRegions()`; constrain only when you mean to.** (`containment:
  'fullyContained'` returns whole blocks only — and nothing at all when the block
  is bigger than your range, which is why it is not the default.)
- **Only the feature-bearing side is yielded** — rectangles carrying none of the
  requested features are elided. You get where the feature *is*, not a tour of
  where it isn't.
- **A region answers *where*.** It is deliberately **not** a range: it carries its
  address, its shape and `getFeatures()`, and it has exactly two bridges —
  `getCell()` (the anchor, whose read *is* the region's payload) and `getRange()`
  (the block, when you want to read or write it). One explicit hop, so you never
  read a region for a feature you did not partition on.
- **A count is presence; a partition is payload identity.** `getCounts(Fill)`
  counts the cells that have a fill; `getRegions(Fill)` splits them by *color*.
  `getCounts(Formula)` counts formula cells; `getRegions(Formula)` splits them by
  *authored R1C1* — so a filled block is ONE region and the value pasted over the
  middle of it is a hole. This is the rule for every feature, and it is what makes
  regions a defect detector rather than a map (`analysis.md`).
- **Regions are rectangles, not runs.** A block filled down *and* across is one
  region, not one per column: `D8:AE45` sharing `=D7*$B$4` is a single 1,116-cell
  region. Nothing here is 1-D.
- **Nothing is computed until you iterate or `getCount()`** the result. Iteration
  order is the standard iterator contract — row-major forward, with `axis` and
  `reverse` in the options — and a report lists them in exactly that order, so
  `[...regions][2]` really is the third row you see. Nothing is re-sorted; a
  small region is a finding you have to *notice*, not one the report floats.

`getCounts(features?)` is the cheap twin — per-feature cell counts straight off
the indexes. It answers "how much of each feature is here"; `getStats()` answers
"what are the values like". Use `getCounts` for *"is anything here?"* before
paying for a partition. The returned object carries **every** public feature as
an explicit key (absent ones are `0`, never missing); its *report* prints only
the non-zero ones.

### Crossing features — one call, two dimensions

Combining features **refines** the partition — a cross-product, never a
coarsening — and each region then carries one payload per feature, joined. That
is what makes a two-dimensional question a single call:

```typescript
wb.getWorksheet('Revenue Build').getRegions(['formula', 'font'])
```

```text
# Regions — formula, font (4)
  C4:C11  8×1  #FF0000
  E4:E7   4×1  =C4*D4  #0000FF
  E8      1×1  #0000FF
  E9:E11  3×1  =C9*D9  #0000FF
```

Every region pairs an authored formula with the color a human sees — and `E8`,
which has the color but no formula, is the typed-in number sitting in a formula
column. No scan, no script, no threshold. Cross only when you mean it: each added
feature multiplies the partition, so three or four at once returns confetti. Two
features and a convention is the shape that works; the catalogue of which
convention lives in which feature is `analysis.md`.

### The contentful profile — what is actually in each block

The contentful partition does not just tell you where the blocks are; each one
comes back described. The follow-up call you would have made is already in the
answer — and it arrives the same way whether you asked for the partition
(`sheet.getRegions()`) or just returned the sheet, since the sheet report embeds
this listing:

```text
# Regions — contentful (4)
  A1      1×1  'Q3 Actuals (draft)
  A3:F11  9×6  header row 3
      A  Date    8  date  1/1/2024 … 1/8/2024
      B  Region  8  text  3 unique (EU 3 · NA 3 · APAC 2)
      C  Units   8  number  0 … 70  Σ 280  x̄ 35
      D  Price   8  number  1.2  Σ 9.6  x̄ 1.2
      E  Total   8  number  0 … 84  Σ 372  x̄ 46.5  · 7 formula
      F  Ref     8  text  8 distinct, no repeats
  H3:H8   6×1  6  mixed  number 5 · text 1
  K20     1×1  'check me
```

Read it the way you read `df.info()`. Five things are load-bearing:

- **The size is the shape** — a `1×1` prints its value, a one-wide run prints its
  one series inline, and a block says where its header is. A header is **detected
  and stated**, never assumed; when the partition left it in a different region
  the note says `(above this block)`. If it says `no header detected`, the labels
  genuinely are not there. `table Sales` means an authored table, and nothing
  else does: on a sheet report a table is one block at its own extent (`part of
  table Sales` when blank cells break it up), and its columns are under `Tables`.
- **The series axis is the longer one**, and the profile says `by row` when it
  went the other way. A column line starts with its grid letter because that is
  what your next read consumes.
- **A column that is formulas all the way down reports its authored formula**,
  not N computed values. A column that is *mostly* formulas reports its values
  and appends the count — `· 7 formula` on an 8-cell column is the hardcode.
- **`mixed` is a finding, not a formatting choice.** `H3:H8 · mixed · number 5 ·
  text 1` is one text cell in a numeric column — exactly the defect the partition
  exists to surface, visible without opening anything.
- **Errors are always reported** — itemized inside a `mixed` breakdown, or as a
  trailing `· error N` note otherwise — so an error never hides inside a type
  label. Blanks cannot appear here at all: a contentful region is solid by
  construction, so a hole splits the block (above) rather than hiding inside a
  profile. Profile a range of your own choosing (the range report, below) and
  blanks do report, as a trailing `· N blank`.

Bounds are explicit. Above the scan budget a block is profiled from a leading
slice and says `profiled from the first N rows of M`; past the profile cap, the
remaining regions still list but say `not profiled`. Neither ever goes quiet, and
neither is a knob you turn — the note names the call that answers it. A sheet
report classifies up to 12 blocks, one line each (shape and header, no column
lines); `getRegions('contentful')` and the range report print the column lines, and
`getStats()` is the call when you want one block exactly.

**A profile is shape, not the cells.** It reports types, bounds, `Σ`, `x̄` and
distinct counts — plus the text that *names* a block, which is why a 1×1 region
prints its value and a table prints its header labels. All of it is bounded by
blocks × series, both capped, so no report above range altitude grows with your
data. The cells themselves are a `Values` section, and only the range report has
one — because only there did you say which cells.

### Many blocks — loop, or fan out

A sheet with dozens of contentful regions is a list of **independent** units of
work, and the right move depends on what each unit needs:

- **Computation over N regions → one `run` with a loop.** Cheap, exact, and it
  costs nothing in context: the loop belongs in the run.
- **Judgment over N regions → fan out**, one sub-agent per region, if your
  harness has them. "Does this block look wrong?" is not something a loop
  answers. Give each agent the region's **address** and the same `session_id`;
  it can call `run` and `grep` against the live workbook itself.

One rule if you fan out: **read in parallel, write through one agent.** Parallel
readers are fine. Parallel writers against one session collide on the drift guard
(`original_commit_id`) and you will spend the round trips you were trying to
save. Collect the findings, then apply the changes in a single sequence.

## Defined names — the symbol table

`=Revenue*(1+TaxRate)` is unreadable without it, and a name broken by a sheet
delete is a root cause that produces no cell error until something references
it. The workbook report carries the roster, so rung 1 already covers it; the
standalone forms are for drilling.

**Names that do not resolve lead the section as their own block** — its mere
existence is the alarm, its intro carries the count, and the `Error` column
carries the cause. A healthy book has no block at all. Below it, the roster is
sorted by **dependents descending, alphabetical on ties**: load-bearing
symbols, then trivia. Unreferenced junk (`_FilterDatabase` and friends) sinks
to the bottom on its own, so nothing has to be filtered out.

```text
## Names (6)
2 do not resolve:
Name           Refers To      Error               Dependents
BadName        =#REF!*12      #REF!               14 · Sheet1!G1:G14
Missing_Name   =NoSuchName*2  #NAME?: NoSuchName  1 · Sheet1!G2

Name           Refers To            Dependents                          Hidden  Comment
TaxRate        =0.21                412 · EffectiveTax, Sheet1!D2:D13, +7 more          2026 statutory rate
Revenue        =Sheet1!$B$2:$B$100  38 · Sheet1!C2:C39
Growth         =RAND()*2 (volatile) 3 · TaxBase, Sheet1!C5, Sheet1!E9
OldRate        =0.18                unused                              hidden
'EU Ops'!Rate  =0.19                unused
```

Reading the row:

- **Refers To is the expression**, so it says at a glance which names are
  navigable (`=Sheet1!$B$2:$B$100`) and which hold a value (`=0.21`). There is
  deliberately no `Value` column: a name with relative references evaluates
  *per calling cell*, so a single value is not well defined — ask one name for
  its value with `evaluate()` below.
- **`Error`** (broken block only) is the cause: `#REF!` for a delete, or
  `#NAME?:` naming the identifiers that no longer exist.
- **`(volatile)`** means the resolved range depends on when it was read, so it
  is not a stable navigation target.
- **Scope is spelled as a qualifier** (`'EU Ops'!Rate`), never as a column.
  That is exactly what a formula writes, so the report's answer is directly the
  next call's argument. No qualifier means global.
- **`Hidden` and `Comment` appear only when some name has one.**

**Dependents is SYMBOL-level, and counts names as well as cells.**
`=SUM(Revenue)` depends on the name; `=SUM(Sheet1!$B$2:$B$100)` does not, even
where that is exactly what `Revenue` designates — only the first goes `#NAME?`
if the name is deleted. Another *name* reading it is a dependent too, and the
heaviest kind: with `Doubled = Base*2`, no cell mentions `Base` and every cell
reading `Doubled` still breaks with it.

**The count leads every populated cell** — `412 · EffectiveTax, Sheet1!D2:D13,
+7 more`. The number is cells plus names, the same weight the sort uses, so the
column compares down its whole length. Behind it, names first, then addresses —
the next call's argument — spelled whole while they fit, with `+N more` for the
rest. `unused` means nothing references it in either currency, and it is the
only safe-to-delete answer; `unused · shadowed` warns that a sheet-scoped twin
wins the binding, which is often exactly why nothing reaches the global one.
`getByName('Revenue')` renders that one name in full — a formula report with an
identity header and its whole dependent tree.

### The three questions a name answers

Each has one call. They are easy to reach for interchangeably and they return
different things, so the wrong one silently answers a question you did not ask.

```typescript
const named = wb.getNames().getByName('Revenue');
named.getExpression()     // 'Sheet1!$B$2:$B$100' — the refers-to TEXT, no leading '='
named.getRanges()         // the cells it designates — null for a name holding an expression
await named.evaluate()    // what it is worth NOW — a Scalar, or an IRange for a reference name
```

(`getValue()` is the same text — it is `INamed`'s generic accessor and a defined name
answers it with its expression.)

**`evaluate()` is the one to reach for when the question is "what is this
worth".** Do not read the text and re-query it: `wb.getRange(String(n.getValue()))`
is a parse of a formula expression that only *sometimes* happens to be an
address, which is why it needs a `try`/`catch` — and it is wrong for the names
that matter (`=0.21`, `=SUM(...)`, anything with a function). `evaluate()` is
async and settles the engine itself.

A name written with **relative** references evaluates per calling cell, so
`evaluate()` gives one anchored answer and each dependent cell sees its own.
That is the same reason the roster carries no `Value` column; for those names
the honest read is the dependent cell, not the name.

There is no `getRange()` (singular) and no `getReferenceText()`. Guessing one
throws — `n.getRange is not a function` — so a run that comes back with empty
columns instead is a `?.()` in your own loop swallowing the mistake. Do not
write defensive optional calls against this API; let it throw and read the
name.

## Features — the vocabulary

Same flat list everywhere a `features` argument appears. **In code, features are
the exact camelCase values** (`'formula'`, `'numberFormat'`) — the typed
`ICellRange.Feature` constants — while the report-option spelling
(`--md.features=…` on the CLI, `md: { features: "…" }` on MCP) is
case-insensitive. There is no group or wildcard grammar: every call names exactly
the features it means, which is a property, not a limitation — an audit that
asserts uniformity should say over which dimensions, and its meaning should not
drift when the engine grows a new feature.

Three categories to orient by (categories, not arguments — asking for several
features at once is the cross-product, see above):

| Category | Members | The question they answer |
|---|---|---|
| appearance | `fill` `borders` `numberFormat` `dateFormat` `alignment` `font` · `conditionals` · `table` `autoTable` | how does this look? |
| calculation | `formula` `spills` `calcError` `calcStatus` | is the calculation healthy? |
| scalar kind | `error` | what kind of data is this? |
| audit | `anomalies` | what breaks the pattern around it? — derived from the other partitions, never authored; a region is a run of cells with the same finding and evidence |

Outside the categories: **`contentful`** (the default — contiguous blocks of data),
`merge`, `dataValidations`, `hyperlink`, `comments`, `sparkline`, and `unlock` — the cells that
still take a write on a protected sheet. `unlock` has no section on a sheet report:
it is the body of the `Protection` section, which prints only when the sheet is
protected.

Features also narrow a *report* rather than a query. A workbook or sheet report
over a heavily formatted sheet is dozens of region sections; passing `features`
as a **report option** — `--md.features=fill,borders` on the CLI, `md: { features:
"fill,borders" }` on MCP's `run` — prints only those sections without changing what
was asked for. Reach for it when the answer is in the report but buried; reach for
`getRegions` when the *question* is about specific features.

`features` and `mode` are the whole report surface — **it selects, it does not
size.** Nothing there makes a report longer, because a truncated report is a signal
to ask a narrower question, not to re-run the same one with a bigger budget. Every
truncation note names the query that returns the rest.

- **`string`/`number`/`boolean` are not features.** The vocabulary is for things
  with a spatial identity — a fill, a formula, an occupied cell — and a scalar
  kind has none: it is read straight off the value, which is the single pass
  `range.getStats().getTypedCounts()` already makes, richer (aggregates and
  frequencies come with it, at no extra cost). `error` is a feature — an error
  value has spatial identity of its own — and it partitions like any other,
  grouped by error value. `dateFormat` is a formatting predicate (the same tier
  as `numberFormat`), not a value kind: it answers "does this cell carry a
  date-category format", which includes a date-formatted blank or text cell. For
  "how many cells are genuinely dates" (a numeric value under that format), read
  values — `getInteropType()` per cell, or the report's own `date` line in its
  `Kind` section, which already does this. That census is disjoint — each cell
  under one kind (`number`, `date`, `text`, `formula`, …), summing to the heading —
  so `formula` against the rest is how much of the sheet is model and how much is
  input.
- Appearance spans all three authored layers on purpose, because "is the
  formatting consistent" cannot know in advance which layer the convention lives
  in — but asking about the layers together never composes them; see *Style*
  below. `table` is in the appearance category as a source of it, and
  `--md.features=table` on its own answers a different, structural question — what
  tables exist, named, with headers.

## Style — authored layers, not resolved effect

Style comes back as **the layers a human authored**, each as its own set of uniform
ranges, never composed into a per-cell answer:

| Layer | What it is |
|---|---|
| direct style | what someone set on the cells — the `Fill`, `Border`, `NumberFormat`, `Alignment`, `Font` features |
| conditional formatting | the ranges the rules apply to, each row carrying the rule itself — the `Conditionals` feature |
| tables | the table, its style, banding and header — `Table` / `AutoTable` |

Colors arrive resolved: `#000000 (theme 1)`, never a bare theme index or tint.

Keeping the layers apart is what makes style readable at range altitude. A banded
table is *one* authored fact — `style Medium9 · banded rows` — and an unbounded
number of resolved ranges; a color scale is one rule and one distinct range per
cell. Composing would also erase the distinction a convention audit depends on: a
cell colored deliberately means something, a cell colored by an inherited band
means nothing, and resolved style makes them identical.

When you do want *what a human sees at one cell, and which layer won*, that is a
cell facet: `getCell().getStyle()`. One cell, on demand — never a column across a
range. (`getStyle(true)` is the *direct* read on a cell, which is what a
convention audit compares.)

**Range-level `getStyle()` reads the anchor** — exactly what the Excel ribbon
shows when a range is selected (writes apply to the whole range; `writing.md`).
The ribbon is safe because a human *sees* the selection disagree; an API return
carries no such signal — so never treat the anchor's read as evidence of
uniformity. The uniformity test is `getRegions(<style features>)` returning a
single region.

## Profiling — `getStats()`

`ICellRange.getStats()` computes an `IRangeStats` in a single pass. Prefer it
over hand-rolled loops for aggregation, profiling, and validation.

It exists at every tier: `wb.getStats()` (every sheet's used cells, hidden
included), `sheet.getStats()` (the used range, null-safe — an empty sheet is
empty statistics, not a `null` branch), and the range form below. All take
iterator options — at workbook scale pass `{ onError: 'ignore' }`, since one
error value anywhere makes the default `'throw'` policy fault the aggregate.

```typescript
const stats = wb.getWorksheet().getRange('A1:A100').getStats();

// Size and counts
stats.getSize()          // total cells, including blanks
stats.getCount()         // non-empty cells

// Type breakdown — keys are ScalarType values:
// 'number' | 'string' | 'boolean' | 'error' | 'richData' | 'null'
const counts = stats.getTypedCounts();
// counts.number ?? 0 → numeric cells; counts.error → error cells;
// counts.null → blanks

// Numeric aggregates (null when no numbers)
const agg = stats.getAggregates();
// agg.sum, agg.min, agg.max, agg.average, agg.stdDev — and more in the types

// Order statistics (LARGE, SMALL, PERCENTILE, RANK, MEDIAN)
const sorted = stats.getSortedNumbers();
sorted.getLarge(1);          // maximum
sorted.getSmall(1);          // minimum
sorted.getPercentile(0.5);   // median
sorted.getRankOf(value);     // 1-based rank
```

**Distinct values are also available directly off `getStats()`.**
`getFrequencies()` gives `getUniqueCount()` and `getValueCounts()` (top-N,
sorted by frequency or value) over the whole population — the contentful
profile's `3 unique (EU 3 · NA 3 · APAC 2)` and `8 distinct, no repeats` lines
are exactly this, already computed for you. Reach for `getFrequencies()`
directly when you want the full breakdown rather than the profile's capped
top-N, or need it restricted to one kind (`{ filterFn }`). `getRanked()`
answers top/bottom-N (LARGE/SMALL) without a full sort.

## Reading values — the contract

**A range is a reference, not its contents** — `getRange('B2:D10')` names a section
of a sheet and holds no values, which is why naming one is free at any size.

**Its report is another matter: returning the range prints the values.** That is
the read, and it is the better one — an A1-labelled table where `getValues()` is a
bare nested array you have to count rows against by eye. Reach past it only when
the values must *leave*, or when you are computing over them; *how* you ask decides
whether the work happens inside the engine or inside your loop.

```text
# Cell Range — 'Cover Page'!C12:C18, 7×1

## Kind (3 cells · 1 kind)
  text      3

## Profile
3  text  3 distinct, no repeats  · 4 blank

## Values (C12:C15, 4×1 · 3 cells)
    C
12  'Profit and Loss Statement
14  'Table of Contents
15  'Profit and Loss (Monthly)
```

The table is capped (1000 cells) and elides rows and columns that came back
empty, which is why its labels are real A1 labels rather than `1..n` — with row
`13` gone, an index would assert an adjacency that is not there. The heading
states the shrunk bounds when they differ from what you asked for. Past the cap
the answer is a narrower address or `getStats()`, not a bigger table.

Below the values, the same report carries the formula blocks (one row per
authored formula, addressed as a range — not one row per cell), any spills, the
tables and names that intersect, and the precedents and dependents trees.

### `IRange` — the vector surface

`getValuesRange()` bridges a cell range into **`IRange`**: an immutable,
two-dimensional, mixed-type sparse array with functional vector semantics, shaped
for NumPy/JAX familiarity. Transforms are zero-copy and reads are SIMD-optimized;
computation happens inside the engine rather than in your runtime.

**The operations are in [vector](vector.md)**, along with `setComputedValues`, the
write half. Load it whenever you are about to transform, filter, sort or broadcast
over more than a couple of cells. The rule for choosing: `getStats()` when you are
**reducing** to an answer, `IRange` when you are **producing** new values,
`getValues()` only when the grid is leaving.

## The dependency graph — the tree and the flat list

The calculation graph renders in two shapes, and they answer different questions.

**The tree** is a section of the cell, range and name reports — `## Precedents`
and `## Dependents` — walked one hop at a time from the subject, so the *shape*
of the chain survives: the top level is one hop, anything nested is where the
influence keeps travelling.

```text
## Dependents

  D2:D4  3×1  =B2*C2
    D6   6308.8  =SUM(D2:D4)
  B6     400  =SUM(B2:B4)
  D6     ↑ shown above
```

- **Each node carries its payload at the right altitude** — a single cell shows
  its value and formula, a multi-cell range its dimensions and the formula
  shared across it. A range node is never expanded into its cells, and addresses
  qualify with their sheet only when the tree spans sheets.
- **A defined name appears as a labeled node** — `EffectiveTax  name`, or
  `Revenue  name → Sheet1!$B$2:$B$13` when it designates a range — and the tree
  continues through it to whatever reads or feeds it. `=SUM(Revenue)` therefore
  shows BOTH the `Revenue` node (the symbol that breaks if the name is deleted)
  and the resolved cells (the data the value came from); they are different
  facts, not a duplicate.
- **Two revisit marks, and they are opposite findings.** `↑ shown above` is
  re-convergence — a node reached by more than one path, ordinary in any model:
  a total fed both directly and through an intermediate. `↺ circular` is a
  genuine cycle — deliberate in an iterative calculation, a defect otherwise.
- **Bounded three ways, each stated rather than silent** — depth (3 hops by
  default), node count, and revisits. A cut at nodes means the graph is *wide*,
  and the answer is to stand on one node and walk from there; a cut at depth is
  what the flat list answers outright.

**The flat list** is what the graph calls themselves render. `getPrecedents()` /
`getDependents()` resolve to the whole transitive reach as ranges (an `ICellRanges`,
whose `getCount()` is the number of areas — cells are `getCounts()`), and
`getDirectPrecedents()` / `getDirectDependents()` one hop — on `ICell`,
`ICellRange` and defined names alike. All four are promises - the engine answers
them - so a script awaits one before reading it (`(await r.getPrecedents())?.getCount()`);
a snippet awaits its final value for you. Returning one prints a `# Ranges` listing
with the same per-row payloads as the tree and no hop structure, which is the
point: it is the countable, diffable form, and the answer to a tree branch cut
at the depth bound. It qualifies the way the tree does — a set that stays on one
sheet names it once in the heading and leaves its rows bare, one that crosses
qualifies every row instead. The same graph the tree above walked, flat:

```text
# Ranges — Sales (3)
  D2:D4  3×1  =B2*C2
  B6     400  =SUM(B2:B4)
  D6     6308.8  =SUM(D2:D4)
```

The graph is indexed by range, so either shape reduces a column of four hundred
dependent cells to one entry. What a defect *feeds* and what a fix *moves* is
`analysis.md` §"Before you edit — the blast radius"; this section is only how
the answer prints.

### `getValues()` materializes — use it deliberately

`getValues()` copies the range into JavaScript. It is for values you are going to
*compute with* in your runtime — hand to another tool, write to a file — not
values you want to look at. **A loop over `getValues()` is the slow path twice
over**: it copies, and it moves the work out of the engine and into your code.

```typescript
range.getValues()   // Scalar[][] — a real array in your runtime, maxSize-capped
range.getValue()    // one Scalar — the top-left cell
range.getCell()     // ICell — per-cell detail, top-left by default
```

Display text is a **cell** lens, not a range one: `getCell().getText()` for the
one cell you are diagnosing — there is no `getText()` on a range. Reaching for it
in a loop is the tell that the range report was the call.

One boundary to keep straight. `getValues()` hands back **typed** scalars, where
the JavaScript type carries the type: `"123"` is a string, `45292` a date serial.
The `'` prefix belongs to the *text* encoding — reports, grep, anything crossing a
text boundary — where type would otherwise be lost, and it is what makes
`'10/23/95` (text) distinguishable from `10/23/95` (a formatted number). The rule
and both directions are the **Value encoding** section of the root `SKILL.md`;
per-cell writing is `writing.md`.

## Common patterns

### Orient to an unfamiliar sheet

```typescript
wb.getSelectedWorksheet()     // kind census, every feature partition, and the data blocks classified
```

One call. Reach for `getRegions()` on its own when you want a *single* partition
without the rest of the report around it, or the column profiles of its blocks.

When the profile is not the question you were sent with, the loop belongs in the
run:

```typescript
const report = [];
for (const region of wb.getWorksheet().getRegions()) {
  const stats = region.getRange().getStats();     // getRange() = "now I want to read"
  report.push(`${region.getAddress()}: ${stats.getCount()} values`);
}
report.join('\n')
```

### Profile a column for data quality

```typescript
const counts = wb.getWorksheet().getRange('B2:B100').getStats().getTypedCounts();
// (counts.null ?? 0) > 0  → blanks present
// (counts.error ?? 0) > 0 → errors present
// (counts.string ?? 0) > 0 in a numeric column → suspect cells
```

Bound the range. `A:A` is a million cells, and its `null` count is the million
empty ones — a true answer to a question nobody asked.

### Check a range is purely numeric

```typescript
const counts = wb.getWorksheet().getRange('B2:B100').getStats().getTypedCounts();
!(counts.string || counts.boolean || counts.error)
```
