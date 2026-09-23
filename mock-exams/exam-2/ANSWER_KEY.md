# Mock exam 2 — answer key

Don't open this until you've attempted the questions. Each row links to the full write-up in
[`scenarios/`](../../scenarios/).

| Question | Full write-up | Topic | Difficulty |
|---|---|---|---|
| [q01](q01.md) | [scenarios/042](../../scenarios/042-taints-and-tolerations.md) | Taints and tolerations | Medium |
| [q02](q02.md) | [scenarios/037](../../scenarios/037-multi-bug-triage.md) | Debugging two stacked bugs | Hard |
| [q03](q03.md) | [scenarios/045](../../scenarios/045-kubectl-cp-file-transfer.md) | `kubectl cp` | Easy |
| [q04](q04.md) | [scenarios/038](../../scenarios/038-secure-microservice-rbac-networkpolicy.md) | RBAC + NetworkPolicy + Secrets together | Hard |
| [q05](q05.md) | [scenarios/041](../../scenarios/041-statefulset-headless-service.md) | StatefulSet + headless Service | Medium |
| [q06](q06.md) | [scenarios/035](../../scenarios/035-horizontal-pod-autoscaler.md) | HorizontalPodAutoscaler | Medium |
| [q07](q07.md) | [scenarios/047](../../scenarios/047-multi-document-yaml.md) | Multi-document YAML | Easy |
| [q08](q08.md) | [scenarios/040](../../scenarios/040-projected-volume.md) | Projected volumes | Medium |
| [q09](q09.md) | [scenarios/044](../../scenarios/044-rolling-update-strategy-tuning.md) | Rolling-update strategy | Medium |
| [q10](q10.md) | [scenarios/036](../../scenarios/036-poddisruptionbudget.md) | PodDisruptionBudget | Easy |
| [q11](q11.md) | [scenarios/046](../../scenarios/046-job-active-deadline-seconds.md) | `activeDeadlineSeconds` vs `backoffLimit` | Medium |
| [q12](q12.md) | [scenarios/039](../../scenarios/039-ambassador-container-pattern.md) | Ambassador container | Medium |
| [q13](q13.md) | [scenarios/043](../../scenarios/043-limitrange-defaults.md) | LimitRange defaults | Easy |

## Things worth knowing after you've tried them

- **q02** stacks two unrelated bugs: a bad `configMapKeyRef` (`CreateContainerConfigError`), then a
  memory limit too low for the workload (`OOMKilled`). Fixing the first only reveals the second, so
  keep checking after each fix.
- **q04** combines a ServiceAccount, a Role/RoleBinding and a NetworkPolicy, which the first exam
  covers only one at a time.
- **q08**: two sources in a projected volume that share a key name don't cause an error. One value
  is silently dropped, and nothing tells you anything went wrong.
