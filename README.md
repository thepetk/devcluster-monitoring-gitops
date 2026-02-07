# Model Observability GitOps

This repository manages an end-to-end observability stack for:

- Locally deployed models on OpenShift (via KServe + vLLM)
- Nvidia GPU utilization (via the Nvidia DCGM exporter)
- External AI Provider usage and cost monitoring (OpenAI + Gemini) via [thepet/robotheus](https://github.com/thepetk/robotheus.git)
- Grafana Dashboards for better monitoring
- Prometheus scraping and basic alerting

Everything is deployed declaratively using **Argo CD and Kustomize**.

## Components

### Grafana

- Managed by the [Grafana Operator](https://github.com/grafana/grafana-operator) (`grafana.integreatly.org/v1beta1`)
- Runs in `model-monitoring` namespace
- Dashboards are declared as `GrafanaDashboard` CRDs (see [grafana dir](./apps/monitoring/base/grafana/))
- Prometheus datasource is configured via `GrafanaDatasource` CRD (`http://prometheus-k8s.openshift-monitoring.svc:9091`)
- All resources connect to the Grafana instance via `instanceSelector: { dashboards: "grafana" }`

#### Dashboards

- **Local Models**:

  - Predictor pod CPU
  - GPU utilization + memory
  - GPU utilization by node
  - Predictor pods by node
  - Request rate + p95 latency (generic HTTP metrics)

- **AI Providers usage**: Currently on team scope
  - OpenAI tokens/sec
  - OpenAI cost/day
  - Gemini tokens/sec
  - Gemini cost/day
  - Robotheus scrape health

---

### Cluster Models (KServe + vLLM)

Scraped via `PodMonitor`:

- Namespace: `vllm`
- Selector: `component=predictor`
- Endpoint: `:8080/metrics`

GPU metrics are scraped from DCGM exporter using `ServiceMonitor`.

### Alerts:

Alerts are set via prometheus rules:

- Predictor scrape down
- Robotheus scrape down

## Required Pre-existing Secrets

These Secrets are NOT managed by this repo and must exist before Argo CD sync:

### Grafana

```
model-monitoring/grafana-admin
```

Keys:

- `username`
- `password`

### Robotheus

```
monitoring/robotheus-secrets
```

Keys:

- OPENAI_API_KEY
- GEMINI_API_KEY

## Installation

1. Ensure the **Grafana Operator** and **Prometheus Operator** are installed on the cluster (via OperatorHub / OLM)
2. Apply Argo CD Project + Application from `argocd/`
3. Argo CD syncs `apps/monitoring/overlays/dev`
4. Monitoring namespace is created
5. Grafana CR, datasource, folder, and dashboards are reconciled by the operator
6. Robotheus, PodMonitors, ServiceMonitors, and alerts are deployed
7. Dashboards appear automatically in Grafana under folder **GitOps**

## Important Notes

- **Grafana Operator** must be installed on the cluster (via OperatorHub or manual install)
- **Prometheus Operator** must be installed on the cluster (provides `PodMonitor`, `ServiceMonitor`, and `PrometheusRule` CRDs)
- Prometheus must already be present in the cluster
- NVIDIA GPU Operator + DCGM exporter must already be installed

## Contirbutions

Contributions are welcomed in the repo. Feel free to open a PR.
