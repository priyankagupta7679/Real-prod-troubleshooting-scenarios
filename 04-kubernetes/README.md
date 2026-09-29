# 04 · Kubernetes Troubleshooting

---

### 4.1 CrashLoopBackOff
**Check (in order):**
```bash
kubectl describe pod <pod> | sed -n '/Last State/,/Ready/p'   # exit code + reason
kubectl logs <pod> --previous                                  # logs of the crashed container
kubectl get events --sort-by=.lastTimestamp | tail -20
```
**Read the exit code first:**

| Exit code | Meaning | Typical cause |
|---|---|---|
| 0 | Process finished | The container runs a script, not a long-running server (wrong CMD) |
| 1 | App error | Missing env var/secret, bad config, DB unreachable at startup |
| 126 / 127 | Not executable / not found | Wrong entrypoint, missing binary, CRLF line endings in shell script |
| 137 | SIGKILL | **OOMKilled**, or liveness probe kill |
| 139 | SIGSEGV | Native crash, wrong CPU arch image (arm64 vs amd64) |
| 143 | SIGTERM | Normal shutdown. If it happens constantly, the probe is killing the pod |

**Fix:** Fix the underlying cause. To debug a container that won't stay up: `kubectl debug <pod> -it --copy-to=dbg --container=app -- sh`.

---

### 4.2 OOMKilled (exit 137)
**Symptoms:** `Reason: OOMKilled` and the restart count keeps rising.

**Check:**
```promql
container_memory_working_set_bytes{pod=~"api.*"} / on(pod,container) kube_pod_container_resource_limits{resource="memory"}
```
**Root cause:** The memory limit is below real peak usage, there is a leak, or a JVM/Node runtime doesn't respect the container limit (see 5.8).

**Fix:** Raise the limit based on the p99 working set plus 20–30% headroom, then profile for leaks. Set **requests = limits for memory** on critical services to avoid node-level OOM surprises.

---

### 4.3 ImagePullBackOff / ErrImagePull
**Check:** `kubectl describe pod` → the Events section shows the exact error.

| Message | Cause |
|---|---|
| `manifest unknown` / `not found` | Wrong tag, or the tag was deleted (ECR lifecycle policy!) |
| `unauthorized` / `no basic auth credentials` | Missing `imagePullSecrets`, or the node IAM role lacks ECR pull permission |
| `toomanyrequests` | Docker Hub rate limit. Mirror images into your own registry |
| `i/o timeout` | Private nodes without a NAT gateway or VPC endpoint to the registry |

**Prevention:** Deploy **immutable tags or digests** and never `:latest`. Make sure ECR lifecycle rules never expire images that are currently deployed.

---

### 4.4 Pod stuck in Pending
```bash
kubectl describe pod <pod> | grep -A10 Events
```

| Event | Fix |
|---|---|
| `0/5 nodes available: insufficient cpu` | Requests are too high, or the Cluster Autoscaler/Karpenter is at its max |
| `node(s) had untolerated taint` | Add tolerations, or check the node group |
| `node(s) had volume node affinity conflict` | The EBS PV is in AZ-a but the free nodes are in AZ-b. Use `WaitForFirstConsumer` in the StorageClass |
| `pod has unbound immediate PersistentVolumeClaims` | Wrong StorageClass name, or the CSI driver is missing |
| `didn't match pod anti-affinity rules` | Hard anti-affinity with more replicas than nodes |

---

### 4.5 Random 5-second DNS delays (conntrack race)
**Symptoms:** p99 latency shows **exactly** 5s spikes on outbound calls, and `curl` sometimes hangs on name resolution.

**Root cause:**
1. A Linux conntrack race in which parallel A and AAAA UDP queries from glibc share one socket, one gets dropped, and the resolver waits for its 5s timeout.
2. `ndots:5` makes every external lookup try five search domains first, which multiplies the query volume.

**Fix:**
- Deploy **NodeLocal DNSCache**.
- Add `dnsConfig.options: [{name: single-request-reopen}]` (glibc) or set `ndots: 2`.
- Use FQDNs with a trailing dot (`api.example.com.`).

**Source:** [Weaveworks: Racy conntrack and DNS lookup timeouts](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts); [XING engineering](https://tech.xing.com/a-reason-for-unexplained-connection-timeouts-on-kubernetes-docker-abd041cf7e02); [Kubernetes docs: NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/).

---

### 4.6 High latency even though CPU usage looks low (CFS throttling)
**Symptoms:** The service shows 30% CPU usage but its p99 latency is terrible.

**Check:**
```promql
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total[5m]))
  / sum by (pod) (rate(container_cpu_cfs_periods_total[5m]))   # > 25% = problem
```
**Root cause:** CPU **limits** are enforced per 100ms CFS period. A multi-threaded app burns its quota in the first few milliseconds, then waits for the rest of the period. Averaged over a minute, usage looks low.

**Fix:** Remove CPU limits and keep accurate CPU **requests** for latency-critical services, or raise the limits. Match runtime thread pools to the CPU count (`GOMAXPROCS` via `automaxprocs`, and `-XX:ActiveProcessorCount` for the JVM).

**Source:** [Buffer: Make your services faster by removing CPU limits](https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/); [Zalando talk via k8s.af](https://k8s.af/).

---

### 4.7 Node NotReady
```bash
kubectl describe node <node> | sed -n '/Conditions/,/Addresses/p'
# on the node (SSM/SSH):
journalctl -u kubelet --since "30 min ago" | tail -50
df -h ; free -m ; systemctl status containerd
```
**Common causes:** `PLEG is not healthy` (container runtime overloaded or hung), disk full from images or logs, kubelet cert expiry, memory exhaustion without `system-reserved`, or the CNI losing routes (see Datadog in [06](../06-real-world-postmortems/README.md)).

**Fix:** Cordon, drain and replace the node rather than fixing it by hand. Set `--kube-reserved` / `--system-reserved` and image GC thresholds.

---

### 4.8 Pods Evicted (`The node was low on resource: ephemeral-storage`)
**Root cause:** An app writes big files to its container filesystem or an `emptyDir`, or container logs aren't rotated.

**Fix:** Set `resources.limits.ephemeral-storage`, use `emptyDir.sizeLimit`, send logs to stdout (the kubelet rotates them), and delete leftover Evicted pods: `kubectl delete pod --field-selector=status.phase=Failed -A`.

---

### 4.9 EKS: pods stuck `ContainerCreating`, "failed to assign an IP address"
**Root cause:** The AWS VPC CNI gives every pod a real VPC IP. Either the subnet ran out of IPs, or the node hit its ENI/IP limit for the instance type.

**Fix:** Enable **prefix delegation** (`ENABLE_PREFIX_DELEGATION=true`), add secondary CIDR subnets (100.64.0.0/10), or use larger instance types.

**Prevention:** Alert on available IPs per subnet (CloudWatch or `awscni_` metrics).

---

### 4.10 HPA not scaling: `<unknown>/70%`
**Root cause:** metrics-server isn't installed or healthy, the pods have **no CPU requests** (HPA percentages are computed against requests), or the custom-metrics adapter is misconfigured.

**Check:** Run `kubectl top pods` (if this fails, metrics-server is the problem) and `kubectl describe hpa`.

---

### 4.11 Readiness/liveness probes cause a cascading outage
**Symptoms:** A small DB slowdown turns into a complete outage and every pod restarts.

**Root cause:** The **liveness** probe checks a downstream dependency (DB, Redis). When the dependency slows down, every pod fails liveness at the same moment, Kubernetes restarts them all, and the cold starts pile even more load on the dependency.

**Fix:**
- Liveness checks only "is my process alive" (a cheap `/healthz`).
- Readiness *may* check dependencies.
- Use a generous `timeoutSeconds` / `failureThreshold`, and use a `startupProbe` for slow boots.

**Source:** Community pattern; [Colin Breck: Kubernetes Liveness and Readiness Probes: How to Avoid Shooting Yourself in the Foot](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/).

---

### 4.12 Namespace stuck in `Terminating`
```bash
kubectl get ns <ns> -o json | jq '.status.conditions'
kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get -n <ns> --ignore-not-found --show-kind
```
**Root cause:** A resource has a finalizer whose controller is gone (e.g. an uninstalled operator), or an APIService is unavailable (a broken `metrics.k8s.io` blocks discovery).

**Fix:** Fix or remove the broken APIService first. Remove a finalizer only on the specific stuck object, and only once you understand what cleanup it was guarding (e.g. a cloud load balancer or volume that will otherwise be leaked).
