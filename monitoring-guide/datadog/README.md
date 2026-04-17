# vCluster Monitoring Guide — Datadog

Two vCluster Platform Apps that collect Kubernetes metrics from vClusters and push them to Datadog via the OTel Datadog exporter. Metrics are enriched with both vcluster identity tags (`vcluster_name`, `vcluster_project`, etc.) and Datadog's standard k8s tags (`kube_cluster_name`, `kube_namespace`, `pod_name`) so Datadog's OOTB Kubernetes dashboards populate without extra wiring.

## Apps

| App | File | Deploys to | Mode |
|---|---|---|---|
| **OTEL Collector - Shared Nodes (Datadog)** | `otel-collector-shared-nodes-datadog-app.yaml` | Host cluster | Deployment (2 replicas) + Target Allocator |
| **OTEL Collector - Private Nodes (Datadog)** | `otel-collector-private-nodes-datadog-app.yaml` | Inside vCluster | DaemonSet |

Same scrape surface and pipeline shape as the Prometheus variants — swap in the `datadog` exporter, pin host/tag behaviour, and add a cardinality guard.

## How It Works

### Shared Nodes

```
Host Cluster
+-----------+     +-------------------+     +--------------------+
| kubelet   |---->| OTel Collector    |---->| Datadog            |
| cAdvisor  |     | Deployment        |     | (datadog exporter) |
+-----------+     | (2 replicas)      |     +--------------------+
                  |                   |
+-----------+     | groupbyattrs      |     Metrics arrive with:
| vCluster  |---->| k8sattributes     |     - kube_cluster_name
| API Server|     | filter/vcluster   |     - kube_namespace
| (mTLS)    |     | + cluster label   |     - kube_deployment
+-----------+     | + set_host        |     - pod_name
       ^          | + cardinality     |     - vcluster_name
       |          +-------------------+     - vcluster_project
  Target Allocator                          - datadog.host.name
  - discovers ServiceMonitors (app=vcluster)  (pinned per-payload)
  - distributes cadvisor/ServiceMonitor targets
  - uses consistent-hashing across replicas
```

**Metrics pipeline:**

```
prometheus receiver (cadvisor + ServiceMonitors + otelcol-self)
    -> memory_limiter
    -> groupbyattrs            (split cadvisor batch into per-pod resource scopes)
    -> transform/pre_enrich    (copy namespace/pod/node to k8s.* resource attributes)
    -> k8sattributes           (resolve pod/namespace metadata, add vcluster tags)
    -> filter/vcluster_only    (drop non-vcluster metrics; keep otelcol_* self-metrics)
    -> resource/add_cluster    (add cluster + k8s.cluster.name from loft.cluster)
    -> transform/set_host      (pin datadog.host.name: node name or synthetic cp host)
    -> transform               (copy resource attributes to datapoint attributes)
    -> transform/cardinality_guard (drop high-cardinality labels; keep_keys on resource)
    -> batch
    -> datadog exporter
```

### Private Nodes

```
Inside vCluster
+-----------+     +-------------------+     +--------------------+
| kubelet   |---->| OTel Collector    |---->| Datadog            |
| cAdvisor  |     | DaemonSet         |     | (datadog exporter) |
| API server|     | (per node)        |     +--------------------+
+-----------+     |                   |
                  | k8sattributes     |     Metrics arrive with:
                  | + resourcedetect  |     - kube_cluster_name
                  | + resource/ident  |     - kube_namespace
                  | + set_host        |     - pod_name
                  | + cardinality     |     - vcluster_name
                  +-------------------+     - vcluster_project
                                            - datadog.host.name
                                              (= K8S_NODE_NAME)
```

**Metrics pipeline:**

```
prometheus receiver (kubelet, cadvisor, apiserver, otelcol-self)
    -> memory_limiter
    -> k8sattributes           (resolve pod metadata)
    -> resourcedetection       ([env, system] — node-level host attributes)
    -> resource/identity       (cluster, k8s.cluster.name, vcluster_* from loft.*)
    -> transform/set_host      (pin datadog.host.name to K8S_NODE_NAME)
    -> transform               (copy resource attributes to datapoint attributes)
    -> transform/cardinality_guard (drop high-cardinality labels; keep_keys on resource)
    -> batch
    -> datadog exporter
        + metric_relabel       (namespace -> vcluster_virtual_namespace,
                                pod -> vcluster_virtual_pod)  [at scrape]
```

The `metric_relabel_configs` on the prometheus scrape jobs copy `namespace`/`pod` to `vcluster_virtual_namespace`/`vcluster_virtual_pod` before the OTel pipeline sees them, so those labels survive all downstream processors and land in Datadog as-is.

## Tag Mapping

Metrics from both apps arrive in Datadog with two families of tags — the Datadog-standard k8s tags (promoted from OTel semantic attributes) and the vcluster-specific identity tags.

### Datadog standard k8s tags (OOTB dashboard compatibility)

| Datadog tag | OTel resource attribute | Source |
|---|---|---|
| `kube_cluster_name` | `k8s.cluster.name` | `resource/add_cluster` / `resource/identity` from `{{ .Values.loft.cluster }}` |
| `kube_namespace` | `k8s.namespace.name` | `k8sattributes` |
| `kube_deployment` | `k8s.deployment.name` | `k8sattributes` |
| `pod_name` | `k8s.pod.name` | `k8sattributes` |
| `container_name` | `k8s.container.name` | `k8sattributes` |
| `host` | `datadog.host.name` | `transform/set_host` (see [Hostname handling](#hostname-handling)) |

The semantic-attribute → tag promotion is done by the Datadog exporter. See <https://docs.datadoghq.com/opentelemetry/mapping/semantic_mapping/>.

### vCluster identity tags

| Tag | Shared Nodes Source | Private Nodes Source |
|---|---|---|
| `cluster` | `resource/add_cluster` from `{{ .Values.loft.cluster }}` | `resource/identity` from `{{ .Values.loft.cluster }}` |
| `vcluster_name` | `k8sattributes` from namespace label `loft.sh/vcluster-instance-name` | `resource/identity` from `{{ .Values.loft.name }}` |
| `vcluster_project` | `k8sattributes` from namespace label `loft.sh/project` | `resource/identity` from `{{ .Values.loft.project }}` |
| `vcluster_user` | `k8sattributes` from namespace label `loft.sh/user` | `resource/identity` from `{{ .Values.loft.user.name }}` |
| `vcluster_project_namespace` | `k8sattributes` from namespace label `loft.sh/vcluster-instance-namespace` | `resource/identity` from `{{ .Values.loft.space }}` |
| `vcluster_virtual_namespace` | `k8sattributes` from pod label `vcluster.loft.sh/namespace` | `metric_relabel_configs` copying `namespace` label |
| `vcluster_virtual_pod` | `k8sattributes` from pod annotation `vcluster.loft.sh/name` | `metric_relabel_configs` copying `pod` label |

Note: `vcluster_virtual_namespace`/`vcluster_virtual_pod` are MISSING on some metrics — vCluster system pods (syncer, CoreDNS) don't carry the syncer labels/annotations since they aren't user workloads synced from inside the vCluster.

## Hostname Handling

The Datadog exporter derives the `host` tag from resource attributes. With no resource attributes set, each collector pod's own hostname is used — meaning 2 phantom hosts per shared-nodes Deployment and one per DaemonSet pod, all churning with restarts. These register as hosts in Datadog's infra list even though they represent nothing real.

Both apps pin `datadog.host.name` explicitly via the `transform/set_host` processor:

- **Shared nodes:** for workload (cadvisor) metrics, set `datadog.host.name = k8s.node.name` so each metric is attributed to the node its pod runs on. For control-plane metrics (ServiceMonitor-sourced), set `datadog.host.name = "<cluster>-vcluster-cp"` — a single synthetic host per cluster representing the vcluster control-plane plane, so those metrics don't get scattered across ephemeral pod hostnames.
- **Private nodes:** set `datadog.host.name = K8S_NODE_NAME` (injected via the downward API). A DaemonSet maps 1:1 to nodes, so this is always the correct node.

The exporter is also pinned to `host_metadata.hostname_source: config_or_system` so the in-config hostname wins over any auto-detected fallback.

Reference: <https://docs.datadoghq.com/opentelemetry/mapping/hostname/>

## Cost Control

Datadog bills custom metrics per (metric name × unique tagset). The cadvisor, kubelet, and apiserver Prometheus metrics carry a lot of high-cardinality labels (`id`, `image`, `name`, `kernelVersion`, `interface`, etc.) that multiply tagset uniqueness per metric.

Both apps ship a `transform/cardinality_guard` processor that:

- Drops the known high-cardinality datapoint attributes outright.
- `keep_keys` on the resource — only the tags referenced by queries survive.

The surviving resource attributes are:

```
k8s.cluster.name, k8s.namespace.name, k8s.deployment.name, k8s.pod.name,
k8s.node.name, k8s.container.name, container.image.tag,
vcluster_name, vcluster_project, vcluster_project_namespace, vcluster_user,
vcluster_virtual_namespace, vcluster_virtual_pod, cluster,
datadog.host.name, host.name, cloud.region, cloud.provider
```

Adjust this list in the YAML if a query needs a different tag.

For further control, use Datadog's [Metrics without Limits](https://docs.datadoghq.com/metrics/metrics-without-limits/) in the UI — *Metrics → Summary → select metric → Manage Tags → Include only*. The cardinality guard caps ingest-side cost; Metrics without Limits caps indexed (queryable) cost.

## Prerequisites

- **Datadog account + API key**
- **Prometheus Operator CRDs** installed on the host cluster (ServiceMonitor, PodMonitor) — required for shared-nodes app only
- **vClusters** with `controlPlane.serviceMonitor.enabled: true` — required for shared-nodes control plane metrics
- **Platform namespace labels** present (added automatically by the vCluster Platform)
- **Kubelet scraping disabled** in any existing kube-prometheus-stack to avoid duplicate cadvisor series (`kubelet.enabled: false`)

## Deploy

### Shared Nodes App

Deploy once per host cluster via the vCluster Platform UI. The `cluster` + `kube_cluster_name` tags are set automatically from the Platform's cluster identity (`{{ .Values.loft.cluster }}`).

| Parameter | Required | Description |
|---|---|---|
| Datadog API Key | Yes | Datadog API key (stored encrypted at rest; ends up plaintext in pod env — externalize via a Secret for production) |
| Datadog Site | No | `datadoghq.com`, `datadoghq.eu`, `us5.datadoghq.com`, etc. (default: `datadoghq.com`) |

### Private Nodes App

Deploy into each private-nodes vCluster via the vCluster Platform UI. All vcluster identity tags are injected automatically by the Platform via `{{ .Values.loft.* }}`.

| Parameter | Required | Description |
|---|---|---|
| Datadog API Key | Yes | Datadog API key |
| Datadog Site | No | Default: `datadoghq.com` |
| Skip Kubelet TLS Verification | No | Skip TLS verify when scraping kubelet endpoints |

## Example Queries (DQL)

Equivalents of the PromQL queries in the parent [README](../README.md#example-queries) and the [fleet-monitoring-otel guide](/home/dan/repos/vcluster-docs/platform/maintenance/monitoring/fleet-monitoring-otel.mdx). All queries filter on `vcluster_name:*` to scope to vcluster-managed metrics (non-vcluster cadvisor series are dropped by `filter/vcluster_only`, but the `:*` wildcard is kept for query-tool compatibility).

```dql
# CPU usage by vcluster
sum:container.cpu.usage{vcluster_name:*} by {vcluster_name}.as_rate()

# Memory usage for a specific project
sum:container.memory.working_set{vcluster_project:team-a} by {vcluster_name}

# API server request rate by vcluster (grouped by status code)
sum:apiserver.requests{vcluster_name:*} by {vcluster_name,code}.as_rate()

# Pod-level CPU by virtual namespace
sum:container.cpu.usage{vcluster_virtual_namespace:*} by {vcluster_name,vcluster_virtual_namespace}.as_rate()
```

### Golden Signals Queries

Same structure as [fleet-monitoring-otel.mdx Golden signals queries](/home/dan/repos/vcluster-docs/platform/maintenance/monitoring/fleet-monitoring-otel.mdx#L943-L1210). Metric names are the OTel-ingested Prometheus names after Datadog's `.`-separated normalization (e.g. `apiserver_request_total` → `apiserver.requests`).

#### Latency

```dql
# kube-apiserver request latency (p99, by verb)
# Works because histograms.mode is pinned to distributions in the exporter.
p99:apiserver.request.duration{vcluster_name:*} by {verb,cluster,vcluster_project,vcluster_name}
```

```dql
# kube-apiserver request latency (p95, non-WATCH)
p95:apiserver.request.duration{vcluster_name:*,verb NOT IN (WATCH,CONNECT)} by {verb,cluster,vcluster_project,vcluster_name}
```

```dql
# etcd backend latency (p99, by operation)
p99:etcd.request.duration{vcluster_name:*} by {operation,cluster,vcluster_project,vcluster_name}
```

#### Traffic

```dql
# kube-apiserver request rate (by verb)
sum:apiserver.requests{vcluster_name:*} by {verb,cluster,vcluster_project,vcluster_name}.as_rate()
```

```dql
# kube-apiserver request rate (by resource) - top 10
top(
  sum:apiserver.requests{vcluster_name:*} by {resource,cluster,vcluster_project,vcluster_name}.as_rate(),
  10, 'mean', 'desc'
)
```

```dql
# Network receive rate (top 10 by virtual namespace)
top(
  sum:container.network.receive.bytes{vcluster_name:*,vcluster_virtual_namespace:*} by {vcluster_name,vcluster_virtual_namespace}.as_rate(),
  10, 'mean', 'desc'
)
```

```dql
# Network transmit rate (top 10 by virtual namespace)
top(
  sum:container.network.transmit.bytes{vcluster_name:*,vcluster_virtual_namespace:*} by {vcluster_name,vcluster_virtual_namespace}.as_rate(),
  10, 'mean', 'desc'
)
```

```dql
# REST client outbound request rate (by code)
sum:rest_client.requests{vcluster_name:*} by {code,cluster,vcluster_project,vcluster_name}.as_rate()
```

#### Errors

```dql
# kube-apiserver error rate (4xx/5xx, by code)
sum:apiserver.requests{vcluster_name:*,code:4*,code:5*} by {code,cluster,vcluster_project,vcluster_name}.as_rate()
```

```dql
# kube-apiserver error ratio (5xx / total)
(sum:apiserver.requests{vcluster_name:*,code:5*} by {cluster,vcluster_project,vcluster_name}.as_rate())
/
(sum:apiserver.requests{vcluster_name:*} by {cluster,vcluster_project,vcluster_name}.as_rate())
```

```dql
# etcd request errors
sum:etcd.request.errors{vcluster_name:*} by {operation,cluster,vcluster_project,vcluster_name}.as_rate()
```

```dql
# Container OOM kills
sum:container.oom.events{vcluster_name:*} by {vcluster_name,vcluster_virtual_namespace,vcluster_virtual_pod}.as_rate()
```

```dql
# REST client error rate (outbound 5xx)
sum:rest_client.requests{vcluster_name:*,code:5*} by {host,cluster,vcluster_project,vcluster_name}.as_rate()
```

#### Saturation

```dql
# Container CPU usage (top 10 pods)
top(
  sum:container.cpu.usage{vcluster_name:*,container:*,vcluster_virtual_pod:*} by {vcluster_name,vcluster_virtual_namespace,vcluster_virtual_pod}.as_rate(),
  10, 'mean', 'desc'
)
```

```dql
# Container memory working set (top 10 pods)
top(
  sum:container.memory.working_set{vcluster_name:*,container:*,vcluster_virtual_pod:*} by {vcluster_name,vcluster_virtual_namespace,vcluster_virtual_pod},
  10, 'mean', 'desc'
)
```

```dql
# CPU throttling ratio (top 10 pods)
top(
  (sum:container.cpu.cfs.throttled.periods{vcluster_name:*,vcluster_virtual_pod:*} by {vcluster_name,vcluster_virtual_namespace,vcluster_virtual_pod}.as_rate())
  /
  (sum:container.cpu.cfs.periods{vcluster_name:*,vcluster_virtual_pod:*} by {vcluster_name,vcluster_virtual_namespace,vcluster_virtual_pod}.as_rate()),
  10, 'mean', 'desc'
)
```

```dql
# kube-apiserver in-flight requests
avg:apiserver.current_inflight_requests{vcluster_name:*} by {request_kind,cluster,vcluster_project,vcluster_name}
```

```dql
# kube-apiserver flow-control queue depth
sum:apiserver.flowcontrol.current_inqueue_requests{vcluster_name:*} by {priority_level,cluster,vcluster_project,vcluster_name}
```

```dql
# Workqueue depth (top 10 by queue name)
top(
  avg:workqueue.depth{vcluster_name:*} by {name,cluster,vcluster_project,vcluster_name},
  10, 'mean', 'desc'
)
```

Key DQL vs PromQL differences:

- Prometheus counters exported via OTel become Datadog COUNT metrics. Wrap with `.as_rate()` to get per-second rates (the default render already per-second-normalizes for COUNT but `.as_rate()` makes it explicit).
- `topk(N, ...)` becomes `top(query, N, 'mean', 'desc')`.
- `histogram_quantile(0.99, rate(..._bucket[5m]))` becomes `p99:metric.name{...}` — works only because the exporter is pinned to `histograms.mode: distributions`. In `counters` mode, server-side percentiles aren't available.
- `label!=""` becomes `label:*` (match any value).
- Metric type is a first-class property — no querying a histogram as a count.

## Key Configuration Details

### Both apps

#### Why pin `datadog.host.name`?

Without it, the collector pod's own hostname is used as `host`, so each collector replica registers as a separate Datadog host. See [Hostname handling](#hostname-handling).

#### Why `k8s.cluster.name`?

Datadog's OOTB Kubernetes dashboards group by `kube_cluster_name`, which is promoted from the `k8s.cluster.name` resource attribute. Without it, the dashboards won't populate for this cluster even though the data is present.

#### Why pin `histograms.mode: distributions`?

Server-side percentile queries (`p95:`, `p99:`) only work on Datadog DISTRIBUTION metrics. The default exporter behaviour is `distributions`, but pinning makes this immune to upstream default changes. `send_aggregation_metrics: true` exposes `.min` / `.max` companions for heatmap visualizations.

#### Why the cardinality guard?

Datadog bills per (metric × tagset). cadvisor alone exports `id`, `image`, `name` labels that explode cardinality per container. The guard drops these outright before the exporter and `keep_keys` on the resource attributes so only queryable tags survive. See [Cost Control](#cost-control).

#### Why the `otelcol-self` scrape job?

The collector's own `/metrics` endpoint (port 8888) exposes OTel-standard `otelcol_*` metrics. Datadog ships a free OOTB OpenTelemetry Collector dashboard that reads these metric names directly. See <https://docs.datadoghq.com/opentelemetry/integrations/collector_health_metrics/>.

### Shared Nodes App

#### Why Deployment mode instead of DaemonSet?

A Deployment with 2 replicas and `consistent-hashing` allocation is more resource-efficient than a DaemonSet on every node. Since we use the `prometheus` receiver (not `kubeletstats`), there's no need for local-node scraping — the Target Allocator distributes all targets (cAdvisor, ServiceMonitors) evenly across replicas.

#### Why `groupbyattrs` before `k8sattributes`?

The prometheus receiver batches all cadvisor metrics from a single node into one resource scope. Without `groupbyattrs`, the `k8sattributes` processor matches one pod and applies its metadata to all metrics in the batch — causing cross-contamination between vclusters on the same node. The `groupbyattrs` processor splits the batch into per-pod resource scopes (by `namespace`, `pod`, `node`) so `k8sattributes` matches each pod correctly.

#### Why `filter/vcluster_only` keeps `otelcol_*` metrics?

The filter drops datapoints with no `vcluster_name` resource attribute — that covers non-vcluster workload cadvisor series. The collector self-metrics (`otelcol_*`) also have no `vcluster_name`, so they are explicitly kept via the `IsMatch(metric.name, "^otelcol_")` exemption.

#### Why `otel/opentelemetry-collector-contrib`?

The default `otel/opentelemetry-collector-k8s` image does not include the `datadog` exporter. The contrib image is set via `opentelemetry-operator.manager.collectorImage.repository`.

#### Why `operator.targetallocator.mtls: true`?

Each vCluster exposes its API server metrics over mTLS using a per-vcluster TLS secret (`vc-<name>`). Without this feature gate, the Target Allocator redacts TLS private keys as `<secret>` when passing scrape configs to collectors, breaking authentication.

#### Why `serviceMonitorSelector.matchLabels.app: vcluster`?

Without filtering, the Target Allocator discovers **all** ServiceMonitors in the cluster, overwhelming collectors with memory pressure. Filtering to `app: vcluster` limits scope to only vCluster control plane ServiceMonitors.

#### Why the post-install webhook patch job?

The OTel operator's validating webhook incorrectly rejects DELETE operations on collector CRs in deployment mode with Target Allocator enabled. The chart's pre-delete cleanup job triggers this webhook and fails, hanging app uninstallation. The post-install Job removes the DELETE webhook rules from the `ValidatingWebhookConfiguration` after each install/upgrade.

### Private Nodes App

#### Why DaemonSet mode?

Private-nodes vClusters have dedicated nodes. A DaemonSet ensures one collector per node, scraping only the local kubelet/cAdvisor via `${env:K8S_NODE_NAME}` filtering. No Target Allocator is needed. The 1:1 pod-to-node mapping also makes the `datadog.host.name = K8S_NODE_NAME` pinning trivially correct.

#### Why `resource/identity` instead of `k8sattributes` for vcluster identity?

The collector runs inside the vCluster, so it can't access host-cluster namespace labels where the `loft.sh/*` identity lives. Instead, the Platform injects `{{ .Values.loft.* }}` template variables at deploy time, which `resource/identity` writes as resource attributes. Static per-vCluster values applied to all exported metrics.

#### Why `metric_relabel_configs` for virtual namespace/pod?

`metric_relabel_configs` runs at scrape time, before the OTel pipeline. The resulting `vcluster_virtual_namespace`/`vcluster_virtual_pod` labels become datapoint attributes that survive the whole pipeline without the resource → datapoint copy dance. Keeps parity with the shared-nodes app's label set without the complexity.

## Architecture

### Shared Nodes

| Component | Role |
|---|---|
| OTel Operator | Manages collector Deployment and Target Allocator lifecycle |
| Target Allocator | Discovers vCluster ServiceMonitors, distributes cAdvisor/ServiceMonitor targets across replicas using consistent-hashing, handles mTLS cert distribution |
| Collector Deployment (2 replicas) | Scrapes assigned targets via `prometheus` receiver; enriches with vcluster + Datadog k8s tags; exports to Datadog |
| `groupbyattrs` processor | Splits cadvisor metric batches into per-pod resource scopes for correct `k8sattributes` matching |
| `k8sattributes` processor | Resolves pod/namespace metadata to enrich metrics with vcluster identity + k8s semantic attributes |
| `filter/vcluster_only` processor | Drops non-vcluster metrics; keeps `otelcol_*` self-metrics |
| `transform/pre_enrich` | Maps Prometheus labels to `k8s.*` resource attributes |
| `resource/add_cluster` | Sets `cluster` + `k8s.cluster.name` from `{{ .Values.loft.cluster }}` |
| `transform/set_host` | Pins `datadog.host.name` per-payload (node for workload, synthetic for control-plane) |
| `transform` | Copies resource attributes to datapoint attributes |
| `transform/cardinality_guard` | Drops high-cardinality labels; `keep_keys` on resource attributes |
| `datadog` exporter | Pushes to Datadog; histograms pinned to distributions mode |

### Private Nodes

| Component | Role |
|---|---|
| Collector DaemonSet | One pod per node, scrapes local kubelet, cAdvisor, and API server metrics |
| `k8sattributes` processor | Resolves pod metadata inside the vCluster |
| `resourcedetection` | Populates host-level attributes from env and system detectors |
| `resource/identity` | Adds static vcluster identity attributes + `k8s.cluster.name` from Platform-injected `{{ .Values.loft.* }}` |
| `transform/set_host` | Pins `datadog.host.name` to `K8S_NODE_NAME` |
| `transform` | Copies resource attributes to datapoint attributes |
| `transform/cardinality_guard` | Drops high-cardinality labels; `keep_keys` on resource attributes |
| `metric_relabel_configs` (scrape-time) | Copies `namespace`/`pod` to `vcluster_virtual_namespace`/`vcluster_virtual_pod` |
| `datadog` exporter | Pushes to Datadog; histograms pinned to distributions mode |

## References

- Datadog OTel exporter config: <https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/datadogexporter/README.md>
- OTel → Datadog semantic mapping: <https://docs.datadoghq.com/opentelemetry/mapping/semantic_mapping/>
- Hostname derivation: <https://docs.datadoghq.com/opentelemetry/mapping/hostname/>
- OTLP metric type mapping: <https://docs.datadoghq.com/metrics/open_telemetry/otlp_metric_types/>
- Metrics without Limits: <https://docs.datadoghq.com/metrics/metrics-without-limits/>
- OOTB Collector health dashboard: <https://docs.datadoghq.com/opentelemetry/integrations/collector_health_metrics/>
