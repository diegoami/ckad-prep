# Mock exams

Two blind practice exams. Each question file has only a **Task** and a **Setup**: no solution, no
hints, and a file name that doesn't give away the topic. It's the closest this repo gets to exam
conditions.

| Exam | Questions | Style |
|---|---|---|
| [exam-1](exam-1/) | 22 | One skill per question, spread across all five curriculum domains. Start here. |
| [exam-2](exam-2/) | 13 | Harder: compound questions, a two-bug debugging task, and topics exam 1 doesn't touch (HPA, PDB, StatefulSet, taints, LimitRange, projected volumes, ambassador containers). |

## How to take one

1. Set up the practice cluster from [../guide/practice-cluster.md](../guide/practice-cluster.md),
   including ingress-nginx and metrics-server.
2. Set a timer. The real exam gives you 2 hours for roughly 15–20 questions, so about 6–7 minutes
   per question.
3. For each question, run its **Setup** block first. That creates the starting state a real exam
   would already have waiting for you. Then solve the **Task**.
4. Skip anything that's taking too long and come back to it. That's the most important exam skill.
5. Grade yourself with `ANSWER_KEY.md`. It links each question to its full write-up in
   [../scenarios/](../scenarios/).
6. Clean up between questions (`kubectl delete ns <namespace>`) so leftover state doesn't leak
   into the next one.

Every question is a stripped copy of a scenario, so don't read `scenarios/` first if you want to
take these blind.
