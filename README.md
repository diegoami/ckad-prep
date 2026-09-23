# Preparing for the CKAD

The notes, drills and practice questions I used to prepare for the
[Certified Kubernetes Application Developer](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/)
exam, which I passed, cleaned up for anyone else going for it.

The CKAD is a hands-on exam: about two hours in a terminal, solving 15–20 tasks on live clusters with
only the official docs to help. Reading doesn't prepare you for that; typing does. So nearly
everything here is meant to be *run* against a local cluster, not just read.

## What's in here

| | What | Use it for |
|---|---|---|
| 📘 | [guide/exam-tips.md](guide/exam-tips.md) | The exam environment, shell/vim/tmux setup, what the curriculum really expects, and the gotchas that cost the most time |
| 🧩 | [guide/task-patterns.md](guide/task-patterns.md) | The question *shapes* that keep recurring (blue/green, ConfigMap injection, PV/PVC, RBAC, probes, ...) with the idiom for each |
| 🛠️ | [guide/practice-cluster.md](guide/practice-cluster.md) | Setting up a local kind cluster with ingress, metrics-server and NetworkPolicy support |
| 🔁 | [drills/drill.md](drills/drill.md) | 170+ short questions with collapsible answers and a docs link each, for building muscle memory |
| ✍️ | [drills/workbook.md](drills/workbook.md) | The same questions without answers, for writing your own |
| 🧪 | [scenarios/](scenarios/) | 88 exam-style tasks, each with a Setup that creates the starting state, a worked Solution and a Cleanup |
| ⏱️ | [mock-exams/](mock-exams/) | Two blind, timed practice exams (22 + 13 questions) drawn from the scenarios |
| 📄 | [examples/](examples/) | Small working manifests (Jobs, Pods, Ingress, 12 Kustomize setups) to apply and poke at |
| 🔗 | [guide/resources.md](guide/resources.md) | Official links, simulators, question banks and write-ups from people who passed |

## A suggested path

1. **Read the [curriculum](https://github.com/cncf/curriculum)** and the first two sections of
   [exam-tips.md](guide/exam-tips.md). Know what's in scope and what the exam environment is like.
2. **Build the [practice cluster](guide/practice-cluster.md).** You'll use it for everything else.
3. **Drill every day.** Work through [drill.md](drills/drill.md) a section at a time: try each
   question, then expand the answer. Mark misses with an `x` and redo them the next day. Aim for
   speed: the imperative commands (`kubectl run/create/expose/set ... $do`) should become automatic.
4. **Do the [scenarios](scenarios/) for each curriculum domain.** Run the Setup, solve the Task
   without looking, then compare with the Solution. The explanations under each Solution are where
   most of the learning is.
5. **Read [task-patterns.md](guide/task-patterns.md)** once you've seen most question types. It's
   the pattern-recognition layer.
6. **Take the [mock exams](mock-exams/) under time pressure**, and your two included
   [killer.sh](https://killer.sh) sessions: one about two weeks out, one in the last few days.
   killer.sh is harder than the real exam, so don't panic about the score.
7. **Before exam day**, reread the environment section of [exam-tips.md](guide/exam-tips.md) and
   practise finding pages on kubernetes.io/docs without a search engine.

## Where each curriculum domain is practised

| Domain (weight) | Curriculum topics | Drill sections | Scenarios |
|---|---|---|---|
| Application Design and Build (20%) | images, workload types, multi-container Pods, volumes | 4, 5, 9, 12, 16 | [see index](scenarios/README.md#application-design-and-build-20) |
| Application Deployment (20%) | Deployments and rolling updates, blue/green and canary, Helm, Kustomize | 3, 10, 17 | [see index](scenarios/README.md#application-deployment-20) |
| Application Observability and Maintenance (15%) | API deprecations, probes, monitoring, logs, debugging | 7, 14 | [see index](scenarios/README.md#application-observability-and-maintenance-15) |
| Application Environment, Configuration and Security (25%) | CRDs and Operators, RBAC, requests/limits/quotas, ConfigMaps, Secrets, ServiceAccounts, SecurityContexts | 6, 11, 15 | [see index](scenarios/README.md#application-environment-configuration-and-security-25) |
| Services and Networking (20%) | NetworkPolicies, Services, Ingress | 8 | [see index](scenarios/README.md#services-and-networking-20) |
| General kubectl fluency | output formatting, labels, `run` flags | 1, 2, 18, 19 | [see index](scenarios/README.md#core-kubectl) |

## The ten things I'd tell a friend

1. **Switch context and namespace at the start of every question.** Doing a question right in the
   wrong place scores zero.
2. **Generate YAML; don't write it.** `k create deploy x --image=nginx $do > x.yaml` (with
   `export do="--dry-run=client -o yaml"`), then edit.
3. **`kubectl explain pod.spec.containers.securityContext`** answers "where does this field go?"
   faster than the docs.
4. **Skip and flag.** A 4% question you're stuck on isn't worth the three easy ones after it.
5. **Verify every answer**: `get`, `describe`, a quick `curl` from a temporary pod. It takes seconds
   and catches typos.
6. **Write output files to the exact path asked for.** Grading reads the file.
7. **Egress NetworkPolicies need a DNS rule** (port 53), or everything "mysteriously" breaks.
8. **Empty `kubectl get endpoints` means the Service selector doesn't match the pod labels.** That's
   the number one Services bug.
9. **Most of a running Pod is immutable.** Export it, edit, then `kubectl replace --force -f`.
10. **Set up vim for YAML** (`expandtab`, `shiftwidth=2`) before the first question that needs it.

The reasoning behind each of these is in [exam-tips.md](guide/exam-tips.md).

## Repository layout

```
guide/          exam tips, task patterns, practice-cluster setup, resources
drills/         drill.md (Q&A deck), workbook.md (generated blank copy), build.py
scenarios/      88 runnable exam-style tasks, indexed by curriculum domain
mock-exams/     exam-1/ and exam-2/: blind question files plus answer keys
examples/       small manifests: pods, jobs, deployments, ingress, kustomize
.claude/skills/ Claude Code skills used to author and verify the drills and scenarios
```

After editing `drills/drill.md`, regenerate the workbook with `python3 drills/build.py`, or with
`python3 drills/build.py --html` to also get printable HTML versions (needs `pip install markdown`).

## Tested with

Scenarios were verified end to end on kind v0.23 (Kubernetes v1.30), and many also on kind v0.31
(Kubernetes v1.35), with kubectl v1.36. The exam
tracks recent Kubernetes releases, so check the current version on the
[curriculum page](https://github.com/cncf/curriculum) and use a matching kind node image if you
can. Where kind behaves differently from a real exam cluster (NetworkPolicy enforcement, the default
StorageClass, metrics-server), the scenario says so.

Found a mistake? Issues and pull requests are welcome.

## License

[MIT](LICENSE). Links in [guide/resources.md](guide/resources.md) point to third-party material under
its own terms.
