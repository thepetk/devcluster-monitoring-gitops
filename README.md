# Model Observability GitOps

This repository manages an end-to-end observability stack for:

- Openshift AI Deployed models.
- Red Hat Developer Hub deployments.

## Structure

### Components

#### Grafana

- Managed by the [Grafana Operator](https://github.com/grafana/grafana-operator) (`grafana.integreatly.org/v1beta1`)
- Runs in `model-monitoring` namespace
- Dashboards are declared as `GrafanaDashboard` CRDs (see [grafana dir](./apps/monitoring/base/grafana/))
- Prometheus datasource is configured via `GrafanaDatasource` CRD (`https://prometheus-k8s.openshift-monitoring.svc:9091`)

Available Dashobards are:

- RHOAI Models Registry (Observability on RHOAI deployed models for RHDH AI DevCluster)
- RHDH AI Rolling Demo

## Alerts

Alerts are set via `PrometheusRule` and routed through `AlertmanagerConfig` to Slack.

Each `PrometheusRule` is deployed in the namespace it monitors (required by `enforcedNamespaceLabel`), while the `AlertmanagerConfig` lives in `model-monitoring`:

- **RHOAIModelsRegistryTargetsDown** (warning, `vllm` namespace) — fires when no predictor scrape targets are up for 1 hour
- **VllmPodNotReady** (critical, `vllm` namespace) — fires when any pod in the `vllm` namespace is stuck in Pending or Failed phase for more than 1 hour
- **RollingDemoPodNotReady** (critical, `rolling-demo-ns` namespace) — fires when any pod in the `rolling-demo-ns` namespace is stuck in Pending or Failed phase for more than 10 minutes

## Required Pre-existing Secrets

These Secrets are NOT managed by this repo and must exist before Argo CD sync:

### Grafana

```
model-monitoring/grafana-admin
```

Keys:

- `username`
- `password`

### Alerting

```
model-monitoring/webhook-secret
```

Keys:

- `webhook-url`

## Installation

1. Ensure the **Grafana Operator** and **Prometheus Operator** are installed on the cluster.
2. Enable **user-workload monitoring** on the cluster (required for ServiceMonitor scraping in user namespaces):
   ```bash
   oc apply -f - <<EOF
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: cluster-monitoring-config
     namespace: openshift-monitoring
   data:
     config.yaml: |
       enableUserWorkload: true
   EOF
   ```
3. Enable **user-workload Alertmanager** (required for `AlertmanagerConfig` routing to Slack):
   ```bash
   oc apply -f - <<EOF
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: user-workload-monitoring-config
     namespace: openshift-user-workload-monitoring
   data:
     config.yaml: |
       alertmanager:
         enabled: true
         enableAlertmanagerConfig: true
   EOF
   ```
4. Create the **webhook secret** with your Slack Workflow webhook URL:
   ```bash
   oc create secret generic webhook-secret \
     -n model-monitoring \
     --from-literal=webhook-url='https://hooks.slack.com/triggers/YOUR/WORKFLOW/URL'
   ```
5. Apply Argo CD Project + Application from `argocd/`

## Contirbutions

Contributions are welcomed in the repo. Feel free to open a PR.
