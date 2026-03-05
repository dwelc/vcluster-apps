# OTel Collector - Shared Node vCluster Monitoring

A vCluster Platform App that deploys an OpenTelemetry Collector on the **host cluster** to monitor shared-node vClusters. Enriches all metrics with vcluster identity labels at ingest time, eliminating the need for complex PromQL joins at query time.

Uses standard Prometheus metric names (`container_cpu_usage_seconds_total`, etc.) for compatibility with existing dashboards and the private node monitoring configs.

## How It Works

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

### Single Metrics Pipeline

All metrics flow through one `prometheus` receiver and a single `metrics` pipeline:

- **Workload metrics** - cAdvisor `/metrics/cadvisor` scraped via `kubernetes_sd_configs` (node role). The Target Allocator distributes node targets across collector replicas using consistent-hashing.
- **Control plane metrics** - API server and controller-manager metrics via vCluster ServiceMonitors (filtered by `app: vcluster`). The Target Allocator handles per-vcluster mTLS certificate distribution automatically.

### Metrics Processing Pipeline

```
prometheus receiver
    -> memory_limiter
    -> groupbyattrs        (split cadvisor batch into per-pod resource scopes)
    -> transform/pre_enrich (copy namespace/pod/node to k8s.* resource attributes)
    -> k8sattributes       (resolve pod/namespace metadata, add vcluster labels)
    -> filter/vcluster_only (drop metrics without vcluster identity)
    -> resource/add_cluster (add cluster label)
    -> transform            (copy resource attributes to datapoint attributes)
    -> batch
    -> prometheusremotewrite
```

### Label Enrichment

The `k8sattributes` processor extracts these from existing platform labels/annotations on the host cluster:

| Metric Label | Source | Kubernetes Label/Annotation |
|---|---|---|
| `vcluster_name` | namespace | `loft.sh/vcluster-instance-name` |
| `vcluster_project` | namespace | `loft.sh/project` |
| `vcluster_project_namespace` | namespace | `loft.sh/vcluster-instance-namespace` |
| `vcluster_user` | namespace | `loft.sh/user` |
| `vcluster_instance_project` | namespace | `loft.sh/vcluster-instance-project` |
| `vcluster_virtual_namespace` | pod label | `vcluster.loft.sh/namespace` |
| `vcluster_virtual_pod` | pod annotation | `vcluster.loft.sh/name` |
| `cluster` | config | Set via `clusterName` parameter |

## Prerequisites

- **Prometheus** with remote write receiver enabled (`web.enable-remote-write-receiver`)
- **Prometheus Operator CRDs** installed on the host cluster (ServiceMonitor, PodMonitor)
- **vClusters** with `controlPlane.serviceMonitor.enabled: true` (creates ServiceMonitors with `app: vcluster` label)
- **Platform namespace labels** present (these are added automatically by the vCluster Platform)
- **Kubelet scraping disabled** in any existing kube-prometheus-stack to avoid duplicate cadvisor series (`kubelet.enabled: false`)

## Deploy

Deploy once per host cluster via the vCluster Platform UI. Parameters:

| Parameter | Required | Description |
|---|---|---|
| Cluster Name | Yes | Used as `cluster` label on all metrics |
| Prometheus Endpoint | Yes | Remote write URL (without `/api/v1/write` suffix) |
| Prometheus Username | No | Basic auth username |
| Prometheus Password | No | Basic auth password |
| Skip TLS Verification | No | Skip TLS verify for Prometheus connection |

## Key Configuration Details

### Why Deployment mode instead of DaemonSet?

A Deployment with 2 replicas and `consistent-hashing` allocation is more resource-efficient than a DaemonSet on every node. Since we use the `prometheus` receiver (not `kubeletstats`), there's no need for local-node scraping — the Target Allocator distributes all targets (cAdvisor, ServiceMonitors) evenly across replicas.

### Why `groupbyattrs` before `k8sattributes`?

The prometheus receiver batches all cadvisor metrics from a single node into one resource scope. Without `groupbyattrs`, the `k8sattributes` processor matches one pod and applies its metadata to all metrics in the batch — causing cross-contamination between vclusters on the same node. The `groupbyattrs` processor splits the batch into per-pod resource scopes (by `namespace`, `pod`, `node`) so `k8sattributes` matches each pod correctly.

### Why `filter/vcluster_only`?

The cadvisor scrape returns metrics for all pods on every node, including non-vcluster workloads. The `filter/vcluster_only` processor drops any metrics where `vcluster.name` is nil after `k8sattributes` enrichment — only vcluster workloads pass through. This also prevents duplicate series with any existing Prometheus scrapes.

### Why `otel/opentelemetry-collector-contrib`?

The default `otel/opentelemetry-collector-k8s` image does not include the `prometheusremotewrite` exporter. The contrib image is set via `opentelemetry-operator.manager.collectorImage.repository`.

### Why `operator.targetallocator.mtls: true`?

Each vCluster exposes its API server metrics over mTLS using a per-vcluster TLS secret (`vc-<name>`). Without this feature gate, the Target Allocator redacts TLS private keys as `<secret>` when passing scrape configs to collectors, breaking authentication. Enabling this feature gate allows the TA to securely pass the full TLS credentials.

### Why `serviceMonitorSelector.matchLabels.app: vcluster`?

Without filtering, the Target Allocator discovers **all** ServiceMonitors in the cluster (cilium, kube-prometheus-stack, etc.), overwhelming collectors with memory pressure. Filtering to `app: vcluster` limits scope to only vCluster control plane ServiceMonitors.

### Why `health_check: endpoint: 0.0.0.0:13133`?

The contrib image defaults the health check endpoint to localhost, making it unreachable by kubelet probes. Explicitly binding to `0.0.0.0:13133` ensures startup, liveness, and readiness probes work.

### Why the post-install webhook patch job?

The OTel operator's validating webhook incorrectly rejects DELETE operations on collector CRs in deployment mode with Target Allocator enabled. The chart's pre-delete cleanup job runs `kubectl delete opentelemetrycollectors`, which triggers this webhook and fails — hanging app uninstallation. Note: `failurePolicy: Ignore` does not help because the webhook returns an active denial, not an error/timeout.

The fix is a post-install Job (via `extraObjects`) that removes the DELETE webhook rules from the `ValidatingWebhookConfiguration` after each install/upgrade. This allows the pre-delete cleanup job to succeed on uninstallation. The Job creates its own ServiceAccount, ClusterRole, and ClusterRoleBinding for the minimum permissions needed (`get` and `patch` on `validatingwebhookconfigurations`), and all hook resources are cleaned up automatically.

### Why `send_batch_max_size: 10000`?

The `opentelemetry-kube-stack` chart defaults `send_batch_max_size` to 1500. Setting `send_batch_size: 10000` without also setting `send_batch_max_size >= send_batch_size` fails validation.

## Example Queries

```promql
# CPU usage by vcluster (cAdvisor metrics, standard Prometheus naming)
sum by (vcluster_name) (rate(container_cpu_usage_seconds_total{vcluster_name!=""}[5m]))

# Memory usage for a specific project
sum by (vcluster_name) (container_memory_working_set_bytes{vcluster_project="team-a"})

# API server request rate by vcluster
sum by (vcluster_name, code) (rate(apiserver_request_total{vcluster_name!=""}[5m]))

# All metrics for a specific vcluster
{vcluster_name="my-vcluster", cluster="my-cluster"}
```

## Architecture

| Component | Role |
|---|---|
| OTel Operator | Manages collector Deployment and Target Allocator lifecycle |
| Target Allocator | Discovers vCluster ServiceMonitors, distributes cAdvisor/ServiceMonitor targets across replicas using consistent-hashing, handles mTLS cert distribution |
| Collector Deployment (2 replicas) | Scrapes assigned targets via `prometheus` receiver; enriches with vcluster labels; remote-writes to Prometheus |
| `groupbyattrs` processor | Splits cadvisor metric batches into per-pod resource scopes for correct `k8sattributes` matching |
| `k8sattributes` processor | Resolves pod/namespace metadata to enrich metrics with vcluster identity |
| `filter/vcluster_only` processor | Drops non-vcluster metrics based on absence of `vcluster.name` after enrichment |
| `transform` processors | Pre-enrich: maps Prometheus labels to OTel resource attributes. Post-enrich: copies resource attributes to datapoint attributes for Prometheus compatibility |
| `resource/add_cluster` processor | Adds the `cluster` label to all metrics |
| `prometheusremotewrite` exporter | Pushes metrics to central Prometheus |
