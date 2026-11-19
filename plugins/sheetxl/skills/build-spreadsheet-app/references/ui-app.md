---
name: ui-app
description: "Building an app or page that shows a spreadsheet: the Studio component (full spreadsheet UI) or the chromeless WorkbookElement in React, the full Studio on a plain web page with no React (script tag, CDN, Vue, Svelte, plain TypeScript), scaffolding with create-sheetxl, packages and peers, loading a workbook into the grid, reaching the engine from app code, matching the page's colors and dark mode, the license key. Use when the user wants a web app, page or component with a spreadsheet in it. Triggers on: React, Studio, @sheetxl/studio-mui, @sheetxl/react, WorkbookElement, @sheetxl/studio-vanilla, script tag, CDN, HTML, vanilla, Vue, Svelte, create-sheetxl, scaffold, Vite, Next.js, embed the grid, chromeless, theme, dark mode."
---

# UI App — Putting the Grid in an Application

> Part of the SheetXL skills. Which page to build from is the `build-spreadsheet-app` skill's `SKILL.md`, the same page as [build-app](build-app.md). The
> engine side (packages, file IO, constructing workbooks) and any Node program or
> service beside the app are [headless-app](headless-app.md).

This is ordinary app development in the user's repo: no agent modes, no
sessions, no `run` context. §1–§5 are React; a page without React is §6. The engine API inside the app is the same one a
`run` snippet uses, called from your own code.

## 1. Start from a template when there is no app yet

```bash
npm create sheetxl@latest my-app -- --template react-studio-mui   # full UI: toolbar, menus, dialogs
npm create sheetxl@latest my-app -- --template react-chromeless   # grid, tabs and scrollbars only
```

`@beta` in place of `@latest` scaffolds against the upcoming SDK. Use `.` as the
name to scaffold into the current directory. The template is Vite + TypeScript
with a VS Code launcher; `npm install && npm run dev` serves it. Its `App.tsx` is
the reference implementation of everything below: copy it, then change it.

**Done when** `npm run dev` shows the grid in the browser.

## 2. Add to an existing React app

Two components, one engine:

| Component | Package | What it is |
|---|---|---|
| `Studio` | `@sheetxl/studio-mui` | The production spreadsheet UI: toolbar, formula bar, sheet tabs, dialogs, theming. Material UI based. |
| `WorkbookElement` | `@sheetxl/react` | Chromeless: the sheet, tabs and scrollbars, no UI framework. Build your own chrome around it. |

**Which one.** `Studio` when people work *in* the spreadsheet: they need formatting,
inserting, sheets, the formula bar — a spreadsheet app, an editor page. `WorkbookElement`
when the spreadsheet is one part of a page the app already designs: a pricing table in a
dashboard, a calculator, a form backed by a model, where the app's own buttons do the few
things users need. "Users can edit it" alone does not decide it: both are editable.
When the request does not say, decide by the page rather than asking:
`WorkbookElement` inside a page the app already designs, `Studio` for a page that *is*
the spreadsheet. Name the choice in your reply so the user can switch it.

Peers for `Studio`: React 19, Material UI 9 (`@mui/material`, `@mui/system`,
`@mui/utils`, `@mui/styled-engine`) and Emotion (`@emotion/react`,
`@emotion/styled`, `@emotion/cache`). `@sheetxl/sdk` arrives as a dependency.
Check the template's `package.json` for the current list rather than this one.

```tsx
import { Studio } from '@sheetxl/studio-mui';

export default function App() {
  return <Studio style={{ position: 'absolute', inset: 0 }} />;
}
```

`WorkbookElement` needs only React 19 and `react-dom` beside it (`npm install
@sheetxl/react @sheetxl/sdk`); no UI framework:

```tsx
import { WorkbookElement } from '@sheetxl/react';

export function PricingTable() {
  return (
    <div style={{ height: 400 }}>
      <WorkbookElement workbook={{ source: '/pricing.xlsx' }} />
    </div>
  );
}
```

Either component fills its container; give the container a size.

**Browser only.** The components touch `window` on import. In a framework that
renders on the server (Next.js), load them with a dynamic import and SSR off:

```tsx
const Studio = dynamic(() => import('@sheetxl/studio-mui').then((m) => m.Studio), { ssr: false });
```

## 3. Load a workbook

The `workbook` prop takes an `IWorkbook`, a `Promise<IWorkbook>`, or read
options (`{ source }`), so a file served by the app opens in one line:

```tsx
<Studio workbook={{ source: '/budget.xlsx' }} />
```

`source` takes a URL or path, and also the bytes themselves: a `File` (from an
`<input type="file">` or a drop), a `Blob`, an `ArrayBuffer` or a `Uint8Array`. So a
file the user uploads opens without a server round trip:

```tsx
const [source, setSource] = useState<File | null>(null);

<input type="file" accept=".xlsx,.csv,.sxl" onChange={(e) => setSource(e.target.files?.[0] ?? null)} />
{source && <Studio workbook={{ source }} />}
```

A `File` names its format; raw bytes do not, so pass the format with them:
`{ source: arrayBuffer, format: 'xlsx' }`. Changing the `workbook` prop opens the new
workbook in place of the old one.

For an app that opens whatever `?source=` names in the URL and wires the tab
title, unsaved marker and close prompt, use the hook the templates use:
`useStudioApp()` for `Studio`, `useWorkbookApp()` for `WorkbookElement`. Spread
its result onto the component.

### Save the user's edits

The workbook lives in the page. To keep the user's changes, write it to bytes and
send them where your app stores files:

```tsx
import { WorkbookIO, type IWorkbook } from '@sheetxl/sdk';

async function save(wb: IWorkbook, id: string) {
  const bytes = await WorkbookIO.writeArrayBuffer(wb, { format: 'xlsx' });   // or 'sxl', 'csv'
  await fetch(`/api/sheets/${id}`, { method: 'PUT', body: bytes });
}
```

Save on an explicit button, or debounced on change (`onWorkbookChange` hands you the
workbook; `Studio`'s `onDirtyChange` says when there is something unsaved). Load it
back with `workbook={{ source: '/api/sheets/' + id }}`. `.sxl` is smaller and keeps
everything SheetXL has; `.xlsx` when users download the file for Excel.

**Several people, one workbook.** SheetXL has no built-in multi-user editing: each
browser holds its own copy. A team app keeps workbooks on its own server (the save
and load above, behind the app's auth) and decides what happens when two people
save the same one (last write wins, or a lock while someone edits). Ask the user
which before building: where workbooks are stored, who can open which, and whether
two people edit at once.

### Coming from SheetJS (or ExcelJS)

Those libraries parse and write files; SheetXL is the spreadsheet itself, so most
of the old code goes away rather than being translated:

| SheetJS | SheetXL |
|---|---|
| `XLSX.read(data)` | `WorkbookIO.read({ source: data, format: 'xlsx' })`, or pass the source straight to the component |
| `XLSX.utils.sheet_to_json(ws)` | `wb.getWorksheet().getUsedRange()?.getValues()`; the values are calculated, not the file's cached ones |
| your own table UI rendering the rows | `Studio` or `WorkbookElement` |
| `XLSX.write(wb, …)` | `WorkbookIO.writeArrayBuffer(wb, { format: 'xlsx' })` |
| formulas shown as text, results stale until Excel opens the file | formulas calculate in the page as users edit |

Keep server code that only stores and moves files; replace the parts that read
cells out to show or compute them.

## 4. Reach the engine from app code

`onWorkbookLoad` and `onWorkbookChange` hand you the live `IWorkbook`. Keep it in
state and call the SDK on it: the same `getRange`, `getRegions`, `getStats`,
`setValue` the `sheetxl` skill describes for workbook work, minus the `run` context (so `await`
is explicit and `console` is yours).

```tsx
const [wb, setWb] = useState<IWorkbook | null>(null);

<Studio workbook={{ source: '/budget.xlsx' }} onWorkbookChange={setWb} />

<button onClick={async () => {
  const sheet = wb!.getWorksheet();
  const anomalies = sheet.getRegions('anomalies');   // members: `api ICellRegions`
  const stats = wb!.getRange('Model!B2:F40').getStats();
  await wb!.getRange('Model!B2').select();           // the grid scrolls there and selects it
}}>Check the model</button>
```

Look names up before calling something new: `npx sheetxl api <name>` answers from
the shipped types in one call (`api ICellRange` lists its members); the types are
the authority. `select()` on a range is how an app
shows a user a cell; the in-grid `highlight` and `prompt` exist only inside an
MCP session, not in app code.

## 5. Match the page's look

The grid is styled by `--sxl-*` tokens, resolved per **mode** (`light`, `dark`)
and per **surface**: `app` (the backdrop and chrome), `header` (row and column
headers) and `grid` (the cells). An `app` value reaches every surface in its
mode; a `header` or `grid` value makes that surface differ.

On `WorkbookElement`, set both schemes and state the tokens:

```tsx
<WorkbookElement
  colorScheme="auto"        // 'light' | 'dark' | 'auto': a dark/light class on <html>/<body>, else the OS
  gridColorScheme="auto"    // the cells; unset, they stay light in a dark app
  tokens={{
    dark: {
      app: { 'color-base-100': '#14102c', 'color-base-200': '#14102c', 'color-primary': '#6e9bff' },
      header: { 'color-base-100': '#1a1538', 'color-base-300': '#312b55' }
    }
  }}
/>
```

Pass `tokens` a stable object (a module constant or a memo); a new object
rebuilds the stylesheet.

The same values as a stylesheet, for a page that themes in CSS. Put it in
`@layer sxl-overrides`; where the sheet loads does not matter:

```css
@layer sxl-overrides {
  [data-sxl-theme-mode="dark"] {
    --sxl-color-base-100: #14102c;
    --sxl-color-base-200: #14102c;
    --sxl-color-primary: #6e9bff;
  }
  [data-sxl-theme-mode="dark"][data-sxl-theme-surface="header"] {
    --sxl-color-base-100: #1a1538;
    --sxl-color-base-300: #312b55;
  }
}
```

Which token paints what:

| Token | `app` | `header` | `grid` |
|---|---|---|---|
| `color-base-100` | chrome paper | header paper | cell paper |
| `color-base-200` | backdrop around the grid, the scrollbar corner | | |
| `color-base-300` | | header dividers | gridlines |
| `color-base-border` | the grid frame, the header/cell seam | | |
| `color-base-content` | text | header labels | |
| `color-primary` | selection, scrollbar thumb | | |

Scrollbars derive from `base-content` and `primary`; they need no token of
their own. The cells' own fills and fonts come from the workbook (its
document theme and styles), not from tokens.

`Studio` takes the same `tokens` shape alongside `muiTheme`: there the values
feed Studio's palette derivations, so an `app` value does not flatten the
header and grid.

Every token, the corner-mark classes and Studio's options:
https://www.sheetxl.com/docs/guides/how-tos/theme-the-spreadsheet

**Done when** the grid's backdrop, headers, gridlines and frame match the
surrounding card in both modes.

## 6. Without React: a plain web page, Vue, Svelte, plain TypeScript

`@sheetxl/studio-vanilla` is the full Studio UI behind one call. React ships
inside it; the page needs none. The `vanilla` template is this in a Vite project
(any framework, or none); `vanilla-cdn` is the no-build-step page. In an existing
site, add the script tag or the package instead of scaffolding.

**Script tag, no JavaScript.** A sized `<div>` and the loader:

```html
<div id="sheetxl" style="height: 600px; width: 100%;"></div>
<script
  src="https://cdn.jsdelivr.net/npm/@sheetxl/studio-vanilla@latest/cdn/loader.js"
  data-target="#sheetxl"
  defer>
</script>
```

Options are `data-*` attributes in kebab-case (`data-auto-focus="true"` becomes
`autoFocus`). Values arrive as strings and any present value is truthy; leave an
attribute out to turn it off. The license key is an attribute too:
`data-license-key="…"`. Anything that is not a string or an on-switch (a workbook
to open, hiding the toolbar) needs the JavaScript form below.

**From JavaScript.** With a bundler, `npm install @sheetxl/studio-vanilla` and import it
by name, as below. On a page with no build step, import the same module from the CDN
inside a module script:

```html
<div id="sheetxl" style="height: 600px;"></div>
<script type="module">
  import { SheetXL } from 'https://cdn.jsdelivr.net/npm/@sheetxl/studio-vanilla@latest/cdn/index.js';
  await SheetXL.attachStudio('#sheetxl', { workbook: { source: '/pricing.xlsx' } });
</script>
```

The full form, with every option the examples use:

```js
import { SheetXL } from '@sheetxl/studio-vanilla';

const studio = await SheetXL.attachStudio(
  { selector: '#sheetxl', licenseKey: '…' },          // or just '#sheetxl'
  {
    workbook: { source: '/pricing.xlsx' },             // any §3 source: a URL, a File, bytes
    renderToolbar: null,                               // a lighter grid: no toolbar,
    showFormulaBar: false,                             // no formula bar,
    showStatusBar: false,                              // no status bar
  }
);

studio.getWorkbook();                                  // the live workbook (§4)
await studio.update({ workbook: { source: file } });   // swap in another file, e.g. a new upload
await studio.detach();                                 // unmount
```

The container needs a height; the grid fills it. The props are `Studio`'s:
loading and saving (§3), the engine (§4), theming (§5), and the callbacks too, so
`onWorkbookChange` and `onDirtyChange` go in the same object. An upload is a `File`
from an `<input type="file">`, passed as `update({ workbook: { source: file } })`.
Hiding the toolbar, formula bar and status bar is how a page without React gets a
light grid; the chromeless `WorkbookElement` itself is React only. In Vue or Svelte, attach in
the component's mount hook (`onMounted`, `onMount`) to an element it renders, and
detach in its unmount hook (`onUnmounted`, `onDestroy`); call `update` when the
file changes rather than attaching again.

## 7. License key

Set it once at startup, before the first component renders (the script tag and
`attachStudio` take it as shown in §6):

```ts
import { LicenseManager } from '@sheetxl/sdk';
LicenseManager.setLicenseKey('…');
```

Without a key the grid runs as an evaluation: full function, a watermark, and no
watermark on `localhost`. Keys come from https://my.sheetxl.com.

## Pointers

- Engine-side setup, file formats, constructing workbooks in code: [headless-app](headless-app.md), §Engine.
- Styling, charts, formulas from code: [writing](writing.md), [charts](charts.md).
- Guides and the component API: https://www.sheetxl.com/docs and https://api.sheetxl.com.
