# vCluster Monitoring Guide

Two vCluster Platform Apps that collect Kubernetes metrics from vClusters and push them to a central Prometheus via remote write. Both apps enrich metrics with vcluster identity labels at ingest time, enabling simple PromQL queries like `sum by (vcluster_name)(rate(container_cpu_usage_seconds_total[5m]))` without complex joins.

## Apps

| App | File | Deploys to | Mode |
|---|---|---|---|
| **OTEL Collector - Shared Nodes** | `otel-collector-shared-nodes-app.yaml` | Host cluster | Deployment (2 replicas) + Target Allocator |
| **OTEL Collector - Private Nodes** | `otel-collector-private-nodes-app.yaml` | Inside vCluster | DaemonSet |

Both apps use standard Prometheus metric names (`container_cpu_usage_seconds_total`, etc.) for dashboard compatibility.

Datadog variants of both apps (with `datadog` exporter, OOTB-dashboard-compatible tagging, hostname pinning, and a cardinality guard) live in [`datadog/`](./datadog/).

## How It Works

### Shared Nodes

```
Host Cluster
+-----------+     +-------------------+     +--------------------+
| kubelet   |---->| OTel Collector    |---->| Prometheus         |
| cAdvisor  |     | Deployment        |     | (remote write)     |
+-----------+     | (2 replicas)      |     +--------------------+
                  |                   |
+-----------+     | groupbyattrs      |     Metrics arrive with:
| vCluster  |---->| k8sattributes     |     - vcluster_name
| API Server|     | filter/vcluster   |     - vcluster_project
| (mTLS)    |     | + cluster label   |     - vcluster_user
+-----------+     +-------------------+     - cluster
       ^                 ^                  - etc.
       |                 |
  Target Allocator ------+
  - discovers ServiceMonitors (app=vcluster)
  - distributes cadvisor/ServiceMonitor targets
  - uses consistent-hashing across replicas
```

**Metrics pipeline:**

```
prometheus receiver
    -> memory_limiter
    -> groupbyattrs        (split cadvisor batch into per-pod resource scopes)
    -> transform/pre_enrich (copy namespace/pod/node to k8s.* resource attributes)
    -> k8sattributes       (resolve pod/namespace metadata, add vcluster labels)
    -> filter/vcluster_only (drop metrics without vcluster identity)
    -> resource/add_cluster (add cluster label from loft.cluster)
    -> transform            (copy resource attributes to datapoint attributes)
    -> batch
    -> prometheusremotewrite
```

### Private Nodes

```
Inside vCluster
+-----------+     +-------------------+     +--------------------+
| kubelet   |---->| OTel Collector    |---->| Prometheus         |
| cAdvisor  |     | DaemonSet         |     | (remote write)     |
| API server|     | (per node)        |     +--------------------+
+-----------+     |                   |
                  | k8sattributes     |     Metrics arrive with:
                  | + external_labels |     - vcluster_name
                  | + metric_relabel  |     - vcluster_project
                  +-------------------+     - cluster
                                            - etc.
```

**Metrics pipeline:**

```
prometheus receiver (kubelet, cadvisor, apiserver)
    -> memory_limiter
    -> k8sattributes       (resolve pod metadata)
    -> transform           (copy resource attributes to datapoint attributes)
    -> batch
    -> prometheusremotewrite
        + external_labels  (cluster, vcluster_name, project, user)
        + metric_relabel   (namespace -> vcluster_virtual_namespace,
                            pod -> vcluster_virtual_pod)
```

The private-nodes collector runs inside the vCluster, so `namespace` and `pod` labels on metrics already ARE the virtual names. The `metric_relabel_configs` copy them to `vcluster_virtual_namespace`/`vcluster_virtual_pod` for dashboard compatibility with the shared-nodes app.

## Metric Labels

All metrics from both apps carry a consistent set of identity labels:

| Label | Shared Nodes Source | Private Nodes Source |
|---|---|---|
| `cluster` | `resource/add_cluster` processor using `{{ .Values.loft.cluster }}` | `external_labels` using `{{ .Values.loft.cluster }}` |
| `vcluster_name` | `k8sattributes` from namespace label `loft.sh/vcluster-instance-name` | `external_labels` using `{{ .Values.loft.name }}` |
| `vcluster_project` | `k8sattributes` from namespace label `loft.sh/project` | `external_labels` using `{{ .Values.loft.project }}` |
| `vcluster_user` | `k8sattributes` from namespace label `loft.sh/user` | `external_labels` using `{{ .Values.loft.user.name }}` |
| `vcluster_project_namespace` | `k8sattributes` from namespace label `loft.sh/vcluster-instance-namespace` | `external_labels` using `{{ .Values.loft.space }}` |
| `vcluster_virtual_namespace` | `k8sattributes` from pod label `vcluster.loft.sh/namespace` | `metric_relabel_configs` copying `namespace` label |
| `vcluster_virtual_pod` | `k8sattributes` from pod annotation `vcluster.loft.sh/name` | `metric_relabel_configs` copying `pod` label |

**Note:** `vcluster_project_namespace` differs between apps. The shared-nodes collector gets the project namespace (e.g. `p-default`) from the host namespace label. The private-nodes collector gets the host namespace (e.g. `loft-default-v-private-nodes`) from `{{ .Values.loft.space }}` since no Platform variable provides the project namespace.

**Note:** `vcluster_virtual_namespace` and `vcluster_virtual_pod` are MISSING on some metrics — these are vCluster system pods (syncer, CoreDNS) that don't have the syncer labels/annotations since they aren't user workloads synced from inside the vCluster.

## Prerequisites

- **Prometheus** with remote write receiver enabled (`--web.enable-remote-write-receiver`)
- **Prometheus Operator CRDs** installed on the host cluster (ServiceMonitor, PodMonitor) — required for shared-nodes app only
- **vClusters** with `controlPlane.serviceMonitor.enabled: true` — required for shared-nodes control plane metrics
- **Platform namespace labels** present (added automatically by the vCluster Platform)
- **Kubelet scraping disabled** in any existing kube-prometheus-stack to avoid duplicate cadvisor series (`kubelet.enabled: false`)

## Deploy

### Shared Nodes App

Deploy once per host cluster via the vCluster Platform UI. The `cluster` label is set automatically from the Platform's cluster identity (`{{ .Values.loft.cluster }}`).

| Parameter | Required | Description |
|---|---|---|
| Prometheus Endpoint | Yes | Remote write URL (without `/api/v1/write` suffix) |
| Prometheus Username | No | Basic auth username |
| Prometheus Password | No | Basic auth password |
| Skip TLS Verification | No | Skip TLS verify for Prometheus connection |

### Private Nodes App

Deploy into each private-nodes vCluster via the vCluster Platform UI. All vcluster identity labels are injected automatically by the Platform via `{{ .Values.loft.* }}`.

| Parameter | Required | Description |
|---|---|---|
| Prometheus Endpoint | Yes | Remote write URL (without `/api/v1/write` suffix) |
| Prometheus Username | No | Basic auth username |
| Prometheus Password | No | Basic auth password |
| Skip TLS Verification | No | Skip TLS verify for Prometheus connection |

## Example Queries

```promql
# CPU usage by vcluster (works across both app types)
sum by (vcluster_name) (rate(container_cpu_usage_seconds_total{vcluster_name!=""}[5m]))

# Memory usage for a specific project
sum by (vcluster_name) (container_memory_working_set_bytes{vcluster_project="team-a"})

# API server request rate by vcluster
sum by (vcluster_name, code) (rate(apiserver_request_total{vcluster_name!=""}[5m]))

# All metrics for a specific vcluster
{vcluster_name="my-vcluster", cluster="loft-cluster"}

# Pod-level CPU by virtual namespace (works across both app types)
sum by (vcluster_name, vcluster_virtual_namespace) (rate(container_cpu_usage_seconds_total{vcluster_virtual_namespace!=""}[5m]))
```

## Key Configuration Details

### Shared Nodes App

#### Why Deployment mode instead of DaemonSet?

A Deployment with 2 replicas and `consistent-hashing` allocation is more resource-efficient than a DaemonSet on every node. Since we use the `prometheus` receiver (not `kubeletstats`), there's no need for local-node scraping — the Target Allocator distributes all targets (cAdvisor, ServiceMonitors) evenly across replicas.

#### Why `groupbyattrs` before `k8sattributes`?

The prometheus receiver batches all cadvisor metrics from a single node into one resource scope. Without `groupbyattrs`, the `k8sattributes` processor matches one pod and applies its metadata to all metrics in the batch — causing cross-contamination between vclusters on the same node. The `groupbyattrs` processor splits the batch into per-pod resource scopes (by `namespace`, `pod`, `node`) so `k8sattributes` matches each pod correctly.

#### Why `filter/vcluster_only`?

The cadvisor scrape returns metrics for all pods on every node, including non-vcluster workloads. The `filter/vcluster_only` processor drops any metrics where `vcluster.name` is nil after `k8sattributes` enrichment — only vcluster workloads pass through. This also prevents duplicate series with any existing Prometheus scrapes.

#### Why `otel/opentelemetry-collector-contrib`?

The default `otel/opentelemetry-collector-k8s` image does not include the `prometheusremotewrite` exporter. The contrib image is set via `opentelemetry-operator.manager.collectorImage.repository`.

#### Why `operator.targetallocator.mtls: true`?

Each vCluster exposes its API server metrics over mTLS using a per-vcluster TLS secret (`vc-<name>`). Without this feature gate, the Target Allocator redacts TLS private keys as `<secret>` when passing scrape configs to collectors, breaking authentication. Enabling this feature gate allows the TA to securely pass the full TLS credentials.

#### Why `serviceMonitorSelector.matchLabels.app: vcluster`?

Without filtering, the Target Allocator discovers **all** ServiceMonitors in the cluster (cilium, kube-prometheus-stack, etc.), overwhelming collectors with memory pressure. Filtering to `app: vcluster` limits scope to only vCluster control plane ServiceMonitors.

#### Why `health_check: endpoint: 0.0.0.0:13133`?

The contrib image defaults the health check endpoint to localhost, making it unreachable by kubelet probes. Explicitly binding to `0.0.0.0:13133` ensures startup, liveness, and readiness probes work.

#### Why the post-install webhook patch job?

The OTel operator's validating webhook incorrectly rejects DELETE operations on collector CRs in deployment mode with Target Allocator enabled. The chart's pre-delete cleanup job runs `kubectl delete opentelemetrycollectors`, which triggers this webhook and fails — hanging app uninstallation. Note: `failurePolicy: Ignore` does not help because the webhook returns an active denial, not an error/timeout.

The fix is a post-install Job (via `extraObjects`) that removes the DELETE webhook rules from the `ValidatingWebhookConfiguration` after each install/upgrade. This allows the pre-delete cleanup job to succeed on uninstallation. The Job creates its own ServiceAccount, ClusterRole, and ClusterRoleBinding for the minimum permissions needed (`get` and `patch` on `validatingwebhookconfigurations`), and all hook resources are cleaned up automatically.

#### Why `send_batch_max_size: 10000`?

The `opentelemetry-kube-stack` chart defaults `send_batch_max_size` to 1500. Setting `send_batch_size: 10000` without also setting `send_batch_max_size >= send_batch_size` fails validation.

### Private Nodes App

#### Why DaemonSet mode?

Private-nodes vClusters have dedicated nodes. A DaemonSet ensures one collector per node, scraping only the local kubelet/cAdvisor via `${env:K8S_NODE_NAME}` filtering. No Target Allocator is needed.

#### Why `external_labels` instead of `k8sattributes` for identity?

The private-nodes collector runs inside the vCluster, so it can't access host-cluster namespace labels. Instead, the Platform injects `{{ .Values.loft.* }}` template variables at deploy time, which are set as `external_labels` on the `prometheusremotewrite` exporter. These are static per-vCluster values applied to all exported metrics.

#### Why `resource_to_telemetry_conversion: false`?

Setting this to `true` causes duplicate labels that break Grafana dashboards. With `false`, only the original Prometheus labels from scraping and `external_labels` are exported.

#### Why `metric_relabel_configs` for virtual namespace/pod?

Since `resource_to_telemetry_conversion` is disabled, OTel transform processor attributes don't reach the exported Prometheus labels. Instead, `metric_relabel_configs` copy `namespace` -> `vcluster_virtual_namespace` and `pod` -> `vcluster_virtual_pod` at scrape time, producing native Prometheus labels that survive the export.

## Architecture

### Shared Nodes

| Component | Role |
|---|---|
| OTel Operator | Manages collector Deployment and Target Allocator lifecycle |
| Target Allocator | Discovers vCluster ServiceMonitors, distributes cAdvisor/ServiceMonitor targets across replicas using consistent-hashing, handles mTLS cert distribution |
| Collector Deployment (2 replicas) | Scrapes assigned targets via `prometheus` receiver; enriches with vcluster labels; remote-writes to Prometheus |
| `groupbyattrs` processor | Splits cadvisor metric batches into per-pod resource scopes for correct `k8sattributes` matching |
| `k8sattributes` processor | Resolves pod/namespace metadata to enrich metrics with vcluster identity |
| `filter/vcluster_only` processor | Drops non-vcluster metrics based on absence of `vcluster.name` after enrichment |
| `transform` processors | Pre-enrich: maps Prometheus labels to OTel resource attributes. Post-enrich: copies resource attributes to datapoint attributes for Prometheus compatibility |
| `resource/add_cluster` processor | Adds the `cluster` label from `{{ .Values.loft.cluster }}` |
| `prometheusremotewrite` exporter | Pushes metrics to central Prometheus |

### Private Nodes

| Component | Role |
|---|---|
| Collector DaemonSet | One pod per node, scrapes local kubelet, cAdvisor, and API server metrics |
| `k8sattributes` processor | Resolves pod metadata inside the vCluster |
| `transform` processor | Copies resource attributes to datapoint attributes |
| `metric_relabel_configs` | Copies `namespace`/`pod` to `vcluster_virtual_namespace`/`vcluster_virtual_pod` at scrape time |
| `external_labels` | Adds static vcluster identity labels (cluster, name, project, user) from Platform-injected `{{ .Values.loft.* }}` |
| `prometheusremotewrite` exporter | Pushes metrics to central Prometheus |
