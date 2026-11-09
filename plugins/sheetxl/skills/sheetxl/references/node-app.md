---
name: node-app
description: "A Node program or service on the SheetXL SDK, and the engine-side setup every app shares: packages, entry setup (IOPlugin), reading and writing files in code, constructing a workbook; a command-line tool, an HTTP service that serves a workbook model, a scheduled job as a runner script (`sheetxl -f`, @sheetxl/runner); scaffolding with create-sheetxl. Use when writing the user's application code, or a standalone program outside `run`. Triggers on: Node, service, API, HTTP, express, Fastify, cron, job, pipeline, CLI tool, runner, -f, @sheetxl/runner, headless, server, @sheetxl/sdk, @sheetxl/io, import, WorkbookIO, IOPlugin, new Workbook, library."
---

# Node App — Programs and Services on the SDK

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. A browser
> UI around the engine is [react-app](react-app.md); it imports the same packages
> and reads this file's §Engine for anything engine-side. Running one-off code
> from a shell is [cli](cli.md), not this.

This is ordinary Node development in the user's repo: your own program, your own
`console`, explicit `await`. No agent modes, no sessions, no `run` context. It is
also the path for a **standalone program** when the run context cannot express a
task (operating on two workbooks at once, say): same imports, your own small
program instead of `run`.

## Engine: packages, entry, files

| Package | Purpose |
|---|---|
| `@sheetxl/sdk` | Core engine: workbook, sheets, ranges, cells, formulas |
| `@sheetxl/io` | Format plugins; register once at app entry |
| `@sheetxl/runner` | Run a runner script against a workbook from your own code (§2) |
| `@sheetxl/primitives` | The complete (and only) import surface for formula/UDF authoring ([udf](udf.md)) |
| `sheetxl` | The CLI: run, grep, info, setup, MCP ([cli](cli.md)) |

Entry setup, once, before any file read or write:

```ts
import { WorkbookIO } from '@sheetxl/sdk';
import { IOPlugin } from '@sheetxl/io';

await WorkbookIO.install(IOPlugin);
```

Files in code. Format is inferred from the extension; options go in the second
argument:

```ts
const wb = await WorkbookIO.read('data.xlsx');
const csv = await WorkbookIO.read({ source: 'data.csv' }, { delimiter: ';' });

await WorkbookIO.write(wb, 'output.sxl');              // recalcs pending formulas first
await WorkbookIO.write(wb, 'out.csv', { delimiter: '\t' });
```

| Extension | Format | Notes |
|---|---|---|
| `.xlsx` | Excel Open XML | Full read/write, Excel compatible |
| `.sxl` | SXL (native) | 50–500x smaller, significantly faster, lossless round-trip |
| `.csv` | CSV | Configurable delimiter |
| `.json` | JSON | SheetXL array format |

`.sxl` for working copies and pipelines; `.xlsx` when Excel opens the result (the
root `SKILL.md` has the trade-off). A workbook from nothing is `new Workbook()`
(`import { Workbook } from '@sheetxl/sdk'`); `wb.getWorksheet()` is its first
sheet. The shipped type definitions are the API surface:
`node_modules/@sheetxl/sdk/types/index.d.ts`, searched with `api` as everywhere
in this skill.

## 1. Start from the template when there is no project yet

```bash
npm create sheetxl@latest my-service -- --template node
```

TypeScript, a VS Code launcher, and an `index.ts` that does the whole entry
sequence: set the license key, install the IO plugin, read a workbook, change
it, write it. Copy it, then change it. Node requirements are in the template's
`engines`.

**Done when** `npm run build && npm run dev` writes the example report.

## 2. A workbook transform: write a runner script, not a program

When the job is *take a workbook, change it, hand it back* (fill a template,
normalize an import, append a month), the program is already written. A **runner
script** is a module with a default export that receives the run context and
returns the changed workbook or a result:

```ts
// fill-report.ts
export default async function main({ wb, args, log }) {
  wb.getRange('Inputs!B2').setValue(args.month);
  log(`filled ${args.month}`);
  return wb;
}
```

Run it anywhere, with no host code:

```bash
npx sheetxl -f fill-report.ts template.xlsx --out report.xlsx -- --month 2026-11
npx sheetxl -f normalize.ts -f summarize.ts import.xlsx --save      # chained, in order
```

From your own Node code, `@sheetxl/runner` runs the same file against a workbook
you hold: `Runner.run(wb, args, { target: 'fill-report.ts' })` returns what the
script returned. The contract (snippet versus script, the five context names, no
`console` inside the script) is [scripting](scripting.md); the flags are
[cli](cli.md). Reach for a script before a bespoke program: a script runs from a
cron line, a Dockerfile, CI, and the agent's own `run`, unchanged.

## 3. A service: one model, many requests

For a pricing model, a rating engine or a calculator behind an HTTP endpoint,
load the workbook once and serve requests against it:

```ts
import { LicenseManager, WorkbookIO, type IWorkbook } from '@sheetxl/sdk';
import { IOPlugin } from '@sheetxl/io';

await LicenseManager.setLicenseKey(process.env.SHEETXL_LICENSE_KEY!);
await WorkbookIO.install(IOPlugin);
const model: IWorkbook = (await WorkbookIO.read('pricing-model.xlsx'))!;

app.post('/quote', async (req, res) => {
  model.getRange('Inputs!B2').setValue(req.body.volume);
  res.json({ quote: model.getRange('Quote!B10').getValue() });
});
```

**One workbook is one mutable model.** Two requests writing inputs at once read
each other's answers. Serve requests through it one at a time (a promise queue),
or load one workbook per worker. Save nothing unless the service's job is to
produce a file; the model on disk is the source of truth.

Pending formulas recalculate before a read, so a set followed by a get is a
correct quote. For a bulk computation over many inputs, the vector surface
([vector](vector.md)) runs it inside the engine in one pass.

## 4. A command-line tool

A bespoke CLI is a program that parses its own arguments and then does §2 or §3.
Before writing one, check that `sheetxl` itself does not already do the job:
`sheetxl book.xlsx --out book.csv` converts, `sheetxl -e` queries, `sheetxl -f`
transforms, `sheetxl grep` finds. [cli](cli.md) has the grammar.

## Operations

- **License.** `LicenseManager.setLicenseKey` at startup, from an environment
  variable. The key is verified locally; nothing phones home. Without it the
  program runs as an evaluation.
- **No display.** Nothing here needs a browser, Excel or LibreOffice. Containers,
  serverless and air-gapped hosts all work; the memory cost is the workbook.
- **Verify your own work.** No one is watching a job run: read back what you
  wrote and count what you changed before the program exits, the same rule as
  unattended `run`.
