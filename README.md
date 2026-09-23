# Preparing for the CKAD

I passed the [Certified Kubernetes Application Developer](https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/)
exam, and this repository is what I prepared with: my notes, a deck of kubectl drills, and a
lot of practice tasks that run on a local cluster. I've tidied it up in the hope it helps you
too.

The CKAD is entirely hands-on. You get about two hours for 15–20 tasks, each on its own VM that you
`ssh` into, with the Kubernetes docs open in a browser. Knowing things isn't enough; you need to
type them quickly and know where to look when you don't. That's why nearly everything here is
meant to be run, not just read.

All tasks, scenarios and examples here are written in my own words, with their own names and
framing. Many are modelled on the *kinds* of task that come up in practice exams, but none reproduce
real exam questions (the exam is under NDA) or questions from paid simulators and courses. Those
are linked in [resources](guide/resources.md), not copied.

## Learn `kubectl -h` and `kubectl explain`

If you take one thing from this repository, make it this. In the exam I had two terminals side by
side: one for running commands, the other for checking files and reading the documentation that's
built into kubectl. That second terminal answered most of my questions faster than the browser
could.

`kubectl <command> -h` tells you **how to create something**. Every help page has an Examples
section with complete commands you can adapt:

```bash
kubectl create -h                    # everything `create` can make: deployment, job, cronjob, ingress, role, secret, ...
kubectl create ingress -h            # the exact --rule="host/path=svc:port" syntax, with examples
kubectl create job -h                # including --from=cronjob/<name>
kubectl run -h                       # pods: there's no `kubectl create pod`
kubectl run -h | grep -A1 -E '^\s+--restart'   # jump to one flag
```

`kubectl explain` tells you **where a field goes**, straight from the cluster's own API schema:

```bash
kubectl explain pod.spec.containers.securityContext        # any depth, in one go
kubectl explain deploy.spec.strategy --recursive           # the whole shape at a glance
kubectl explain pod.spec --recursive | grep -i toleration  # "where did that field live again?"
```

Practise with these instead of the browser until it's a habit. There's a full guide with more
examples in [guide/kubectl-help.md](guide/kubectl-help.md).

## What's in here

- **[guide/](guide/)**: the written part.
  - [kubectl-help.md](guide/kubectl-help.md): the section above, in depth.
  - [exam-tips.md](guide/exam-tips.md): my exam setup, how to read the curriculum, and the gotchas
    that cost the most time.
  - [task-patterns.md](guide/task-patterns.md): the question shapes that keep coming back, with the
    idiom for each.
  - [practice-cluster.md](guide/practice-cluster.md): setting up the local kind cluster everything
    else runs on.
  - [resources.md](guide/resources.md): simulators, question banks, and write-ups from people who
    passed.
- **[drills/drill.md](drills/drill.md)**: 176 short questions, each with a collapsible answer and a
  docs link, for building speed. [workbook.md](drills/workbook.md) has the same questions with space
  for your own answers.
- **[scenarios/](scenarios/)**: 88 exam-style tasks. Each has a Setup that builds the starting
  state on your cluster, the Task, a worked Solution with the traps explained, and a Cleanup.
- **[mock-exams/](mock-exams/)**: two blind practice exams (22 and 13 questions) drawn from the
  scenarios, to take under time pressure.
- **[examples/](examples/)**: small manifests to apply and poke at: Jobs, Pods, a StatefulSet,
  NetworkPolicies, Ingress, and 12 Kustomize setups.

## How I'd use it

1. Read the [curriculum](https://github.com/cncf/curriculum) and
   [exam-tips.md](guide/exam-tips.md), so you know what's in scope.
2. Build the [practice cluster](guide/practice-cluster.md), and get to know your tools: the
   terminal (I practised in an Xfce desktop on Ubuntu) and vim, especially indenting and
   unindenting. See [my exam setup](guide/exam-tips.md#my-exam-setup).
3. Work through the [drills](drills/drill.md) a section at a time. Try each question before
   opening the answer, and redo the ones you missed the next day.
4. Do the [scenarios](scenarios/), a curriculum domain at a time. Solve each Task before reading
   the Solution. The explanations underneath are where most of the learning is.
5. Take the [mock exams](mock-exams/) against the clock. If you buy the exam through the Linux
   Foundation you also get two days in a practice environment beforehand. Use it to get used to
   the desktop, but know that the real exam is slower and puts every question on its own VM (see
   [my exam setup](guide/exam-tips.md#my-exam-setup)).

Throughout, reach for `kubectl -h` and `kubectl explain` before the browser.

## Where each curriculum domain is practised

| Domain (weight) | Curriculum topics | Drill sections | Scenarios |
|---|---|---|---|
| Application Design and Build (20%) | images, workload types, multi-container Pods, volumes | 4, 5, 9, 12, 16 | [see index](scenarios/README.md#application-design-and-build-20) |
| Application Deployment (20%) | Deployments and rolling updates, blue/green and canary, Helm, Kustomize | 3, 10, 17 | [see index](scenarios/README.md#application-deployment-20) |
| Application Observability and Maintenance (15%) | API deprecations, probes, monitoring, logs, debugging | 7, 14 | [see index](scenarios/README.md#application-observability-and-maintenance-15) |
| Application Environment, Configuration and Security (25%) | CRDs and Operators, RBAC, requests/limits/quotas, ConfigMaps, Secrets, ServiceAccounts, SecurityContexts | 6, 11, 15 | [see index](scenarios/README.md#application-environment-configuration-and-security-25) |
| Services and Networking (20%) | NetworkPolicies, Services, Ingress | 8 | [see index](scenarios/README.md#services-and-networking-20) |
| General kubectl fluency | output formatting, labels, `run` flags | 1, 2, 18, 19 | [see index](scenarios/README.md#core-kubectl) |

## Tested with

The scenarios were verified end to end on kind v0.23 (Kubernetes v1.30), many of them also on
kind v0.31 (Kubernetes v1.35), with kubectl v1.36. The exam tracks recent Kubernetes releases, so
check the current version on the [curriculum page](https://github.com/cncf/curriculum) and use a
matching kind node image if you can. Where kind behaves differently from a real exam cluster
(NetworkPolicy enforcement, the default StorageClass, metrics-server), the scenario says so.

After editing `drills/drill.md`, regenerate the workbook with `python3 drills/build.py` (add
`--html` for printable HTML versions; needs `pip install markdown`). The `.claude/skills/` folder
holds the Claude Code skills I used to write and check the drills and scenarios.

Found a mistake? Issues and pull requests are welcome.

## License

[MIT](LICENSE). Links in [guide/resources.md](guide/resources.md) point to third-party material under
its own terms.
