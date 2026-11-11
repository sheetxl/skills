---
name: react-app
description: "Building a React app around SheetXL: the Studio component (full spreadsheet UI) or the chromeless WorkbookElement, scaffolding with create-sheetxl, packages and peers, loading a workbook into the grid, reaching the engine from app code, matching the page's colors and dark mode, the license key. Use when the user wants a web app, page or React component that shows a spreadsheet. Triggers on: React, Studio, @sheetxl/studio-mui, @sheetxl/react, WorkbookElement, create-sheetxl, scaffold, Vite, Next.js, embed the grid, chromeless, theme, dark mode."
---

# React App — Putting the Grid in an Application

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. The
> engine side (packages, file IO, constructing workbooks) and any Node program or
> service beside the app are [node-app](node-app.md).

This is ordinary React development in the user's repo: no agent modes, no
sessions, no `run` context. The engine API inside the app is the same one a
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

The grid fills its container; give the container a size.

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

For an app that opens whatever `?source=` names in the URL and wires the tab
title, unsaved marker and close prompt, use the hook the templates use:
`useStudioApp()` for `Studio`, `useWorkbookApp()` for `WorkbookElement`. Spread
its result onto the component.

## 4. Reach the engine from app code

`onWorkbookLoad` and `onWorkbookChange` hand you the live `IWorkbook`. Keep it in
state and call the SDK on it: the same `getRange`, `getRegions`, `getStats`,
`setValue` the rest of this skill describes, minus the `run` context (so `await`
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

Look names up with `api` before calling something new, as everywhere in this
skill; the shipped types are the authority. `select()` on a range is how an app
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
https://www.sheetxl.com/docs/guides/concepts/theming

**Done when** the grid's backdrop, headers, gridlines and frame match the
surrounding card in both modes.

## 6. License key

Set it once at startup, before the first component renders:

```ts
import { LicenseManager } from '@sheetxl/sdk';
LicenseManager.setLicenseKey('…');
```

Without a key the grid runs as an evaluation: full function, a watermark, and no
watermark on `localhost`. Keys come from https://my.sheetxl.com.

## Pointers

- Engine-side setup, file formats, constructing workbooks in code: [node-app](node-app.md), §Engine.
- Styling, charts, formulas from code: [writing](writing.md), [charts](charts.md).
- Guides and the component API: https://www.sheetxl.com/docs and https://api.sheetxl.com.
