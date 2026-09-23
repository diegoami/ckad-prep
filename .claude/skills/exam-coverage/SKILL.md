---
name: exam-coverage
description: Verify and backfill CKAD study-material coverage for a topic across guide/exam-tips.md, guide/task-patterns.md and drills/drill.md (then regenerate drills/workbook.md). Use when the user names a command/feature ("command line ingress", "kubectl debug", "startup probes") and asks whether it's covered, wants it added, or says things like "make sure X is in the drills" / "did we cover X". Also use with no topic to audit drill.md for structural problems.
---

# Exam coverage check

The study material has distinct roles. Keep each file doing its own job rather than merging content.

| File | Role | Style |
|---|---|---|
| `drills/drill.md` | Q&A drill deck: question + collapsible canonical answer + doc link | Terse, `<details><summary>answer</summary>` blocks, headings `### N.M question — [doc](url)` |
| `drills/workbook.md` | Blank copy of drill.md for writing answers | **Generated** by `python3 drills/build.py`. Never edit by hand |
| `guide/exam-tips.md` | Prose gotchas not obvious from the curriculum: *why* something is tricky | Succinct, scannable, headed sections, no exhaustive command catalogs |
| `guide/task-patterns.md` | Recurring exam question shapes, with the idiom for each | Organised by task type |
| `guide/practice-cluster.md` | Local kind cluster setup (ingress, metrics-server, CNI) | Out of scope for exam content |

The readers are other people preparing for the CKAD. Write for them: no repo-maintenance meta
commentary, no references to this skill or to "the user".

## When invoked with a topic

1. **Search** for existing coverage, by English name *and* by the underlying command or flag (for
   "command line ingress", also grep `create ingress` and `--rule=`):
   ```bash
   grep -ni "<keywords>" drills/drill.md guide/*.md scenarios/*.md
   ```
2. **Report what you found** before writing: which files cover it and how thoroughly (a passing
   mention or a dedicated question). Read the context around each hit.
3. **Decide what's missing**, one file at a time:
   - **drill.md**: missing if no dedicated `### N.M` question exercises the command. Add questions
     under the most relevant existing section, using the next free number in that section, placed
     after its current last question (grep the section's headings first). Never renumber existing
     questions. Only create a new top-level section if nothing fits.
   - **exam-tips.md**: only for a genuinely non-obvious gotcha (a surprising default, missing docs,
     behaviour that differs from the naive expectation). A command that works as documented
     belongs in drill.md alone. Extend an existing section where one fits.
   - **Scope**: only what is exercised *on the exam*. Local environment setup goes in
     `guide/practice-cluster.md`. Exam-*terminal* mechanics (vim, tmux, browser-eaten shortcuts)
     belong in exam-tips.md's environment section.
4. **Verify every command** you write, with `--dry-run=client -o yaml`, `kubectl explain` or
   `-h` against a live cluster if one is reachable (the local kind context is `kind-ckad`). If none
   is reachable, say so. Fetch any `kubernetes.io` doc link you aren't sure exists.
5. **Regenerate the workbook** and check it:
   ```bash
   python3 drills/build.py
   diff <(grep -oE '^### [0-9]+\.[0-9]+' drills/drill.md) <(grep -oE '^### [0-9]+\.[0-9]+' drills/workbook.md)
   ```
   The diff must be empty. If it isn't, a drill heading doesn't match the
   `### N.M title — [link](url)` format that `build.py` expects.

## When invoked with no topic: structural audit

- Numbering: `### N.M` ascending within each `## N.` section, no duplicates or gaps.
- Every question has exactly one `<details>` answer block and a working doc link.
- Answers use current syntax (`--dry-run=client`, `patches:` rather than `patchesStrategicMerge:`,
  no removed API versions).
- `python3 drills/build.py` runs, and the diff above is empty.

Fix small, unambiguous problems directly. Report anything judgment-heavy (renumbering, merging
sections) and ask first.

## A stray `## TODO` is a question directed at you

If `drill.md` or a guide file gains a `## TODO`/`## Todo` heading with a `[ ] <question>` line that
you didn't write, that's the user asking something in place. Answer it by editing the content it's
about (the relevant `### N.M` answer or guide section), then remove the heading and its content.
Don't answer only in chat. If the question is unclear, ask. Regenerate the workbook afterwards.
`ckad-scenarios` follows the same convention.

## Notes

- Don't duplicate content between drill.md and the guides. The answer goes in drill.md; the reason
  it's tricky goes in exam-tips.md. Cross-reference in plain text ("see drill 7.10") or with
  relative links, matching whatever the surrounding text already does.
- Several related gotchas surfacing together belong in one exam-tips section with sub-points, not
  in several thin sections.
- Scenarios carry their own `## Documentation` sections (owned by `ckad-scenarios`), which point at
  concept pages. Don't add that format to drill.md, and don't try to keep the two in sync.
