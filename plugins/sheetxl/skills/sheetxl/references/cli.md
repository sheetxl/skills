---
name: cli
description: "SheetXL terminal commands: executing SDK code against a workbook file, grep, info, setup, and the interactive session. Flags (-e, -c, -f, -i, --out, --save, --json, --md.features), chaining scripts, shell composition, CI. Triggers on: npx sheetxl, sheetxl grep, sheetxl info, -e, -c, -f, --out, --save, --json, REPL, terminal, command line, automate."
---

# CLI — Terminal Commands

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.
> How to *write* the code you run is `scripting.md`; this is how to invoke it.

**In this file:** The grammar · compile — typecheck without running · Large
workbooks — parsing repeats · Structured questions are one-liners · grep ·
Interactive sessions · Composition

| Command | Purpose |
|---|---|
| `sheetxl [options] [workbook]` | The main event: execute SDK TypeScript against a workbook — including every structured question. Return `getRegions()`, `getStats()`, a range, and the report renders. With no code, an interactive session. |
| `sheetxl info` | Read-only environment report: versions, skills, MCP status. Once per session. |
| `sheetxl setup` | Converge the environment (skills, engine, MCP registration) — consent at each step. |
| `sheetxl grep <pattern>` | Find cells by content or pattern. |
| `sheetxl compile <files…>` | Typecheck (and optionally emit) without executing. |

Zero-install: prefix any command with `npx -y` (`npx -y sheetxl info`).

## The grammar

```
sheetxl [options] [workbook]
```

There is **no `run` verb** — the operation is the default, the way `duckdb -c
"<sql>" my.db` needs no verb. One rule decides the mode:

> Give it code and it runs, prints, and exits. Give it none and you get an
> interactive session.

```
sheetxl -e "<code>" [workbook]      # inline snippet
sheetxl -f <script.ts> [workbook]   # script file; repeat -f to chain on the same workbook
sheetxl [workbook] < script.ts      # code from stdin
sheetxl [workbook]                  # no code → interactive session
```

The positional argument is the **workbook**, the same slot it fills in `grep`.
Code is never positional: it arrives through `-e`, `-f`, or stdin. Omit the
workbook and you get a blank one.

Prefer stdin (e.g. with a heredoc) over `-e` when the code is multi-line or
shell quoting gets awkward. **On PowerShell, make it the default** — a
single-quoted here-string piped in is literal, where `-e "…"` expands `$` and
backticks and dies on `!` the moment an inner quote breaks the outer string
(root `SKILL.md` has the form).

Flags:

```
-e, -c <code>         inline code, snippet or script; the result is rendered.
                      Repeatable (-c is an alias, for duckdb/python muscle memory)
-f <script>           script file; repeatable, chained on the same workbook —
                      a chain renders the last one's result
-i, --interactive     stay in an interactive session after the code runs
--out <file>          write the workbook here; `-` writes it to stdout, for piping
--save                write back to the workbook that was opened; naming a different
                      destination is `--out <file>`. With no input workbook, an error
--json                the result as structured JSON, for piping (default is rendered text)
--no-delta            omit the run delta: what a write changed, printed after the result
                      (`Changed:`, `Dependents:`, …). A read-only run prints none, and
                      neither does `--json`; a chain labels each target that wrote
--in.<key>=<value>    format options for input  (e.g. --in.csv.delimiter=";")
--out.<key>=<value>   format options for output (e.g. --out.format=csv)
--md.<key>=<value>    report options (--md.features=fill,borders, --md.mode=value)
-- <args…>            forwarded to the run (in scope as `args`)
```

`-v` prints the CLI, engine and node versions on one line; `sheetxl info` is the full handshake.
`-h` is a short help, `--help` the complete one.

Anything else beginning with `-` is an error, suggesting the flag you probably meant. A script's
own arguments go after `--` — there is no implicit passthrough.

`--md.<key>` configures the *report*, not the query — the expression already says what was asked
for — which is why it is grouped under `--md` the way input and output options are grouped under
`--in.*` and `--out.*`. `--md.features` is what makes a formatting audit readable on a heavily styled
sheet:

```bash
sheetxl -e "wb.getWorksheet()" book.xlsx --md.features=fill,borders
sheetxl -e "wb.getWorksheet('Model')" book.xlsx --md.features=formula,error   # the audit view of one sheet
```

The other key is `mode` — `display` (number format applied) or `value` (full precision,
formatting stripped). An unknown key or feature is an error listing the whole vocabulary, checked
before the workbook is read. `--json` does not read these and says so rather than ignoring them.

**Those two are the whole surface: it selects, it does not size.** There is no option for a longer
report, because a longer report is never the answer — every listing that truncates names the query
that returns the rest, and the query is both cheaper and exact:

| The report cut | Ask this instead |
|---|---|
| `…truncated at N cells` | `--json`, or `getValues()`, or `--out data.csv` |
| `…truncated at N of M regions` | `getRegions()` and loop them in the run |
| `…more regions listed but not profiled` | the sheet report, or `getStats()` on the one you want |
| `…truncated at N of M tokens` | `getTokens()` and search it |
| `…N branches not shown` | `getPrecedents()` / `getDependents()` — the whole reach, flat (the heading already counts it) |

Same for wanting *less*: `--md.features` names the sections you want, which beats capping a count
you would then have to reason about. A workbook report without the region firehose is a narrower
question too — `wb.getWorksheets()`, `wb.getNames()`, `wb.getStats()` each answer one part of it
outright.

On MCP the same options are the `run` tool's `md` argument (`{ features: "fill,borders" }`).

`-- --region EU report.csv` reaches the run as `args.region === 'EU'` (named flags are also under
`args.kv`) and `args._ === ['report.csv']`, with the untouched tail in `args.raw`. Reading
`args.region` is the form that ports unchanged to MCP, where `args` is the caller's object.

The root `SKILL.md` shows the basic forms; these are the ones it doesn't:

```bash
sheetxl book.xlsx --out book.sxl                                    # convert format (no code)
sheetxl -f normalize.ts -f validate.ts raw.xlsx --out out.xlsx      # pipeline
sheetxl -e "wb.getWorksheet().getRange('A1:C9').getValues()" \
  data.csv --in.csv.delimiter=";"                                   # format options
sheetxl book.xlsx --save <<'EOF'                                    # stdin: multi-line code
const sheet = wb.getWorksheet();
sheet.getRange('B2').setValue('42%');
sheet.getRange('B3').setValue('=SUM(B1:B2)');
EOF
```

Formats (inferred from extension): `.xlsx`, `.sxl` (native — smaller, faster),
`.csv` (configurable delimiter), `.json`.

## compile — typecheck without running

```
sheetxl compile <files…> [--out <path>] [--json] [--quiet]
```

Compiles and typechecks without executing anything. Diagnostics are positioned,
so they point at the source line; the exit code is non-zero when there are
errors, which makes it a usable CI gate.

```bash
sheetxl compile audit.ts                          # typecheck one script
sheetxl compile src/*.ts                          # or a set
sheetxl compile audit.ts --out dist/audit.sxlm    # also emit the compiled module
```

Reach for `compile` when there is no workbook in play — a CI gate, or a check on
a script a human will run later. **You do not need it before executing code**: a
run that fails to compile returns the same positioned diagnostics, so the run
*is* the typecheck. Likewise `-f` is not a way to run code you just wrote — that
goes through `-e` or stdin; what a file is for is `scripting.md`.

## Large workbooks — the CLI is stateless, so parsing repeats

Every invocation parses the workbook from scratch. A 50 MB workbook that opens in a
few seconds costs those seconds *again* on the next call, so a ten-call
explore-then-edit loop pays them ten times. Three fixes:

**Move the iteration inside one run.** A loop that would be N invocations is one
script and one parse. This is where `log` earns its keep — report per item as you
go, return the summary:

```bash
sheetxl -f audit.ts big.xlsx     # log() per finding, return the tally
```

**Convert once, then work on `.sxl`.** The native format is 50–500x smaller and
significantly faster to load, lossless both ways — so pay the xlsx parse a single
time and export at the end:

```bash
sheetxl big.xlsx --out scratch.sxl                                # pay xlsx once
sheetxl -e "wb.getWorksheet('Assumptions').getRegions()" scratch.sxl  # …fast from here
sheetxl -f fix.ts scratch.sxl --out out.xlsx                      # deliver xlsx
```

**Or use an MCP session**, where the engine stays warm across calls and forking,
undo and restore are in-memory operations rather than file rewrites
(`sessions.md`). For a large workbook and a long edit–verify loop, this is the
right answer.

## Structured questions are one-liners

There is no query command, because every result renders: name the subject and its
report comes back. Which subject answers which question is `reading.md`; the
order to take them in is the root `SKILL.md`. On a terminal that looks like:

```bash
sheetxl -e "wb" book.xlsx                                             # the workbook report
sheetxl -e "wb.getWorksheet('Sheet1').getRegions('formula')" book.xlsx    # the formula blocks
sheetxl -e "wb.getWorksheet('Sheet1').getRegions('anomalies')" book.xlsx  # what breaks a pattern, ranked
sheetxl -e "wb.getAnomalies({ focus: 'unitMismatch', limit: 25 })" book.xlsx  # ordered for one kind of defect, every sheet
sheetxl -e "wb.getRange('B2:B847').getStats()" book.xlsx              # aggregates + typed counts
sheetxl -e "wb.getCell('WACC!H2')" book.xlsx                          # one cell, every facet
sheetxl -e "wb.createFormula('=NPV(0.08,B2:B9)')"            # a formula, unwritten
```

A script's own plain objects render too — `sheets.map(s => ({ name: …, rows: … }))`
comes back as a table — so a probe needs no formatting code. `--json` emits the
same result as structured JSON, for piping to `jq` and friends.

## grep

```
sheetxl grep <pattern> [file]
```

The pattern is a **regex by default** (JS syntax); `-F` makes it literal.
Excel-style `?`/`*` wildcards are not supported. Matches are per-cell, in
reading order, each tagged with the layer it matched; truncation is always
reported.

Text that is in no cell is searched too, by default: comments ride their cell;
a sheet's shapes and charts follow its cells, addressed by the range they are
anchored over; defined names follow every sheet, addressed by the name.

```
grep "Fund Size" · workbook · fields display,formula,comment,shape,chart,name · 1 cell · 2 objects in 1 of 3 sheets
  Summary!B4      display  Fund Size
  Summary!F2:H6   shape    Fund Size is committed capital, not NAV.
  Summary!J4:P20  chart    Fund Size by Vintage
```

```
--fields <list>        layers to search: display (as shown, format applied),
                       value (the stored scalar, what getValue() returns — 0.4
                       behind 40%, dates as serials; suppressed per cell when
                       identical to display),
                       formula (leading = restored), hyperlink (the target),
                       numberFormat (the format code, e.g. "0.00%"),
                       comment (comments, notes, replies), shape (text on a shape,
                       alt text), chart (title, axis titles, series names written
                       into the chart, alt text), name (defined names and tables:
                       the name, what it refers to, its comment)
                       [default: display,formula,comment,shape,chart,name]
-F, --fixed-strings    literal string, not a regex
-i, --ignore-case      case-insensitive (case-sensitive by default, like rg)
-w, --word             match on word boundaries (text edges count as edges)
-v, --invert           cells and objects where no requested field matched
--scope <scope>        a sheet name (globs allowed: "Q*"), Sheet1!A1:D9, or a bare
                       range (applied to every sheet) [default: every used range].
                       A range narrows shapes and charts to those anchored over it;
                       a name is reached by the sheet it is scoped to or the cells
                       it designates
--hidden               search hidden rows/columns/sheets, shapes and names (excluded
                       by default; naming a hidden sheet in --scope always searches it)
-C, --context [mode]   annotate + group matches by their enclosing block:
                       contentful (bare -C) | table — not a ±N line count
-m, --max-count <n>    stop after n matches, cells and objects together
                       [default: 200; 0 = unlimited]
--output <mode>        content | addresses | count (per sheet) | sheets
                       [default: content]
```

Error tokens are display text, so they grep directly: `sheetxl grep "#REF!" book.xlsx`.
"Which cells are percentages" is a format question, not a text question:
`sheetxl grep "%" --fields numberFormat book.xlsx`.

## Interactive sessions

No code means an interactive session — there is no `repl` verb either. `sheetxl`
opens one on a blank workbook; `sheetxl <file.xlsx>` preloads the file into `wb`.
Use `-i` to run code first and stay (`sheetxl -i -f fixtures.ts book.xlsx`).

Useful dot commands: `.help`, `.api` (the API help screen), `.md [expr]`
(render a result as markdown), `.info`, `.skills`, `.write <file>` (persist —
workbook mode only), `.exit`.

Every read command is also a dot command, generated from the same descriptors —
`.grep <pattern> [--flag …]`, with the session's workbook implied. Each is a
context **function** too (`grep('shipped', { ignoreCase: true })`), which returns
the data rather than the rendered text, so it composes with everything else at
the prompt: `(await grep('TODO')).cellCount`.

Two session-only globals: `read('<file-or-url>')` loads a workbook (paths resolve
against the shell directory) and `write(wb [, target])` persists one — bare
promises are auto-awaited, so neither needs `await`. They exist only in the
interactive session; a one-shot run receives `wb` and persists via `--save`.

## Composition

Everything is a normal process: exit codes, stdout, stdin. Chain with shell
pipes, run in CI, script with `npx -y sheetxl …` for zero-install environments.
