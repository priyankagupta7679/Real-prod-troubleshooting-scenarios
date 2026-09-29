# 07 · Field Notes (anonymised, from real production)

These come from running monitoring and alerting for several production Kubernetes environments (EKS, Prometheus/Alertmanager, Grafana, Loki, Zabbix). Names are removed, but the failure modes are exactly what happened.

---

### 7.1 Alert channels silent for weeks: empty notification templates
- **What we saw:** New per-category alert channels (Infra / DB / App) were created and routed. Rules fired and the UI showed "sent". **Nothing ever reached the chat channel.**
- **Root cause:** The new notification media types had **empty message templates**. The platform rendered an empty body and the webhook rejected it silently. Separately, a Grafana contact point template contained **stray backslashes**, which broke rendering on every environment.
- **Why nobody noticed:** The "Test" button uses a default template, so it passed.
- **Fix:** Fill in and validate the templates, then fire real alerts through each route and confirm they're **received**, not just "sent".
- **Lesson:** An alert that has never been observed arriving should count as broken. Run a daily canary alert through every route.

### 7.2 A crashing monitoring sidecar starved Prometheus and caused SLA breaches
- **What we saw:** Many days of intermittent "service down" SLA breaches that looked like application failures.
- **Root cause:** The `prometheus-msteams` webhook bridge had **no resource limits**, and its downstream (a Power Automate webhook) was broken. It retried in a loop, grew without bound, and **starved the node running Prometheus**. Prometheus then missed scrapes, which the breach calculation read as service downtime.
- **Fix:** Set requests and limits on the sidecar, then moved alerting to Alertmanager's native `msteamsv2_configs` and removed the bridge.
- **Lesson:** Give observability components limits, alerts and node isolation of their own. **Monitoring that has no monitoring of its own becomes a single point of failure.** (Compare OpenAI in [6.3](../06-real-world-postmortems/README.md).)

### 7.3 Log dashboard empty for one service: container label mismatch
- **What we saw:** The Loki dashboard for one service showed no application logs, while every other service was fine.
- **Root cause:** The dashboard filtered on `container="svc-internal"`, but the real container name was `svc-scheduler-internal`.
- **Fix:** Check the real label values in the Explore label browser, then switch the dashboard to regex or template variables.
- **Lesson:** "No data" is often a *query* bug and not a pipeline bug. Check label values before you debug Promtail.

### 7.4 Hand-patched fix silently at risk from the next `helm upgrade`
- **What we saw:** Resource limits and an Alertmanager config fix had been applied with `kubectl edit` / `kubectl patch` on a Helm-managed stack.
- **Risk:** The next `helm upgrade kube-prometheus-stack` renders the chart's values again and **reverts the fix**, which brings the outage back.
- **Fix:** Put every change into `values.yaml` in Git, and use `helm diff upgrade` in CI to catch drift.
- **Lesson:** A hotfix is only finished once it's in the source of truth.

### 7.5 Alert fatigue from per-pod rules: alert on the workload, not the pod
- **What we saw:** `KubePodCrashLooping` / `KubePodNotReady` / `ReplicasMismatch` fired dozens of times a day during normal rollouts and single-pod restarts, and on-call started ignoring the channel.
- **Fix:** Replaced them with a workload-level rule that fires only when **all replicas** of a Deployment or StatefulSet are unavailable:
```yaml
- alert: WorkloadAllPodsDown
  expr: |
    (kube_deployment_status_replicas_available == 0 and kube_deployment_spec_replicas > 0)
  for: 3m
  labels: { severity: critical }
```
  The noisy per-pod rules were null-routed in Alertmanager.
- **Lesson:** Page on user-facing symptoms. Single-pod churn is the reason you run replicas.

### 7.6 Deleted "unused" CI/CD cleanup: know what's live before you delete
- **What we saw:** During a cost cleanup of old pipelines and images, a few deployments still pulled images from repositories that were about to be deleted.
- **Fix:** Before deleting anything, map every running pod's image to its registry: `kubectl get pods -A -o jsonpath='{..image}' | tr ' ' '\n' | sort -u`.
- **Lesson:** Only call something unused once you've checked the running workloads, not the last-commit date.

### 7.7 A 1.9 GB monitoring server hit memory and disk limits twice in one month
- **What we saw:** The self-hosted monitoring server returned 504s. It was undersized: 1.9 GB RAM, no swap, and a 20 GB disk.
- **Fix:** Tuned PHP-FPM workers, added 2 GB of swap, and vacuumed journald (`journalctl --vacuum-size=200M`), then set disk and memory alerts **on the monitoring box itself**.
- **Lesson:** Monitor the monitor, and alert from somewhere other than the box you're watching.
