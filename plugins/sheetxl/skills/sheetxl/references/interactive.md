---
name: interactive
description: "SheetXL attended mode: working with a human who has the workbook open right now. Session ownership rules, the highlight tool (mark ranges in their grid — a ranked, individually labelled list of findings they can walk; asks nothing), the prompt tool (one blocking dialog — a question, or a few as steps — shown at the cells each is about), and safe experimentation via forks. Load whenever an attended session exists, or when guiding a user in the live UI. Triggers on: attended session, attended, highlight, mark ranges, show the user, present, presence, prompt user, guide user, draw attention."
---

# Interactive — Working With a Human

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing;
> `sessions.md` for session mechanics and cell deep links.

**In this file:** Attended-mode rules · The two human-loop tools — `highlight`
and `prompt` · Common patterns

An `attended` session means a human has that workbook open in a live UI **right
now**. This reference covers the rules that keep you and the user in the same
reality, and the tools for communicating spatially in their UI.

## Attended-mode rules

1. **The attended workbook is the context.** Unqualified references ("B10", "this
   sheet") mean the active attended session — never a file you find on disk.
2. **Reach it only through its session** (`run` with that `session_id`).
   Editing the underlying file via CLI or SDK forks state the user's UI cannot
   see.
3. **Never open or close attended sessions.** The user's UI owns them. For your own
   isolated work, create a sandbox session.
4. **Experiment on a fork, not on the user.** To try something risky against live
   attended state, fork the attended session into a private copy, verify the result
   there, then apply the verified change through the attended session (guarded by
   `original_commit_id`). The full what-if loop is `sessions.md` §"Experiments
   and what-ifs".
5. **Narrate spatially.** When you change or discuss cells the user is looking
   at, point — `highlight` a range, and deep-link the addresses you cite in prose
   (`sessions.md`). A highlight beats a cell address in prose.

## The two human-loop tools

They split on one question — **am I asking for anything?**

| | `highlight` | `prompt` |
|---|---|---|
| response | none — informational | blocking — one dialog, answered once |
| ranges | many, in review order (`working`: one) | one per question (its subject) |
| lifetime | `finding`/`question`: until the user dismisses them; `working`: until the workbook next changes or you highlight anything else | until answered |

Both are UI-only: they never modify workbook data, and they need an **attended**
session — a sandbox has no grid to render into. Neither will pretend otherwise:
on a sandbox, `highlight` tells you nothing was marked and `prompt` errors. Take
either at its word rather than narrating a highlight the user cannot see.

Both also **resolve their addresses before rendering anything** — a range that
does not exist is an error naming the reason, not a silent miss, and what comes
back is what the grid actually marked (sheet-qualified). If you sent `B2` you
will see `Sheet1!B2`.

(The selection itself belongs to the user — point with `highlight` and deep
links; never move their cursor.)

### `highlight` — show, ask nothing

`highlight` is **declarative within its intent**. A `finding` or `question` call
**adds** a list the user reviews alongside the earlier ones, and those stay until
the user dismisses them, one at a time or all together. An empty `ranges` clears
the intent you pass, and only that one. Nobody has to respond.

`working` is different: it is your **cursor** — the one range you are about to
write. It replaces your previous `working` mark and clears itself as soon as the
workbook changes (your write landing, or the user's own edit) or you highlight
anything else. The user never manages it and never sees it in their findings, so
never use it for something you want them to look at.

- `ranges` — `{ range, label? }` entries. **The order is the order the user walks
  them**, so rank a findings list by what matters, not by what your scan hit
  first. Each entry carries its own `label` because that is what the user reads
  on arriving at it — one label across seven findings says the same thing at
  every stop.
- `intent` — what the marking means: `finding` (look at this — the default),
  `question` (these are the candidates; judge them), `working` (I am about to
  write here — one range).

**This is how you hand over a list of results.** An audit that found six
suspicious cells is one call, ranked, each with its own reason — the user steps
through it in the grid instead of reading six addresses out of a chat message
and clicking each one:

```text
highlight {
  ranges: [
    {range: "P&L!D69", label: "hardcoded — the rest of the row is =-(D44+D54)"},
    {range: "P&L!F12", label: "sign flipped vs. the column"},
    {range: "P&L!B7",  label: "General format in a currency column"}
  ],
  intent: "question"
}
```

It also narrates a single edit:

1. Before the edit — `highlight` the target with `working`. It shows while your
   write runs and clears itself when the write lands.
2. After the edit — `highlight` what changed as a `finding`, and say what you did
   in prose with deep links.

A walkthrough — several places, explained in order — is a `finding` list, not a
series of `working` marks: the user steps through it at their own pace.

### `prompt` — one blocking dialog, at the cells

`prompt` puts a question to the user inside the grid and waits for the answer.
Use it when the work cannot proceed without a decision that is genuinely the
user's — a missing input, a destructive choice — not for confirmations you can
express in chat.

- **One prompt at a time per session.** While one is waiting on the user,
  another is refused. Wait for the answer, or cancel your task, before asking
  again.
- **Several related decisions are one prompt.** Pass `questions` (each with its
  own `id`, `message`, `options`, `range`, and a short `header` for its tab)
  instead of `message`. The user moves between them as tabs of one dialog, each
  scrolled to and marked at its own cells, and the answers come back together
  under their ids. If
  they dismiss part-way, you get the answers given so far.
- **Say what each option does.** An option may carry a `description` — a second,
  quieter line under its label ("Change B20 to use TEXT(B17, \"0.00%\")").
- **The user can always answer in their own words.** A question with `options`
  also offers "Other" — a multi-line text box; that answer comes back as text
  (`in their own words: "…"`) instead of an option id. Read it — it is usually
  the case your options missed.

- `range` — the single range the question is **about**. The question appears at
  those cells, which is what makes it a question about *them* rather than a
  floating dialog. Pass it whenever the question has a subject.
- Omit `range` only when the question is not about a place ("which quarter should
  I model?"). It is then centred. It never falls back to the user's selection —
  that is theirs, and it is almost never what you are asking about.

Asking about cells does **not** need a `highlight` first: the prompt is already
at them. Reach for both only when the surrounding context matters too — mark the
whole block with `highlight`, then `prompt` at the one cell in question.

## Common patterns

### Orient the user before making changes

```text
highlight {ranges: [{range: "Sheet1!B2:D8", label: "Updating these prices"}], intent: "working"}
→ run (the edit, through the attended session)
→ highlight {ranges: [{range: "Sheet1!B2:D8", label: "Updated — 3 cells changed"}], intent: "finding"}
```

### Flag errors found during analysis

One `highlight` with `question` intent, ranked worst-first, each entry labelled
with its reason. The grid shows *where* and *why* together; your prose adds what
you propose to do about it.

### Ask before a destructive change

```text
prompt {
  range: "Budget!D69",
  message: "D69 is hardcoded where the column computes. Replace it with the formula?",
  options: [{id: "replace", label: "Replace"}, {id: "keep", label: "Keep as-is"}]
}
```

The question renders at D69, so there is nothing to cross-reference.

### Several decisions that belong together

```text
prompt {
  questions: [
    {id: "d69", header: "Hardcode", range: "Budget!D69",
     message: "D69 is hardcoded. Replace it with the formula?",
     options: [{id: "replace", label: "Replace", description: "Use =-(D44+D54) like the rest of the row."},
               {id: "keep", label: "Keep"}]},
    {id: "f12", header: "Sign", range: "Budget!F12",
     message: "F12's sign is flipped vs. the column. Fix it?",
     options: [{id: "fix", label: "Fix"}, {id: "keep", label: "Keep"}]}
  ]
}
→ User answered:
  - d69: replace
  - f12: keep
```

One dialog, two tabs, each marking its cells — not two prompts in a row.

### Walk a user through a model

Highlight each range in turn as you explain it in chat — inputs, then the
formulas that consume them, then the output — one `working` highlight per step,
each replacing the last.
