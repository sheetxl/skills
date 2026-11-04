---
name: setup
description: "Fallback install instructions for when SheetXL is not available and `sheetxl info` cannot run — bootstrap requirements (Node), per-client MCP registration, the VS Code extension, and skills installation. The primary setup path is always `npx -y sheetxl setup`; use this reference only when no SheetXL surface can run in this environment."
---

# Setup — Fallback Instructions

> Part of the SheetXL skill suite. The **primary** setup path is dynamic:
> `npx -y sheetxl setup` detects the environment and converges it with consent at
> each step, and `npx -y sheetxl info` reports what is missing. Use this reference
> only when neither can run here (no shell, or no Node yet) — its job is to give
> the human the instructions to run elsewhere.

## Requirements

- **Node.js** (a current LTS release) with `npm`/`npx` on PATH — everything below
  is delivered through npm.

## The one-command path (preferred)

Have the user run in any terminal:

```bash
npx -y sheetxl setup
```

It installs the skills, ensures the engine, registers the MCP server for detected
clients, and offers the VS Code extension — asking consent at each install
boundary. Re-running is always safe: already-converged steps are no-ops.

## Manual steps (when the one-command path is unavailable)

Each step is optional and independent. Confirm before installing anything into a
user's editor or client configuration.

**Skills** (ambient knowledge for any agent):

```bash
npx -y skills add sheetxl/skills -g --all
```

**Engine/CLI** (zero-install via npx, or global):

```bash
npx -y sheetxl info            # zero-install: npx fetches and caches
npm install -g sheetxl         # or: global install
```

**MCP server** (sessions, audited edits, undo — and human detection):

- Claude Code: `claude mcp add sheetxl -- npx -y sheetxl mcp`
- Other clients: add to the client's MCP configuration:

  ```json
  { "command": "npx", "args": ["-y", "sheetxl", "mcp"] }
  ```

- Newly registered servers usually attach at the next session — say so, and do
  not claim the MCP tools until they are actually connected.

**VS Code extension** (human in the loop: attended sessions, live highlights,
prompts). Ask first — this installs software into their editor:

```bash
code --install-extension sheetxl.sheetxl-vscode
```

A running VS Code window needs a reload to activate it; if the `code` command is
not on PATH, point the user to the marketplace instead.

## If something fails

Run `npx -y sheetxl info` and report its diagnostic output. Do not fall back to
editing spreadsheet files with generic tools — `.xlsx` / `.sxl` are binary.
