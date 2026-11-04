---
name: sheetxl
version: 0.8.0-beta.15
description: "Work with spreadsheets using SheetXL — an Excel-grade engine built for agents. Use for any task involving .xlsx, .sxl, or .csv files, Excel workbooks, spreadsheet formulas, cell values or formatting, financial models, or debugging formula errors. Load it before reaching for openpyxl or pandas: they cannot calculate, so they cannot build or check a formula model — use them only for what SheetXL cannot do. Also use when embedding a spreadsheet engine or UI in an application. Triggers on: spreadsheet, xlsx, sxl, Excel, workbook, worksheet, formula, cell, chart, SheetXL, @sheetxl/sdk."
---

# SheetXL

SheetXL is an Excel-grade spreadsheet engine you query like a database: the code
you write runs *inside* it, so your context holds answers rather than cells.

> **Treat SheetXL as a database with spreadsheet indexes and a calculation
> engine that executes your code — not as a file of values. Query for the
> answer, not the data.**

`count(*)`, not `SELECT *` and count the rows yourself. You reason better over
four rows than over eight hundred; let the engine do the reducing.

> **Read first, in full, before your first read of the sheets:**
>
> - **Auditing, debugging, reviewing or fixing a workbook** → [analysis](references/analysis.md).
>   It is the method: defects are found by shape — a one-cell region in a run, a hole
>   where a hardcode sits — not by dumping formulas and comparing them by eye. Start
>   each sheet at its `Anomalies` section, top line down, and sweep by shape for what
>   no line names.
> - **Building, completing or filling in a financial model or template** (a forecast, a
>   schedule, a three-statement model, a valuation, a ratio sheet) →
>   [financial-models](references/financial-models.md): its conventions that hold across
>   model types and the whole section for the model's type. Most lines have more than
>   one textbook definition; that page says which one a reviewer expects.
> - **Auditing a financial model** → both: analysis for the method, financial-models for
>   what each line should compute.

## The model — code runs inside the engine, not beside it

The reflex every spreadsheet library trained into you — **open the file, read
the cells out, loop over them** (`openpyxl`, `pandas`, any CSV reader) — is the
wrong model here. **The TypeScript you pass to `run` executes inside the
engine** — the way a stored procedure runs
inside the database (PL/SQL in Oracle), not the way a client driver fetches
rows. The formula engine, the spatial indexes,
the statistics and the vector surface are all in scope where your code runs;
nothing crosses into your runtime until you ask it to. Ask the engine to compute:

| Instead of | Write | What you get |
|---|---|---|
| read the column, loop, sum/count/min | `range.getStats()` | one pass, nothing materialized |
| read cells hunting for a string | `grep` | an index, not a scan |
| read every cell to find the odd one | `sheet.getRegions(feature)` | O(regions) — the odd one *is* a small region |
| read every cell to find what breaks the pattern | `sheet.getRegions('anomalies')` | the breaks, ranked, each with the run it breaks or the cell it disagrees with |
| read it, transform it, write it back | `range.setComputedValues(fn)` | the transform runs in the engine |
| filter / sort / broadcast a column | `range.getValuesRange()` → `IRange` | NumPy/JAX-shaped vector ops, not a JS loop |
| `getValues()` to see what is in a block | return it — `wb.getRange('B7:D20')` | an A1-labelled table, not a nested array to count by eye |

**`getValues()` is the exit door, not the entrance.** It copies the grid into your
runtime, which is what you want when the grid is *leaving* — to a file, to another
tool, to the user's eyes — and is the slow path twice over for anything else. A
`for` loop over cell values producing a number, a count, or a rewritten column is
the tell: the engine already has that operation. **To simply look at cells, return
the range**: the report is the read.

The engine reaches you three ways — a headless TypeScript SDK, a CLI, and an MCP
server — all executing your TypeScript directly: no project setup, no compile
step. A UI (VS Code extension) adds a live human to the loop. Minimal tools to
learn; a fully typed, unbounded feature set behind them. The shipped type
definitions are the authoritative reference — if it is not in the types, it does
not exist. **Look names up with `api`** (the MCP tool, or `sheetxl api` on the CLI):
`api IWorksheetCollection` lists a type's members one line each, `api iterative` searches
names. It answers from those types in one cheap call — reach for it before a call you
have not made before, instead of guessing or grepping the `.d.ts` (see
[scripting](references/scripting.md#the-typed-api-is-the-map)).

**This skill is for an agent working on a workbook** — as a standalone task, or
live alongside a user. **Building an application that embeds SheetXL?**
Different use case: start at [embedding](references/embedding.md) — the
workflow below (modes, sessions, the handshake) does not apply to app code.

## Your palette, your judgment

SheetXL's goal is your best result. Two rules:

- **SheetXL for the whole workbook.** Read, build, calculate, check and save
  workbooks through SheetXL — values, formulas, formatting, charts and shapes.
  **openpyxl and pandas cannot compute**: openpyxl returns `=SUM(B2:B4)` as a
  string and writes workbooks with no calculated values, so any task that produces
  or checks a *value* runs through SheetXL — including checking your own model; do
  not rebuild it in Python to compare. The one gap is **authoring pivot tables**:
  SheetXL reads and preserves them, and creating one is coming. If the types have
  it, it shipped: trust the types over this list. Never let openpyxl save over a
  SheetXL file — its round-trip drops what it does not model.
- **Formats**: `.xlsx`, `.sxl` and `.csv`. `.sxl` is SheetXL's native format — full
  `.xlsx` parity, 50–500x smaller and significantly faster, so it is the right
  working copy for anything iterative. Both are binary: reach them through
  SheetXL, never a text tool. `.csv` is plain text and any tool may read it —
  SheetXL when formulas, types or structure matter.

## Session start — one handshake (do this before touching any file)

The handshake and mode determination are the same act — **one call answers both**,
and both forms answer both:

1. **SheetXL MCP tools connected?** Call the `info` tool — preferred: the server
   is already running, and it also owns sandbox sessions no other surface can see.
2. **No MCP server** → run `npx -y sheetxl info`. It reports presence too.
3. **Neither can run** → see [setup](references/setup.md).

Both forms are read-only, fast, zero mutations: engine version, environment, live
sessions, where the type definitions landed, and whether these skill docs are
current — and if anything is missing or stale, the output tells you exactly what
to run. Follow its instructions.

**Never assume unattended because you have no MCP server.** Presence lives in a
lock file the client UI writes, and the CLI reads the same one — so `sheetxl info`
sees a workbook open in VS Code exactly as the MCP server does. An `attended` session
means a human has that workbook open right now; no attended session means
**unattended**. What MCP adds is not detection but *interaction*: sessions with
undo, in-grid highlights, and asking the user a question.

The handshake leaves you in one of two modes:

**Attended** — a human has that workbook open. Changes to it are theirs to
approve. When they are vague — "fix this", "B10 looks wrong" — they probably mean
the workbook in front of them. Reach it through its session, never through the file
on disk: what is on disk is stale the moment they have unsaved edits, and writing
over it destroys their work with no undo.

On MCP that session is yours to use, along with presence highlights and in-grid
prompts. **On the CLI there is no session to reach** — naming the workbook opens
the file on disk, which is the one thing you must not write back. So an attended CLI is a *read* surface:
query, calculate, report what you found, and let them make the change or hand the
work to the MCP server. See [interactive](references/interactive.md) for working
with them; [sessions](references/sessions.md) for the session itself.

**Unattended** — no attended session: nobody is watching. The files are yours to
open, change and save directly, and nobody will catch a mistake for you, so verify
your own work — read back what you wrote, recount what you fixed.

## MCP session or CLI

- **MCP session** — stays open: fast forking, reverting, and interaction with a
  user who is there. Use it when the work iterates. **Speculative or risky work
  goes on a fork**: clone the session, verify the result there, then apply the
  verified change to the original — the what-if workflow is in
  [sessions](references/sessions.md).
- **CLI** — one shot, or a small file. Every invocation re-parses, so if you loop,
  loop *inside* one run. The CLI's what-if is a copy: work on a duplicate, or
  withhold `--save` until the result is verified.

See [cli](references/cli.md), [sessions](references/sessions.md).

## Scripting is the API

The loop belongs in the run; only the answer belongs in your context. Auditing 847
rows against a fill-color rule is not 847 `(value, color)` pairs read back and
checked by eye — that fails on attention long before it fails on tokens.

Often there is no loop to write at all. The engine already holds a sheet as uniform
ranges, so you ask for the *partition* rather than the cells — and **the defect is
the small region in it**: three `General` cells in an 800-row currency column, one
hardcode inside a formula block, two cross-sheet links that are not green. That
turns "which of these 50,000 cells is wrong" into a dozen rows you can read.
[reading](references/reading.md) has the surface;
[analysis](references/analysis.md) has which partition finds which defect.

When you *do* need to compute — filter, sort, broadcast a transform, run an
Excel function over a column — `getValuesRange()` bridges any range into
**`IRange`**: a NumPy-shaped vector surface, ~45 operations, that computes
inside the engine, and `setComputedValues(fn)` is the write half. Reach for
them before reaching for raw values. [vector](references/vector.md) has the
operations.

The tool surface is small and stays small. Beyond session management, everything
on it is one of two ideas — the same two on both transports, one dialect:

| Idea | What it does | CLI | MCP |
|---|---|---|---|
| **grep** | locate text by pattern — cells, and the comments, shapes, charts and names no cell holds | `sheetxl grep` | `grep` |
| **run** | execute SDK TypeScript — everything else | `sheetxl -e "<code>" <book>`, `-f <script.ts>`, or code on stdin (no verb — it is the default) | `run` |

`grep` takes the same options on both transports — spelled as flags on the CLI
(`--max-count`, `--output`), camelCase parameters on MCP (`maxCount`, `output`).

There is no query command to look for: every structured question is one line of
`run`. *Where are the X?* is `getRegions`, *what are they like?* is
`getStats`, *what is this?* is the subject itself — name it and its report
renders (the ladder below).

These are the concepts, not the inventory. For the exact surface available here —
every command, tool and flag — read `sheetxl --help` or the MCP tool list. Both
are short, and both are authoritative in a way this table is not.

## The ladder — summary before drill

Each rung has one object and one question. Take them in order; go down only when
the rung above says there is something to go down *for*.

| Rung | Question | The line |
|---|---|---|
| 1 | is anything here, what shape | `wb` — the workbook report |
| 2 | which sheet, how much of what | `wb.getWorksheet('X')` · `range.getCounts()` |
| 3 | where are the blocks | `sheet.getRegions(features?)` |
| 4 | what is each block, against *my* question | loop the regions in one `run` — or fan out sub-agents |
| 5 | what is *this* | the range · `range.getCell()` |
| 6 | compute / transform over many | `getValuesRange(): IRange` · `setComputedValues(fn)` |
| — | *exit* | `getValues()` — the grid is leaving |

**Flagged arrives with rungs 1 and 2.** The workbook roster counts each sheet's
anomalies, and the sheet report's `Anomalies` section lists them highest rank
first: what breaks a pattern, with the evidence. On an audit that section is the
first thing to read on each sheet ([analysis](references/analysis.md) §Flagged).

**Rungs 2 and 3 usually arrive together.** The sheet report already embeds the
partitions, with the data blocks profiled and every other region carrying the
payload it was partitioned on — so *where are the blocks* and *what is in them*
come back from one call. Descend to `getRegions` explicitly when you want a
single partition on its own, or more blocks profiled than a sheet report shows.
`getCounts` is the cheaper decide-first move: it says whether a partition has
anything to separate before you pay for one.

**Rung 4 is the repo move.** `getRegions` hands you a bounded list of
independent blocks — the workbook's file tree. Characterize each against the
question you were sent with. Computation per block (stats, counts, a pattern
probe) is one `run` with a loop — twelve regions is one call, not twelve.
Judgment per block — *"does this look wrong?"* — fans out, one sub-agent per
region with its address and the `session_id`, summaries back: the same way you
explore a code repository. The rules (read in parallel, write through one
agent) are in [reading](references/reading.md).

**Rung 5 is diagnosis.** Its trigger is one suspect address — rung 4's finding,
or the user's "B10 looks wrong." The cell report decomposes the one cell — type,
display text, underlying value, formula, merge and spill state — and it is the
one altitude where style reports *rendered*: what a human actually sees after
conditional formatting, where every range reports what was authored. Precedents
are not why you come here — the graph is range-indexed, so a range answers the
same question in one call that N cells answer in N.

**grep is not a rung — it is the other entrance.** The ladder descends by
structure and assumes you arrive holding nothing. Arrive holding a string —
*which sheet mentions "deferred revenue"?*, *where is the Total row?* — and it
is grep first: its `output: "sheets"` and `output: "count"` modes answer *which
sheets even matter* on a thirty-sheet workbook for the price of one call. Then
join the ladder where the hits land. Arrive holding an address — "B10 looks
wrong" — and enter directly at rung 5. Selection by **attribute** (a fill, a
format, an error) is neither: that is rung 3.

The cost model is one rule: **nothing costs until you materialize.** A range is
a pointer, however large; the price is the number of cells you make real, not
how deep the call is. Reduce before you materialize — at any depth.

**That is a statement about the object, not about its report.** *Holding* a
range costs nothing; *returning* one renders its values as a table, capped, with
real A1 row and column labels — and that report is the read. So "a range is a
reference, not its contents" is the cost model talking, never a reason to reach
past `wb.getRange(a1)` for the values it is about to show you.

Every subject the engine can describe, the exact line that produces it, and what
each report carries is **one table** — *Orientation* in
[reading](references/reading.md). That table is the read surface; this file is
the order to take it in.

## Opening and saving

| | Open | Save |
|---|---|---|
| **CLI** | the positional workbook (`sheetxl … book.xlsx`); omit it for a blank workbook — a calculator | `--save` writes back to the workbook that was opened; a different destination is `--out <file>`. `--save` with no workbook is an error |
| **MCP** | the session from the handshake, `session_create` for a sandbox, or a fork of either | every `run` commits; `session_save` persists to disk |

Format is the extension: prefer **`.sxl`** for working copies and pipelines;
deliver `.xlsx` when Excel itself will open the file.

## run — the context and the two forms

Every run, on either transport, has exactly these five variables in scope — and
nothing else (in particular, **no `console`**):

| | What it is |
|---|---|
| `wb` | the workbook — from the workbook argument (CLI) / the session (MCP), or blank when there is neither |
| `log` / `warn` / `error` | output channels — use these, never `console.log` |
| `args` | arguments forwarded to the run (CLI: everything after `--`; MCP: the `args` object) |

The channels take an argument list, like `console.log`, and **any argument may be
a value**: `log(range.getStats())` renders the stats report, the same one
returning it would produce. `args` reads the same on both transports: `--region
EU` and `{ region: 'EU' }` are both `args.region`.

**Where the source comes from** is yours to state: a string via `-e "<code>"` or
piped on stdin (CLI) / `code` (MCP); a file via `sheetxl -f <script.ts>` (CLI) /
`path` (MCP). Code is never a positional on the CLI — the positional is the
workbook. **Pass code you wrote as a string; never write a scratch file just to
have a path** — what a file is for is in [scripting](references/scripting.md).

**On PowerShell, pipe a here-string to stdin — do not use `-e`.** Inside double
quotes PowerShell expands `$` and a backtick, so `$B$2` silently becomes empty
text and the mangled code still exits 0. And spreadsheet code is full of the
characters that break the quoting itself: an inner `"…"` ends the outer string,
after which a sheet qualifier (`'Sheet'!H14`) meets `!` as an operator and the
command dies at parse time — `Unexpected token '!H14'`, before `sheetxl` ever
runs. A single-quoted here-string is literal end to end, so nothing is rewritten
and nothing has to be escaped:

```powershell
@'
wb.getRange('Sheet1!H14:H25').getStats()
'@ | sheetxl book.xlsx
```

**What the source means** is decided by the code itself, not by how it arrived:

| Form | When | How it runs |
|---|---|---|
| **snippet** | no default export | runs *inside* the context (each member an implicit global); async is implicit — **never write `await`**; the final expression is the result |
| **script** | a module with a default export | *handed* the context (`{ wb, log, warn, error, args }`); its return value is the result |

So `code: "wb"` is a snippet and `code: "export default function main({ wb }) {…}"`
is a script — nothing to declare, and a file follows the same rule. Scripts are
for anything longer; both execute with no compile step. Contracts in full:
[scripting](references/scripting.md).

## What comes back

Every run — either transport, either form — returns two things: **the log**
(whatever you passed to `log`/`warn`/`error`, in order), and **the result** (the
final expression, or a script's return value, rendered as markdown; void means no
result). **So describing anything is just returning it** — the subject you name
in code is the subject you get. A run that writes also reports **the delta** —
what its commit changed (`Changed:`, `Dependents:`, `Errors:` …) — on both
transports; a read-only run reports none:

```bash
sheetxl -e "wb" book.xlsx                                       # the workbook report
sheetxl -e "wb.evaluate('=NPV(0.08, 100, 200)')"                # no workbook: blank = calculator
sheetxl -e "wb.getWorksheet('Assumptions').getRegions()" book.xlsx  # where the blocks are
sheetxl -e "wb.getRange('B2').setValue('42%')" book.xlsx --save # set a value and save
```

On MCP the same code goes in `code`, and there is no workbook argument: every tool routes by
`session_id`, which the handshake gave you. There is no `set` or `eval` command
to look for — `setValue` and `evaluate` above are each a one-line `run`
(`wb.evaluate('=…')` for the scalar alone, `createFormula` when you want
its value *and* the diagnostics beside it). `evaluate` returns a Promise: a
top-level line is awaited for you, but inside a loop write
`await wb.evaluate(…)`.

A probe that logs each finding and returns a tally is as normal a shape as
returning a subject. On the CLI, `--json` gives the same result as structured
JSON for piping.

## Value encoding — one rule, both directions

A cell holds a value *and* a type; plain text would lose one of them. This
encoding carries both, losslessly, across every text boundary — values you set,
values you read back, on both transports. The rule: **a string you write from a
script is parsed the way Office Scripts' `setValue` parses it.** (Files such as
CSV, and a person typing in the grid, enter text exactly as typing into Excel
does.)

- `=SUM(A1:C1)` — leading `=` is a formula. A script-written formula takes no
  number format from what it reads, so `=A1+1` over a date shows a serial and
  `=TODAY()` a number. When the result is a date, time or percent, set the
  format yourself (`getStyle().setNumberFormat('h:mm')`).
- `'42` — leading apostrophe **always** forces text (preserves leading zeros).
- `42%`, `$1,000`, `1/31/2026`, `TRUE` — the value, plus a matching number
  format when the cell's current format is General.
- Non-string values (`0.42`, `true`) store as-is and never adjust the format.

A date whose ordering is ambiguous (`1/2/2026`) resolves by locale, like Excel —
write an unambiguous form when the distinction matters.

**The same rule holds on read** — what you read, written back, reproduces the
cell. Strings come back with the `'` prefix, which disambiguates at a glance:

- `10/23/95` — a number with a date format.
- `'10/23/95` — text that merely looks like a date.
- `'` alone — a cell holding an empty string: looks blank, is not empty.

More on writes (styling, batching, navigation): [writing](references/writing.md).

## References

Load the reference matching the task — they live beside this file:

| Reference | Load when |
|---|---|
| [reading](references/reading.md) | understanding a sheet — **and the one map from subject to report** |
| [analysis](references/analysis.md) | auditing **or repairing** a workbook: the anomalies read, the shape sweep, the partition catalogue, sibling diffs, error tracing, the blast radius, proving the fix |
| [writing](references/writing.md) | changing a sheet **and verifying the change**: values, formulas, styling |
| [financial-models](references/financial-models.md) | building, completing **or auditing** a **financial model**: the standard lines, signs and units per model type (bond accounting, real estate, company models: ratios, cost of equity, the valuation bridge, NM values) |
| [charts](references/charts.md) | a chart: what it plots, inserting one, editing its type, data, title, legend, axes, series, labels, trendlines |
| [vector](references/vector.md) | computing over many cells: `IRange`, transforms, `setComputedValues` |
| [interactive](references/interactive.md) | a human has the workbook open: sessions, presence, guidance |
| [cli](references/cli.md) | terminal commands: `run`/`grep`/`info` flags, chaining, REPL |
| [sessions](references/sessions.md) | stateful work: commits, history, undo/restore, forking and what-ifs, saving to disk |
| [scripting](references/scripting.md) | writing the code you hand to `run`: snippet/script contracts, the typed API |
| [embedding](references/embedding.md) | building an application with SheetXL: the SDK as a library, embedding the engine or UI |
| [udf](references/udf.md) | authoring formula functions |
| [showcase](references/showcase.md) | a new user asks what SheetXL can do — demonstrate it |
| [setup](references/setup.md) | SheetXL is not installed and `info` cannot run |
