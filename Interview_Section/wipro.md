# Wipro SRE Engineer Interview — 100 Q&A · Set 2 (4 Years Experience)

**Focus:** Advanced & scenario-based — the depth a 4-YOE SRE client round actually probes.
**Complements:** the earlier *Wipro SRE 100 Q&A* (fundamentals). This set goes harder on distributed-systems reliability, scenario/troubleshooting, deep PromQL, Kubernetes-for-SRE, capacity planning, and behavioral.
**How to use:** Answers are talking points. Anchor everything to your real experience. 💡 = framing tip. Scenario questions ("what would you do if…") are where 4-YOE candidates win — always narrate *mitigate → diagnose → prevent*.

---

## Section 1 — Advanced SLO, Error Budget & Reliability (Q1–Q10)

**Q1. How do you choose the right SLO window and target for a new service?**
Start from user expectations, not system capability. Identify critical user journeys, define event-based SLIs (good events / valid events), measure a baseline over a few weeks, then set the target slightly above current reality if reliability must improve. Use a rolling window (e.g., 28–30 days) so the SLO reflects recent behavior, not one bad day forever. Review quarterly. 💡 Say "SLA ≤ SLO ≤ actual target buffer" — internal targets are always stricter than the external contract.

**Q2. Explain multi-window multi-burn-rate alerting and why it's better than a static threshold.**
Static thresholds either page on every blip (noisy) or miss slow degradations. Burn-rate alerting measures how fast you're consuming the error budget. You combine a fast/severe window (e.g., 14.4× burn over 1h + 5m short window) for sudden outages with a slow window (e.g., 3× over 6h) for gradual erosion. The short second window prevents flapping — the alert only fires if the burn is sustained. This catches both acute and chronic problems with far less noise.

**Q3. Your error budget is exhausted mid-quarter. Product wants to ship a big feature. What do you do?**
Fall back to the pre-agreed **error budget policy** — this decision was made before the incident precisely to avoid arguing now. Typically: freeze non-essential releases, prioritize reliability work, require extra review. But I'd present the data (what's burning the budget, what the feature risks), and offer risk-managed options — ship behind a feature flag, canary to 1% traffic, or ship with added safeguards. The policy is the default; the conversation is about safe paths within it.

**Q4. What's the difference between an event-based SLI and a time-based SLI?**
Event-based: good events / valid events (e.g., successful requests / total requests) — precise, aggregatable, the SRE Workbook-preferred approach. Time-based: fraction of time windows that were "good" (e.g., minutes where error rate < threshold) — simpler but coarser and can hide sub-minute problems. Prefer event-based for request-driven services.

**Q5. How do you set SLOs for a service that depends on 5 downstream services?**
Recognize that your achievable SLO is bounded by your dependencies — if each of 5 dependencies is 99.9% and you call all of them serially, your theoretical max is ~99.5%. So either loosen your SLO, reduce hard dependencies (make some calls optional/async), add caching/fallbacks/circuit breakers so a downstream failure degrades gracefully, or negotiate stricter SLOs with the dependency owners. Map the dependency graph and identify which are on the critical path.

**Q6. What is the "nines" table and what does each nine cost?**
99% ≈ 3.65 days downtime/year; 99.9% ≈ 8.76 hrs; 99.95% ≈ 4.38 hrs; 99.99% ≈ 52.6 min; 99.999% ≈ 5.26 min. Each nine is ~10× harder and costs exponentially more (redundancy, automation, multi-region). Use this to push back on unrealistic asks — quantify what five nines actually requires.

**Q7. How do you handle an SLI that's technically met but users are still complaining?**
The SLI is measuring the wrong thing — it's not capturing the user's real experience. Re-examine: is the metric measured at the right point (client-side vs server-side)? Are you missing a dimension (specific region, endpoint, or percentile)? Maybe you track availability but the pain is tail latency, or you measure p50 but users feel p99. Add the SLI that reflects the actual complaint. SLIs are hypotheses about user happiness — refine them.

**Q8. What's the difference between reliability and resilience?**
Reliability is the system consistently doing what it should (measured via SLOs). Resilience is the system's ability to absorb and recover from failures/disruptions gracefully — through redundancy, graceful degradation, circuit breakers, retries, and self-healing. You engineer resilience to achieve reliability; chaos engineering tests resilience.

**Q9. How do you measure and reduce toil quantitatively?**
Track time spent on operational work vs engineering (aim for the ~50% cap). Log recurring manual tasks, their frequency, and time cost. Prioritize automating the highest frequency × time-cost items. Measure the reduction after automating (hours saved/week). Present it to stakeholders as capacity reclaimed for reliability work. 💡 "We automated cert rotation, saving ~6 engineer-hours/week and eliminating a recurring outage cause."

**Q10. What's the relationship between SLOs and prioritization?**
SLOs turn reliability into a data-driven prioritization tool. When the budget is healthy, prioritize features/velocity. When it's burning, prioritize reliability. It replaces subjective "is this important enough" debates with an objective signal, and gives you a defensible reason to push back on velocity when needed.

---

## Section 2 — Distributed Systems & Reliability Patterns (Q11–Q22)

**Q11. Explain the CAP theorem and its practical implication.**
In a distributed system you can guarantee at most two of Consistency, Availability, and Partition tolerance. Since network partitions are unavoidable, the real choice under a partition is between consistency (CP — reject requests to stay correct, e.g., etcd/ZooKeeper) and availability (AP — keep serving possibly-stale data, e.g., Cassandra/Dynamo). As an SRE you pick based on the use case — a bank ledger wants CP, a shopping cart often tolerates AP.

**Q12. What is a circuit breaker and why is it important for reliability?**
A pattern that stops calling a failing dependency after a failure threshold, "opening" the circuit to fail fast instead of piling up requests and threads waiting on timeouts. After a cooldown it goes "half-open" to test recovery. It prevents cascading failures — one slow dependency dragging down the whole service via resource exhaustion.

**Q13. What is a retry storm / cascading failure, and how do you prevent it?**
When a service slows or errors, clients retry, multiplying load, which worsens the slowdown — a feedback loop that can take down the whole system. Prevent with: exponential backoff **with jitter** (so retries don't synchronize), retry budgets/caps, circuit breakers, load shedding, and idempotency so retries are safe. Never retry infinitely or without backoff.

**Q14. What is load shedding and when do you use it?**
Deliberately dropping or rejecting low-priority requests when the system is overloaded, to protect its ability to serve high-priority traffic. Better to serve 90% of users well than fail 100% under overload. Implement via rate limiting, priority queues, and returning fast 429/503 responses. Graceful degradation is the same idea — reduce functionality rather than fail entirely.

**Q15. Explain idempotency and why it matters for reliability.**
An idempotent operation produces the same result whether called once or many times. It matters because networks fail and retries happen — if "charge card" isn't idempotent, a retry double-charges. Use idempotency keys, upserts, and design APIs so retries are safe. Critical for at-least-once delivery systems (queues, retries).

**Q16. What is the difference between at-least-once, at-most-once, and exactly-once delivery?**
At-least-once: message delivered one or more times (possible duplicates — needs idempotent consumers). At-most-once: delivered zero or one time (possible loss, no duplicates). Exactly-once: delivered precisely once — hardest to achieve, usually simulated via at-least-once + idempotency/deduplication. Most real systems do at-least-once + idempotency.

**Q17. How do caching strategies affect reliability?**
Caching reduces load on backends and improves latency, but introduces staleness and consistency concerns. Patterns: cache-aside (app manages cache), read-through/write-through, write-behind. For reliability, a cache can serve stale data during a backend outage (graceful degradation), but watch for cache stampede (many misses hitting the backend at once — mitigate with request coalescing, staggered TTLs, or a lock). A cache going down shouldn't take the service down.

**Q18. What is a health check, and what's the difference between shallow and deep health checks?**
A shallow health check confirms the process is up (returns 200). A deep health check verifies dependencies (DB, cache reachable). Deep checks catch more but risk cascading failures — if a shared DB blips, all instances fail health checks and get pulled simultaneously, causing an outage. Best practice: shallow for liveness, careful deep checks for readiness, and don't let a non-critical dependency fail the whole health check.

**Q19. How do you prevent a single point of failure?**
Redundancy at every layer: multiple instances across AZs/regions behind a load balancer, replicated databases (multi-AZ, read replicas), no single shared component without failover, stateless services so any instance can serve any request, and DNS/traffic failover. Then test it with chaos experiments — assume everything fails eventually.

**Q20. What is the thundering herd problem?**
When many clients/processes wake up and act simultaneously — e.g., all caches expire at once and hammer the backend, or all clients reconnect at the same instant after an outage. Mitigate with jittered timeouts/TTLs, backoff with jitter, request coalescing, and staggered restarts.

**Q21. Explain graceful degradation with an example.**
The system reduces functionality instead of failing completely when a dependency is down. Example: an e-commerce product page — if the recommendations service is down, still show the product and hide recommendations rather than erroring the whole page. If the review service is slow, show the page with a "reviews loading" placeholder. Core function survives; nice-to-haves degrade.

**Q22. What is a bulkhead pattern?**
Isolating resources (thread pools, connection pools) per dependency or workload so a failure in one area can't consume all resources and sink the whole service — like watertight compartments in a ship. If dependency A's calls are slow, they exhaust only A's pool, not the pool serving dependency B.

---

## Section 3 — Observability Deep Dive (Q23–Q32)

**Q23. Explain the three pillars of observability and how you use them together in an incident.**
Metrics (Prometheus) tell you *something* is wrong and quantify it (error rate up). Traces (Jaeger/Tempo) tell you *where* — which service/hop in the request path is slow or erroring. Logs (Loki/ELK) tell you *why* — the specific error/stack trace. Workflow: alert fires on a metric → open the trace to localize the failing component → read that component's logs for the root cause.

**Q24. What is OpenTelemetry and why does it matter?**
A vendor-neutral standard and set of SDKs/collectors for generating and exporting telemetry (traces, metrics, logs). It matters because it decouples instrumentation from the backend — you instrument once with OTel and can send to any backend (Prometheus, Tempo, Datadog) without re-instrumenting. It's becoming the industry standard, replacing vendor-specific agents.

**Q25. What is distributed tracing and what is a span?**
Distributed tracing follows a single request as it flows across multiple services, using a propagated trace ID. A **trace** is the whole journey; a **span** is one unit of work within it (one service call), with start/end time, parent span, and attributes. It's how you find *where* latency accumulates in a microservices call chain.

**Q26. How do you handle high-cardinality metrics without blowing up Prometheus?**
Never put unbounded values (user IDs, request IDs, full URLs, timestamps) in labels — each unique combination is a separate time-series and eats memory. Use bounded labels (status code, method, normalized endpoint). Drop noisy metrics with `metric_relabel_configs`. Push high-cardinality data to logs/traces instead of metrics. Monitor `prometheus_tsdb_head_series` for cardinality growth and set limits.

**Q27. How would you monitor a service you didn't build and can't modify?**
Black-box monitoring: synthetic probes (blackbox_exporter) against its endpoints, and infrastructure metrics around it (node_exporter, container metrics via cAdvisor). Scrape any exposed metrics endpoint. Monitor its dependencies and the load balancer/proxy in front of it (nginx/envoy metrics give latency and error rates without touching the app). Logs from stdout if containerized.

**Q28. What's the difference between logs, structured logs, and log levels — and why do structured logs matter?**
Plain logs are free-text; structured logs are machine-parseable (JSON with fields). Structured logs matter because you can query/filter/aggregate them (e.g., all logs where `status=500 AND region=us-east`) instead of regex-scraping text. Log levels (DEBUG/INFO/WARN/ERROR) let you control verbosity and alert on ERROR. Always log a correlation/trace ID to tie logs to traces.

**Q29. How do you decide what to alert on vs what to just dashboard?**
Alert only on actionable, user-impacting conditions (symptoms) that need a human now. Dashboard everything useful for diagnosis (causes, resource metrics, trends). If an alert fires and there's no clear action, it shouldn't page. Page on symptoms (SLO burn, error rate), ticket on slow-building causes (disk filling over days), dashboard the rest.

**Q30. What is RED vs USE, and when do you apply each?**
RED (Rate, Errors, Duration) — request-centric, for services/APIs. USE (Utilization, Saturation, Errors) — resource-centric, for infrastructure (CPU, disk, network). Apply RED to your microservices, USE to the hosts/nodes/queues underneath. Together they cover the request layer and the resource layer.

**Q31. How do you build an SLO dashboard that stakeholders actually understand?**
Top line: current SLO attainment vs target (e.g., 99.94% / 99.9% ✓) and error budget remaining as a percentage or gauge. Then burn-rate over time with threshold lines, and a breakdown by service/endpoint. Keep it non-technical — product/business should read it at a glance. Build it on recording rules so it's fast and consistent with your alerts.

**Q32. A dashboard shows normal metrics but users report problems. How do you reconcile this?**
Your monitoring has a blind spot. Check: are you measuring at the right vantage point (server-side looks fine but the CDN/LB/client-side path is broken)? Is it a specific segment (one region, one device, one endpoint) averaged away in aggregate metrics? Is there a percentile issue (p50 fine, p99 terrible)? Add synthetic monitoring from the user's perspective and RUM to catch what server-side metrics miss.

---

## Section 4 — Advanced Prometheus & PromQL (Q33–Q44)

**Q33. Write a PromQL query for the 99th percentile latency over 5 minutes.**
```promql
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```
Aggregate the histogram buckets by `le` first, then apply the quantile function. Bucket boundaries should straddle your SLO threshold for accuracy.

**Q34. How do you write a PromQL query for error budget burn rate?**
Define the SLI (success ratio), then compare failure fraction to the allowed budget. For a 99.9% SLO the allowed failure rate is 0.001; burn rate = actual failure rate / 0.001:
```promql
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
  / sum(rate(http_requests_total[1h]))
) / 0.001
```
A result of 14.4 means burning 14.4× the budget — page-worthy.

**Q35. Explain `rate()` vs `increase()` vs `irate()`.**
`rate()` = per-second average increase over the window (smooth, for alerting). `increase()` = total increase over the window (a count, e.g., "errors in the last hour"). `irate()` = instantaneous rate from the last two points (reactive, noisy, only for high-res graphs). All handle counter resets. Use `rate()` for most alerting, `increase()` for totals, `irate()` rarely.

**Q36. Why do you see gaps or weird values in a `rate()` graph, and how do you fix them?**
Common causes: window too short relative to scrape interval (need at least ~4 scrapes in the window — rule of thumb: rate window ≥ 4× scrape interval), targets going up/down (staleness), or counter resets on restart. Fix by widening the window (e.g., `[5m]` with a 15s scrape), and ensure scrape reliability.

**Q37. What are recording rules and give a real use case.**
Precomputed queries saved as new time-series, evaluated on a schedule. Use them for expensive/repeated aggregations, SLI calculations, and speeding up dashboards. Real case: precompute per-service success ratio once, then dashboards and alerts both read the cheap recorded metric instead of recomputing the heavy query every time.
```yaml
- record: job:request_success_ratio:rate5m
  expr: sum(rate(http_requests_total{status!~"5.."}[5m])) by (job)
        / sum(rate(http_requests_total[5m])) by (job)
```

**Q38. How does Prometheus service discovery work in Kubernetes?**
`kubernetes_sd_configs` queries the K8s API for targets (pods, services, endpoints, nodes). You then use `relabel_configs` to filter (keep only pods with a `prometheus.io/scrape: true` annotation), set the scrape port/path from annotations, and map K8s metadata (namespace, pod, labels) into clean metric labels. This auto-discovers targets as pods come and go.

**Q39. What's the difference between `by` and `without` in aggregation?**
`by (labels)` keeps only the listed labels and aggregates away the rest. `without (labels)` drops the listed labels and keeps all others. Example: `sum by (job) (...)` collapses to per-job totals; `sum without (instance) (...)` aggregates across instances but keeps everything else.

**Q40. How do you alert on a target being down?**
```promql
up == 0
```
`up` is a synthetic metric Prometheus generates per scrape — 1 if the scrape succeeded, 0 if it failed. Wrap with `for: 2m` to avoid flapping on a single missed scrape. Also use a dead-man's-switch / watchdog to detect the whole monitoring pipeline failing.

**Q41. How do you scale Prometheus for a large environment?**
Prometheus isn't clustered. Options: functional sharding (separate Prometheus per team/region), HA pairs (two identical servers, dedupe downstream), and for global view + long retention add **Thanos** or **Cortex/Mimir** (object-storage-backed, deduplication, downsampling, global query). Use remote-write to ship to the long-term store.

**Q42. What is remote-write and when do you use it?**
Prometheus streams scraped samples to a remote endpoint (Thanos Receive, Mimir, Cortex, a managed TSDB) for long-term storage, global querying, or centralized aggregation. Use it when local retention isn't enough, you need a unified view across many Prometheus servers, or you're offloading storage to a scalable backend.

**Q43. What does the `for` clause do in an alert, and how do you tune it?**
`for` requires the condition to hold continuously before the alert fires (pending → firing), preventing flapping on brief spikes. Tune it against the signal's volatility and how fast you need to know: short (1–2m) for severe outages, longer (5–15m) for noisier or slower-building conditions. Too short = noise; too long = delayed detection.

**Q44. How would you monitor certificate expiry with Prometheus?**
Use blackbox_exporter's `probe_ssl_earliest_cert_expiry` metric and alert when expiry is near:
```promql
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 14
```
Fires when a cert expires within 14 days — a classic preventable outage synthetic monitoring catches.

---

## Section 5 — Kubernetes for SRE (Q45–Q56)

**Q45. How do you make a Kubernetes deployment reliable?**
Multiple replicas across nodes/AZs (pod anti-affinity + topology spread), proper resource requests/limits, liveness/readiness/startup probes, a PodDisruptionBudget, rolling updates with sensible `maxUnavailable`, HPA for load, graceful shutdown handling (SIGTERM + `preStop` + `terminationGracePeriodSeconds`), and health checks that don't cascade.

**Q46. What is a PodDisruptionBudget and why does it matter?**
A PDB limits how many pods of an app can be voluntarily disrupted at once (during node drains, upgrades, autoscaling). Example: `minAvailable: 2` ensures at least 2 replicas stay up during maintenance. Without it, a node drain could take down all replicas simultaneously and cause an outage.

**Q47. Explain readiness vs liveness probe failures and their impact.**
Readiness failure → pod removed from Service endpoints (no traffic) but *not* restarted — good for temporary unavailability (warming up, dependency blip). Liveness failure → container is *restarted* — for unrecoverable states (deadlock). Misconfiguring liveness (too aggressive) causes restart loops; misconfiguring readiness causes traffic to hit unready pods. Get the distinction right.

**Q48. A pod is OOMKilled repeatedly. How do you investigate and fix?**
`kubectl describe pod` shows the OOMKilled reason and exit code 137. Check the memory limit vs actual usage (metrics/`kubectl top`). Options: raise the memory limit if the app legitimately needs it, fix a memory leak in the app, or check if a spike (large request/dataset) needs pagination/streaming. Ensure requests are set so the scheduler places it correctly. Don't just keep raising the limit without understanding why.

**Q49. How does horizontal pod autoscaling work and what are its limits?**
HPA adjusts replica count based on observed metrics (CPU/memory or custom/external metrics) against a target. It queries the metrics server / adapter and scales toward the target. Limits: it reacts (there's lag), needs headroom (can't scale instantly), depends on good metrics, and can't help if you're node-constrained — pair with Cluster Autoscaler. For request-based scaling use custom metrics (e.g., requests-per-second via Prometheus Adapter).

**Q50. What happens during a rolling update, and how do you make it zero-downtime?**
K8s gradually replaces old pods with new ones per `maxSurge`/`maxUnavailable`. For zero downtime: readiness probes so traffic only goes to ready pods, `maxUnavailable: 0` (or low) to keep capacity, graceful shutdown (handle SIGTERM, `preStop` sleep to drain connections), and PDBs. Verify with `kubectl rollout status`; rollback with `kubectl rollout undo`.

**Q51. How do you debug intermittent 5xx errors in a Kubernetes service?**
Correlate timing with deploys (annotations) and pod restarts (`kubectl get pods` — restart counts). Check if specific pods are erroring (a bad replica) via per-pod metrics. Look at readiness — is traffic hitting not-ready or terminating pods (missing graceful shutdown/preStop)? Check resource saturation (throttling/OOM), downstream dependency latency (traces), and LB/ingress logs. Intermittent + rolling = often a graceful-shutdown or readiness gap.

**Q52. What is the difference between a Deployment and a StatefulSet from a reliability standpoint?**
Deployments treat pods as interchangeable/stateless — easy to scale and replace. StatefulSets give stable identity, ordered rollout, and stable persistent storage per pod — needed for databases/clustered stateful systems where identity and data matter. StatefulSet updates are more delicate (ordered, one at a time) to protect quorum/data.

**Q53. How do you handle node failures gracefully?**
Spread replicas across nodes/AZs (anti-affinity, topology spread constraints) so one node's loss doesn't take the service down. K8s reschedules pods from a lost node automatically. Use PDBs for planned disruptions, Cluster Autoscaler to replace capacity, and ensure the app tolerates the reschedule (stateless or proper persistent volume handling). Chaos-test by killing a node.

**Q54. What are taints, tolerations, and node affinity used for?**
Taints repel pods from a node unless they have a matching toleration — used to reserve nodes (e.g., GPU nodes, or cordoning). Node affinity attracts pods to nodes with certain labels. Together they control scheduling — e.g., keep system pods on system nodes, or ensure a latency-sensitive app runs on the right hardware. Relevant to reliability via workload isolation.

**Q55. How do you ensure graceful shutdown of a pod?**
On termination, K8s sends SIGTERM, waits `terminationGracePeriodSeconds`, then SIGKILL. The app must catch SIGTERM, stop accepting new requests, finish in-flight ones, and close connections. Add a `preStop` hook (e.g., a short sleep) so the pod is removed from Service endpoints *before* it stops serving, avoiding requests routed to a dying pod. This prevents 5xx spikes during rollouts/scale-down.

**Q56. What Kubernetes-native reliability signals would you alert on?**
Pods in CrashLoopBackOff, pods not Ready, high restart counts, OOMKills, deployments with unavailable replicas, pending pods (scheduling failures), node NotReady, PVC issues, and HPA at max replicas (capacity ceiling). kube-state-metrics exposes most of these.

---

## Section 6 — Incident Management & On-Call (Q57–Q66)

**Q57. Walk me through your incident response process end to end.**
Detect (alert) → acknowledge → assess severity/impact → assign roles (Incident Commander, ops, comms, scribe for big ones) → **mitigate first** (rollback/failover/scale — restore service before root-causing) → communicate to stakeholders on a cadence → resolve → blameless postmortem with tracked action items. Priority order: stop the bleeding, then understand.

**Q58. Scenario: latency spiked 10× at 2 AM, you're on call. First 5 minutes?**
Acknowledge the page. Confirm it's real (check the dashboard, not just the alert). Assess blast radius — all endpoints or one? All regions? Check the deploy timeline (recent release? → likely rollback candidate). Check golden signals — is it saturation (scale), errors (dependency), or traffic (spike/attack)? Communicate that I'm investigating. Then mitigate the most likely cause (rollback if a deploy correlates) before deep debugging.

**Q59. How do you decide incident severity?**
By user/business impact and scope. SEV1: major outage, broad impact, revenue/data at risk — all hands, immediate page. SEV2: significant degradation, partial impact. SEV3: minor, limited. Severity drives urgency, who's paged, and comms cadence. Define it in advance so on-call doesn't guess.

**Q60. What makes a good blameless postmortem?**
Focus on system/process, not people. Include: timeline, impact (users, duration, budget consumed), root cause + contributing factors (5 whys), what went well, what went poorly, where you got lucky, and concrete action items with owners and due dates. Share it widely and track the actions to completion — an untracked postmortem means the incident recurs.

**Q61. How do you run a sustainable on-call rotation?**
Adequate team size and rotation length, tuned alerting so pages are actionable (minimize false pages), good runbooks and dashboards, clear escalation paths, follow-the-sun if global, compensation/time off, and a solid handoff process. Track paging volume — high off-hours paging is a signal to fix underlying reliability, not to tough it out.

**Q62. A dependency team's service keeps causing your incidents. How do you handle it?**
Bring data, not blame — show the correlation (their outages → your incidents → budget burned). Propose SLOs for the dependency and an interface contract. On your side, add resilience (circuit breakers, caching, graceful degradation) so their failures don't cascade. Escalate through the right channels with the impact quantified. Collaboration + protecting yourself, not finger-pointing.

**Q63. What is the difference between mitigation and resolution?**
Mitigation restores service (rollback, failover, scale, feature-flag off) — stops user impact fast, even if the underlying bug remains. Resolution fixes the actual root cause. Always mitigate first; users don't care about root cause while they're down. Root-cause and permanent fix come after, tracked as postmortem actions.

**Q64. How do you prevent the same incident from recurring?**
Postmortem action items with owners and deadlines, tracked to completion. Add detection (an alert that would've caught it earlier), add prevention (a test/gate, a canary stage, a guardrail), and reduce blast radius. Measure recurrence — if similar incidents keep happening, the actions aren't landing.

**Q65. What is an Incident Commander and why separate that role?**
The IC coordinates the response, makes decisions, and manages the flow — deliberately *not* hands-on-keyboard, so someone maintains the big picture while others fix. Separating it prevents the common failure where everyone dives into debugging and nobody coordinates, communicates, or decides. For small incidents one person does both.

**Q66. How do you communicate during a major incident to non-technical stakeholders?**
Regular, jargon-free updates on a predictable cadence: what's impacted, who's affected, what we're doing, next update time. Use a status page for external users. Under-promise on ETAs. A dedicated comms lead owns this so responders stay focused. Maps directly to the JD's "open communication with Engineering and Product teams."

---

## Section 7 — Chaos Engineering, Game Days & Synthetic (Q67–Q74)

**Q67. What is Chaos Engineering and what are its core principles?**
Deliberately injecting failures to verify resilience and find weaknesses before real outages. Principles: define steady-state behavior (a measurable metric), hypothesize it holds under a fault, inject a controlled failure with a limited blast radius, run in production ideally (or realistic staging), observe, and fix what breaks. Automate and expand confidence gradually.

**Q68. Walk me through designing a chaos experiment for a payment service.**
Define steady state (payment success rate > 99.9%, p99 < 500ms). Hypothesis: "losing one replica / adding 200ms DB latency won't breach the SLO." Start in staging, small blast radius. Ensure monitoring and an abort switch are in place first. Inject (kill a pod / inject latency via a chaos tool like Chaos Mesh/Litmus). Observe against steady state. If it breaches, that's a real weakness — fix (retries, circuit breaker, more replicas) and re-run. Only move to prod with a tiny blast radius once confident.

**Q69. Chaos Engineering vs Game Day — what's the difference?**
Chaos Engineering tests the *system* (automated/controlled fault injection for technical resilience). A Game Day tests *people and process* — a scheduled drill of the whole incident response (runbooks, escalation, comms), often triggered by a chaos experiment. They overlap; you often run chaos during a Game Day to exercise both.

**Q70. How would you introduce chaos engineering to a risk-averse organization (the JD's "progressively adopt")?**
Start where risk is lowest and value is clear: synthetic monitoring first, then non-prod chaos experiments, then Game Days with willing teams, then carefully scoped prod chaos once observability and trust are mature. Always document blast radius and rollback. Tie each step to data (fewer incidents, lower MTTR). Need a blameless culture first, or people won't engage.

**Q71. What is Synthetic Monitoring and why is it valuable even with real traffic?**
Scheduled, scripted probes simulating user actions from outside the system. Valuable because it's consistent and proactive — catches issues (cert expiry, broken flows, DNS) even during low/no traffic and off-hours, gives you availability/latency SLIs from the user's perspective, and detects problems before real users do. blackbox_exporter or Grafana Synthetic Monitoring for HTTP/TCP/DNS probes.

**Q72. Synthetic vs Real User Monitoring (RUM)?**
Synthetic: proactive, controlled, consistent baselines, runs without real traffic — good for uptime SLIs and early detection. RUM: measures actual users' real experiences across devices/geos/networks — good for real-world performance truth. They complement each other; use both.

**Q73. How do you validate that your monitoring/alerting actually works?**
Test it deliberately: trigger a known failure (chaos experiment / Game Day) and confirm the alert fires, routes correctly, and the runbook resolves it. Use a dead-man's-switch to confirm the pipeline is alive. Periodically review fired alerts for actionability. An alert you've never tested is a hope, not a control.

**Q74. What's a hypothesis-driven approach to reliability?**
Treat reliability like science: state a hypothesis ("the system stays within SLO if a dependency is 30% slower"), design an experiment (chaos/load test) to test it, measure against steady state, and either confirm resilience or find and fix the gap. It replaces "we think it's fine" with evidence.

---

## Section 8 — Capacity Planning & Performance Testing (Q75–Q82)

**Q75. How do you approach capacity planning?**
Understand current usage and growth trends (from metrics), identify the constraining resource (saturation signals), model demand (organic growth + known events like sales/launches), add headroom for spikes and failover (e.g., N+1 or N+2 redundancy so you survive losing capacity), and validate with load testing. Re-plan regularly. Autoscaling handles short-term variance; capacity planning handles the baseline and peaks.

**Q76. What types of performance testing are there and when do you use each?**
Load (expected peak — meets SLOs?), Stress (beyond capacity — where/how it breaks), Soak/endurance (sustained load — memory leaks, resource exhaustion over time), Spike (sudden surge — handling and recovery), Scalability (does adding resources scale throughput linearly). Tools: k6, JMeter, Gatling, Locust.

**Q77. How do you write a test plan to validate scalability (from the JD)?**
Define objectives tied to SLOs (e.g., p95 < 300ms at 5000 rps, error rate < 0.1%). Establish a baseline, design load profiles (ramp-up, steady, spike), specify metrics to capture (golden signals + resource saturation), run in a prod-like environment, incrementally raise load to find the breaking point and scaling limit, and document bottlenecks + remediation. Define pass/fail criteria before running.

**Q78. How do you identify the bottleneck under load?**
Watch saturation per tier during the test — the resource that maxes first (CPU, DB connections, thread pool, network, disk I/O) is the bottleneck. Correlate with the point where latency/errors climb. Use traces to find the slow component and profiling for hot paths. Fix it, re-test — the bottleneck usually moves to the next tier.

**Q79. Little's Law — what is it and how is it useful?**
L = λ × W: the average number of concurrent requests in a system equals arrival rate × average time in system. Useful for capacity math — e.g., if you receive 100 req/s and each takes 0.2s, you have ~20 concurrent requests, so you need enough workers/threads/connections to handle that. Helps size thread pools and connection pools.

**Q80. How do you decide between vertical and horizontal scaling for reliability?**
Horizontal (more instances) is generally better for reliability — fault tolerance (lose one, others serve), no single big point of failure, and near-limitless scale — but requires statelessness. Vertical (bigger instance) is simpler but has a ceiling, often needs downtime, and a single large instance failing hurts more. SRE favors horizontal + autoscaling.

**Q81. What's the difference between throughput and latency, and can you optimize both?**
Throughput = requests handled per unit time; latency = time per request. They can conflict — batching improves throughput but adds latency; more concurrency raises throughput until saturation, where latency spikes. You optimize for the SLO that matters to users (usually tail latency) while meeting required throughput. Watch the "knee" where latency degrades as throughput climbs.

**Q82. How do you plan capacity for a known traffic spike (e.g., a sale)?**
Estimate peak load from history/projections, load-test at that level ahead of time, pre-scale (don't rely solely on reactive autoscaling which lags), add headroom, verify dependencies (DB, cache, downstream) also scale, prepare load-shedding/graceful-degradation for overflow, and have a rollback/runbook ready. Run a Game Day simulating the spike.

---

## Section 9 — Automation, Toil & SWE Practices for SRE (Q83–Q90)

**Q83. The JD says "apply software development best practices" — what does that mean for an SRE?**
Treat operations as a software problem: everything as code (IaC, config, dashboards, alerts), version control and code review for infra/automation, testing (including for automation scripts and IaC), CI/CD for operational tooling, modular reusable code, and documentation. It means building tools/automation to eliminate toil rather than doing repetitive ops by hand.

**Q84. How do you decide what to automate?**
Prioritize by frequency × time cost × risk of human error. Automate high-frequency, repetitive, error-prone toil first (deploys, cert rotation, failover, scaling, common remediations). Don't automate rare one-offs where automation costs more than it saves. The goal: automation should reduce future toil more than it costs to build and maintain.

**Q85. What is self-healing and give an example.**
Automated detection and remediation without human intervention. Examples: Kubernetes restarting failed pods, autoscaling replacing unhealthy instances, a watchdog restarting a hung service, auto-failover to a standby, or an auto-remediation runbook triggered by an alert (e.g., clear a full disk, restart a stuck consumer). Reduces MTTR and toil — but log it and alert so humans know it happened.

**Q86. How do you test infrastructure code?**
Static analysis/linting (tflint, checkov, tfsec for security), `terraform validate` and `plan` review in CI, policy-as-code (OPA/Sentinel) for guardrails, and integration tests (Terratest) that actually spin up and verify resources in a sandbox. For Ansible, `--check` mode and Molecule. Treat IaC like application code — reviewed, tested, versioned.

**Q87. How do you handle configuration across many environments without drift?**
IaC as the single source of truth, parameterized per environment (tfvars/values files), GitOps to continuously reconcile live state to Git, and immutable infrastructure (replace, don't modify). Drift detection (`terraform plan` in CI on a schedule) flags manual changes. No manual production changes — everything through the pipeline.

**Q88. What is a "runbook" and how do you make them useful (not stale)?**
Step-by-step operational docs for a specific alert/scenario. Keep them useful by: linking directly from the alert, keeping them concrete (exact commands/queries), reviewing/updating them after incidents (postmortem action), and testing them during Game Days. Ideally, automate the runbook so it becomes self-healing. Stale runbooks are worse than none.

**Q89. How do you balance the 50% ops / 50% engineering split in practice?**
Track operational load (paging, tickets, toil hours). If ops consistently exceeds 50%, that's a signal — push work back to dev teams, invest in automation, or fix the underlying reliability driving the load. Protect engineering time so you're reducing future toil, not just treading water on today's.

**Q90. How do you drive reliability improvements across teams (the JD's core ask)?**
Make reliability visible and shared: SLOs/dashboards everyone sees, blameless postmortems with tracked actions, error-budget-driven prioritization, partnering with dev teams early on design (build for scale/observability from the start), and quantifying impact (incidents down, MTTR down, availability up). Influence through data and collaboration, not mandates.

---

## Section 10 — Scenario Troubleshooting & Behavioral (Q91–Q100)

**Q91. Scenario: CPU is fine, memory is fine, but requests are timing out. What do you check?**
Likely not compute-bound — check other saturation: connection pool exhaustion, thread pool starvation, file descriptor limits, DB connection limits, downstream dependency latency (traces), lock contention, GC pauses (for JVM), or network saturation. A request can wait forever on an exhausted pool while CPU sits idle. Check the resource the requests are *blocked on*, not just CPU/mem.

**Q92. Scenario: after a deploy, error rate slowly climbs over 30 minutes rather than spiking. What's likely?**
A gradual issue: a memory leak filling up (OOM approaching), a connection/file-descriptor leak, cache filling, a slow-growing queue backlog, or a resource pool slowly exhausting. A spike suggests an immediate config/code break; a slow climb suggests resource accumulation. Check memory/FD/connection trends since the deploy — and consider rolling back while investigating.

**Q93. Scenario: one of five app instances serves errors, the other four are fine. How do you handle it?**
Immediate: remove the bad instance from the load balancer (or let readiness probes do it) to stop user impact. Then diagnose *that* instance — bad node/hardware, corrupted local state, a config difference, resource exhaustion on its host, or a partial deploy. Replace it (in K8s, delete the pod to reschedule). Then ask *why only one* — that's the interesting question (node issue, race condition, uneven load).

**Q94. Scenario: your monitoring shows everything green but customers can't reach the site. What now?**
Your monitoring is blind to the failing path. Check the layers your metrics don't cover: DNS resolution, CDN, load balancer/ingress, TLS/cert, and the network path from the user's side. Run a synthetic probe from an external location. It's often something outside the app tier (expired cert, DNS misconfig, LB health-check misconfig pulling all backends). Add external synthetic monitoring so this isn't invisible next time.

**Q95. Scenario: a query to your database has suddenly gotten slow. How do you approach it?**
Check if it correlates with a deploy (schema/query change), data growth (missing/ineffective index as the table grew), a bad query plan, lock contention, increased load, or resource saturation on the DB (CPU, I/O, connections). Look at slow query logs and the execution plan. Mitigate (add an index, cache, rate-limit, scale reads) then fix root cause. Consider connection pool exhaustion cascading to the app.

**Q96. How do you handle competing demands from multiple stakeholders? (JD)**
Make trade-offs explicit and data-driven — use SLOs and error budgets as the shared, neutral language so it's not opinion vs opinion. Prioritize by user/business impact, communicate constraints and timelines transparently, and when two urgent things truly conflict, surface it to the right owner with the trade-off stated rather than silently dropping one. 💡 Have a real STAR example of balancing a feature deadline against reliability.

**Q97. Tell me about a time you improved reliability. (STAR)**
Situation (recurring incident / high MTTR / noisy alerts / repeated outages), Task (your goal), Action (what you built — burn-rate SLO alerting, an SLO dashboard, automated a manual recovery into self-healing, added a canary + auto-rollback, ran a postmortem action to closure), Result (quantified — false pages down X%, MTTR from Y→Z, availability 99.5%→99.9%). Always land on a measurable outcome.

**Q98. How do you push back on a product team that wants to ship despite reliability risk? (JD)**
Frame in their terms — user/business risk, not internal jargon. Bring data: current error budget, what shipping now risks, and options (feature flag, canary to 1%, delay, add a safeguard). Don't just say "no" — present the trade-off and a safe path, then let the owner make an informed call. Preserve the relationship; you'll need it next time.

**Q99. Describe a difficult incident you handled and what you learned.**
Pick a real one. Structure: what broke and the impact, how you detected and mitigated it (emphasize mitigate-first thinking), the root cause, and — most important — the systemic fix and what changed afterward (a new alert, a guardrail, a process improvement). Show that you turn incidents into lasting improvements, and be blameless in how you tell it.

**Q100. Why SRE, and how do you stay current? (closer)**
Honest framing: SRE sits at the intersection of software engineering and operations, is data-driven (SLOs, error budgets) rather than heroics, and has direct, measurable user impact. Staying current: hands-on with the tooling (Prometheus/Grafana/K8s), the Google SRE books, public incident writeups from other companies, postmortems as learning, and side projects. Show curiosity and a bias toward automating your own toil away.

---

## Rapid-Fire Cheat Sheet (Set 2 — advanced)

| Concept | One-liner |
|---|---|
| Burn rate | actual failure rate ÷ allowed budget rate |
| Multi-window alert | fast severe window + slow window; short window kills flapping |
| CAP theorem | under partition, choose Consistency or Availability |
| Circuit breaker | fail fast on a failing dependency; prevents cascade |
| Retry storm | retries multiply load → backoff + jitter + budgets |
| Load shedding | drop low-priority traffic to protect the rest |
| Idempotency | same result on retry; needed for at-least-once |
| Bulkhead | isolate resource pools so one failure can't sink all |
| Graceful degradation | reduce function, don't fail entirely |
| Thundering herd | synchronized wakeups → jitter TTLs/backoff |
| Deep vs shallow health check | dependency check (can cascade) vs process-up |
| Three pillars | metrics (what) → traces (where) → logs (why) |
| OpenTelemetry | vendor-neutral instrumentation standard |
| High cardinality | never label with user/request IDs; explodes series |
| histogram_quantile | aggregate buckets by le, then quantile |
| Thanos/Mimir | long-term, global, scalable Prometheus |
| PDB | limits voluntary pod disruptions |
| Readiness vs liveness | remove from LB vs restart |
| OOMKilled | exit 137, hit memory limit |
| preStop + SIGTERM | graceful pod shutdown, no 5xx on rollout |
| Little's Law | L = λ × W (concurrency = rate × time) |
| Mitigate vs resolve | restore service first, root-cause later |
| Incident Commander | coordinates, not hands-on-keyboard |
| 50% rule | cap ops work; rest engineers away future toil |

---

*Client-round tips for 4 YOE:*
1. **Scenario questions are the differentiator.** Always answer *mitigate → communicate → diagnose → prevent*, out loud, in that order.
2. **Whiteboard one real architecture** and one real incident end to end — that's where depth shows.
3. Expect **"why" and trade-off follow-ups** (CP vs AP, deep vs shallow health checks, vertical vs horizontal). The reasoning separates 4 YOE from junior.
4. **Know your resume cold** — every tool listed is fair game for a deep dive.
