# 3 AM Triage Cheatsheet

## First 5 minutes
1. **What changed?** Deploys, config changes, upgrades, cert rotations, cloud provider status pages.
2. **How big is it?** One pod, one service, one node, one AZ, or everything?
3. **Who returns the error?** CDN, LB, ingress, app or database? Check headers and each layer's logs.
4. **Mitigate first, then find the root cause.** Roll back, scale up or fail over.

## Kubernetes
```bash
kubectl get pods -A | grep -vE "Running|Completed"
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous --tail=100
kubectl top nodes ; kubectl top pods -n <ns> --sort-by=memory
kubectl get nodes -o wide | grep -v " Ready"
kubectl rollout history deploy/<d> -n <ns> ; kubectl rollout undo deploy/<d> -n <ns>
kubectl get endpoints <svc> -n <ns>
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash   # network debug
```

## PromQL essentials
```promql
up == 0                                                           # dead targets
sum by (service) (rate(http_requests_total{code=~"5.."}[5m]))
  / sum by (service) (rate(http_requests_total[5m]))              # error ratio
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
increase(kube_pod_container_status_restarts_total[1h]) > 3
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}
rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])
(node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
prometheus_tsdb_head_series
rate(alertmanager_notifications_failed_total[5m]) > 0
```

## LogQL essentials
```logql
{namespace="prod", app="api"} |= "error" != "healthz"
{app="api"} | json | status >= 500 | line_format "{{.method}} {{.path}} {{.status}}"
sum by (app) (rate({namespace="prod"} |= "error" [5m]))
topk(10, sum by (app) (bytes_over_time({namespace=~".+"}[5m])))   # noisiest apps
```

## Exit codes
`0` finished · `1` app error · `126/127` entrypoint · `137` OOM/SIGKILL · `139` segfault · `143` SIGTERM

## HTTP codes by layer
`499` client gave up · `502` upstream closed or crashed (check keep-alive) · `503` no healthy endpoints · `504` upstream too slow · `429` rate limited (check retry storms)
