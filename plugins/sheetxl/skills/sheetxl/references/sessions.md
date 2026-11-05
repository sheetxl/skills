---
name: sessions
description: "SheetXL sessions — stateful spreadsheet work: workbooks kept in memory between calls, per-run commits with undo/restore, commit history, forking and the what-if workflow (experiment on a copy, verify, then apply), saving to disk, and the audited write path. Use whenever the SheetXL MCP server is connected. Triggers on: session, list_sessions, list_commits, session_create, session_save, session_close, commit, revert, restore, drift, fork, sandbox, attended session, what-if, experiment, try it out, compare scenarios."
---

# Sessions — Stateful Spreadsheet Work

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing;
> `interactive.md` for the human-loop tools (`highlight`, `prompt`).

**In this file:** Start here · Tools · Discovery · History — `list_commits` ·
Working in a sandbox · Experiments and what-ifs — fork first · The write path —
`run` · Linking to cells · Agent operating rules · Reading a session

The MCP server gives you live read/write access to spreadsheet **sessions**. A
session is a workbook plus where its engine lives:

- **sandbox** — owned by the server, headless, in-process. You create these on demand.
- **attended** — owned by a client UI (a VS Code workbook), reached over the relay.

**Scripting is the API here — the short tool list is deliberate.** The server
exposes discovery and session management as tools; every write goes through
`run`, which executes SDK TypeScript you author against the live workbook.
Writing code is the primary, fully supported path — not a fallback for missing
tools.

## Start here — before doing anything else

1. **Workbook files come through these tools, not the file system.** Do NOT
   read `.xlsx` or `.sxl` from disk — they are binary. (`.csv` is plain text:
   any tool may handle it; come through these tools when formulas, types, or
   structure matter.)
2. **Know your sessions.** The `info` handshake (root `SKILL.md`) already seeds
   the open sessions and connected users; call `list_sessions` to refresh that
   view at any point. Every tool requires a `session_id` — pass the id of the
   session you mean; there is no default, and a call without one is refused
   rather than routed to whatever the user has open. An `attended`
   session means a user has that workbook open right now — when they reference
   a cell without naming a file ("what's wrong with B10?"), they mean the
   active attended session, never a file on disk.
3. **Never open or close attended sessions.** The client UI owns those. To work
   in isolation, create your own sandbox with `session_create`.

## Tools

| Tool | Purpose |
|---|---|
| `info` | The session-start handshake: environment, sessions, users. Read-only. |
| `grep` | Find cells by content or pattern. |
| `run` | Execute SDK TypeScript as one undoable commit — the write path, and every structured read (return the subject and it renders). |
| `list_sessions` | List all sessions (sandbox + attended). |
| `list_commits` | A session's history: who changed what, when — the source of `commitId`s. |
| `session_create` | Create a sandbox: empty, from a `path`, or a fork of a `source` session. |
| `session_close` | Dispose a sandbox you no longer need (sandboxes only). |
| `session_revert` | Undo a specific change by its `commitId`. |
| `session_restore` | Move a session back to a previous commit point. |
| `session_save` | Write a session to disk. No `path` (or its own file) saves back to the file it was opened from; another `path` is a Save As, which a sandbox then belongs to. `overwrite` is needed only to replace a different existing file. Relative paths resolve against the workspace; format guidance is in the root `SKILL.md`. |
| `highlight` / `prompt` | Human-loop tools — see `interactive.md`. |

The feature vocabulary — what `getRegions` / `getCounts` take inside `run` — is
documented in `reading.md`.

## Discovery

`list_sessions` returns every open session. Each entry has:

- `id` — pass this as `session_id`.
- `kind` — `sandbox` (yours, writable now) or `attended` (a client-UI workbook).
- `origin` — `blank`, `file`, or `fork`; `forked from` names the source session,
  and a fork is named after what it forked (`Budget.xlsx (fork)`).
- `view` or `headless` — whether a grid is showing it. `highlight` and `prompt`
  only do anything on a session with a view.
- `path` — the workbook's file, when known.
- `opened … ago` — the session's **history epoch**: when its workbook was
  (re)loaded. Commit ids from before this moment are gone.
- `head` — the current commit id (pass as `original_commit_id` for a guarded write).

If a chat context chip is attached (value `<id>:<address>`), the part before `:`
is an attended `session_id`.

**Attended ids are stable; histories are not.** An attended session's id is
derived from its file path, so the same file has the same id across editor
reloads and restarts — an id you learned earlier keeps working. What does NOT
survive a reload is the workbook's commit/undo history: the `opened` stamp
resets, and commit ids from before it are dead. Sandboxes are the perishable
kind — they live only as long as the server process.

**You are told when any of this happens.** Changes you did not make — the user
opening, closing, or reloading a workbook — are announced at the top of your
next tool result. `list_sessions` also reports **recently closed** sessions in
their own section: not usable, listed so an id you are holding resolves to a
reason rather than to silence. Passing a dead id to any tool gives you the same
explanation.

## History — `list_commits`

`list_commits(session_id, limit?)` returns the session's commit log, newest
first — `commitId`, description, who wrote it (or unattributed), timestamp. It
is how you obtain a `commitId` for `session_revert` / `session_restore`.

**Read it in order, whole.** Resuming a session, or working alongside a user,
start here: *"three commits since you last looked, two of them not mine"* is the
signal that your assumptions about the workbook may be stale — re-read before
writing. The order matters as much as the authors: a change of theirs that
landed after yours may have moved what you wrote.

## Working in a sandbox

1. `session_create` with no arguments (empty workbook), a `path` (open from
   disk), or a `source` (clone another session — including an attended one, for
   safe experiments). The origin (`blank` | `file` | `fork`) is inferred from
   which you pass. Returns the new `session_id`.
2. Make changes with `run` (below).
3. `list_commits` to read the history; `session_revert` / `session_restore` to undo.
4. `session_save` to persist the result to disk — commits live in the session;
   the file only changes when you save.
5. `session_close` when done (frees the in-memory workbook). Only sandboxes can
   be closed.

## Experiments and what-ifs — fork first

Any work you are not yet sure you want is done on a **fork**, not on the
original: a risky multi-step restructure, "what happens to the totals if this
assumption changes", comparing two approaches before committing to one. A fork
is an in-memory clone — cheap to make, free to discard — and the original never
sees the experiment.

The loop:

1. **Fork** — `session_create` with `origin: "fork"` + `source: <session_id>`.
   Works on any session, including an attended one.
2. **Try** — make the changes with `run` against the fork.
3. **Verify** — read the results back off the fork; compare against the original
   by running the same read on both sessions ("original says X, the fork says Y"
   is one `run` per session).
4. **Apply or discard** — apply the verified change to the original as a fresh
   `run` (on an attended session, guarded by `original_commit_id` — the user may
   have edited meanwhile); or just `session_close` the fork and nothing ever
   happened.

Do **not** re-play the experiment on the original by saving the fork over its
file — the fork's history includes the dead ends, and on an attended session the
file is not the workbook (the UI's unsaved state is). The fork is where you
*learn* the change; the original is where you *make* it, once, cleanly.

Two degenerate forms are already safe without a fork: a `session_revert` after a
single bad commit (every `run` is undoable), and a pure read — reading never
needs isolation. And `createFormula` is the zero-cost end of the same
spectrum: a one-formula what-if that commits nothing (`analysis.md`).

On the **CLI** there are no forks; the same discipline is a file copy — work on a
duplicate, or simply withhold `--save` until the run's output proves the change,
since nothing touches disk without it.

## The write path — `run`

The source comes from exactly one of two parameters — never both: `code` (the
source as a string) or `path` (a local file this server reads). Which to use, and
what the code means once it arrives — snippet versus script, the context, the
ambient `SheetXL` API — is `scripting.md`. Either form executes as a single
undoable commit against the session.

Both forms:

- Always pass a `description` — it becomes the undo/commit label.
- On success you get a `commitId`; use it with `session_revert`.
- To guard against concurrent edits, pass `original_commit_id` (the session
  `head`); if the session changed since, the write is rejected with
  `status: "drift"` — re-read and retry.
- If compilation fails, the report lists positioned diagnostics with a `fix`
  for each — apply them and call `run` again.

### Attended sessions

The read tools, `run`, `session_revert`, `session_restore`, and the human-loop
tools all work against an attended session — edits land in the user's grid the
moment they commit, and the user reviews and reverts them from the agent panel at
the top right of the grid, so there is nothing to warn them about. Because the
user may also be editing, pass `original_commit_id` on writes so a concurrent
edit is caught as `drift` rather than silently overwritten. Never open or close attended sessions;
the UI owns their lifecycle. The full attended-mode rules are `interactive.md`.

## Linking to cells (attended sessions only)

When you cite a cell or range that lives in an **attended** session, render it as a
clickable deep link so the user can jump straight to it. Clicking the link
reveals that workbook and selects the range in the grid.

Markdown form — link the address text to a `vscode://` URI:

```text
[Sheet1!B10](vscode://sheetxl.sheetxl-vscode/select?session=<session_id>&range=Sheet1!B10)
```

- `session` — the attended session's `id` (from `list_sessions`). Include it whenever
  more than one workbook is open; if you omit it, the link targets the active one.
- `range` — a sheet-qualified address (`Sheet1!A1:B4`) or a named range. URL-encode
  it (a space becomes `%20`, `!` may stay literal).
- The link text is yours — use the human-readable address (e.g. `Sheet1!B10` or
  "the total"), not the URI.

Do **not** emit these links for **sandbox** sessions — they are headless and have
no grid to reveal. Prefer linking the specific cells you reference in prose
(totals, errors, the range you changed) rather than every address.

## Agent operating rules

- Do not perform arithmetic or computation on cell values yourself — the
  calculation engine handles this correctly. To compute: write a formula
  (e.g. `=SUM(A1:A10)`) or use the SDK via `run`, and let the engine evaluate.
- Use cell values to understand what the user sees and to write back what the
  user would type. The value-encoding contract (the `'` prefix and friends) is
  the root `SKILL.md` — it applies to every tool input and response carrying
  cell data.

## Reading a session

Reading routes through two tools. `grep` takes an **A1-scoped** pattern search
(a range, `Sheet1!A1:D9`, or a bare sheet name for its used range); everything
else is `run`, where naming a subject in code *is* how you describe it. Which
subject answers which question is `reading.md`; the order to take them in is the
root `SKILL.md`.

Every response is self-describing — it names what it does *not* contain
(truncated, or a different lens available), so the next call is always obvious.
Elision is explicit; results are never silently truncated.
