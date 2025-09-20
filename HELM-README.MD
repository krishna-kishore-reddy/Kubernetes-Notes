# Helm — Complete Cheat Sheet (Day 1 → Day 3.5)

> A single-file reference covering everything we've learned so far: quickstart commands, chart anatomy, values, templating (helpers, conditionals, loops), advanced templating utilities (`tpl`, `with`, `toYaml`, `lookup`), and repo/chart exploration. Keep this as your day-to-day `.md` reference.

---

## Preface

* These notes assume **Helm 3** (Helm 2/Tiller is deprecated). Helm 3 uses `apiVersion: v2` in charts.
* Use `helm version` to check your client version.

---

# Day-1: Quickstart & Core Commands

### Quick setup (local cluster)

* Use `minikube`, `kind`, or any Kubernetes cluster to practice.
* Install Helm CLI from the official docs for your OS.

### Common Helm commands (fast list)

```bash
helm create <name>            # scaffold a new chart
helm install <release> <chart>   # install the chart into cluster
helm upgrade <release> <chart>   # upgrade an installed release
helm uninstall <release>     # remove a release
helm template <release|chart>    # render manifests locally without installing
helm lint <chart>            # validate chart format and best practices
helm package <chart>         # package chart into .tgz
helm repo add <name> <url>   # add a chart repository
helm repo update             # update local repo index
helm repo list               # list added repos
helm search repo <term>      # search charts only in added repos
helm search hub <term>       # search Artifact Hub (global)
helm pull <repo/chart>       # download a packaged chart (.tgz)
helm show chart <repo/chart> # show Chart.yaml for a repo chart
helm show values <repo/chart> # show default values.yaml for a repo chart
helm show all <repo/chart>   # show chart/chart templates/values etc
helm get values <release>    # show values used for an installed release
helm get manifest <release>  # get rendered manifests from release
helm history <release>       # release revision history
helm status <release>        # status of release
helm rollback <release> <rev># rollback to a previous revision
```

### Common flags (used with many commands)

* `-f, --values <file>` — supply a YAML file of values.
* `--set key=value[,key2=value2]` — override values on command-line.
* `--namespace <ns>` — specify Kubernetes namespace.
* `--create-namespace` — create namespace if missing when installing.
* `--version` — specify chart version to install from repo.
* `--atomic` — roll back on failure (install/upgrade).
* `--wait` — wait for resources to be ready.
* `--timeout` — timeout for wait.
* `--dry-run` — simulate and render without sending to cluster.
* `--debug` — extra logging and info.

---

# Day-1.5: Exploring & Reading Existing Charts (very important)

### Add repo & update index

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### List/search charts in a repo

```bash
helm search repo bitnami         # list all charts in bitnami repo
helm search repo bitnami/nginx  # filter to nginx in that repo
helm search hub nginx           # search Artifact Hub (global) for nginx
```

### Show chart metadata and values (without pulling)

```bash
helm show chart bitnami/nginx    # prints Chart.yaml
helm show values bitnami/nginx   # prints default values.yaml
helm show all bitnami/nginx      # prints full info (templates / values / chart)
```

### Pull and unpack a chart

```bash
helm pull bitnami/nginx             # downloads nginx-<version>.tgz
# unpack
tar -xvzf nginx-<version>.tgz
cd nginx
# see Chart.yaml, values.yaml, templates/
```

### Install from repo directly

```bash
helm install my-nginx bitnami/nginx --version <chart-version> -n myns --create-namespace
```

**Why study repo charts?**

* Learn structure & best-practices from production charts (Bitnami, Prometheus, etc.).
* Discover available configuration keys (`values.yaml`) before you install.

---

# Day-2: Chart Anatomy & `Chart.yaml`

## Chart Directory (scaffolded by `helm create myapp`)

```
myapp/
├── Chart.yaml          # chart metadata
├── values.yaml         # default configuration values
├── charts/             # subcharts / dependencies (packed or folders)
├── templates/          # Kubernetes YAML templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # reusable template snippets
│   └── NOTES.txt
└── .helmignore
```

## `Chart.yaml` fields (what to write and why)

```yaml
apiVersion: v2         # required for Helm 3
name: myapp            # chart name
description: A Helm chart for Kubernetes
type: application      # 'application' or 'library'
version: 0.1.0         # chart semver (chart package version)
appVersion: "1.27.0" # the version of the app (informational)

# Optional
icon: https://example.com/logo.png
home: https://example.com
sources:
  - https://github.com/myorg/myapp
maintainers:
  - name: Mamatha
    email: mamatha@example.com
keywords:
  - nginx
  - webserver
dependencies:
  - name: redis
    version: 17.3.2
    repository: https://charts.bitnami.com/bitnami
annotations:
  category: web
```

**Rules of thumb**

* `version` (chart) differs from `appVersion` (application image). Update `version` when chart templates/behavior change. Update `appVersion` when the packaged app version changes.

---

# Day-2: `values.yaml` & Using `.Values`

* `values.yaml` is the single place for a chart's default configuration.
* Templates use `.Values` to read keys. You can override with `--set` or `-f`.

**Example `values.yaml`**

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
extraPorts: []
env: {}
```

**Using values in a template (deployment.yaml snippet)**

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

**Override examples**

* On CLI: `helm install myapp ./myapp --set replicaCount=5 --set image.tag=1.28`
* With values file: `helm install myapp ./myapp -f custom-values.yaml`
* Inspect applied values: `helm get values myapp`

---

# Day-3: Templating Deep Dive — Basics

Helm templates use Go templates + Sprig library functions. Templates are written in `templates/*.yaml` and helpers in `_helpers.tpl`.

## Built-in Context Objects

* `.Values` — values from values.yaml & overrides.
* `.Chart` — chart metadata from Chart.yaml (e.g., `.Chart.Name`, `.Chart.Version`).
* `.Release` — release information (e.g., `.Release.Name`, `.Release.Namespace`).
* `.` (dot) — current context (passed to `include`/`tpl`).

## Whitespace control

* `{{` and `}}` standard. Use `{{-` and `-}}` to trim whitespace at edges when needed.

## Helpers and include

* Define reusable templates in `_helpers.tpl` with `define`.
* Call with `include "chartname.template" .` — it returns a string you can pipe or indent.

**Example `_helpers.tpl`**

```gotemplate
{{- define "myapp.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

Use it:

```yaml
metadata:
  name: {{ include "myapp.fullname" . }}
```

---

# Conditionals (`if`, comparison functions)

* `if` evaluates truthiness: non-empty string, non-zero number, non-empty list/map, `true` → truthy. `"", 0, false, nil, []` → falsy.
* Comparison helpers: `eq`, `ne`, `lt`, `gt`, `le`, `ge`.
* Combine with `and`, `or`, and `not`.

### Example usages

```gotemplate
{{- if .Values.service.enabled }}
# Service YAML rendered only if enabled is true
{{- end }}

{{- if eq .Values.env "production" }}
# production only
{{- else }}
# non-prod
{{- end }}

{{- if and .Values.service.enabled (eq .Values.service.type "ClusterIP") }}
# both true
{{- end }}
```

---

# Day-3: `if` — 10 Practical Examples

1. **Simple boolean flag**

```gotemplate
{{- if .Values.service.enabled }}
kind: Service
{{- end }}
```

2. **Equality check**

```gotemplate
{{- if eq .Values.service.type "NodePort" }}
# only render if NodePort
{{- end }}
```

3. **Inequality**

```gotemplate
{{- if ne .Values.env "production" }}
# special config for non-prod
{{- end }}
```

4. **Greater than**

```gotemplate
{{- if gt .Values.replicaCount 3 }}
# extra config if replicas > 3
{{- end }}
```

5. **Less than or equal**

```gotemplate
{{- if le .Values.replicaCount 1 }}
# single replica only
{{- end }}
```

6. **Nested if**

```gotemplate
{{- if .Values.ingress.enabled }}
  {{- if eq .Values.ingress.class "nginx" }}
  # nginx specific annotations
  {{- end }}
{{- end }}
```

7. **If with else**

```gotemplate
{{- if eq .Values.env "production" }}
replicas: 5
{{- else }}
replicas: 1
{{- end }}
```

8. **Defaulting values** (uses `default` from Sprig)

```gotemplate
image: {{ default "nginx:latest" .Values.image }}
```

9. **Check for presence of list**

```gotemplate
{{- if .Values.extraPorts }}
# render ports
{{- end }}
```

10. **Combining `and` / `or`**

```gotemplate
{{- if and .Values.service.enabled (eq .Values.service.type "ClusterIP") }}
# only if enabled AND ClusterIP
{{- end }}
```

---

# Day-3: Loops (`range`) — Syntax, meaning & outcomes

## Syntax

```gotemplate
{{- range $index, $value := .Values.list }}
  # use $index and $value
{{- end }}
```

* If you only need the value: `{{- range .Values.list }} ... {{- end }}` and `.` inside loop refers to current value.
* For maps: `{{- range $k, $v := .Values.map }} ... {{- end }}`

## 10 Loop examples (practical)

1. **Basic list loop**

```gotemplate
# values.yaml:
# extraPorts:
#   - 8080
#   - 9090

ports:
{{- range .Values.extraPorts }}
- port: {{ . }}
{{- end }}
```

2. **Loop with index**

```gotemplate
{{- range $i, $port := .Values.extraPorts }}
- name: port-{{ $i }}
  containerPort: {{ $port }}
{{- end }}
```

3. **Loop over map (key\:value) for env var rendering**

```gotemplate
env:
{{- range $k, $v := .Values.env }}
- name: {{ $k }}
  value: "{{ $v }}"
{{- end }}
```

4. **Conditional inside loop**

```gotemplate
{{- range .Values.volumes }}
  {{- if eq .type "emptyDir" }}
- name: {{ .name }}
  emptyDir: {}
  {{- end }}
{{- end }}
```

5. **Loop + function (`upper`)**

```gotemplate
labels:
{{- range .Values.labels }}
  - {{ . | upper }}
{{- end }}
```

6. **Range with `default`**

```gotemplate
{{- range (default (list) .Values.tolerations) }}
- key: {{ .key }}
  operator: {{ .operator }}
{{- end }}
```

7. **Nested loops**

```gotemplate
{{- range .Values.deployments }}
- name: {{ .name }}
  containers:
  {{- range .containers }}
    - image: {{ .image }}
  {{- end }}
{{- end }}
```

8. **Loop with template include**

```gotemplate
{{- range .Values.services }}
{{ include "myapp.service" . }}
{{- end }}
```

9. **Numeric sequence (using Sprig `until`)**

```gotemplate
{{- range $i, $e := until 3 }}
- containerPort: {{ add 8000 $i }}
{{- end }}
```

10. **Loop producing multiple documents**

```gotemplate
{{- range .Values.jobs }}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .name }}
spec:
  template:
    spec:
      containers:
      - name: job
        image: {{ .image }}
{{- end }}
```

---

# Indentation Helpers: `indent` vs `nindent` (why they matter)

* YAML requires correct indentation. When you inject a block (like `toYaml` output), you must indent it to fit the surrounding YAML.

* `indent N` — add N spaces in front of every line in the input.

* `nindent N` — add a newline and then indent N spaces (useful when you need a newline before the block).

### Example (broken vs fixed)

Without indent (broken):

```gotemplate
metadata:
  annotations:
{{ toYaml .Values.annotations }}
```

Might render:

```yaml
metadata:
  annotations:
foo: bar
```

With `indent` (fixed):

```gotemplate
metadata:
  annotations:
{{ toYaml .Values.annotations | indent 4 }}
```

Renders:

```yaml
metadata:
  annotations:
    foo: bar
```

With `nindent` (often cleaner):

```gotemplate
metadata:
  labels:{{ toYaml .Values.labels | nindent 4 }}
```

Renders:

```yaml
metadata:
  labels:
    app: nginx
```

---

# `toYaml` — dump a values map into YAML

* `toYaml` converts a Go object/map into a YAML formatted string. Combine with `indent`/`nindent` to place into templates.

**Example**
`values.yaml`:

```yaml
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
```

Template:

```gotemplate
metadata:
  annotations:
{{ toYaml .Values.podAnnotations | indent 4 }}
```

Renders:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

---

# Day-3.5: Advanced templating utilities (must know)

## `tpl` (template inside a value)

* `tpl` renders a string value as a template using the provided context `.`.

**When to use**: When a value in `values.yaml` itself contains template expressions (e.g., `"release={{ .Release.Name }}"`).

**Example**
`values.yaml`:

```yaml
customAnnotation: "release={{ .Release.Name }}"
```

Template:

```gotemplate
annotations:
  custom: {{ tpl .Values.customAnnotation . }}
```

If release = `myapp`, it becomes:

```yaml
annotations:
  custom: release=myapp
```

## `with` (shorter nested scope)

* `with` changes the dot (`.`) inside the block to a nested object, so you don’t repeat long paths.

**Example**

```gotemplate
{{- with .Values.resources }}
resources:
  limits:
    cpu: {{ .limits.cpu }}
    memory: {{ .limits.memory }}
{{- end }}
```

## `required` (fail fast if a value missing)

* `required "error message" .Values.password` will abort rendering with the message if the value is missing or empty.

**Example**

```gotemplate
password: {{ required "password is required" .Values.password | b64enc }}
```

## `lookup` — query live cluster resources while templating

* Use: `lookup apiVersion kind namespace name`
* Returns the resource object (or `nil` if not found). Useful for conditional logic based on existing cluster state.

**Example**

```gotemplate
{{- $cm := lookup "v1" "ConfigMap" "default" "my-config" }}
{{- if $cm }}
# Found: use its data
dataFromExisting: {{ index $cm.data "foo" }}
{{- else }}
# Not found
{{- end }}
```

**Important**: `lookup` performs an API read during `helm template` / `helm install` rendering; it requires kubeconfig access and is typically used carefully in advanced charts.

## `include` vs `template`

* `include` returns the called template as a **string** so you can pipe it (for `indent`, `nindent`, `quote`, etc.).
* `template` writes directly to output and is less flexible.

Prefer `include` for reusability.

## Pipelines (`|`) and chaining functions

* Example: `{{ .Release.Name | trunc 10 | lower }}` — truncate then lowercase.
* Common Sprig functions you’ll use: `default`, `upper`, `lower`, `trim`, `replace`, `quote`, `b64enc`, `b64dec`, `sha256sum`, `uuidv4`, `until`, `join`, `split`.

---

# Examples: Putting It All Together

### Example: Conditional Service + helper + include + indent

`_helpers.tpl`:

```gotemplate
{{- define "myapp.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

`templates/service.yaml`:

```gotemplate
{{- if .Values.service.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
{{ toYaml .Values.service.ports | nindent 4 }}
{{- end }}
```

`values.yaml`:

```yaml
service:
  enabled: true
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

### Example: `tpl` usage for custom annotation

`values.yaml`:

```yaml
customAnnotation: "installed={{ .Release.Name }};app={{ .Chart.Name }}"
```

`templates/any.yaml`:

```gotemplate
metadata:
  annotations:
    custom: {{ tpl .Values.customAnnotation . }}
```

---

# Day-3 Cheats: Common Template Patterns & Gotchas

* Always use `helm lint` to find template errors.
* Use `helm template ./myapp` to render locally and inspect final YAML (fast feedback loop).
* Use `| nindent <N>` when injecting multi-line `toYaml` output into a block.
* Prefer `include` + `nindent` for labels/annotations so indentation and newlines are correct.
* Use `required` for values that must be supplied (DB passwords, secret keys) to fail fast.
* Avoid complex `lookup` logic unless necessary; it couples rendering to live cluster state.

---

# Comprehensive Helm Command Cheat Sheet (explanations)

* `helm create <name>`: scaffold a new chart directory with sensible defaults.
* `helm lint <chart>`: static analysis of chart; catches templating issues and common structure problems.
* `helm template <release> <chart> [-f values.yaml] [--set ...]`: render chart templates locally without applying to a cluster.
* `helm install <release> <chart> [-f values.yaml] [--set ...] [--namespace <ns>] [--create-namespace] [--atomic] [--wait]`: install a chart as a release.
* `helm upgrade <release> <chart> [flags like -f/--set]`: upgrade an existing release.
* `helm uninstall <release>`: removes a release and associated resources.
* `helm repo add <name> <url>`: add a remote chart repository.
* `helm repo update`: refresh local index of repos.
* `helm repo list`: list added repos.
* `helm search repo <term>`: search charts in added repos.
* `helm search hub <term>`: search global Artifact Hub (may require network).
* `helm pull <repo/chart> --untar --untardir <dir>`: download chart (use `--untar` to unpack automatically).
* `helm show chart|values|readme|all <repo/chart>`: inspect chart metadata, default values, README or everything.
* `helm package <chart-dir>`: package a chart into `.tgz` for distribution.
* `helm dependency update <chart-dir>`: update dependencies (reads Chart.yaml dependencies and downloads them into `charts/`).
* `helm dependency build <chart-dir>`: build the charts/ directory (useful with Chart.lock).
* `helm history <release>`: shows revision history.
* `helm status <release>`: current status and notes.
* `helm rollback <release> <revision>`: roll back to given revision.
* `helm get values <release>`: show values currently applied to release.
* `helm get manifest <release>`: show the last rendered manifest.
* `helm test <release>`: run chart tests (if defined in `templates/tests/`).
* `helm plugin list`: list installed helm plugins.

---

# Best Practices / Tips

* Keep `values.yaml` small and document keys in `README.md`.
* Use semantic versions for `version` in Chart.yaml and update on chart changes.
* Prefer explicit `include` helpers for names/labels so naming is consistent across templates.
* Use `helm lint` & `helm template` in CI for PR validation.
* Use `--atomic` and `--wait` in automation to avoid partial failures.
* When packaging for distribution, use Chart Releaser or an OCI registry.

---

# Next steps

* Day-4: **Chart dependencies & subcharts** — how to add Redis/MySQL as dependencies, `requirements.yaml`/Chart.yaml dependencies, `helm dependency update`, and best practices for subcharts vs external charts.

---

*End of notes (Day 1 → Day 3.5). Keep this file handy. I will maintain and append Day-4+ as we progress.*
