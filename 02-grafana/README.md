# 02 · Grafana Troubleshooting

---

### 2.1 Every panel shows "No data" after importing a dashboard
**Symptoms:** An imported JSON dashboard (from grafana.com or another environment) loads, but every panel is empty. Panel edit shows `Datasource ${DS_PROMETHEUS} was not found`.

**Root cause:** The panels point to a datasource **UID** that doesn't exist in this Grafana instance, or to an unresolved `__inputs` variable.

**Fix:**
```bash
# find the real UID
curl -s -u admin:$PW http://grafana:3000/api/datasources | jq '.[] | {name,uid,type}'
# replace in JSON
sed -i 's/"uid": "OLD_UID"/"uid": "NEW_UID"/g' dashboard.json
```
Better: add a `datasource` template variable (type `prometheus`) and reference `${datasource}` in every panel.

**Prevention:** Provision datasources with **fixed UIDs** through YAML so the UID is identical in every environment.

---

### 2.2 `Templating init failed` / variables stuck loading
**Symptoms:** A red banner appears at the top of the dashboard, and the dropdowns are empty.

**Root cause:** A variable query is invalid for the datasource (e.g. `label_values()` against Loki), a chained variable references one that fails first, or the query times out on a large label set.

**Fix:** Open *Dashboard settings → Variables* and preview each query. Scope `label_values(kube_pod_info{namespace="$namespace"}, pod)` narrowly, and set *Refresh: On time range change* instead of *On load* for heavy queries.

---

### 2.3 Dashboard takes 30+ seconds and times out
**Symptoms:** Panels spin and then fail with `context deadline exceeded` or `504`.

**Check:** The Query inspector shows query time, the number of series returned and the data points.

**Root cause:** A 30-day range at 15s resolution means millions of points. Other causes are missing recording rules for heavy aggregations, too many panels firing queries at once, and a Loki query without a stream selector that scans everything.

**Fix:**
- Set **Min interval** / **Max data points** on panels.
- Precompute with **recording rules** (e.g. `job:http_requests:rate5m`).
- Collapse rows so panels lazy-load.
- For Loki, always start with a tight stream selector `{namespace="x", app="y"}` before any `|=` filters.

---

### 2.4 Grafana behind nginx/ALB: 502, redirect loop, or broken assets
**Symptoms:** The login page redirects to `localhost:3000`, CSS doesn't load, or you get `origin not allowed` after login.

**Root cause:** `root_url` / `serve_from_sub_path` don't match the proxy path, or the proxy isn't passing `Host`/`X-Forwarded-*` headers. Grafana Live WebSockets are blocked.

**Fix:**
```ini
[server]
domain = grafana.example.com
root_url = https://grafana.example.com/grafana/
serve_from_sub_path = true
```
nginx: `proxy_set_header Host $host;` plus `Upgrade`/`Connection` headers for `/api/live/`.

---

### 2.5 `database is locked` (SQLite backend)
**Symptoms:** Saving dashboards or alert rules fails intermittently. Logs show `database is locked`.

**Root cause:** The default SQLite database is on a network PV (EFS/NFS), or more than one Grafana replica shares the same SQLite file.

**Fix:** For HA or more than one replica, move to **PostgreSQL/MySQL** (`[database] type=postgres`). With a single replica, keep SQLite on block storage (EBS), not NFS.

**Source:** Community pattern; [Grafana docs: HA setup](https://grafana.com/docs/grafana/latest/setup-grafana/set-up-for-high-availability/).

---

### 2.6 Grafana alerts fire `DatasourceNoData` / `DatasourceError` at night
**Symptoms:** You get pages titled `DatasourceNoData` for rules that have nothing to do with an outage.

**Root cause:** The query legitimately returns nothing (e.g. an error counter with no errors, or a job that runs hourly), and the rule's *No data handling* defaults to `NoData` → alerting.

**Fix:** Set **"Alert state if no data" = OK** (or Normal) for sparse metrics. Wrap the query: `sum(rate(errors_total[5m])) or vector(0)`. Keep `DatasourceError` routed separately, because that one *is* a real monitoring outage.

---

### 2.7 Contact point "Test" works, but real alerts never arrive
**Symptoms:** Clicking *Test* on the Teams/Slack contact point delivers a message, but real firing alerts deliver nothing.

**Root cause:** The *Test* button renders a **default** template, while real alerts go through your **custom message template**. If that template is empty, has a syntax error (stray `\`, unclosed `{{ }}`) or references a missing field, rendering fails and the notification is dropped. The only sign is a line in the Grafana logs.

**Check:**
```bash
kubectl logs deploy/grafana | grep -i "template\|failed to send\|notify"
```

**Fix:** Validate the template with the preview in *Alerting → Contact points → Templates*, and fire a **real** test rule (e.g. `vector(1) > 0`) through the full routing tree.

**Prevention:** Run a synthetic "canary" alert that fires daily through every contact point.

**Source:** Field experience (see [7.1](../07-field-notes/README.md)); [Grafana alerting templating docs](https://grafana.com/docs/grafana/latest/alerting/configure-notifications/template-notifications/).
