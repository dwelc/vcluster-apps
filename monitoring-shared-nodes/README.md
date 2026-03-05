# OTel Collector - Shared Node vCluster Monitoring

A vCluster Platform App that deploys an OpenTelemetry Collector on the **host cluster** to monitor shared-node vClusters. Enriches all metrics with vcluster identity labels at ingest time, eliminating the need for complex PromQL joins at query time.

## How It Works

```
Host Cluster
+-----------+     +-------------------+     +--------------------+
| kubelet   |---->| OTel Collector    |---->| Prometheus         |
| (per node)|     | DaemonSet         |     | (remote write)     |
+-----------+     |                   |     +--------------------+
                  | k8sattributes     |
+-----------+     | processor adds:   |     Metrics arrive with:
| vCluster  |---->| - vcluster_name   |     - vcluster_name
| API Server|     | - vcluster_project|     - vcluster_project
| (mTLS)    |     | - vcluster_user   |     - vcluster_user
+-----------+     | - cluster         |     - cluster
                  +-------------------+     - etc.
       ^
       |
  Target Allocator
  discovers ServiceMonitors
  (label: app=vcluster)
```

### Two Metric Pipelines

**Workload metrics** (`metrics/workload`) - Pod CPU, memory, network, filesystem via `kubeletstats` receiver. The `k8sattributes` processor resolves each pod's namespace to extract platform labels (`loft.sh/project`, `loft.sh/vcluster-instance-name`, etc.) and maps them to `vcluster.*` attributes.

**Control plane metrics** (`metrics/controlplane`) - API server and controller-manager metrics via `prometheus` receiver. The Target Allocator discovers vCluster ServiceMonitors (filtered by `app: vcluster`) and handles per-vcluster mTLS certificate mounting automatically.

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

- **Prometheus Operator CRDs** installed on the host cluster (ServiceMonitor, PodMonitor)
- **vClusters** with `controlPlane.serviceMonitor.enabled: true` (creates ServiceMonitors with `app: vcluster` label)
- **Platform namespace labels** present (these are added automatically by the vCluster Platform)

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

### Why `otel/opentelemetry-collector-contrib`?

The default `otel/opentelemetry-collector-k8s` image does not include the `prometheusremotewrite` exporter. The contrib image is set via `opentelemetry-operator.manager.collectorImage.repository`.

### Why `operator.targetallocator.mtls: true`?

Each vCluster exposes its API server metrics over mTLS using a per-vcluster TLS secret (`vc-<name>`). Without this feature gate, the Target Allocator redacts TLS private keys as `<secret>` when passing scrape configs to collectors, breaking authentication. Enabling this feature gate allows the TA to securely pass the full TLS credentials.

### Why `serviceMonitorSelector.matchLabels.app: vcluster`?

Without filtering, the Target Allocator discovers **all** ServiceMonitors in the cluster (kubelet, cilium, kube-prometheus-stack, etc.), overwhelming the DaemonSet collectors with memory pressure. Filtering to `app: vcluster` limits scope to only vCluster control plane ServiceMonitors.

### Why `send_batch_max_size: 10000`?

The `opentelemetry-kube-stack` chart defaults `send_batch_max_size` to 1500. Setting `send_batch_size: 10000` without also setting `send_batch_max_size >= send_batch_size` fails validation.

## Example Queries

```promql
# CPU usage by vcluster
sum by (vcluster_name) (rate(k8s_pod_cpu_time_seconds_total{vcluster_name!=""}[5m]))

# API server request rate by vcluster
sum by (vcluster_name, code) (rate(apiserver_request_total{vcluster_name!=""}[5m]))

# Memory usage for a specific project
sum by (vcluster_name) (k8s_pod_memory_working_set_bytes{vcluster_project="team-a"})

# All metrics for a specific vcluster
{vcluster_name="my-vcluster", cluster="my-cluster"}
```

## Architecture

| Component | Role |
|---|---|
| OTel Operator | Manages collector DaemonSet and Target Allocator lifecycle |
| Target Allocator | Discovers vCluster ServiceMonitors, handles mTLS cert distribution, assigns targets to collectors per-node |
| Collector DaemonSet | Runs on each node; scrapes kubeletstats locally, scrapes assigned ServiceMonitor targets via TA |
| `k8sattributes` processor | Resolves pod/namespace metadata to enrich metrics with vcluster identity |
| `transform` processor | Copies resource attributes to datapoint attributes for Prometheus compatibility |
| `resource/add_cluster` processor | Adds the `cluster` label to all metrics |
| `prometheusremotewrite` exporter | Pushes metrics to central Prometheus |
