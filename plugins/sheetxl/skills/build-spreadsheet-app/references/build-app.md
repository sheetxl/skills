---
name: build-app
description: "Build an app, web page, program or service that has a spreadsheet in it or works with spreadsheets, on SheetXL: a full Excel-like spreadsheet app, a spreadsheet grid embedded in an existing React app or site, a spreadsheet on a plain HTML page from a CDN script tag, or a headless Node program, API or scheduled job that reads, calculates or writes .xlsx files. What to ask the user first, which create-sheetxl template fits, and what done looks like. Use when the request is to build or embed something, not to work on one workbook. Triggers on: build me an app, spreadsheet app, Excel-like app, embed a spreadsheet, spreadsheet component, React spreadsheet, data grid with formulas, add a spreadsheet to my site, web page, HTML, CDN, script tag, Vue, Svelte, Node service, API that calculates a workbook, automate spreadsheets, scaffold, template, create-sheetxl, @sheetxl/sdk, @sheetxl/studio-mui."
---

# Build an App — Ask, Pick, Build, Show

> This file decides *what* to build; [ui-app](ui-app.md) (anything people see) and
> [headless-app](headless-app.md) (anything without a UI) hold the *how*. Working on
> a workbook itself, rather than building something around one, is the `sheetxl`
> skill.

## 1. Look before you ask

Read the working folder first. It answers most questions for free:

- **A project is already here** (`package.json`, a framework, an `index.html`):
  you are adding SheetXL to it. Match its framework, package manager and style.
- **Empty folder, or no project:** you are scaffolding one (§3).
- **Spreadsheet files here** (`.xlsx`, `.csv`, `.sxl`): likely the data the app is
  for. Name them in your first question.

**Asked what it would take, not to build it** ("how would we switch to…", "what's
involved in…"): answer with the plan instead of building. Name the route from §3,
the packages, what changes in their code and what stays, and the open decisions
(§2's questions); then offer to build it.

## 2. Ask, briefly

Ask **at most three questions, in one message**, and only the ones the prompt and
the folder did not already answer. Then build. Do not interview in rounds.

1. **What is it for?** One sentence: a budgeting tool, a pricing calculator, a
   report viewer, a nightly import. This decides what you put in the workbook.
2. **What should people be able to do?** Edit like a full spreadsheet, fill in a
   few inputs, or only look. This decides the template (§3).
3. **What data does it start with?** One of their files, a workbook you build, or
   blank.

When more than one person will use it ("our team", "our customers"), the third
question becomes: **where are workbooks kept, and do two people edit the same one
at once?** SheetXL runs in each user's browser and has no built-in shared editing,
so storage, access and conflicts are the app's ([ui-app](ui-app.md) §3, "Several
people, one workbook"). "Like Google Sheets" usually means live co-editing: say
plainly that the app will not have it, and what it will have instead.

For a service or job, the third question is **which cells are the inputs and which
are the outputs**. Find them in the workbook yourself first (its labels say most of
it; [reading](reading.md) reads a workbook's structure in one call) and ask only to
confirm.

When the user says "you decide" or does not answer one, pick the default in §3 and
say which you picked. When the request already answers all three, ask nothing.

## 3. Pick the starting point

| The user wants | New project: template | Existing project: install | Then read |
|---|---|---|---|
| A full spreadsheet app: toolbar, formula bar, menus, dialogs | `react-studio-mui` | `@sheetxl/studio-mui` and its peers | [ui-app](ui-app.md) §1 new, §2 existing |
| A grid inside their own app or design, with their own buttons around it | `react-chromeless` | `@sheetxl/react` `@sheetxl/sdk` | [ui-app](ui-app.md) §1 new, §2 existing |
| A spreadsheet on a web page that has no build step | `vanilla-cdn` | nothing: a script tag | [ui-app](ui-app.md) §6 |
| A spreadsheet in a non-React bundled app (Vue, Svelte, plain TypeScript) | `vanilla` | `@sheetxl/studio-vanilla` | [ui-app](ui-app.md) §6 |
| A program, service or scheduled job with no UI | `node` | `@sheetxl/sdk` `@sheetxl/io` | [headless-app](headless-app.md) |
| A one-off change to some files, not a program | no app | the `sheetxl` CLI | [cli](cli.md) |

"Full app" or "grid inside their design" when the request does not say: see
[ui-app](ui-app.md) §2, "Which one".

Defaults when the goal is open: a new app with a UI starts from
**`react-studio-mui`**; a spreadsheet added to a page an existing app already designs is
**`WorkbookElement`** (`react-chromeless`); anything without a UI is **`node`**.

Scaffold with the template named, so nothing prompts:

```bash
npm create sheetxl@latest my-app -- --template react-studio-mui
```

Use `.` as the name to scaffold into the current (empty) folder. In an existing
project, do not scaffold: install the packages in the table's "Existing project"
column and follow the reference.

## 4. Build what they asked for, then show it

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
