---
name: ckad-scenarios
description: Author, verify, or extend runnable CKAD practice scenarios under scenarios/ (Task + Setup + Solution + Cleanup that actually run on the local kind cluster), and keep their blind copies in mock-exams/ in sync. Use when the user wants to turn a question they pasted or a guide/task-patterns.md pattern into a scenario, add a new scenario, or review/fix an existing one. Distinct from the exam-coverage skill, which governs drills/drill.md and the guide/ prose.
---

# CKAD scenarios authoring

`scenarios/` holds runnable, exam-style practice tasks. They're different from isolated command
drills (`drills/drill.md`, owned by the `exam-coverage` skill) and from the pattern reference
(`guide/task-patterns.md`). Read `scenarios/README.md` first: it's the format spec this skill
enforces. This file is about the *process*.

The readers are other people preparing for the CKAD. Write for them: no repo-maintenance meta
commentary, no references to this skill or to "the user".

## The core problem

Exam questions describe a starting state that already exists ("there is a Deployment named X").
Locally, nothing exists until you create it. A scenario is only useful if someone can paste its
commands into a terminal and have them work end to end, so every scenario needs an accurate
**Setup** that builds the "given" state, not just a Solution.

## Adding a scenario

1. **Never copy question text from paid or proprietary sources** (killer.sh, KodeKloud, Udemy
   courses, exam dumps). If the idea comes from one, write an original task that tests the same
   skill: your own framing, your own names, your own wording. Candidates are also bound by the exam
   NDA, so never reproduce real exam questions.
2. **Design the Setup before the Solution.** Build exactly what the Task claims already exists,
   nothing more. Don't let the Setup do part of the task.
3. **Verify every command against the live cluster** (context `kind-ckad`, node
   `ckad-control-plane`; see `guide/practice-cluster.md`): Setup, then Solution, then Cleanup,
   exactly as written. Things that were silently wrong until checked live:
   - kind ships a default StorageClass, so a PVC without `storageClassName` binds to it instead of
     waiting for your PV. Use `storageClassName: ""` on both sides.
   - Since 1.24, ServiceAccounts don't get a long-lived token Secret automatically.
   - kindnet on kind v0.23 doesn't enforce NetworkPolicy (v0.31 does). Take a before/after baseline.
   - `items[0]` right after a rollout can pick a terminating Pod. Select the newest Pod or filter
     by phase.

   If no cluster is reachable, say so, and base the commands on an already verified pattern
   elsewhere in the repo.
4. **Never hardcode a guessed resource name** (ReplicaSet hashes, generated Pod names). Look it up
   in the step before, e.g.
   `POD=$(kubectl get pods -l app=X -o jsonpath='{.items[0].metadata.name}')`, so everything can be
   pasted end to end.
5. **Say so when something surprises you.** If verified behaviour contradicts the naive
   expectation, explain it in the Solution. If it's about the exam in general rather than kind,
   mention it to the user as a candidate for `guide/exam-tips.md` (owned by `exam-coverage`).
6. **Flag environment mismatches clearly.** A scenario that "works" without testing what it claims
   (a NetworkPolicy on a CNI that doesn't enforce it) is worse than one that errors. Put the note in
   the Setup or Solution, and point to `guide/practice-cluster.md`.
7. **Use writable local paths** (`~/ckad/NNN/`) instead of exam-style `/opt/...` paths that need
   root, and public or local substitutes (a Bitnami chart, a local `registry:2`) for resources that
   don't exist locally.
8. **Numbering** is sequential across the collection: take the next free `NNN`. Promote a scenario
   to a directory (`NNN-slug/README.md` plus helper files) only when it really needs supporting
   files.
9. **Update `scenarios/README.md`'s index** under the right curriculum domain, and set the
   `**Domain:** ... · **Difficulty:** ...` line to match.
10. **Every scenario has a `## Documentation` section** between Task and Setup: where to look the
    task up in the exam's allowed docs (kubernetes.io, or helm.sh for Helm scenarios), not a
    restatement of the solution. **Never construct a doc URL from memory.** Fetch it and confirm the
    page covers what you cite it for. Leave this section out of the blind copies in `mock-exams/`,
    because it names the topic.
11. **Note faster alternatives where they really exist**, especially `kubectl edit` vs
    `kubectl patch`. Solutions often use `patch` because it runs non-interactively, but a person on
    the exam often wants `edit`. After the relevant `patch`, say which case applies:
    - **Faster, and avoids a trap** (strategic-merge vs JSON-merge semantics, `"field": null` to
      clear a map key): say so explicitly, since `edit` saves the whole object instead of a diff.
      See 051, 053, 061, 063.
    - **Just faster to type**, with no trap: a one-line "faster by hand: `kubectl edit ...`".
    - **No help at all**, for immutable fields (071, 088): say that `edit` hits the same rejection.
    Prefer a dedicated command (`kubectl set image|env|serviceaccount`, `scale`, `label`) over a
    raw patch when one exists, and mention it even if the Solution keeps the patch (see 084).
12. **A stray `## TODO` section in a scenario is the user asking a question in place.** Answer it by
    editing the content it's about, as a short aside in Setup or Solution, then remove the heading
    and its content. If the question is cut off or unclear, ask instead of guessing. Afterwards,
    check that the number of code-fence lines is even; inserting prose inside a fenced block is an
    easy way to break it.
13. **Add a verification footer** (`*Verified end-to-end on a local kind cluster (Kubernetes vX.Y)
    on YYYY-MM-DD.*`) only after a successful live run.

## Blind copies in mock-exams/

Each `mock-exams/exam-N/qNN.md` is a stripped copy of one scenario: the same `## Task` and
`## Setup`, with no Solution and no hints (including hints hidden in Setup comments). The mapping is
in each exam's `ANSWER_KEY.md`. Whenever you change a scenario's Task or Setup, update its blind copy
as well.

## Reviewing existing scenarios

Spot-check them against the live cluster rather than assuming they still work. Kubernetes and kind
upgrades change defaults. A scenario whose Setup no longer produces the state its Task assumes is a
bug; fix it.

## Relationship to other skills and files

- `exam-coverage` owns `drills/drill.md` (and the generated `drills/workbook.md`) and
  `guide/exam-tips.md`. If writing a scenario reveals a drill gap or a gotcha, mention it rather
  than editing those files from this skill.
- `guide/task-patterns.md` is a pattern reference, and it's fine to use as inspiration for new
  scenarios.
- A scenario's Documentation section points at the *concept* page; a drill.md link points at the
  specific command reference. They're independent; one doesn't need to mirror the other.
