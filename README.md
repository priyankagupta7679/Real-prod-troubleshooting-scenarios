# Production Troubleshooting Playbook

**Kubernetes · Prometheus · Grafana · Promtail/Loki · Application error codes**

Real troubleshooting scenarios, each written the same way: what you see, what to check, the root cause, the fix, and how to prevent it next time. They come from three kinds of source:

1. **Public postmortems** from companies that published their RCAs: Reddit, Datadog, OpenAI, Cloudflare, Google Cloud, Grafana Labs, Monzo, Buffer and others. Every one links to its original write-up.
2. **Failure patterns the community reports again and again** on Reddit (r/kubernetes, r/devops, r/sre), Stack Overflow, GitHub issues and Quora. These describe common patterns and are not tied to any one company.
3. **Field notes** from running observability for several production Kubernetes environments (details anonymised).

> **Honesty note:** A scenario is attributed to a company only when that company published it. Many companies, including Rapido, Swiggy and Zomato, don't publish detailed infra postmortems. Their failures are almost certainly the same generic patterns described here, but this repo won't invent stories for them.

---

## How to use this repo

- **Paged at 3 AM?** Go to [`cheatsheets/triage-commands.md`](cheatsheets/triage-commands.md).
- **Preparing for an SRE/DevOps interview?** Read each scenario and answer *"what would I check first?"* before you open the resolution.
- **Writing runbooks?** Copy the scenario template below.

## Index

| # | Area | Scenarios |
|---|------|-----------|
| 01 | [Prometheus](01-prometheus/README.md) | Cardinality OOM, slow WAL replay, `context deadline exceeded`, out-of-order samples, many-to-many matching, `rate()` misuse, flapping alerts, Alertmanager silent, remote-write lag |
| 02 | [Grafana](02-grafana/README.md) | "No data" after import, `Templating init failed`, slow dashboards, 502 behind a proxy, `database is locked`, alert `DatasourceNoData`, broken contact point templates |
| 03 | [Promtail / Loki](03-promtail-loki/README.md) | No logs shipped, CRI vs docker parsing, `entry too far behind`, 429 rate limit, stream limit, duplicate logs, `max entries limit`, label mismatch |
| 04 | [Kubernetes](04-kubernetes/README.md) | CrashLoopBackOff, OOMKilled (137), ImagePullBackOff, Pending, DNS 5s timeouts, CPU throttling, NotReady, evictions, IP exhaustion, HPA stuck, probe cascades, stuck namespace |
| 05 | [Application & error codes](05-application-error-codes/README.md) | 502, 503, 504, 499, 429, connection reset, exit codes, JVM OOM in containers, DB pool exhaustion, TLS expiry |
| 06 | [Real-world postmortems](06-real-world-postmortems/README.md) | Reddit Pi-Day, Datadog systemd, OpenAI telemetry, Cloudflare feature file, Google Cloud Service Control, Grafana Labs preemption, Monzo linkerd, Buffer CPU limits, conntrack DNS race |
| 07 | [Field notes](07-field-notes/README.md) | Silent alert channels, a monitoring sidecar that starved Prometheus, a label mismatch that hid logs, helm drift, per-pod alert noise |

## Scenario template

```markdown
### <Title>
**Symptoms:** what the on-call engineer actually sees
**Check:** commands / queries, in the order you'd run them
**Root cause:** the real reason
**Fix:** immediate mitigation → permanent fix
**Prevention:** alert / guardrail / process change
**Source:** link (postmortem, GitHub issue, docs) or "community pattern"
```

## The five lessons that repeat across every postmortem

1. **The monitoring stack is part of production.** OpenAI's telemetry service took down its own control plane. A crashing sidecar can starve Prometheus. Give your observability stack resource limits and alerts of its own.
2. **An alert that never fires tells you nothing.** Empty templates, dead webhooks and a missing `for:` all fail silently. Test the full delivery path end to end on a schedule.
3. **Config is code, and it needs the same protection.** Cloudflare, Google Cloud and Skyscanner were all taken down by config or data changes. Validate it, roll it out gradually, and put feature flags on new code paths.
4. **Upgrades hide breaking changes.** Reddit hit a removed label and Datadog hit a systemd behaviour change. Read the changelogs and test on a canary node pool first.
5. **Undocumented infrastructure is a time bomb.** If only one person knows how a component works, it's a single point of failure.

## Contributing

PRs welcome. Please include a **source** for anything attributed to a company, and anonymise anything from your own workplace.

## License

MIT
