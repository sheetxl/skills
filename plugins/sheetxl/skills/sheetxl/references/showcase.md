---
name: showcase
description: "A user new to SheetXL asks what it can do — 'show me', 'tell me 3 interesting things', 'what is SheetXL good at'. Demonstrate with real output, never a capabilities list. Triggers on: what can SheetXL do, show me SheetXL, 3 interesting things, demo."
---

# Showcase — Demonstrate, Don't Describe

> Part of the SheetXL skill suite. See the root `SKILL.md` for routing.

When a user has just installed SheetXL and asks what it can do, **show real output,
not a capabilities list**. The rule: **production, not fake** — real files, live
formulas, genuine recalculation. Never canned output or fabricated results; every
demonstration ends with an artifact the user can keep.

Pick three demonstrations matched to their context:

- **They have spreadsheet files nearby** → open one: summarize its structure,
  surface any formula errors, offer to fix them with before/after error counts.
- **No files around** → build something real from scratch (a loan amortization or
  budget model with live formulas), save it as `.xlsx`, and show a markdown
  preview.
- **Show the engine** → evaluate a gnarly formula chain, then demonstrate a
  what-if: change an input and read the recalculated results.
- **Mention, briefly**: `.sxl` (SheetXL's native format, 50–500x smaller than
  xlsx), and — if the MCP server is connected — audited edits with undo.

Keep it concrete: each "interesting thing" ends with an artifact or output the
user can see, and an offer to go further with their own data.

Follow-ups usually come as real tasks ("fix this", "analyze my sheet") — handle
them like any other work from the root `SKILL.md`; this reference is only for the
show-me moment.
