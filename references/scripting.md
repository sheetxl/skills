---
name: scripting
description: "Writing SheetXL scripting code — the code you hand to `run` on either transport: runner scripts and inline snippets against the ambient SheetXL API; the typed API reference. Triggers on: script, snippet, ScriptContext, SheetXL namespace, default export, types, .d.ts, macro."
---

# Scripting — The Code You Hand to `run`

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.

**In this file:** Scripts (default export) · Inline snippets (no default
export) · The typed API is the map · When scripting cannot

One dialect, two transports: the same TypeScript executes via the CLI
(`sheetxl`) and the MCP macro tool, written against the ambient `SheetXL`
API — no imports. This is **scripting**, and it is the normal way agents write
SheetXL code. The environment is pre-provisioned: file IO is already installed,
the workbook is handed to you, and saving belongs to the invocation (`--save`
on the CLI; commits on a session) — your code never opens, saves, or constructs
a workbook. When a task exceeds what scripting can express, **drop to the SDK**
(last section).

Both run forms share **one context** — these five members and nothing else (in
particular, no `console`). The relationship is simple: a script is *handed* the
context; a snippet runs *inside* it, every member an implicit global.

| Context member | What it is |
|---|---|
| `wb` | the workbook — from the workbook argument (CLI) / the session (MCP), or blank when there is neither |
| `log` / `warn` / `error` | output channels — use these, never `console.log` |
| `args` | arguments forwarded to the run (CLI: everything after `--`; MCP: the `args` object) |

`log` is not a debug afterthought — it is the reporting channel. It takes an
argument list like `console.log` (`log('rows:', n, 'ok', true)`), and any of
those arguments may be a **value, not just a string**: `log(range.getStats())`
renders the stats exactly as returning them would. In query-shaped runs (scans,
audits, anything with per-item findings) stream the findings with `log` as you
go and let the return value carry the tally — a probe's logs *are* its output.
`warn`/`error` are diagnostics rather than findings, and travel separately
(stderr on the CLI).

`args` reads the same on both transports — `--region EU` on the CLI and
`{ region: 'EU' }` on MCP are both `args.region`, so a parameterized script
moves between them without an edit. The CLI additionally exposes the raw tail
(`args.raw`), the parsed flags (`args.kv`) and the positionals (`args._`).

## Scripts (default export)

For anything longer than a few lines. A module with a default export is a
script; it is **handed the context as its one parameter** and destructures what
it needs:

```typescript
export default function main({ wb, log }: SheetXL.IScript.ScriptContext): void {
  wb.getWorksheet().getRange('B2').setValue(99);
  log('Updated B2');
}
```

That entry is portable: the same file runs on the CLI (`sheetxl -f script.ts`)
and as an MCP macro, because both hand it the same five members. Write this form
unless you need the exception below.

**The exception — a macro that wants "the selected" something.** A macro may
take its target as the *first* parameter and the context second, which is how it
asks the runtime for the current sheet or range rather than the whole workbook:

```typescript
export default function main(sheet: SheetXL.IWorksheet, ctx: SheetXL.IScript.ScriptContext): void {
  ctx.log('running against', sheet.getName());          // the SELECTED sheet
}
```

Valid targets are `IWorkbook`, `IWorksheet`, `ICellRanges` and `ICellRange`. This
form is macro-only — the CLI runner passes one argument, so a two-parameter
entry leaves its second undefined there.

- **Name the default export** (anonymous fails to compile).
- **Annotate the parameter and the return type** (untyped fails to compile).
- The `SheetXL` API is **ambient and a runtime global** — types, enums, and
  statics all reachable (e.g. `SheetXL.ISheet.Visibility.Visible`); no
  `import` statements, ever.
- Async is fine: `async function main(...): Promise<...>`.
- **The return value is the run result** — exactly like a snippet's final
  expression, and **every result renders**: scalars via the value-encoding
  rules, ranges/stats/formulas via their own shapes (`--json` on the CLI when
  you need to pipe it). Return nothing when there is nothing to report — and
  stream per-item findings with `log` as you go.
- Where the workbook comes from and goes: the workbook argument / `--save`
  (CLI), the session and its commit (MCP). Never `WorkbookIO`, never
  `new Workbook()`.

**A script does not need to live in a file.** Pass the whole module text as a
string — `code` on MCP, `-e` or stdin on the CLI. Code you just wrote never
earns a file: writing one to have a path to point at costs a round trip and
leaves litter, and a failed compile answers with positioned diagnostics either
way, so the run is also the typecheck.

A **file** (`path` on MCP, `-f` on the CLI) is for a script with a life beyond
this turn — human-authored, checked into the repo, reused across sessions — or,
in attended mode, when the user should be able to *read* what is about to run
against their live workbook. `sheetxl compile` typechecks one without executing
it, which makes it a CI gate rather than a prerequisite.

## Inline snippets (no default export)

For one-liners and short transforms. The snippet runs inside the context — same
names, no destructuring. **Async is implicit at the top level**: each top-level
statement that yields a promise is awaited automatically, so a top-level line
never needs `await`. The final expression's (resolved) value is the result:

```typescript
wb.getWorksheet().evaluate('=SUM(A:A)')
```

Only top-level statements are awaited for you. A promise made **inside** a loop,
a callback or a larger expression is still a promise — `evaluate` returns one —
so write `await` there:

```typescript
const sheet = wb.getWorksheet();
for (let r = 2; r <= 10; r++) {
  const serial = await wb.evaluate(`=DATEVALUE(A${r})`);  // without await: a Promise
  sheet.getRange(`B${r}`).setValue(serial);
}
```

```typescript
const sheet = wb.getWorksheet();
sheet.getRange('B2').setValue('42%');
log('B2 updated');
sheet.getRange('B2').getValue()          // final expression → result
```

Statements, not lines — the same snippet is one shell-friendly `-e` argument:

```typescript
const sheet = wb.getWorksheet();sheet.getRange('B2').setValue('42%');log('B2 updated');sheet.getRange('B2').getValue()
```

End on an expression — that is the result. A snippet ending in a declaration or
loop returns nothing.

"Top-level" is literal, and the separation is a parser's: the snippet is parsed
as TypeScript, so semicolons are optional here exactly as they are in any file
and the same ASI rules apply. The implicit await reaches every top-level
statement — including each declarator of a `const a = …, b = …` — but not a
statement nested inside a loop or block. If a body needs awaiting inside control
flow, that is the signal it has outgrown a snippet — write it as a script, where
`await` is yours to place.

Need a blank workbook? Omit the workbook argument; do not construct one.

The form is decided by the code itself — a default export makes it a script;
anything else runs as a snippet. Where the source comes from (CLI file, `-e`,
stdin; MCP `code` or `path`) never changes what it means.

## The typed API is the map

The shipped type definitions are the authoritative API surface — if it is not in
the types, it does not exist. Look a name up instead of guessing it:

- **`api`** — `sheetxl api ICellRange` on the CLI, the `api` tool over MCP, `.api` in a
  session. A type lists what you can call on it, one line per member (signature and
  summary); `api ICellRange.setValues` prints that member's full documentation;
  `api merge` searches type and member names; bare `api` lists where to start. This is
  the cheap read — reach for it before writing a call you have not made before.
- **`scripting.d.ts`** — the same surface as one file, every declaration with its full
  documentation (`sheetxl info` prints the path). It is what a script is type-checked
  against, and it names types the way a script writes them: `SheetXL.ICellRange`, never a
  bare `ICellRange`. It is large and mostly documentation — grep it for a name and read
  around the hit; do not read it whole.
- **`scripting-api.json`**, beside it — the index `api` answers from, one line per
  member. For when you have file access and nothing else.
- Docs: https://www.sheetxl.com/docs

The guesses that still fail, and the call that exists:

| Guessed | Exists |
|---|---|
| `wb.getSheets().getAt(0)` · `.getItems()` | `getSheets()` returns a plain array. By index: `wb.getWorksheetAt(0)`; by name: `wb.getWorksheet('Name')`; all grids: `wb.getWorksheets().getItems()` |
| `cell.setValue(…)` · `cell.setStyle(…)` | `ICell` is a read lens. Writes go through the range: `range.setValue`, `range.getStyle()` |
| `range.setValues(42)` | `setValues` takes a 2-D array; one value is `setValue(42)` |
| `cell.getFormula().replace(…)` | `getFormula()` is an `IFormula` object; the text is `cell.getFormulaText()`, or `range.getFormulas()` for a block |

Names from other spreadsheet APIs (Office Scripts, Apps Script) often still run as
deprecated aliases, but `api` does not list them. Write the name `api` documents.

For sheet interaction patterns see `reading.md` / `writing.md`; for custom
formula functions see `udf.md`.

## When scripting cannot

Some tasks exceed the run context — e.g. operating on two workbooks at once.
The next rung is a standalone program written against `@sheetxl/sdk` with real
imports: see [embedding](embedding.md). Building the user's application is the same path.
Under `run`, none of that applies — the workbook is handed to you, and a blank
one is what you get by omitting the workbook argument; constructing your own is
basically never right.
