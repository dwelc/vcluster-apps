# vCluster Monitoring — Datadog Agent (annotation-based)

Reference artifacts for the **Datadog Agent + autodiscovery annotations** path, as opposed to the [OTel-collector-based path](../datadog/) we ship as a Platform App. Use this when the customer is already running a Datadog Agent fleet and wants to integrate vcluster monitoring without standing up a separate OTel pipeline.

These files were validated end-to-end on a homelab cluster (k3s, vCluster Platform, multiple shared-nodes vclusters). Lessons captured below.

## When this path applies

- Customer already runs Datadog Agent (Operator-installed or Helm-installed) on the host cluster.
- Customer wants per-vcluster identity tags (`vcluster_name`, `vcluster_project`, `vcluster_virtual_namespace`, etc.) on all metrics.
- Customer wants vcluster control plane Golden Signals (`apiserver_*`, `etcd_*`, `workqueue_*`).
- They do NOT want a separate OTel collector pipeline running alongside the Datadog Agent.

If they're greenfield or willing to run a parallel collector, [the OTel-based Datadog app](../datadog/) is the simpler path — it bundles tag enrichment, hostname pinning, and cardinality control into one collector deployment.

## Three-part fix

The three parts are independent and can be applied in any order. Workload metrics need Part A; control plane metrics need Part B. Part C is optional security hardening (replace `system:anonymous` with an SA token) — not required for either Part A or Part B to work.

### Part A — Datadog Agent env vars (one-time, fleet-wide)

Promotes vcluster syncer labels/annotations and Platform namespace labels onto every metric the Agent emits. **Without this, dashboards filtered/grouped by `vcluster_name` etc. return empty even though the labels are visible in Infrastructure → Kubernetes Explorer.**

See [`datadog-agent-cr.yaml`](./datadog-agent-cr.yaml) for the DatadogAgent CR form, or [`datadog-agent-helm-values.yaml`](./datadog-agent-helm-values.yaml) for the Helm-values form. Same effect either way.

Rolling restart of the Node Agent DaemonSet required for new tags to attach. New tags only appear on metrics ingested **after** the restart.

### Part B — Autodiscovery annotation in vcluster.yaml (per vcluster)

Tells the Datadog Agent to scrape the vcluster control plane `/metrics` endpoint. Produces `apiserver_*`, `etcd_*`, `workqueue_*`, `rest_client_*` metrics tagged with `vcluster.*` namespace prefix.

See [`vcluster-annotation-snippet.yaml`](./vcluster-annotation-snippet.yaml). Apply to each vcluster's vcluster.yaml — content is identical across vclusters; the Datadog Agent fills in pod-specific values via `%%host%%`.

Triggers a rolling restart of the vcluster control plane StatefulSet (~10-30s API server unavailability for single-replica; no downtime for HA control planes).

### Part C — SA-token replacement for `system:anonymous` (optional)

The annotation in Part B requires the Datadog Agent to authenticate to the vcluster apiserver. The lazy-but-insecure way is a `system:anonymous` ClusterRoleBinding on `/metrics` (let any unauthenticated request through). The clean way is an SA + token + projected volume mount.

See:
- [`auth-sa-rbac.yaml`](./auth-sa-rbac.yaml) — SA + ClusterRole + ClusterRoleBinding + token Secret, applied **inside** the vcluster.
- [`agent-token-mount.yaml`](./agent-token-mount.yaml) — DatadogAgent CR override that mounts the host-side mirrored Secret onto the Node Agent.
- [`vcluster-annotation-with-auth.yaml`](./vcluster-annotation-with-auth.yaml) — the annotation form that uses the mounted token.

The token mirroring step (extract from inside the vcluster, recreate as a Secret on the host in the Datadog Agent's namespace) is per-vcluster operational work. At fleet scale, automate via a small sync controller or build it into your vcluster provisioning pipeline.

## Lessons captured (lessons we'll forget if not written down)

### 1. The OpenMetrics check `bearer_token_auth: true` does NOT take a custom path

**Symptom:** check returns 403 even though the token at `bearer_token_path` works perfectly when used via `curl` from inside the same agent pod.

**Cause:** `bearer_token_auth: true` in the OpenMetrics v2 check is hardwired to use the kubelet's own SA token at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Setting `bearer_token_path` alongside it does NOT redirect the auth — it's silently ignored.

**Fix:** use the `auth_token` config block instead:

```json
{
  "auth_token": {
    "reader": {
      "type": "file",
      "path": "/etc/datadog-agent/vcluster-tokens/<vcluster-name>"
    },
    "writer": {
      "type": "header",
      "name": "Authorization",
      "value": "Bearer <TOKEN>"
    }
  }
}
```

The literal string `Bearer <TOKEN>` is a template — the agent reads the file from `reader.path` and substitutes the file's content for `<TOKEN>` at runtime.

This is documented (https://docs.datadoghq.com/integrations/openmetrics/) but not surfaced in most quick-start examples. Easy to miss.

### 2. Labels in Infrastructure Explorer ≠ tags on metrics

The Datadog Cluster Agent's resource collection sees pod/namespace labels and shows them in the Kubernetes Explorer's grouping/filter UI. This is **independent** of metric tagging. The Node Agent's metric tagger only attaches labels to metrics that are explicitly listed in `podLabelsAsTags` / `podAnnotationsAsTags` / `namespaceLabelsAsTags`.

Customers will say "I can see the label in Infrastructure but my dashboards are empty." That's exactly this — the Cluster Agent shows them, but the metric pipeline doesn't tag them.

### 3. Two redundant promotion paths exist for vcluster_name

vCluster Platform labels the host namespace with `loft.sh/vcluster-instance-name`. The vcluster syncer also labels every synced workload pod with `vcluster.loft.sh/managed-by` (= the same vcluster name). Both can be promoted to the same `vcluster_name` Datadog tag.

We promote **both** for redundancy:
- `namespaceLabelsAsTags: { "loft.sh/vcluster-instance-name": "vcluster_name" }` covers control-plane scrapes (since the control plane runs in the host namespace, not as a synced pod).
- `podLabelsAsTags: { "vcluster.loft.sh/managed-by": "vcluster_name" }` covers synced workload pods directly without going through namespace traversal.

If both are set, Datadog deduplicates and the same `vcluster_name` tag is applied with the correct value either way.

### 4. The "every pod becomes a host" trap doesn't apply here (mostly)

The OTel-exporter path has a known trap where collector pods register as ephemeral hosts in Datadog's Infrastructure list. The Datadog-Agent path doesn't have this issue because:
- The Node Agent runs as a DaemonSet (one pod per node, mapped 1:1 to nodes)
- The Agent's hostname detection automatically uses the underlying k8s node name
- Datadog Agent has built-in container/host scoping

**One exception:** if they install the Cluster Agent as multiple replicas, those replicas can show up as ephemeral hosts. Default install has a single Cluster Agent replica, so this is rare.

### 5. Custom metrics cost — the Agent has no equivalent of OTTL `keep_keys`

Our OTel collector path uses `transform/cardinality_guard` (OTTL `keep_keys` + `delete_matching_keys`) to drop high-cardinality labels before they become billable Datadog tags. The Datadog Agent's OpenMetrics check has no per-label drop mechanism — only metric-level `metrics:` allow-list and `exclude_metrics:` block-list.

Customers using the Agent path should:
1. Set a tight `metrics:` allow-list in the autodiscovery annotation (we ship one — `apiserver_*`, `etcd_*`, `workqueue_*`, etc.).
2. After ingestion, use Datadog UI's **Metrics without Limits** to drop high-cardinality tags from indexed (queryable) metrics. https://docs.datadoghq.com/metrics/metrics-without-limits/

### 6. Restart effects

| Change | Restarts | Impact |
|---|---|---|
| Operator CR / Helm values change for env vars | Node Agent DaemonSet rolling restart | None — new tags attach to metrics ingested after restart. No data loss. |
| vcluster.yaml annotation change | vcluster control plane StatefulSet rolling restart | Single-replica: ~10-30s vcluster apiserver downtime. HA: zero downtime. Workload pods inside the vcluster are not affected. |
| Adding `auth_token` to existing annotation | vcluster control plane rolling restart | Same as above — annotation change updates pod template. |

## Verification

After Part A applied:

```
sum:kubernetes.cpu.usage.total{vcluster_name:*} by {vcluster_name,vcluster_virtual_namespace}
```

Should return one series per (vcluster, virtual namespace) pair within ~2 minutes.

After Part B applied:

```
sum:vcluster.apiserver_request_total{vcluster_name:*} by {verb}
```

Should return data within ~1 minute of the StatefulSet rollover.

After Part C applied (token-based auth working):

```bash
# On the Node Agent on the same node as the vcluster control plane pod:
kubectl exec -n datadog <agent-pod> -c agent -- agent status | grep -A5 "openmetrics:vcluster"
```

Look for `[OK]` status (not `[ERROR]`) and a non-zero "Metric Samples" line.
