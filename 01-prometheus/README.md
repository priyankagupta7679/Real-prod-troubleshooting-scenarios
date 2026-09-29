# 01 · Prometheus Troubleshooting

---

### 1.1 Prometheus OOMKilled every few hours (cardinality explosion)
**Symptoms:** The Prometheus pod restarts with `OOMKilled`. Memory climbs steadily after a deploy, and dashboards show gaps.

**Check:**
```bash
kubectl describe pod prometheus-0 -n monitoring | grep -A3 "Last State"
# Top series by metric name
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.seriesCountByMetricName[:10]'
# Top label names by distinct values
curl -s localhost:9090/api/v1/status/tsdb | jq '.data.labelValueCountByLabelName[:10]'
```
```promql
topk(10, count by (__name__)({__name__=~".+"}))
prometheus_tsdb_head_series
```

**Root cause:** A team added a label with unbounded values (`user_id`, `request_id`, full URL path, `pod_ip`). Every unique value creates a new series, and each head series costs roughly 3–8 KB of RAM.

**Fix:**
- Immediate: drop the label at scrape time with `metric_relabel_configs`, then restart.
```yaml
metric_relabel_configs:
  - action: labeldrop
    regex: "user_id|request_id"
  - source_labels: [__name__]
    regex: "http_request_duration_seconds_bucket"
    action: drop        # if the histogram itself is the problem
```
- Permanent: fix the instrumentation. Use path templates (`/users/:id`), not raw paths.

**Prevention:** Set `sample_limit` / `label_limit` per scrape job. Alert on `prometheus_tsdb_head_series` growth, for example `> 1.5x` its 7-day average.

**Source:** Community pattern (one of the most common r/PrometheusMonitoring threads). See the [Prometheus docs on labels](https://prometheus.io/docs/practices/naming/#labels) and [Robust Perception on cardinality](https://www.robustperception.io/cardinality-is-key/).

---

### 1.2 Prometheus takes 20+ minutes to start (WAL replay)
**Symptoms:** After a restart, `/-/ready` returns 503 for a long time. The liveness probe kills the pod before replay finishes, so it replays again, in a loop.

**Check:**
```bash
kubectl logs prometheus-0 -c prometheus | grep -i "replay\|WAL"
du -sh /prometheus/wal
```

**Root cause:** A large head block (high series count) combined with slow EBS/PV IOPS. The liveness probe `initialDelaySeconds` is shorter than the replay time.

**Fix:** Use a `startupProbe` with a long `failureThreshold` (e.g. 60 × 10s) so liveness doesn't start until Prometheus is ready. As a last resort, move `wal/` aside and accept losing about 2 hours of data.

**Prevention:** Cut cardinality (1.1), use gp3 volumes with provisioned IOPS, and never put an aggressive liveness probe on a stateful TSDB.

**Source:** Community pattern; [prometheus/prometheus GitHub issues on WAL replay](https://github.com/prometheus/prometheus/issues?q=wal+replay+slow).

---

### 1.3 Target DOWN: `context deadline exceeded`
**Symptoms:** On `/targets` a target is red with `Get "http://10.x.x.x:8080/metrics": context deadline exceeded`.

**Check:**
```bash
kubectl exec -n monitoring prometheus-0 -- wget -qO- --timeout=5 http://<pod-ip>:<port>/metrics | head
time curl -s http://<pod-ip>:<port>/metrics | wc -l   # how big / how slow?
```

**Root cause (usually one of these):**
1. A NetworkPolicy or Security Group blocks the Prometheus → pod port.
2. The `/metrics` endpoint takes longer than `scrape_timeout` (default 10s) because it returns 500k lines.
3. The app binds to `127.0.0.1` instead of `0.0.0.0`.
4. On an IPv6 cluster the target address isn't bracketed (`[fd00::1]:8080`).

**Fix:** Open the port, raise `scrape_timeout` (it must stay ≤ `scrape_interval`), shrink the exposition, or fix the bind address.

**Prevention:** Alert on `up == 0` per job, with `for: 5m`.

**Source:** Community pattern.

---

### 1.4 `Error on ingesting out-of-order samples` / `duplicate sample for timestamp`
**Symptoms:** Prometheus logs are full of warnings, and `prometheus_target_scrapes_sample_out_of_order_total` keeps rising.

**Root cause:** Two scrape jobs produce **identical label sets**, for example the same pod discovered by both a `ServiceMonitor` and a `PodMonitor`, or two Prometheus HA replicas remote-writing without distinct `external_labels`.

**Fix:** Deduplicate the scrape configs. For HA pairs, set a unique `external_labels.replica` and deduplicate in Thanos/Mimir.

**Source:** Community pattern; [Prometheus operator FAQ](https://prometheus-operator.dev/).

---

### 1.5 `many-to-many matching not allowed: matching labels must be unique on one side`
**Symptoms:** A recording rule or alert that used to work starts failing after a new exporter version.

**Root cause:** The join (`on(...)` / `group_left`) now finds more than one series on the "one" side, because a new label such as `container` or `endpoint` appeared.

**Fix:**
```promql
# collapse the extra dimension before joining
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
  * on(namespace, pod) group_left(node)
max by (namespace, pod, node) (kube_pod_info)
```

**Prevention:** Unit-test rules with `promtool test rules` in CI.

---

### 1.6 Dashboards show absurd spikes (`rate()` misuse)
**Symptoms:** A request-rate graph shows 10M req/s for one point after a deploy.

**Root cause:** One of these:
- `rate()` applied to a **gauge**.
- `rate(sum(...))` instead of `sum(rate(...))`. Summing first hides counter resets.
- A range window shorter than 2× the scrape interval.

**Fix:** Always write `sum(rate(counter[5m]))`, keep the window at 4× the scrape interval or more, and only use `rate`/`increase` on `_total` counters.

**Source:** [Prometheus docs: rate()](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate).

---

### 1.7 Alert storm, flapping every scrape
**Symptoms:** The Teams/Slack channel gets hundreds of FIRING/RESOLVED pairs.

**Root cause:** The rule has no `for:`, the threshold sits right at normal load, or it alerts on individual pods that restart routinely.

**Fix:** Add `for: 5m`, add hysteresis (a separate lower threshold for resolving), alert on **symptoms at the workload level** (e.g. *all replicas down*, error-rate SLO burn) rather than per-pod causes, and set `group_by` / `group_interval` / `repeat_interval` in Alertmanager.

**Source:** [Google SRE book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/). Also see field note [7.5](../07-field-notes/README.md).

---

### 1.8 Alertmanager shows alerts, but nobody gets notified
**Symptoms:** The alert shows as active in the Alertmanager UI, but nothing arrives in Slack/Teams/PagerDuty.

**Check:**
```bash
kubectl logs alertmanager-0 -c alertmanager | grep -i "notify\|error\|template"
```
```promql
rate(alertmanager_notifications_failed_total[5m]) > 0
alertmanager_notification_latency_seconds
```

**Root cause (common):** An expired or rotated webhook URL, a template rendering error, the route matching a `null` receiver first, the alert being inhibited, or a silence someone forgot about.

**Fix:** Test the route with `amtool config routes test --config.file=alertmanager.yml severity=critical`, fire a synthetic test alert, and check the webhook with `curl`.

**Prevention:** Run an always-firing **Watchdog/DeadMansSwitch** alert, routed to a heartbeat service that pages when it *stops* arriving. Alert on `alertmanager_notifications_failed_total`.

---

### 1.9 Remote-write falling behind (Thanos/Mimir/VictoriaMetrics)
**Symptoms:** The long-term store lags real time by several minutes, and `prometheus_remote_storage_samples_pending` keeps rising.

**Check:**
```promql
prometheus_remote_storage_highest_timestamp_in_seconds - ignoring(remote_name, url) group_right prometheus_remote_storage_queue_highest_sent_timestamp_seconds
rate(prometheus_remote_storage_samples_failed_total[5m])
```

**Root cause:** The receiver returns 429/5xx, `max_shards` is too low, or there is network egress throttling.

**Fix:** Tune `queue_config` (`max_shards`, `capacity`, `max_samples_per_send`) and scale the receiver. Once remote-write falls behind the WAL retention window, samples are lost.
