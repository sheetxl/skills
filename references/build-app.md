---
name: build-app
description: "Starting point when the user asks for an app, page or program built on SheetXL: what to ask before building, which create-sheetxl template fits the goal (full spreadsheet app, a grid inside an app, a spreadsheet on a plain web page, a Node program), the plain-HTML and CDN route, and what done looks like. Use first, before react-app or node-app, whenever the request is to build something rather than to work on a workbook. Triggers on: build me, spreadsheet app, embed a spreadsheet, add a spreadsheet to my site, web page, HTML, CDN, script tag, vanilla, @sheetxl/studio-vanilla, template, scaffold, create-sheetxl, automate spreadsheets."
---

# Build an App — Ask, Pick, Build, Show

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing. This file
> decides *what* to build; [react-app](react-app.md) and [node-app](node-app.md)
> hold the *how*.

## 1. Look before you ask

Read the working folder first. It answers most questions for free:

- **A project is already here** (`package.json`, a framework, an `index.html`):
  you are adding SheetXL to it. Match its framework, package manager and style.
- **Empty folder, or no project:** you are scaffolding one (§3).
- **Spreadsheet files here** (`.xlsx`, `.csv`, `.sxl`): likely the data the app is
  for. Name them in your first question.

## 2. Ask, briefly

Ask **at most three questions, in one message**, and only the ones the prompt and
the folder did not already answer. Then build. Do not interview in rounds.

1. **What is it for?** One sentence: a budgeting tool, a pricing calculator, a
   report viewer, a nightly import. This decides what you put in the workbook.
2. **What should people be able to do?** Edit like a full spreadsheet, fill in a
   few inputs, or only look. This decides the template (§3).
3. **What data does it start with?** One of their files, a workbook you build, or
   blank.

When the user says "you decide" or does not answer one, pick the default in §3 and
say which you picked.

## 3. Pick the starting point

| The user wants | Start from | Then read |
|---|---|---|
| A full spreadsheet app: toolbar, formula bar, menus, dialogs | `react-studio-mui` | [react-app](react-app.md) |
| A grid inside their own app or design, with their own buttons around it | `react-chromeless` | [react-app](react-app.md) |
| A spreadsheet on a web page that has no build step | `vanilla-cdn`, or the script tag in §4 | §4 |
| A spreadsheet in a non-React bundled app (Vue, Svelte, plain TypeScript) | `vanilla` | §4 |
| A program, service or scheduled job with no UI | `node` | [node-app](node-app.md) |
| A one-off change to some files, not a program | no app | [cli](cli.md) |

Defaults when the goal is open: **`react-studio-mui`** for anything with a UI,
**`node`** for anything without one.

Scaffold with the template named, so nothing prompts:

```bash
npm create sheetxl@latest my-app -- --template react-studio-mui
```

Use `.` as the name to scaffold into the current (empty) folder. In an existing
project, do not scaffold: install the package the template would have used and
follow the reference.

## 4. A plain web page (no React in the user's code)

`@sheetxl/studio-vanilla` is the full Studio UI behind one call. React ships
inside it; the page needs none.

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
attribute out to turn it off.

**From JavaScript**, with a bundler (`npm install @sheetxl/studio-vanilla`) or
straight from the CDN (`…/studio-vanilla@latest/cdn/index.js`):

```js
import { SheetXL } from '@sheetxl/studio-vanilla';

const studio = await SheetXL.attachStudio('#sheetxl', { /* the same props Studio takes */ });
```

The container needs a height; the grid fills it. Props, loading a workbook and
theming are the `Studio` ones in [react-app](react-app.md).

## 5. Build what they asked for, then show it

A scaffold is not the deliverable. After it runs:

1. **Put their purpose in it.** Load their file, or build the starting workbook
   with real formulas ([writing](writing.md); for a model,
   [financial-models](financial-models.md)). A blank grid answers nobody's request.
2. **Run it** (`npm install && npm run dev`, or open the HTML file) and confirm
   the grid renders.
3. **Tell the user** the URL or file to open, what you built, and one or two things
   they could ask for next.

**Done when** the app is running, shows the user's data or model, and the user has
the link.

Without a license key the app runs as an evaluation: full function, a watermark,
and no watermark on `localhost`. Mention it once, at the end; keys come from
https://my.sheetxl.com.
