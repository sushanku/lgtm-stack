# LGTM Stack (Alloy + Grafana + Loki + Mimir + Tempo) with Beyla auto-instrumentation

This repo provides a deployable **LGTM** stack for Kubernetes using **Helm charts in this repository**, plus an **example app** (Flask + Postgres). The tracing auto-instrumentation is done via **Beyla** which is installed/configured as part of the **Alloy** deployment in this repo.

## What’s included (focus)

- **Grafana Operator**: manages Grafana resources via CRDs (dashboards, datasources, alerting) as GitOps-friendly Kubernetes objects
- **Alloy**: collection/processing pipeline (includes Beyla for auto-instrumentation)
- **Grafana**: explore UI for metrics/logs/traces
- **Loki**: logs backend
- **Mimir**: metrics backend
- **Tempo**: traces backend
- **Example app**:
  - `example-app/postgreschart` (Postgres)
  - `example-app/flaskchart` (Flask app)

## Components (brief)

### Grafana Operator (why we use it)

The Grafana Operator makes Grafana configuration **declarative** by managing resources like **datasources, dashboards, folders, and alerting** using Kubernetes CRDs. Benefits vs “just running Grafana + an agent”:

- **GitOps-friendly**: dashboards/datasources live as YAML, reviewed and promoted across environments
- **Repeatable setups**: consistent Grafana configuration across clusters without manual clicks / UI drift
- **Safer changes**: rollbacks are just `git revert` + sync
- **Multi-tenant patterns**: easier to manage per-namespace/team dashboards and access patterns (when you adopt them)

### Alloy

Collector/agent for **metrics, logs, and traces**. In this repo, Alloy is also where **Beyla** runs for **auto-instrumentation** so your workloads can emit traces with minimal/no code changes.

### Grafana

Visualization and exploration UI. This chart is set up to query **Mimir (metrics)**, **Loki (logs)**, and **Tempo (traces)** from a single place (Grafana Explore / dashboards).

### Loki

Log aggregation backend optimized for Kubernetes workloads. Grafana queries Loki using **LogQL**.

### Mimir

Horizontally scalable metrics backend compatible with the **Prometheus remote_write / PromQL** ecosystem. Grafana uses Mimir as the primary metrics datasource.

### Tempo

Distributed tracing backend for storing and querying traces. Grafana Explore uses Tempo to visualize traces/spans and correlate with logs/metrics.

### Example app (Flask + Postgres)

Two small Helm charts used to generate real traffic and telemetry:
- `postgres`: Postgres database used by the Flask app
- `flaskapp`: a simple Flask service that talks to Postgres

## Repository structure

```
├── alloy
├── grafana
├── loki
├── mimir
├── tempo
├── app-of-apps                 # ArgoCD App-of-Apps chart (deploys the above + example app)
└── example-app
    ├── flaskchart
    └── postgreschart
```

## Deploy with Helm (copy/paste)

These commands install **everything** into the `monitoring` namespace.

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

# Grafana Operator (required by this setup)
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade -i grafana-operator grafana/grafana-operator --namespace monitoring --create-namespace

# Metrics, logs, traces backends
helm upgrade --install mimir  ./mimir  -n monitoring -f mimir/values-development.yaml
helm upgrade --install loki   ./loki   -n monitoring -f loki/values-development.yaml
helm upgrade --install tempo  ./tempo  -n monitoring -f tempo/values-development.yaml

# Collector / auto-instrumentation (Beyla is part of this)
helm upgrade --install alloy  ./alloy  -n monitoring -f alloy/values-development.yaml

# UI (datasources point at the backends above)
helm upgrade --install grafana ./grafana -n monitoring -f grafana/values-development.yaml

# Example app (Postgres first, then Flask)
helm upgrade --install postgres ./example-app/postgreschart -n monitoring -f example-app/postgreschart/values.yaml
helm upgrade --install flaskapp ./example-app/flaskchart   -n monitoring -f example-app/flaskchart/values.yaml
```

## Deploy with ArgoCD (App of Apps)

The `app-of-apps` Helm chart renders ArgoCD `Application` resources for:
`grafana-operator`, `alloy`, `grafana`, `loki`, `mimir`, `tempo`, `flaskapp`, `postgres`.

### Install the App of Apps via Helm

```bash
# Assumes ArgoCD is installed and the `argocd` namespace exists
helm upgrade --install app-of-apps-development ./app-of-apps \
  -n argocd \
  -f app-of-apps/values-development.yaml
```

### Install the App of Apps via ArgoCD CLI

You must be logged in to the ArgoCD API first (port-forward + `argocd login`):

```bash
# Port-forward ArgoCD API locally (adjust if your svc name differs)
kubectl -n argocd port-forward svc/argocd-server 8080:443

# In another terminal, login (use your credentials / SSO flow)
argocd login localhost:8080 --insecure

# Create the App-of-Apps (development)
argocd app create app-of-apps-development \
  --repo git@github.com:sushanku/lgtm-stack.git \
  --path app-of-apps \
  --revision main \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace argocd \
  --values values-development.yaml \
  --sync-policy automated
```

### Notes

- The example app charts live under `example-app/…`. The App-of-Apps chart is configured to deploy them from those subpaths.
- If you use a different environment, swap the values file:
  - `app-of-apps/values-staging.yaml`
  - `app-of-apps/values-production.yaml`

