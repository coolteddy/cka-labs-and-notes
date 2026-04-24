# CKA Notes: Helm

## Fast Commands

- List configured repos:
  - `helm repo list`
- Add repo:
  - `helm repo add <name> <url>`
- Update repo metadata:
  - `helm repo update`
- Search locally added repos:
  - `helm search repo <term>`
- Search Artifact Hub:
  - `helm search hub <term>`
- Search Artifact Hub with structured output:
  - `helm search hub <term> -o yaml`
  - `helm search hub <term> -o json`
- Show chart metadata:
  - `helm show chart <repo/chart>`
- Show default values:
  - `helm show values <repo/chart>`
- Show CRDs shipped by a chart:
  - `helm show crds <repo/chart>`
- Render manifests without installing:
  - `helm template <release> <repo/chart>`
- Install or upgrade:
  - `helm upgrade --install <release> <repo/chart> -n <ns> --create-namespace`
- List releases:
  - `helm list -A`
- Release history:
  - `helm history <release> -n <ns>`
- Inspect effective values:
  - `helm get values <release> -n <ns>`
- Roll back:
  - `helm rollback <release> <revision> -n <ns> --wait --timeout 5m`
- Uninstall:
  - `helm uninstall <release> -n <ns>`

## Mental Model

- A chart is a packaged app definition for Kubernetes.
- Helm repos and OCI registries are two different chart distribution models.
- `helm install` sends resources to the cluster.
- `helm template` only renders YAML locally.
- `helm upgrade --install` means:
  - upgrade if the release exists
  - install if it does not

## Chart Layout

Typical chart structure:

```text
chart-name/
  Chart.yaml
  values.yaml
  templates/
  crds/
  charts/
```

Command mapping:

- `helm show chart` -> `Chart.yaml`
- `helm show values` -> `values.yaml`
- `helm show crds` -> files under `crds/`

Useful local inspection:

```bash
helm pull bitnami/nginx --untar
find nginx -maxdepth 2 -type f
sed -n '1,120p' nginx/Chart.yaml
sed -n '1,160p' nginx/values.yaml
```

## Repo vs OCI

Classic Helm repo:

- add with `helm repo add`
- refresh with `helm repo update`
- search with `helm search repo`
- chart reference looks like `bitnami/nginx`

OCI chart:

- no `helm repo add`
- use direct path like `oci://ghcr.io/nginx/charts/nginx-gateway-fabric`
- inspect with `helm show ... oci://...`
- pull locally with `helm pull oci://... --untar`

## Discovery Patterns

If the repo is already added:

- `helm search repo <term>`
- `helm search repo <repo/chart> --versions`

If the repo is not added and you must discover it:

- `helm search hub <term> -o yaml`

Why `-o yaml` matters:

- plain `helm search hub` truncates URLs
- structured output lets you find the actual repo block

Example:

```bash
helm search hub argo-cd -o yaml | grep -A2 'name: argo$'
```

Useful interpretation:

- `repository.name` and `repository.url` tell you what to pass to `helm repo add`
- the `url` field pointing to `artifacthub.io` is not the repo-add URL

## Search Habits

- Search by product, not broad umbrella names.
- `argo` is too broad.
- `argo-cd` is the specific product.

Better:

```bash
helm search hub argo-cd -o yaml
```

Worse:

```bash
helm search hub argo -o yaml | grep -i argo
```

## Install and Upgrade

Common install shape:

```bash
helm upgrade --install web1 bitnami/nginx \
  -n helm-lab-1 \
  --create-namespace \
  --set replicaCount=2 \
  --set service.type=NodePort \
  --wait --timeout 5m
```

Useful verification:

```bash
helm list -n helm-lab-1
helm get values web1 -n helm-lab-1
kubectl get deploy,svc -n helm-lab-1
```

## Rollback

- `helm history` shows revisions.
- `helm rollback` creates a new revision that restores an older one.
- Rollback restores the target revision's values, including previous mistakes or omissions.

Example:

```bash
helm rollback web1 5 -n helm-lab-1 --wait --timeout 5m
```

## Template Instead of Install

Render YAML without installing:

```bash
helm template ag argo/argo-cd \
  --version 7.7.3 \
  --set crds.install=false \
  -n argocd > ~/argo-helm.yaml
```

Useful checks:

```bash
ls -l ~/argo-helm.yaml
grep -c 'kind: CustomResourceDefinition' ~/argo-helm.yaml
grep -n 'apiextensions.k8s.io' ~/argo-helm.yaml
```

Expected CRD-free proof:

- no matching output from `grep -n`
- count `0` from `grep -c`

Argo CD exam-style flow when CRDs already exist:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm search repo argo/argo-cd --versions
helm show chart argo/argo-cd --version 7.7.3
```

Important:

- chart version and app version are different things
- for example:
  - chart: `7.7.3`
  - app: `v2.13.0`

If you must render to a file and install only the non-CRD resources:

```bash
helm template argocd argo/argo-cd \
  --namespace argocd \
  --version 7.7.3 \
  --set crds.install=false \
  > argocd-rendered.yaml

kubectl apply -f argocd-rendered.yaml
```

If you need the Argo CD CRDs first for a local simulation, install them from upstream version-pinned manifests using the **app version**:

```bash
kubectl apply -k "https://github.com/argoproj/argo-cd/manifests/crds?ref=v2.13.0"
```

## CRD Handling

- CRDs are cluster-scoped resources.
- Check them with `kubectl get crd`, not with `-n <namespace>`.
- `helm show crds <chart>` tells you if the chart ships CRDs.

Two different approaches:

- `--skip-crds`
  - Helm-level behavior
  - generic flag
- `--set crds.install=false`
  - chart-level behavior
  - only works if the chart defines that value

Important distinction:

- They can produce similar outcomes.
- They are not the same mechanism.
- Ideal habit:
  - first check `helm show values <chart> | grep -i crd`
  - then check `helm show crds <chart>`
  - prefer the chart's own CRD control value if it exists
  - use `--skip-crds` as a fallback when the chart does not expose a value

Argo CD specific note:

- For `argo/argo-cd`, `helm show crds` may return nothing useful because the chart does not expose CRDs via the normal Helm `crds/` directory path.
- In that case:
  - install CRDs separately from upstream Argo CD manifests
  - then render/apply the chart with `--set crds.install=false`

## Gotchas

- `helm repo add` does not download the whole repo. It stores repo config locally.
- `helm repo update` refreshes cached index metadata after repo add.
- `helm search hub` default table output truncates useful fields.
- Unknown `--set` keys do not necessarily fail.
- A typo like `replicCount=2` can be accepted and silently do nothing.
- `helm upgrade` can unintentionally drop earlier values if you do not restate them.
- Verify with `helm get values` after install or upgrade.
- For CRD-sensitive charts, do not jump straight to `--skip-crds`.
- First inspect `values.yaml` for chart-defined CRD controls such as `crds.install`, `installCRDs`, or `crds.enabled`.
- If the chart exposes a CRD value, prefer that over the generic Helm flag.
- `helm template` does not need the namespace to already exist.
- Redirecting to `/home/file.yaml` can fail for a normal user. Use `~/file.yaml` or `/home/<user>/file.yaml`.
- `helm pull <repo/chart>` downloads the chart artifact locally, not the whole Git repository.
- By default `helm pull` creates a `.tgz` file.
- Use `helm pull <repo/chart> --untar` to unpack the chart directory locally.
- `<repo_alias>/<chart>` syntax only works after the repo alias has been added with `helm repo add`.

## Useful Local Paths

Helm stores repo config and cache locally:

- repo config:
  - `~/.config/helm/repositories.yaml`
- repo cache:
  - `~/.cache/helm/repository/`

Useful command:

```bash
helm env
```

## Exam Habits

- Prefer `helm search repo` over `helm search hub` when possible.
- Use `helm search hub -o yaml` when you really need repo discovery.
- Inspect before installing:
  - `helm show chart`
  - `helm show values`
  - `helm show crds`
- Use `--wait --timeout 5m` on installs, upgrades, and rollbacks.
- Verify with both Helm and kubectl.
- For CRD-sensitive tasks, prove CRDs are present or absent explicitly.
- Use `helm template` when the task asks for generated YAML rather than direct install.

## Cheat Sheet

- add repo: `helm repo add <name> <url>`
- refresh indexes: `helm repo update`
- local search: `helm search repo <term>`
- hub discovery: `helm search hub <term> -o yaml`
- chart metadata: `helm show chart <repo/chart>`
- default values: `helm show values <repo/chart>`
- shipped CRDs: `helm show crds <repo/chart>`
- install or upgrade: `helm upgrade --install ...`
- render only: `helm template ... > file.yaml`
- inspect effective values: `helm get values <release> -n <ns>`
- release history: `helm history <release> -n <ns>`
- rollback: `helm rollback <release> <rev> -n <ns> --wait`
- uninstall: `helm uninstall <release> -n <ns>`
- CRDs are cluster-scoped: `kubectl get crd`
