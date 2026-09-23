# yq vs JSONPath

`yq` isn't in the curriculum and its docs aren't allowed in the exam (see
[exam-tips.md](exam-tips.md#yq-available-but-dont-depend-on-it)). If you use it anyway, this maps the
common jsonpath queries to yq.

kubectl's `-o jsonpath` is applied by kubectl itself (client-side) to the object the API server returned.
`yq` doesn't hook into kubectl's output flag — you get full output (yaml or json) and pipe it into `yq`, which then filters using its own (jq-like) expression syntax.

## Example

jsonpath:
```
kubectl get pod pod1 -o jsonpath='{.status.phase}'
```

yq equivalent:
```
kubectl get pod pod1 -o yaml | yq '.status.phase'
```
(or `-o json | yq '.status.phase'` — yq reads both)

## Cheatsheet: jsonpath -> yq

| jsonpath | yq |
|---|---|
| `{.status.phase}` | `.status.phase` |
| `{.items[0].metadata.name}` | `.items[0].metadata.name` |
| `{.items[*].metadata.name}` | `.items[].metadata.name` |
| `{.spec.containers[0].image}` | `.spec.containers[0].image` |
| `{range .items[*]}{.metadata.name}{"\n"}{end}` | `.items[].metadata.name` |
| `{.items[?(@.status.phase=="Running")].metadata.name}` | `.items[] \| select(.status.phase=="Running") \| .metadata.name` |

Key difference: jsonpath is a single template string with `{}` blocks; yq expressions read closer to jq (dot-paths, `select()`, pipes `|`).

## Writing: add resources/limits with yq

jsonpath is read-only (query only). yq can also mutate files in place with `-i` — useful for adding `resources.requests`/`resources.limits` to a container without hand-editing YAML.

Set one field at a time:
```
yq -i '.spec.containers[0].resources.requests.cpu = "250m"' pod.yaml
yq -i '.spec.containers[0].resources.requests.memory = "64Mi"' pod.yaml
yq -i '.spec.containers[0].resources.limits.cpu = "500m"' pod.yaml
yq -i '.spec.containers[0].resources.limits.memory = "128Mi"' pod.yaml
```

Chain them into one call with `|`:
```
yq -i '
  .spec.containers[0].resources.requests.cpu = "250m" |
  .spec.containers[0].resources.requests.memory = "64Mi" |
  .spec.containers[0].resources.limits.cpu = "500m" |
  .spec.containers[0].resources.limits.memory = "128Mi"
' pod.yaml
```

Or set the whole `resources` block at once with an inline map:
```
yq -i '.spec.containers[0].resources = {"requests": {"cpu": "250m", "memory": "64Mi"}, "limits": {"cpu": "500m", "memory": "128Mi"}}' pod.yaml
```

Notes:
- `-i` edits the file in place (like `sed -i`); drop it to print to stdout instead and check the result first.
- If there are multiple containers, target by name instead of index: `(.spec.containers[] | select(.name == "app")).resources...`
- CPU/memory values still need quotes ("250m", "64Mi") since yq would otherwise try to parse them as numbers.
