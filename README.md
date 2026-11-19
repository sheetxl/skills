<!-- markdownlint-disable -->
# @sheetxl/skills

Agent skills for **SheetXL** — an Excel-grade spreadsheet built for agents and developers.

Two skills:

| Skill | For |
|---|---|
| `sheetxl` | Working on a workbook: read, query, audit and edit `.xlsx`, `.sxl` and `.csv` files. Query a workbook like a database rather than pulling cells into context, execute TypeScript inside the calculation engine with no compile step, and work alongside a human who has the file open. |
| `build-spreadsheet-app` | Building an app, page or service with a spreadsheet in it: what to ask first, which `create-sheetxl` template fits, the React and plain-web-page routes, and headless Node programs. |

## Install

```bash
npx -y skills add sheetxl/skills -g --all
```

Copies both skills, each a `SKILL.md` with its `references/`, into your agent's skill
directories. `npx skills update` refreshes them. `--skill sheetxl` or
`--skill build-spreadsheet-app` in place of `--all` installs one.

### Upgrading from an earlier install

Installing copies files and never deletes them, so a reference retired in a later release
stays in your skill directory, and an agent that greps the folder reads it as current. Remove
the old copy before reinstalling:

```bash
for d in ~/.claude/skills ~/.codex/skills ~/.cursor/skills ~/.agents/skills; do
  rm -rf "$d/sheetxl" "$d/build-spreadsheet-app"
done
npx -y skills add sheetxl/skills -g --all
```

Check the project-local `.claude/skills/`, `.codex/skills/` and `.cursor/skills/` too.
`sheetxl info` reports which version of the skill an agent will find.

### Plugin — the skill and the MCP server together

For Claude Code and Codex, the `sheetxl` plugin installs both skills with the SheetXL MCP
server, at the same version:

```bash
# Claude Code
claude plugin marketplace add sheetxl/skills
claude plugin install sheetxl@sheetxl

# Codex
codex plugin marketplace add sheetxl/skills
codex plugin add sheetxl@sheetxl
```

The MCP server works with any MCP client on its own: `npx -y @sheetxl/mcp`.

The SheetXL MCP server (`@sheetxl/mcp`) depends on this package and serves it to clients
as `sheetxl://skills/*` resources, so a copy released with the engine lands beside it.

## Contents

`SKILL.md` is the `sheetxl` skill's entry point: framing, the once-per-session
`sheetxl info` handshake, the two ideas (grep / run), the ladder, the cost model, the
value-encoding contract, and routing. `build-app.md` is the `build-spreadsheet-app`
skill's entry point. Each reference is loaded on demand and owns its surface outright —
the entry point routes, the references answer. Both skills carry every reference, so
links between them resolve whichever skill an agent loaded.

| Reference | Load when |
|---|---|
| `reading.md` | understanding a sheet: the subject→report map, regions, profiling, stats, style |
| `analysis.md` | auditing or repairing a workbook: the shape sweep, the partition catalogue, error tracing, the blast radius, proving the fix |
| `writing.md` | changing a sheet **and verifying the change** |
| `vector.md` | computing over many cells: `IRange`, transforms, `setComputedValues` |
| `interactive.md` | a human has the workbook open: sessions, presence, guidance |
| `cli.md` | terminal commands: flags, chaining, the REPL |
| `sessions.md` | stateful work: commits, history, undo, forking and what-ifs, saving |
| `scripting.md` | writing the code you hand to `run` |
| `build-app.md` | building an app, page or service on SheetXL: what to ask, which template |
| `ui-app.md` | an app or page with the grid in it: Studio, WorkbookElement, the CDN script tag |
| `headless-app.md` | a program or service with no UI on the SDK; packages, file IO, runner scripts |
| `udf.md` | authoring formula functions |
| `showcase.md` | demonstrating what SheetXL can do |
| `setup.md` | SheetXL is not installed and `info` cannot run |

## Versioning

Report-only — no pinning machinery. `sheetxl info`, which the skill instructs the agent
to run once per session, detects a stale skill or engine and prints the one-line fix.

## Package

Markdown only — **this package ships no code.** Consumers resolve it
(`require.resolve('@sheetxl/skills/package.json')`) and read the files in place. Also
mirrored on jsDelivr:

```
https://cdn.jsdelivr.net/npm/@sheetxl/skills@<version>/references/reading.md
```

## Contributing

Authored and released from the [SheetXL monorepo](https://github.com/sheetxl/sheetxl).
Issues and corrections are very welcome —
[open an issue](https://github.com/sheetxl/sheetxl/issues).

## License

MIT
