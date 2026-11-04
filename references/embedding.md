---
name: embedding
description: "Building an application with SheetXL — embedding the engine or UI, using @sheetxl/sdk as a library: packages, imports, entry setup, file IO in code, constructing workbooks. Use when writing the user's application code, or a standalone program outside `run`. Triggers on: embed, application, build an app, @sheetxl/sdk, @sheetxl/io, import, WorkbookIO, IOPlugin, new Workbook, library, Studio."
---

# Embedding SheetXL — Building an App on the SDK

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.

This is ordinary software development in the user's repo — no agent modes, no
sessions, no run context. Import normally and follow the app's conventions.

It is also the path for a **standalone program** when the run context cannot
express a task (e.g. operating on two workbooks at once) — same imports, same
setup, just your own small program instead of `run`.

## Packages

| Package | Purpose |
|---|---|
| `@sheetxl/sdk` | Core engine: workbook, sheets, ranges, cells, formulas |
| `@sheetxl/io` | Format plugins — register once at app entry |
| `@sheetxl/primitives` | The complete (and only) import surface for formula/UDF authoring |
| `sheetxl` | The CLI: run, grep, info, setup, MCP |

## Entry setup

```typescript
import { WorkbookIO } from '@sheetxl/sdk';
import { IOPlugin } from '@sheetxl/io';

WorkbookIO.install(IOPlugin);  // once, before any file read/write
```

## Files in code

`IOPlugin` registers the default supported formats (the table below). Format is
inferred from the file extension; options go in the third argument:

```typescript
const wb = await WorkbookIO.read('data.xlsx');
const csv = await WorkbookIO.read({ source: 'data.csv' }, { delimiter: ';' });

await WorkbookIO.writeFile('output.sxl', wb);   // recalcs pending formulas first
await WorkbookIO.writeFile('out.csv', wb, { delimiter: '\t' });
```

| Extension | Format | Notes |
|---|---|---|
| `.xlsx` | Excel Open XML | Full read/write, Excel compatible |
| `.sxl` | SXL (native) | 50–500x smaller, significantly faster, lossless round-trip |
| `.csv` | CSV | Configurable delimiter |
| `.json` | JSON | SheetXL array format |

The format trade-off (`.sxl` for working copies, `.xlsx` when Excel opens the
file) is the root `SKILL.md`; it reads the same from application code.

The shipped type definitions are the authoritative API surface —
`node_modules/@sheetxl/sdk/types/index.d.ts`.

## Constructing a workbook

```typescript
import { Workbook } from '@sheetxl/sdk';

const wb = new Workbook();
const sheet = wb.getWorksheet();
```

## Embedding the UI

Studio — the production React UI with full Excel shortcuts and visual
fidelity — embeds alongside the engine. See https://www.sheetxl.com/docs for
components and integration guides.
