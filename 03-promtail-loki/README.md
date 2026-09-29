# 03 · Promtail / Loki Troubleshooting

> **Note:** Promtail is deprecated. Grafana Labs moved it to LTS and recommends migrating to **Grafana Alloy** (`alloy convert --source-format=promtail`). Every scenario below applies equally to Alloy's `loki.source.*` components.

---

### 3.1 Promtail running, but no logs in Loki
**Symptoms:** Promtail pods are `Running` and Grafana Explore returns nothing for `{namespace="app"}`.

**Check (in this order):**
```bash
# 1. Is promtail discovering targets?
kubectl port-forward ds/promtail 9080 -n logging &
curl -s localhost:9080/targets | grep -c ready
# 2. Is it reading files?
kubectl exec ds/promtail -n logging -- ls /var/log/pods | head
# 3. Is it pushing / getting errors?
kubectl logs ds/promtail -n logging | grep -iE "error|429|500|dropping"
# 4. Metrics
curl -s localhost:9080/metrics | grep -E "promtail_sent_entries_total|promtail_dropped_entries_total|promtail_request_duration_seconds_count"
```

**Root cause (in order of frequency):**
1. `/var/log/pods` (or `/var/lib/docker/containers`) is **not hostPath-mounted** into the DaemonSet.
2. The `__path__` relabel is wrong, so no files match.
3. Promtail has no tolerations for tainted nodes, so it doesn't run on the nodes where the app runs.
4. The Loki URL is wrong or `X-Scope-OrgID` (tenant) is missing in multi-tenant Loki, which returns `401 no org id`.
5. The positions file claims everything is already read (see 3.6).

---

### 3.2 Logs arrive but JSON fields / levels are garbage (CRI vs docker format)
**Symptoms:** Each line looks like `2024-01-01T10:00:00.1Z stdout F {"level":"error"...}` and `| json` fails.

**Root cause:** Since Kubernetes 1.24 (dockershim removal), nodes use containerd, which writes the **CRI** log format. The pipeline still uses the `docker: {}` stage.

**Fix:**
```yaml
pipeline_stages:
  - cri: {}        # not docker: {}
  - json:
      expressions: { level: level, msg: message }
  - labels: { level: }
```

---

### 3.3 `entry too far behind` / `entry out of order`
**Symptoms:** Loki returns `400` and Promtail logs show `entry with timestamp ... ignored, reason: 'entry too far behind'`.

**Root cause:** Promtail restarted after a long outage and is replaying old files, or a `timestamp` pipeline stage parsed a wrong time zone so entries land hours in the past.

**Fix:** Loki 2.4+ accepts out-of-order writes within `max_chunk_age / 2` (default 1h). For older backfill, set `reject_old_samples_max_age` and check the time zone (`location:`) in the timestamp stage.

---

### 3.4 `429 Too Many Requests: ingestion rate limit exceeded`
**Symptoms:** Promtail logs are full of 429s and log lines are dropped during traffic spikes.

**Check:**
```promql
sum by (tenant, reason) (rate(loki_discarded_samples_total[5m]))
```

**Root cause:** The per-tenant `ingestion_rate_mb` / `ingestion_burst_size_mb` limits are too low, or one noisy app is logging in DEBUG at 50 MB/s.

**Fix:** Find the noisy stream:
```logql
topk(10, sum by (namespace, app) (bytes_over_time({namespace=~".+"}[5m])))
```
Drop or sample it at the agent (`drop` stage with `expression: ".*healthz.*"`), or raise the limits in `limits_config`.

---

### 3.5 `Maximum active stream limit exceeded` (label cardinality)
**Symptoms:** Loki refuses new streams, and some apps' logs disappear completely.

**Root cause:** High-cardinality **labels**: `pod_ip`, `request_id`, `trace_id` or `filename` set as a label. Loki indexes labels, so every unique combination becomes a stream.

**Fix:** Keep labels to about 10–15 low-cardinality values (namespace, app, container, level). Put IDs in the log **body** and query them with `|= "trace_id=abc"` or `| json | trace_id="abc"`. Loki 3.x *structured metadata* is the right place for trace IDs.

**Source:** [Grafana: label best practices](https://grafana.com/docs/loki/latest/get-started/labels/bp-labels/).

---

### 3.6 Duplicate logs, or missing logs after a Promtail restart
**Symptoms:** After a Promtail DaemonSet rollout, logs appear twice, or a gap appears.

**Root cause:** The positions file (`/run/promtail/positions.yaml`) is on an **emptyDir**. It is lost on restart, so Promtail rereads from the start (duplicates) or from the end (gap).

**Fix:** Mount the positions path from a **hostPath** (e.g. `/var/lib/promtail`) so it survives pod restarts.

---

### 3.7 Grafana Explore: `max entries limit per query exceeded`
**Root cause:** The query returns more lines than `max_entries_limit_per_query` (default 5000).

**Fix:** Narrow the time range or stream selector, or aggregate instead of listing: `sum by (level) (count_over_time({app="api"}[5m]))`.

---

### 3.8 Dashboard is empty even though Loki has the logs (label value mismatch)
**Symptoms:** `{container="payments"}` returns nothing, but `{namespace="prod"}` shows the payments logs.

**Root cause:** The dashboard hardcodes a label value that doesn't match the **real** container name (e.g. it expects `payments`, but the Deployment's container is called `payments-scheduler`).

**Fix:** Run `logcli labels container` or check *Explore → Label browser* and correct the value. Use regex (`container=~"payments.*"`) or a template variable instead of a hardcoded value.

**Source:** Field experience (see [7.3](../07-field-notes/README.md)).
