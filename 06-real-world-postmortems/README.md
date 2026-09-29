# 06 · Real-World Postmortems (public, sourced)

Each entry is summarised from the company's **own published postmortem**. Read the originals, because they're far better than any summary.
Wider catalogue: [k8s.af: Kubernetes Failure Stories](https://k8s.af/) (curated by Henning Jacobs, Zalando).

---

### 6.1 Reddit: "The Pi-Day Outage" (14 Mar 2023, ~314 min)
- **Trigger:** An upgrade from Kubernetes 1.23 to 1.24.
- **Root cause:** Kubernetes 1.24 removed the `node-role.kubernetes.io/master` label (renamed to `control-plane`). Calico **route reflectors** had been configured years earlier to select nodes by that label. After the upgrade no route reflectors matched, BGP route distribution stopped, and pod-to-pod networking died.
- **What made it worse:** The route reflector configuration was undocumented, had been set up by engineers who had since left, and wasn't managed as code. Restoring from backup took hours.
- **Lessons:** Grep your whole cluster config for deprecated labels before an upgrade (`kubectl get <all> -A -o yaml | grep "node-role.kubernetes.io/master"`). Treat undocumented infrastructure as a risk to fix, and keep a *tested* rollback path.
- **Source:** "You Broke Reddit: The Pi-Day Outage" on r/RedditEng ([discussion + link](https://lobste.rs/s/l8m6xb/r_redditeng_you_broke_reddit_pi_day_outage), [summary](https://postmortem.io/incidents/reddit--2023-03-21--pi-day-outage/))

### 6.2 Datadog: multi-region outage (8 Mar 2023, ~24h to full recovery)
- **Trigger:** An automatic Ubuntu **systemd security update** was applied to tens of thousands of VMs in the same window.
- **Root cause:** When systemd-networkd restarted, it **deleted routes it didn't manage**. Those were Cilium's CNI routes, so the nodes dropped off the network in five regions across three clouds. A fresh boot didn't show the problem because networkd starts before Cilium installs its routes.
- **Lessons:** Unattended OS upgrades are production changes, so stage them. Set `ManageForeignRoutes=no` / `ManageForeignRoutingPolicyRules=no` when a CNI owns the routes. Make sure your recovery doesn't depend on the system that is down.
- **Source:** [Datadog: 2023-03-08 incident](https://www.datadoghq.com/blog/2023-03-08-multiregion-infrastructure-connectivity-issue/), [Pragmatic Engineer deep dive](https://newsletter.pragmaticengineer.com/p/inside-the-datadog-outage)

### 6.3 OpenAI: new telemetry service took down every Kubernetes control plane (11 Dec 2024, ~4h)
- **Trigger:** A new **observability/telemetry** service was deployed fleet-wide.
- **Root cause:** On large clusters, every node ran agents making expensive Kubernetes **API calls**, which overloaded the API servers. Kubernetes DNS-based service discovery depended on the control plane, so services failed once their DNS caches expired. The caching also hid the problem long enough for the rollout to reach every cluster.
- **What made it worse:** The rollback needed `kubectl`, and `kubectl` needed the dead API server. Engineers had to scale down, block network access to the API server, and use out-of-band access.
- **Lessons:** **The monitoring stack can take down production.** Load-test agents against a cluster at *production size*, roll out one cluster at a time, and keep break-glass access that doesn't depend on the control plane.
- **Source:** [OpenAI status: incident report](https://status.openai.com/incidents/ctrsv3lwd797)

### 6.4 Cloudflare: bot-management feature file panics the core proxy (18 Nov 2025)
- **Trigger:** A ClickHouse **permissions change**.
- **Root cause:** After the change, the query that generates the Bot Management "feature file" returned **duplicate rows**, doubling the file past a hard limit of 200 features. The proxy code hit that limit and **panicked** (a Rust `unwrap()` on an error), causing widespread 5xx. The file was regenerated every few minutes and only some ClickHouse nodes had been updated, so the network *flapped* between good and bad. That initially looked like an attack.
- **Lessons:** Validate generated config like user input. A failure in a non-critical module must fail open or fall back to the last good version instead of crashing the request path. Kill switches need to be quick to use.
- **Source:** [Cloudflare blog: 18 Nov 2025 outage](https://blog.cloudflare.com/18-november-2025-outage/)

### 6.5 Google Cloud: Service Control null pointer (12 Jun 2025, ~7h)
- **Trigger:** A quota **policy with blank fields** was written to Spanner and replicated globally within seconds.
- **Root cause:** Code added two weeks earlier had no error handling for those fields and **no feature flag**. It crashed Service Control binaries in every region at once, and the API management failure spread to dozens of GCP products and to customers built on them.
- **What made it worse:** In us-central1, the restarting tasks overloaded the Spanner database behind them, because there was no randomized exponential backoff (a thundering herd).
- **Lessons:** Put new code paths behind a feature flag. Roll out **data/policy changes** progressively, just like binaries. Use backoff with jitter on restart.
- **Source:** [Google Cloud incident report](https://status.cloud.google.com/incidents/ow5i3PPK96RduMcb1SsW)

### 6.6 Grafana Labs: pod priority preemption cascade (2019, ~26 min)
- **Trigger:** PriorityClasses were introduced while a new cluster was being added.
- **Root cause:** New pods with *default* (higher-than-expected) priority **preempted Cortex ingesters**. Each preempted ingester's replacement then preempted another ingester, until they had all been evicted. Cortex couldn't reach its write quorum.
- **Lessons:** PriorityClass is a weapon, so design the priority tiers deliberately (and review `globalDefault`). Protect stateful sets with PDBs and enough spare capacity.
- **Source:** [Grafana Labs blog](https://grafana.com/blog/2019/07/24/how-a-production-outage-was-caused-using-kubernetes-pod-priorities/)

### 6.7 Monzo: Kubernetes + linkerd + etcd interaction (Oct 2017, ~1h)
- **Root cause:** Three things combined: an **etcd upgrade**, a bug in the **Kubernetes etcd client**, and a bug in **linkerd 1.x**. After a new service was deployed, linkerd got an empty or stale endpoint view and routed traffic to nowhere. Outgoing payments failed and incoming ones were delayed. Taking the service mesh down resolved it.
- **Lessons:** Know how every component behaves when its dependency returns *nothing*. Upgrade the control plane, etcd and the service mesh with compatibility testing, and keep a way to bypass the mesh.
- **Source:** [Oliver Beattie, KubeCon EU 2018 keynote: Anatomy of a Production Kubernetes Outage (video)](https://www.youtube.com/watch?v=OUYTNywPk-s), [slides](https://speakerdeck.com/obeattie/anatomy-of-a-production-kubernetes-outage-kubecon-eu-2018)

### 6.8 Cloudflare: a single regex ate every CPU (2 Jul 2019, ~27 min)
- **Root cause:** A new WAF rule containing a regex prone to **catastrophic backtracking** was deployed globally at once, and CPUs hit 100% worldwide.
- **Lessons:** Roll out *rules* in stages, use regex engines with runtime guarantees (RE2/Rust regex), and set CPU budgets per request.
- **Source:** [Cloudflare blog](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/)

### 6.9 Buffer: CPU limits made services slower
- **Finding:** Services with CPU limits were being heavily **CFS-throttled** even at low average usage. Removing the limits from latency-critical services (and keeping accurate requests) cut latency sharply.
- **Source:** [Eric Khun: Kubernetes, faster services with no CPU limits](https://erickhun.com/posts/kubernetes-faster-services-no-cpu-limits/). See scenario [4.6](../04-kubernetes/README.md).

### 6.10 Weaveworks / XING: 5-second DNS timeouts
- **Finding:** A Linux conntrack race with parallel UDP DNS queries dropped packets and caused **exactly 5s** delays.
- **Source:** [Weaveworks: Racy conntrack and DNS lookup timeouts](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts). See scenario [4.5](../04-kubernetes/README.md).

---

## Cross-cutting patterns

| Pattern | Seen in |
|---|---|
| Config/data change, not code, caused it | Cloudflare 2025, Google Cloud 2025, Cloudflare 2019 |
| Upgrade removed or changed a default | Reddit, Datadog, Monzo |
| Observability/tooling caused the outage | OpenAI |
| Recovery path depended on the broken system | OpenAI (kubectl), Datadog |
| Global rollout with no staging | Cloudflare 2019 & 2025, Google Cloud, OpenAI |
| Retry or restart herd made recovery slower | Google Cloud 2025, Grafana Labs 2019 |
