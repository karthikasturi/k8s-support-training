# Reference: YAML Syntax for Kubernetes Manifests

Every Kubernetes manifest you'll write or troubleshoot is YAML. Most YAML
bugs in support tickets aren't Kubernetes problems at all — they're
indentation or quoting mistakes. This doc covers the syntax rules that
actually matter for writing and reading manifests.

---

## 1. The One Rule That Matters Most: Indentation

YAML uses **indentation with spaces** to represent nesting — never tabs.

```yaml
spec:
  containers:
    - name: nginx
      image: nginx:1.27
```

- `containers` is nested one level under `spec` (2 spaces in).
- The list item (`- name: nginx`) is nested one level under `containers`.
- `image` lines up with `name` — they're sibling keys of the *same* list item.

**Tabs are invalid YAML.** Most editors and `kubectl apply` will reject a
tab character outright, or worse, accept it inconsistently depending on
your editor's tab-width setting. Configure your editor to insert spaces
when you press Tab in any `.yaml` file.

The number of spaces per level (2 is conventional) doesn't matter as long
as it's **consistent within the same nesting level**. Mixing 2 spaces for
one sibling and 4 for another at the same level is a parse error.

---

## 2. Key-Value Pairs (Maps)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: default
```

- `key: value` — note the **space after the colon**. `name:web` (no space) is parsed as a single string, not a key-value pair.
- Nested keys (like `name` and `namespace` under `metadata`) are indented under their parent.
- This maps directly to a JSON object — `metadata:` with nested keys is the YAML form of `"metadata": { "name": "web", "namespace": "default" }`.

---

## 3. Lists (Sequences)

Two valid forms — you'll see both in Kubernetes docs:

**Block form** (most common in manifests):
```yaml
containers:
  - name: nginx
    image: nginx:1.27
  - name: sidecar
    image: busybox:1.36
```

**Flow form** (compact, used for short lists):
```yaml
command: ["sh", "-c", "echo hello"]
```

- Each `-` starts a new list item.
- A list item can itself be a map (`name:`/`image:` pairs above) — that's how `containers` ends up as a list of objects, matching `spec.containers[]` in the JSON/API schema.
- Indentation of the `-` matters: it must line up with its sibling list items, and be indented relative to the parent key (`containers:`).

**Common mistake:**
```yaml
containers:
- name: nginx          # this is valid — '-' can sit at the same indent as the key
  image: nginx:1.27
```
Both of these indentation styles are valid YAML, but **don't mix them
within the same file** — pick one and be consistent, since reviewers and
diffs get harder to read otherwise.

---

## 4. Strings, Numbers, and the Quoting Trap

```yaml
name: web              # unquoted string — fine
port: 80                # number, no quotes
version: "1.27"          # quoted string — needed because it looks like a number
enabled: true            # boolean
```

YAML tries to guess types for unquoted scalars. This causes real bugs:

| Value you meant | What you wrote | What YAML parses it as | Fix |
|---|---|---|---|
| The string `"1.27"` | `version: 1.27` | A float `1.27` | `version: "1.27"` |
| The string `"yes"` | `enabled: yes` | Boolean `true` (YAML 1.1 quirk) | `enabled: "yes"` if you mean the literal string |
| The string `"0123456"` (e.g. an account ID with a leading zero) | `accountId: 0123456` | Parsed as **octal**, giving a completely different number | `accountId: "0123456"` |
| A null/empty field | `value:` (nothing after the colon) | `null` | `value: ""` if you mean an empty string |

**Rule of thumb:** if a value could be mistaken for a number, boolean, or
null, quote it explicitly with `"..."`. Kubernetes-specific case: container
ports and replica counts should stay unquoted numbers (the API schema
expects an integer); environment variable values and labels that happen to
look numeric should usually be quoted strings.

---

## 5. Multi-line Strings

You'll hit this writing ConfigMaps, scripts in `command`/`args`, or
multi-line annotations.

```yaml
data:
  config.txt: |
    line one
    line two
    trailing newline preserved
```

- `|` (literal block) keeps line breaks exactly as written, plus a trailing newline.
- `>` (folded block) joins lines with a single space instead of a newline — used for prose, not config files or scripts.

```yaml
data:
  message: >
    This will all be
    joined onto one line.
```

→ becomes `"This will all be joined onto one line.\n"`

**For ConfigMaps/Secrets holding actual files (configs, scripts), always
use `|`** — `>` will silently mangle a script's line breaks.

---

## 6. Comments

```yaml
# This is a comment — only YAML supports this, JSON does not
replicas: 3  # inline comments are also valid
```

Comments start with `#` and run to the end of the line. They're stripped
before the YAML is converted to JSON and sent to the API server — `kubectl
get ... -o yaml` on a live object will never show your original comments.

---

## 7. Multiple Documents in One File

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
```

`---` separates independent YAML documents in the same file. `kubectl
apply -f file.yaml` applies every document in the file, in order — this is
how you bundle a Service + Deployment + ConfigMap into one manifest file.

---

## 8. Anchors and Aliases (less common, worth recognizing)

```yaml
defaults: &defaults
  image: nginx:1.27
  imagePullPolicy: IfNotPresent

containerA:
  <<: *defaults
  name: a

containerB:
  <<: *defaults
  name: b
```

- `&defaults` defines a reusable block (an **anchor**).
- `*defaults` re-uses it (an **alias**).
- `<<:` merges the anchor's keys into the current map.

You won't write this often in plain Kubernetes manifests, but you'll see
it in Helm chart templates and CI pipeline YAML — worth recognizing even
if you don't author it yourself.

---

## 9. Common Pitfalls Checklist

| Symptom | Cause | Fix |
|---|---|---|
| `error converting YAML to JSON: yaml: line N: found character that cannot start any token` | A tab character, or inconsistent indentation | Check for tabs; re-indent with consistent spaces |
| A numeric-looking field rejected, or behaving like the wrong type | Unquoted value YAML auto-typed (e.g. `1.27` as float, `yes` as boolean) | Quote it: `"1.27"`, `"yes"` |
| Multi-line script/config loses its line breaks | Used `>` instead of `|` | Switch to `|` for anything that must keep exact line breaks |
| `kubectl apply` only applies the last object in a multi-doc file | Missing or malformed `---` separator between documents | Ensure each document starts with `---` on its own line |
| "mapping values are not allowed in this context" | Missing the space after a colon (`key:value`), or a colon inside an unquoted string value | Add the space after `:`; quote values containing a literal `:` |

---

## 10. Quick Reference

```yaml
# Map (object)
metadata:
  name: web
  namespace: default

# List (array) of scalars
command: ["sh", "-c", "sleep 3600"]

# List of maps
containers:
  - name: nginx
    image: nginx:1.27

# Quoted string that looks like another type
version: "1.27"

# Literal multi-line block (preserves newlines)
script: |
  #!/bin/sh
  echo "hello"

# Multiple documents in one file
---
apiVersion: v1
kind: Service
---
apiVersion: apps/v1
kind: Deployment
```
