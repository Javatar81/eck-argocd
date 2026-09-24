# eck-argocd

GitOps repository for deploying [Elastic Cloud on Kubernetes (ECK)](https://www.elastic.co/guide/en/cloud-on-k8s/current/index.html) with [Argo CD](https://argo-cd.readthedocs.io/).

It installs the ECK operator once per cluster, then deploys one or more Elasticsearch + Kibana **stacks**. Each stack has a **dev** and **prod** environment. Argo CD promotes all stacks’ `dev` applications first, then `prod` (RollingSync).

## Repository layout

```text
platform/          # Cluster-wide foundations (operator, CRDs, Argo CD config)
apps/              # Argo CD project + ApplicationSet that creates stack apps
stacks/            # Helm values per stack (what Elasticsearch/Kibana look like)
  logs/
  search/
```

| Path | What lives here | Why |
|------|-----------------|-----|
| `platform/` | ECK CRDs, ECK operator, Argo CD health customizations | Shared once per cluster. Upgraded deliberately and separately from workloads. |
| `apps/` | `AppProject` + `ApplicationSet` | Declares *which* stacks/envs exist and *where* they may deploy. Keeps policy out of values files. |
| `stacks/<name>/` | Helm values for the `eck-stack` chart | One folder per product stack. Env-specific sizing stays next to that stack. |

### Platform (`platform/`)

- **`appproject.yaml`** — Argo CD project `platform`. Allowed to install cluster-scoped resources (CRDs, webhooks, RBAC) into `elastic-system` and `argocd`.
- **`eck-crds/`** — Application that installs the `eck-operator-crds` Helm chart (CRDs only).
- **`eck-operator/`** — Application that installs the `eck-operator` chart. `installCRDs: false` because CRDs are owned separately. `managedNamespaces` lists every stack namespace the operator should reconcile.
- **`argocd-cm/`** — Custom health checks and ignoreDifferences for Elasticsearch/Kibana. Yellow cluster health counts as Healthy so a one-node **dev** cluster can unblock RollingSync to **prod**.

### Apps (`apps/`)

- **`appproject.yaml`** — Argo CD project `elastic-stacks`. Whitelists destinations (`elastic-<stack>-<env>` namespaces). Stack apps cannot touch CRDs or webhooks.
- **`applicationset-stacks.yaml`** — One Application per `(stack, env)` from a list generator. Uses the Elastic `eck-stack` Helm chart plus values from this repo (multi-source Application).

### Stacks (`stacks/`)

Each stack directory holds:

| File | Role |
|------|------|
| `values-common.yaml` | Shared shape: versions, TLS, labels, `fullnameOverride`, which chart components are enabled |
| `values-dev.yaml` | Dev sizing (e.g. 1 Elasticsearch node, smaller resources) |
| `values-prod.yaml` | Prod sizing (e.g. 2 nodes, larger resources/storage) |

Helm **replaces** lists rather than merging them, so `nodeSets` live in the env files, not in common.

Today:

| Stack | Apps | Namespaces |
|-------|------|------------|
| `logs` | `logs-dev`, `logs-prod` | `elastic-logs-dev`, `elastic-logs-prod` |
| `search` | `search-dev`, `search-prod` | `elastic-search-dev`, `elastic-search-prod` |

## Design rationale

**Platform vs stacks** — The operator and CRDs are cluster-scoped and sensitive. Separating them from Elasticsearch/Kibana apps limits blast radius and lets you upgrade the operator without touching every stack Application.

**One folder per stack** — Adding a stack is copy-and-wire, not a new chart. Versions and identity (`fullnameOverride`, labels) stay local to that stack so `logs` and `search` can diverge safely.

**Common + env overlays** — Shared settings in one place; capacity and risk (node count, PVC policy, CPU/memory) in `values-dev` / `values-prod`.

**ApplicationSet list, not a chart per env** — The matrix of stack × env is explicit in Git. RollingSync steps on the `env` label: all `dev` applications sync and become Healthy before any `prod`.

**Namespaces per stack and env** — Isolates resources and credentials. The operator only watches namespaces listed in `platform/eck-operator/values.yaml`.

## Prerequisites

1. A Kubernetes cluster with **Argo CD** already installed (this repo does not install Argo CD).
2. Argo CD can reach:
   - this Git repository (`https://github.com/Javatar81/eck-argocd.git`, branch `main`)
   - the Elastic Helm repo (`https://helm.elastic.co`)
3. Changes intended for the cluster are **committed and pushed** to `main`. Argo CD reads Git, not your local working tree.

```bash
# Public GitHub repo: usually no credentials needed.
argocd repo add https://helm.elastic.co --type helm --name elastic
```

## Deploy (bootstrap)

There is no app-of-apps parent yet. Sync-wave annotations on Applications only apply when a parent Application owns them, so **apply in this order** on first install.

All destinations use the in-cluster API (`https://kubernetes.default.svc`).

### 1. Projects

```bash
kubectl apply -f platform/appproject.yaml
kubectl apply -f apps/appproject.yaml
```

Apply (or re-apply) `apps/appproject.yaml` **before** the ApplicationSet creates new stack namespaces. Destinations must list every `elastic-<stack>-<env>` namespace.

### 2. Platform

```bash
kubectl apply -f platform/eck-crds/application.yaml
argocd app wait eck-crds --health --timeout 300

kubectl apply -f platform/eck-operator/application.yaml
argocd app wait eck-operator --health --timeout 300

kubectl apply -f platform/argocd-cm/application.yaml
argocd app wait argocd-cm-eck --health --timeout 180
```

Wait for each app to become Healthy. The operator webhook must be ready before any Elasticsearch custom resource is created.

### 3. Stacks

```bash
kubectl apply -f apps/applicationset-stacks.yaml
```

The ApplicationSet creates `logs-dev`, `logs-prod`, `search-dev`, and `search-prod`. RollingSync syncs **all `dev` apps first**, then **all `prod` apps**.

After bootstrap, Argo CD keeps applications in sync (`automated` + `selfHeal`). Day-2 changes are Git commits: stack values under `stacks/`, operator settings under `platform/eck-operator/values.yaml`, or chart/pin bumps on the Applications.

## Add another stack

Example for a stack named `metrics`:

1. Copy `stacks/logs/` → `stacks/metrics/` and set `fullnameOverride`, labels, and `elasticsearchRef` to `metrics`.
2. Add two list elements in `apps/applicationset-stacks.yaml` (`metrics` / `dev` and `metrics` / `prod` with namespaces `elastic-metrics-dev` / `elastic-metrics-prod`).
3. Allow those namespaces in `apps/appproject.yaml`.
4. Add them to `managedNamespaces` in `platform/eck-operator/values.yaml`.
5. Commit, push to `main`, and ensure the AppProject (and operator values) sync **before** or together with the ApplicationSet change.

## Useful checks

```bash
kubectl get applications -n argocd
kubectl get elasticsearch,kibana -A
argocd app get logs-dev
argocd app get search-prod
```

## Versions (pinned in Git)

| Component | Version | Where |
|-----------|---------|--------|
| ECK operator / CRDs charts | `3.5.0` | `platform/eck-*/application.yaml` |
| `eck-stack` chart | `0.20.0` | `apps/applicationset-stacks.yaml` |
| Elasticsearch / Kibana | `9.5.4` | `stacks/*/values-common.yaml` |
