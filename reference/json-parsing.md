# Reference: Understanding & Parsing JSON

JSON (JavaScript Object Notation) is the wire format Kubernetes actually
speaks — YAML manifests get converted to JSON before they're sent to the
API server, and `kubectl get ... -o json` is one of the most useful
troubleshooting tools you have. This doc covers reading JSON, and parsing
it from the command line with `jq` and `kubectl -o jsonpath`.

---

## 1. JSON Syntax, Minimal and Complete

JSON has exactly six building blocks:

| Type | Example | Notes |
|---|---|---|
| Object | `{"name": "web", "port": 80}` | Unordered key/value pairs, keys are always double-quoted strings |
| Array | `["a", "b", "c"]` | Ordered list, items can be any type, including mixed |
| String | `"hello"` | Double quotes only — single quotes are invalid JSON |
| Number | `42`, `3.14`, `-7` | No quotes; no distinction between int/float in the format itself |
| Boolean | `true`, `false` | Lowercase, no quotes |
| Null | `null` | Lowercase, no quotes |

A few rules that trip people up:

- **No trailing commas.** `{"a": 1, "b": 2,}` is invalid JSON — the comma after `2` breaks it.
- **No comments.** JSON has no `//` or `#` — if you need comments, you're probably looking for YAML.
- **Keys must be double-quoted strings.** `{name: "web"}` is invalid; it must be `{"name": "web"}`.
- **Whitespace outside strings doesn't matter.** JSON can be minified to one line or pretty-printed; both are the same data.

### Example: a Kubernetes object as JSON

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "web",
    "labels": {
      "app": "web",
      "tier": "frontend"
    }
  },
  "spec": {
    "containers": [
      {
        "name": "nginx",
        "image": "nginx:1.27",
        "ports": [
          { "containerPort": 80 }
        ]
      }
    ]
  }
}
```

Reading this as nested structures:
- The whole thing is an **object**.
- `metadata` is a nested **object**.
- `metadata.labels` is another nested **object** (key/value pairs, not a list).
- `spec.containers` is an **array** of objects — even with one container, it's still a list, because a Pod can have more than one.
- `spec.containers[0].ports` is an array nested inside an array item.

This nesting — object → array → object → array — is exactly what you'll
navigate when parsing real `kubectl -o json` output.

---

## 2. Why This Matters for Kubernetes Support Work

```bash
kubectl get pod web -o json
```

returns the full object as JSON — same shape as the YAML you'd write, just
JSON-encoded. Anything you can read in `kubectl describe` or YAML, you can
extract precisely and scriptably from JSON. This matters when you need to:

- Pull one field out of a large object without scrolling through `describe` output
- Feed exact values into another command (an IP, a node name, a container's restart count)
- Compare the *actual* state of an object against what you expect

---

## 3. Parsing JSON with `jq`

`jq` is a command-line JSON processor — install via your package manager
(`apt install jq`, `brew install jq`, etc.).

### Basic filtering

```bash
kubectl get pod web -o json | jq '.metadata.name'
# "web"

kubectl get pod web -o json | jq '.spec.containers[0].image'
# "nginx:1.27"

kubectl get pod web -o json | jq '.metadata.labels'
# { "app": "web", "tier": "frontend" }
```

- `.` means "the whole document."
- `.metadata.name` walks down object keys.
- `[0]` indexes into an array — `[0]` is the first item, `[1]` the second, and so on.

### Iterating over arrays

```bash
kubectl get pods -o json | jq '.items[].metadata.name'
```

`.items[]` (note the empty `[]`, no index) means "for each item in this
array" — it streams every pod name, one per line, instead of pulling a
single index.

### Filtering with conditions

```bash
kubectl get pods -o json | jq '.items[] | select(.status.phase != "Running") | .metadata.name'
```

Reads as: for each pod, keep it only if its phase isn't `Running`, then
print its name. This is the single most useful one-liner for "show me
what's broken."

### Building a custom table

```bash
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name)\t\(.status.phase)"'
```

`-r` outputs raw strings (no surrounding quotes); `\(...)` interpolates a
value into the string — this prints a quick `name<TAB>phase` table without
writing a script.

### Counting

```bash
kubectl get pods -o json | jq '.items | length'
```

`length` on an array returns its item count — useful for "how many pods are
actually running in this namespace right now," scriptable in an alert.

---

## 4. Parsing with `kubectl -o jsonpath` (no extra tool required)

JSONPath is built into `kubectl` directly — no `jq` install needed, but the
syntax is less forgiving.

```bash
kubectl get pod web -o jsonpath='{.metadata.name}'
# web

kubectl get pod web -o jsonpath='{.spec.containers[0].image}'
# nginx:1.27

kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
# web
# api
# redis
```

- `{...}` wraps every JSONPath expression.
- `.items[*]` means "every item" (the JSONPath equivalent of `jq`'s `[]`).
- `{range ...}{end}` is JSONPath's loop construct — needed because plain `{.items[*].metadata.name}` would print all names on one line with no separator.

**Rule of thumb:** reach for `jsonpath` for a single quick field on a
single command; reach for `jq` the moment you need filtering, math, or
anything beyond pulling one value.

---

## 5. Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| `jq: error: Cannot index array with string` | Treated an array (`[...]`) like an object (`{...}`) | Add `[]` or `[0]` before the field you're accessing |
| `parse error: Invalid numeric literal` | Piped non-JSON (e.g. `kubectl get pod` table output) into `jq` | Always pass `-o json` to the `kubectl` command first |
| Output is empty / `null` | Field path doesn't exist on this object, or a typo in the field name | Run `... -o json | jq .` with no filter first, and read the actual structure before assuming the path |
| `jsonpath` returns nothing for an array field | Used `.items[0]` syntax incorrectly, or missing `{range}{end}` for multiple items | Use `{range .items[*]}...{end}` for "for each", index `[0]` only for a single specific item |

---

## 6. Quick Reference

```bash
# Pretty-print full JSON
kubectl get pod web -o json | jq .

# Get one field
kubectl get pod web -o json | jq '.status.podIP'

# List a field across all items
kubectl get pods -o json | jq -r '.items[].metadata.name'

# Filter then extract
kubectl get pods -o json | jq -r '.items[] | select(.status.phase=="Pending") | .metadata.name'

# Same single-field lookup, no jq required
kubectl get pod web -o jsonpath='{.status.podIP}'
```
