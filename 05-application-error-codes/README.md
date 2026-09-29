# 05 · Application & HTTP Error Codes

**Rule 1:** Find out **who** generated the error. A 502 from the ALB, from nginx-ingress and from your app are three different problems. Check the `Server` header, the response body, and which layer's logs contain the request.

---

### 5.1 502 Bad Gateway, intermittent and low-rate (~0.1%)
**Symptoms:** Random 502s with no pattern, usually right after a period with no traffic on a connection.

**Root cause:** **Keep-alive timeout mismatch.** The upstream app (Node.js defaults to 5s, gunicorn to 2s) closes an idle connection *just as* the LB (ALB idle timeout 60s) or nginx reuses it. The LB sends a request down a socket that is already closed.

**Fix:** The app's keep-alive timeout must be **longer** than the proxy's:
```js
server.keepAliveTimeout = 65000;   // > ALB 60s
server.headersTimeout  = 66000;
```
For nginx upstream, set `keepalive_timeout` below the app's timeout.

**Source:** Well-documented community pattern. See [the AWS ALB troubleshooting doc on HTTP 502](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html).

---

### 5.2 502/503 spikes during every deployment
**Root cause:** When a pod terminates, Kubernetes removes the endpoint **asynchronously**, so kube-proxy, ingress controllers and the ALB keep sending traffic for a few seconds after SIGTERM. If the app exits immediately, those in-flight requests fail.

**Fix:**
```yaml
lifecycle:
  preStop:
    exec: { command: ["sleep", "15"] }
terminationGracePeriodSeconds: 45
```
The app should also handle SIGTERM gracefully: stop accepting new requests and drain the in-flight ones. Use `maxUnavailable: 0` in the rollout strategy, set a readiness probe, and add a PodDisruptionBudget.

---

### 5.3 503 Service Unavailable, `no healthy upstream` / `no endpoints available`
**Check:** Run `kubectl get endpoints <svc>`. If it's empty, work through these causes:
- The Service `selector` doesn't match the pod labels (a typo, or labels changed during a refactor).
- Every pod is failing readiness.
- `targetPort` points to the wrong port or port name.

---

### 5.4 504 Gateway Timeout
**Root cause:** The upstream took longer than the proxy timeout (ALB 60s, nginx `proxy_read_timeout` 60s). Usual culprits are a slow DB query, a lock, a synchronous call to a slow third party, or thread pool starvation.

**Fix:** Profile the slow path, add DB indexes, and move long-running work to async jobs with polling or webhooks. Raise the timeout only as a stop-gap.

---

### 5.5 499 Client Closed Request (nginx)
**Meaning:** The **client** gave up before nginx could respond. This is often a mobile app's 10s timeout or a load balancer health check timing out.

**Action:** Treat a rise in 499s as a **latency** problem on your side, not as client noise.

---

### 5.6 429 Too Many Requests / retry storms
**Symptoms:** A short dependency blip turns into a sustained outage.

**Root cause:** Clients retry immediately with no backoff or jitter, which multiplies load by 3–5×. When the dependency recovers, the queued retries knock it straight back down.

**Fix:** Use exponential backoff with **jitter**, retry budgets, circuit breakers and load shedding at the edge.

**Source:** Google Cloud's June 2025 incident report cites the lack of randomized exponential backoff as prolonging recovery in us-central1 (see [06](../06-real-world-postmortems/README.md)). Also see the [AWS Builders' Library: Timeouts, retries and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).

---

### 5.7 `connection reset by peer` / `ECONNRESET` / `broken pipe`
**Common causes:**
- A NAT gateway or conntrack idle timeout (350s on AWS NAT GW) silently drops a long-idle DB connection, and the pool hands it out anyway.
- An upstream was OOMKilled mid-request.
- An MTU mismatch (VPN, overlay networks).

**Fix:** Enable TCP keepalive shorter than the NAT timeout, turn on pool connection validation (`testOnBorrow`, `pool_pre_ping=True`), and set a max connection lifetime.

---

### 5.8 Java app OOMKilled even though `-Xmx` "looks fine"
**Root cause:** Heap is only part of JVM memory. Metaspace, thread stacks, direct buffers (Netty) and the code cache sit on top of it, so `-Xmx` equal to the container limit guarantees an OOMKill. Old JVMs (before 8u191) don't detect cgroup limits at all.

**Fix:** Use `-XX:MaxRAMPercentage=70` instead of a hard-coded `-Xmx`. Check the full breakdown with `-XX:NativeMemoryTracking=summary`.

---

### 5.9 Database connection pool exhaustion
**Symptoms:** You see `Timeout waiting for connection from pool` / `too many clients already` and latency spikes across every endpoint.

**Root cause:** HPA scaled the app from 5 to 40 pods, and each pod has a pool of 20, so 800 connections hit a Postgres with `max_connections=200`. Other causes are leaked connections that are never closed and long transactions holding connections open.

**Fix:** Use **PgBouncer** or RDS Proxy. Size the pool so that `pods_max × pool_size < db_max_connections`. Alert on `pg_stat_activity` counts and on pool-wait metrics.

---

### 5.10 Everything breaks at the same moment: TLS certificate expired
**Symptoms:** `x509: certificate has expired or is not yet valid`, `SSL_ERROR_EXPIRED_CERT_ALERT`.

**Real examples:** The Let's Encrypt **DST Root CA X3** expiry (Sept 2021) broke older clients worldwide. Microsoft Teams (Feb 2020) and many others have gone down because of an expired certificate.

**Fix:** Renew, and restart pods that cache certificates. Use cert-manager for automatic renewal.

**Prevention:** Alert on `probe_ssl_earliest_cert_expiry - time() < 86400 * 21` (blackbox exporter) and on cert-manager's `certmanager_certificate_expiration_timestamp_seconds`.
