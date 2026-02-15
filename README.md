# Model Observability GitOps

This repository manages an end-to-end observability stack for:

- Locally deployed models on OpenShift (via KServe + vLLM)
- Nvidia GPU utilization (via the Nvidia DCGM exporter)
- External AI Provider usage and cost monitoring (OpenAI + Gemini) via [thepet/robotheus](https://github.com/thepetk/robotheus.git)
- Grafana Dashboards for better monitoring
- Prometheus scraping and alerting (including Slack notifications)

Everything is deployed declaratively using **Argo CD and Kustomize**.

## Components

### Grafana

- Managed by the [Grafana Operator](https://github.com/grafana/grafana-operator) (`grafana.integreatly.org/v1beta1`)
- Runs in `model-monitoring` namespace
- Dashboards are declared as `GrafanaDashboard` CRDs (see [grafana dir](./apps/monitoring/base/grafana/))
- Prometheus datasource is configured via `GrafanaDatasource` CRD (`https://prometheus-k8s.openshift-monitoring.svc:9091`)
- All resources connect to the Grafana instance via `instanceSelector: { dashboards: "grafana" }`

#### Dashboards

- **Local Models (KServe vLLM)**:

  - Predictor pods status table
  - Predictor pods by GPU node
  - GPU utilization by node (timeseries + gauge)
  - GPU memory utilization by node (timeseries + gauge)
  - CPU utilization by node (timeseries + gauge)
  - CPU memory utilization by node (timeseries + gauge)
  - Predictor token throughput by pod (prompt + generation tokens/s)
  - Failed pods count (vllm namespace)

- **AI Providers (OpenAI)**:
  - Exporter health (robotheus up/down)
  - OpenAI token throughput (tokens/s)
  - OpenAI cost today (USD)
  - OpenAI tokens by project

---

### Cluster Models (KServe + vLLM)

Scraped via `PodMonitor`:

- Namespace: `vllm`
- Selector: `component=predictor`
- Endpoint: `:8080/metrics`

GPU metrics are scraped from DCGM exporter using `ServiceMonitor`.

### Alerts

Alerts are set via `PrometheusRule` and routed through `AlertmanagerConfig`:

- **LocalModelTargetsDown** (warning) — fires when no predictor scrape targets are up for 1 hour.
- **RobotheusTargetsDown** (warning) — fires when Robotheus scrape targets are down
- **VllmPodNotReady** (critical, Slack) — fires when any pod in the `vllm` namespace is stuck in Pending or Failed phase for more than 1 hour

#### Slack Integration

The `VllmPodNotReady` alert is routed to Slack via an `AlertmanagerConfig` CR. Alerts matching `alert_route: slack` are sent to the `#team-rhdh-ai-devcluster-alerts` channel using a webhook URL stored in a Secret (see below). Resolved notifications are also sent, and alerts repeat every 4 hours.

## Required Pre-existing Secrets

These Secrets are NOT managed by this repo and must exist before Argo CD sync:

### Grafana

```
model-monitoring/grafana-admin
```

Keys:

- `username`
- `password`
- `token` — Bearer token for Prometheus access (service account token with `cluster-monitoring-view` role)

### Slack Alerting

```
model-monitoring/slack-webhook-secret
```

Keys:

- `webhook-url` — Slack incoming webhook URL for the target channel

### Robotheus

```
monitoring/robotheus-secrets
```

Keys:

- OPENAI_API_KEY

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
- **Prometheus Operator** must be installed on the cluster (provides `PodMonitor`, `ServiceMonitor`, `PrometheusRule`, and `AlertmanagerConfig` CRDs)
- **User-workload-monitoring** must be enabled on OpenShift for `AlertmanagerConfig` to work
- Prometheus must already be present in the cluster
- NVIDIA GPU Operator + DCGM exporter must already be installed

## Contirbutions

Contributions are welcomed in the repo. Feel free to open a PR.
