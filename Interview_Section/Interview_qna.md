
---
### Cloud Networking & Infra Q&A Set

Same format as before: question, concise model answer, difficulty. Azure-primary since that's your platform. Active recall — cover the answer, say it out loud, check.

### Core networking fundamentals

**[Easy] What is a VPC / VNet, and what's a subnet?**
A VNet (Azure) / VPC (AWS) is an isolated private network in the cloud where your resources live. A subnet is a segment of that network's IP range — you split a VNet into subnets to separate tiers (e.g., a public subnet for load balancers, private subnets for app and database tiers) and apply different routing/security per subnet.

**[Easy] Public vs private subnet — what's the difference?**
A public subnet has a route to the internet gateway, so resources can be directly reachable from / reach the internet. A private subnet has no direct internet route — resources reach out via a NAT gateway (outbound only) and aren't directly reachable from the internet. Databases and app servers go in private; load balancers / bastion go in public.

**[Medium] What's a NAT gateway and why do you need it?**
It lets resources in a private subnet make *outbound* connections to the internet (e.g., to pull packages or call an API) without being directly reachable *inbound*. It does source NAT — translating the private IP to a public one for outbound traffic, and tracking the connection so responses return. It's how you give private resources internet access without exposing them.

**[Medium] Security group / NSG — stateful or stateless, and what's the implication?**
NSGs (Azure) and security groups (AWS) are **stateful** — if you allow an inbound connection, the return traffic is automatically allowed; you don't need a matching outbound rule. Network ACLs (AWS) are stateless — you must explicitly allow both directions. Stateful is simpler and the common case; the gotcha with stateless is forgetting the return rule.

**[Hard] Connection refused vs connection timed out — what does each tell you about the network path?**
Refused = the packet reached the host and something actively rejected it (RST) — the service is down, wrong port, or not bound; an app/host-layer issue. Timed out = no response at all — usually a firewall/NSG/security group silently dropping the packet, a routing problem, or a network policy blocking it. The distinction tells you whether to look at the application/host (refused) or the network/firewall (timeout).

### Load balancing

**[Easy] L4 vs L7 load balancer?**
L4 routes on IP and port — fast, protocol-agnostic, no payload inspection (Azure Load Balancer, AWS NLB). L7 routes on HTTP attributes like host, path, headers — enables TLS termination, path-based routing, rate limiting (Azure Application Gateway, AWS ALB). Use L4 for raw throughput or non-HTTP; L7 for HTTP routing and richer control.

**[Medium] How does a load balancer know not to send traffic to an unhealthy instance?**
Health checks (health probes) — the LB periodically pings a configured endpoint (e.g., `/healthz`) on each backend. If an instance fails the probe threshold, the LB removes it from the rotation and stops sending traffic until it passes again. This is what gives you automatic failover at the LB layer.

**[Medium] In Azure, when do you use Azure Load Balancer vs Application Gateway vs Front Door?**
Azure Load Balancer = L4, regional, raw TCP/UDP distribution. Application Gateway = L7, regional, HTTP routing + WAF + TLS termination. Front Door = global, L7, edge routing across regions with CDN and global failover. You pick based on layer (L4 vs L7) and scope (regional vs global).

### DNS

**[Medium] How does DNS resolution work at a high level?**
A resolver queries the DNS hierarchy: root servers → TLD servers (.com) → authoritative nameservers for the domain, which return the record (A/AAAA for IP, CNAME for alias). Results are cached per TTL to avoid repeating the chain. In cloud, you also have private DNS zones for internal name resolution within a VNet.

**[Medium] Public vs private DNS zone in cloud?**
A public DNS zone resolves names reachable from the internet (your customer-facing domain). A private DNS zone resolves names only within your VNet — used for internal service discovery, private endpoints, and resources that shouldn't be publicly resolvable. Same DNS mechanics, different scope.

**[Hard] A service intermittently fails to resolve an external hostname. How do you debug?**
Check whether it's DNS specifically: `nslookup`/`dig` the name, time it. In Kubernetes, check CoreDNS health and the `ndots:5` amplification (short names trying search domains first). Look for CoreDNS pod resource pressure or conntrack exhaustion on UDP 53. Check the upstream resolver the cloud provides. Solutions: scale CoreDNS, NodeLocal DNSCache, use FQDNs, fix ndots. Intermittent DNS is often a capacity or conntrack issue, not a config one.

### AKS / Kubernetes networking (your platform — know cold)

**[Medium] Azure CNI vs kubenet in AKS — what's the difference and the tradeoff?**
Azure CNI gives every pod a real IP from the VNet subnet — pods are first-class VNet citizens, directly routable, better performance, integrates with VNet features; but it consumes a lot of VNet IP space (nodes pre-allocate IP blocks), so you can exhaust the subnet at scale. Kubenet gives pods IPs from a separate overlay range and NATs through the node — conserves VNet IPs but adds a hop and has routing limitations. Choose Azure CNI when you need VNet integration and have IP space; kubenet when conserving IPs matters more.

**[Medium] How does traffic from the internet reach a pod in AKS?**
Internet → Azure Load Balancer (or Application Gateway via AGIC) → the Ingress controller pods → the Service (ClusterIP) → the pod. The LB is provisioned when you create a Service of type LoadBalancer (typically just for the ingress controller); the ingress controller does L7 host/path routing to internal Services, which kube-proxy routes to pods.

**[Hard] A pod can't reach an Azure SQL database. Walk through debugging.**
Layer by layer: from inside the pod, test DNS (`nslookup` the DB hostname), then connectivity (`nc -zv host 1433`). If DNS fails — check private DNS zone / private endpoint config. If connect times out — check the NSG rules on the subnet, the DB's firewall rules (is the AKS subnet allowed?), and whether you're using a private endpoint correctly. If refused — DB-side issue. Also check if a NetworkPolicy is blocking egress from the pod. Isolate whether it's the pod, the node, DNS, the network path, or the DB firewall.

### Reliability / infra design

**[Medium] Availability Zone vs Region — what's the difference and why does it matter for SRE?**
A Region is a geographic location (e.g., Central India). An Availability Zone is a physically separate datacenter within a region with independent power/cooling/network. Spreading across AZs survives a single-datacenter failure with low latency between zones; spreading across regions survives a whole-region outage but with higher latency and complexity. SRE cares because designing for AZ failure is table stakes, and region failure is your DR strategy.

**[Medium] How do you design an AKS workload to survive an AZ failure?**
Use a multi-zone node pool spread across AZs, use `topologySpreadConstraints` to distribute pod replicas across zones (not all on one), run enough replicas with headroom so losing a zone doesn't drop you below capacity, use a multi-zone-redundant managed database, and set PodDisruptionBudgets. The goal: losing one zone degrades capacity but doesn't cause an outage.

**[Hard] Explain RTO and RPO and how they drive DR design.**
RTO (Recovery Time Objective) = how long you can afford to be down before recovery. RPO (Recovery Point Objective) = how much data loss you can tolerate, measured in time. They drive cost/complexity: a tight RPO (near-zero data loss) needs synchronous replication or frequent backups; a tight RTO (fast recovery) needs warm/hot standby rather than cold restore. You design backups, replication, and failover strategy to meet the agreed RTO/RPO — looser targets are cheaper, tighter targets cost more.

**[Hard] What reliability patterns prevent a slow dependency from cascading into a full outage?**
Timeouts (don't wait forever on a slow call), retries with backoff and jitter (recover from transient failures without thundering-herd), circuit breakers (stop calling a failing dependency to let it recover and fail fast), bulkheads (isolate resource pools so one failing dependency can't exhaust all threads/connections), load shedding (drop excess load to protect core function), and graceful degradation (serve reduced functionality rather than failing entirely). Together they contain blast radius.

### Connection / kernel-level (overlaps Linux)

**[Hard] What is conntrack and how does it cause cloud/K8s networking incidents?**
conntrack is the kernel's connection-tracking table that remembers NAT mappings so return traffic routes correctly. It has a max size; under high connection churn it fills, and new connections get dropped — you see `nf_conntrack: table full` in dmesg and intermittent failures that look like the app's fault. Common in high-traffic Kubernetes nodes. Fix: raise `nf_conntrack_max`, reduce churn via connection pooling/keepalive, and reduce DNS-driven churn with NodeLocal DNSCache.
---

# SECTION 3 — System Design / Scenario Questions

Mid-level SRE design rounds aren't "design Google" — they're "design something real and reason about reliability." For each, I give you a **structured framework** to answer, not a memorized solution. Interviewers grade your *thinking process*, so always narrate your reasoning.

## Scenario 1: "A service is returning 5xx errors. Walk me through debugging it."

This is the most common SRE scenario. Structure your answer as a funnel — broad to narrow.

**Framework:**
1. **Scope the impact first.** "How many users? All requests or a percentage? When did it start? Did anything change — a deploy, a config push, a traffic spike?" (The "what changed" question solves most incidents.)
2. **Check the dashboards.** Error rate, latency, traffic, saturation (golden signals). Is it a spike or a ramp? All endpoints or one?
3. **Localize the layer.** Is it the LB/ingress, the service itself, or a downstream dependency? Check the `up` metric and per-service error rates.
4. **Use traces.** Open a failing request's trace — which span is erroring? Is the time spent in the service or in a downstream call? This answers "is it us or a dependency?"
5. **Read the logs.** From the trace, jump to logs for that trace ID. What's the actual error — DB timeout? OOM? 500 from a dependency?
6. **Mitigate, then fix.** If it correlates with a recent deploy → roll back first. If it's a dependency → check that dependency's health, consider circuit-breaking.
7. **Root cause + postmortem after.**

**What to emphasize:** "My first question is always *what changed* — most 5xx spikes correlate with a deploy or config change. And I roll back before I deep-debug if users are actively impacted, because the error budget is burning while I investigate."

## Scenario 2: "Design a highly available web service."

**Framework — reason through each layer:**
1. **Redundancy at every layer.** Multiple replicas across multiple availability zones; no single point of failure.
2. **Load balancing.** LB in front, health checks to remove unhealthy instances automatically.
3. **Stateless app tier.** Keep app servers stateless so any replica can serve any request; push state to a shared data store / cache.
4. **Data tier HA.** Managed DB with multi-AZ failover (replica promotion); read replicas for scaling reads.
5. **Autoscaling.** HPA on the app tier to handle load; keep headroom for failover capacity.
6. **Graceful degradation.** Timeouts, retries with backoff, circuit breakers so a slow dependency doesn't cascade.
7. **Observability + SLOs.** Define SLIs (availability, latency), set SLOs, alert on burn rate.
8. **DR.** Backups, and for higher tiers, a secondary region with defined RPO/RTO.

**What to emphasize:** "HA isn't one thing — it's removing single points of failure at every layer: compute, network, and data. And I'd define an SLO upfront so 'highly available' has a number, not just a vibe."

## Scenario 3: "Design a monitoring/observability system for a microservices platform."

This plays directly to your strength (and your Athena project).

**Framework:**
1. **Three pillars.** Metrics (Prometheus), logs (Loki), traces (Tempo/Jaeger via OpenTelemetry) — unified in Grafana.
2. **Instrumentation strategy.** Standardize on OpenTelemetry so instrumentation is vendor-neutral and consistent across languages.
3. **What to measure.** Golden signals per service (latency, traffic, errors, saturation); RED method for request-driven services.
4. **SLI/SLO definition.** Define SLIs per critical service, set SLOs, track error budgets.
5. **Alerting.** Multi-window burn-rate alerts on SLOs (not raw thresholds); route by severity through Alertmanager; inhibition to prevent alert storms.
6. **Correlation.** Exemplars and shared trace IDs so you can pivot metric → trace → log for one request.
7. **Cost/scale.** Control cardinality; use object storage for long-term retention; consider downsampling.

**What to emphasize:** "I actually built exactly this in my Athena project — and the part most people skip is *alerting on SLOs with burn rate*, not raw thresholds, plus controlling cardinality so the metrics system doesn't collapse under its own weight."

## Scenario 4: "Latency is high for a service but error rate is normal. How do you investigate?"

**Framework:**
1. **Confirm and quantify.** Which percentile — p50 or just p99 (tail latency)? Which endpoints? Started when?
2. **Rule out load.** Did traffic spike? Is it saturation (CPU throttling, memory pressure, connection pool exhaustion)?
3. **Use traces — the key tool here.** Open slow traces. Where is the time spent? In the service's own code, or waiting on a downstream call (DB, cache, external API)?
4. **If downstream:** check that dependency's latency and health. Slow DB query? Lock contention? Undersized connection pool?
5. **If in-service:** CPU throttling (hitting CPU limits)? GC pauses? A slow code path?
6. **Check resource saturation.** `container_cpu_cfs_throttled` for throttling; memory pressure; connection pool metrics.

**What to emphasize:** "Latency-without-errors usually means saturation or a slow dependency, not a bug. Traces are decisive — they tell me in seconds whether the time is in our code or downstream, which is the slowest question to answer otherwise."

## Scenario 5: "An alert is paging on-call repeatedly but the issue self-resolves each time. What do you do?"

**Framework:**
1. **Short-term:** Acknowledge the pattern; don't just keep silencing blindly.
2. **Assess the alert quality.** Is it actionable? If it self-resolves and needs no human action, it's a bad alert — a dashboard metric masquerading as an alert.
3. **Fix the alert.** Add a `for` duration so transient spikes don't fire; switch from a raw threshold to a burn-rate or sustained condition; raise the threshold if it's too sensitive.
4. **But investigate the underlying flapping** — is something actually degrading and recovering (e.g., a pod OOMing and restarting every few hours)? The alert might be noisy *and* hiding a real slow problem.
5. **Document and improve the runbook.**

**What to emphasize:** "My rule: every alert must require a human action. A self-resolving page is either a tuning problem or it's masking a real recurring issue — and I check for both. I did exactly this cleanup at Cognizant, cutting weekend pages from 8-10 to 2-3."

---

# SECTION 4 — Behavioral Questions (STAR skeletons)

STAR = Situation, Task, Action, Result. I've added the angle each question tests and a skeleton anchored to your real experience. **Fill in your specifics.**

**1. "Tell me about a difficult production incident you handled."**
*Tests: technical depth + composure.*
> S: [The OOMKilled service — pods dying every few hours, error spikes]
> T: [I was on-call; needed to find why memory was growing]
> A: [Grafana memory slope → confirmed leak not load → exit 137 → log correlation to an endpoint → handed L3 full diagnostic trail]
> R: [Fixed within a day, memory flattened]
> Reflection: [Handing off a complete diagnostic trail cut their investigation time in half]

**2. "Tell me about a time you improved a process or reduced toil."**
*Tests: ownership, proactiveness.*
> S: [On-call was getting paged 8-10x per weekend, mostly non-actionable]
> T: [I took ownership of cleaning up the alert rules]
> A: [Exported 90 days of alerts, categorized by actionability, found 12 rules = 70% of noise, fixed thresholds/added `for`/inhibition]
> R: [Pages dropped to 2-3/weekend, almost all actionable]
> Reflection: [Every alert should require a human action]

**3. "Describe a time you disagreed with a teammate or senior."**
*Tests: communication, handling conflict professionally.*
> S: [A proposed alert threshold / config change you thought was wrong]
> T: [Needed to raise the concern without overstepping]
> A: [Brought data — showed the historical pattern / impact — proposed an alternative, discussed rather than insisted]
> R: [Reached a better decision together / or disagreed-and-committed]
> Reflection: [Bring data, not opinions, to disagreements]

**4. "Tell me about a time you failed or made a mistake."**
*Tests: humility, growth. Pick a real, recoverable mistake — never "I have no failures."*
> S: [A change/action that didn't go as planned]
> T: [What you were trying to do]
> A: [How you caught it, owned it, fixed it]
> R: [Recovered; no lasting damage]
> Reflection: [The specific lesson + what you now do differently]

**5. "Tell me about a time you had to learn something new quickly."**
*Tests: learning agility — your Athena story is perfect here.*
> S: [Got SRE interview feedback that I lacked hands-on observability building]
> T: [Needed to close that gap fast]
> A: [Built Athena — full LGTM stack, OpenTelemetry, SLO alerting — in ~2 weeks, learning by doing]
> R: [Now have hands-on depth + a portfolio project]
> Reflection: [The fastest way I learn is building the real thing and breaking it]

**6. "Tell me about a time you took ownership of something outside your role."**
*Tests: initiative.*
> [The alert cleanup, or building the health-check/disk-cleanup automation that wasn't assigned but reduced team toil]

**7. "Describe a high-pressure situation and how you handled it."**
*Tests: composure under stress.*
> S: [A P1 during on-call — CrashLoopBackOff during a release]
> A: [Stayed systematic: checked events, found liveness probe failures, identified slow startup vs probe timing, recommended fix]
> R: [Resolved, documented in runbook]
> Reflection: [Under pressure, systematic beats fast — and escalate rather than thrash alone]

**8. "Why are you leaving Cognizant?"**
*Tests: motivation, and that you're running *toward* something.*
> [Positive framing: "I've grown a lot operating at scale, but I want to go deeper on the engineering and building side of reliability — designing observability and platforms, not just operating them. Building Athena confirmed that's the work I want to do, and I'm looking for a product-focused team where that's the core of the role."]

**9. "Why this company / this role?"**
*Tests: genuine interest. Research the company first.*
> [Tie their stack/domain to your strengths: "You run a large observability practice / you're in healthcare where I have domain experience / you have a strong SRE culture — that's exactly the direction I'm building toward."]

**10. "Tell me about a time you handled an on-call escalation well."**
*Tests: on-call maturity.*
> S: [An incident where you correctly triaged and escalated]
> A: [Gathered signal, attempted ops-level fixes, escalated to L3 with full context when it was an app bug]
> R: [Faster resolution because of the context you provided]
> Reflection: [Knowing when to escalate with good context is a strength, not a weakness]

---

# SECTION 5 — High-Yield Cheat Sheet (revision notes)

Bullet-point recall, focused on what's commonly tested.

**Linux**
- Load avg = runnable + uninterruptible (D-state/IO). High load + low CPU = I/O wait.
- `df` full but `du` not = deleted file held open → `lsof +L1`.
- Port owner: `ss -tlnp`. CPU/mem hog: `top`, `ps aux --sort=-rss`.
- OOM: `dmesg | grep -i oom`; container OOM = exit 137.
- RSS = real RAM (double-counts shared); VSZ = virtual (misleading); PSS = fair share.
- Hung process: `strace -p`, `/proc/<pid>/stack`, `lsof -p`.

**Networking**
- Refused = nothing listening (RST). Timeout = dropped (firewall/route).
- TLS: hello → cert → verify → key exchange → encrypted. 1.3 = 1-RTT.
- L4 = IP/port, fast. L7 = HTTP host/path, TLS termination, routing.
- conntrack table full → dropped packets (`dmesg`); raise `nf_conntrack_max`.
- ndots:5 → short names try search domains first (DNS latency); FQDN with trailing dot skips it.

**Kubernetes**
- Service = virtual ClusterIP, no process; kube-proxy iptables/IPVS DNAT to Pod IP; conntrack tracks return.
- iptables mode O(n), IPVS O(1) at scale.
- Probes: readiness=traffic, liveness=restart, startup=slow-start grace.
- `kubectl apply`: API server (authn/authz/admission/validate) → etcd → Deployment ctrl → ReplicaSet ctrl → scheduler (filter/score) → kubelet (CRI pull, CNI IP) → endpoints → kube-proxy.
- QoS eviction order: BestEffort → Burstable → Guaranteed.
- CrashLoopBackOff debug: `describe` → `logs --previous` → probes/resources/config.
- etcd lose quorum = read-only (no changes), existing pods keep running.

**Docker**
- Image = template (layers); container = running instance + writable layer.
- Multi-stage build = build heavy, ship slim. Order Dockerfile: rarely-changing first (cache).
- Don't run as root; use slim/distroless; `.dockerignore`.

**CI/CD**
- Pipeline: test → build → scan → push → deploy/sync.
- Secrets: encrypted store, injected as masked env at runtime; OIDC for cloud (short-lived).
- Push vs pull: pull (GitOps) keeps creds in-cluster, Git = source of truth, drift detection.
- Image tags: immutable (SHA/semver), never `latest` in prod.
- Blue-green = instant switch/rollback, double resources. Canary = gradual %, low blast radius.

**IaC (Terraform)** ⚠️
- State = config↔real mapping; remote backend + locking prevents concurrent corruption.
- Modules = reusable, per-env variables. Drift = real ≠ state; `plan` detects.
- Secrets: `sensitive` vars, pull from Key Vault/Vault, encrypt state.

**Cloud (Azure)**
- AKS: Azure manages control plane (free); you manage node pools + workloads.
- Log Analytics = store/KQL; App Insights = APM on top; Azure Monitor = umbrella.
- Workload Identity = pods get short-lived AAD tokens, no stored secrets.

**Observability**
- Metrics (dashboards/alerts), logs (detail), traces (cross-service latency) — correlate.
- Pull model → free `up` signal. Pushgateway for short-lived jobs.
- Golden signals: latency, traffic, errors, saturation. RED: rate, errors, duration.
- `rate()` before `sum()` (reset-aware). p99 = `histogram_quantile(0.99, sum(rate(bucket[5m])) by (le))` — can't avg percentiles.
- Cardinality = #1 killer; never label with unbounded values (IDs).
- Loki indexes labels not content (cheap); ELK full-text (expensive).

**Reliability**
- SLI = measured; SLO = target; SLA = contract with penalties.
- Error budget = 1 − SLO; spend on features when healthy, reliability when burning.
- Multi-window burn-rate alert: short (now) + long (sustained) both breach.
- Toil = manual, repetitive, automatable, scales with growth → cap and automate.

**Incident / On-call**
- Fix-forward vs rollback: users impacted + cause unclear → roll back first.
- Postmortem = blameless, timeline, root cause, action items with owners.
- On-call: stabilize → gather signal → runbook → escalate with context → write up.

**Scripting**
- Bash: `set -euo pipefail` = fail fast. Truncate (not delete) open log files to free space.
- Python over Bash for JSON parsing, logic, maintainability.

---

# SECTION 2 — Core Technical Q&A (tagged by difficulty)

Concise model answers. I've kept Azure-primary per your stack.

## Linux internals & troubleshooting

**[Easy] Difference between `top` load average and CPU usage?**
Load average counts processes runnable *and* in uninterruptible sleep (D-state, usually I/O-blocked). High load + low CPU = I/O wait, not CPU saturation. Check `iostat` and the `wa` column in top.

**[Medium] `df` shows full but `du` doesn't add up. Why?**
A process holds an open handle to a deleted file — blocks aren't freed until it closes. Find with `lsof +L1` / `lsof | grep deleted`. Fix: restart the process or truncate the open fd. (This is why log cleanup truncates rather than deletes.)

**[Medium] How do you find what's using a port?**
`ss -tlnp | grep :PORT` — shows the listening PID/process. `ss` is the modern, faster replacement for `netstat`.

**[Hard] A process is hung. How do you find what it's blocked on without killing it?**
`strace -p <pid>` shows the blocking syscall (read on socket = network wait, futex = lock contention). `cat /proc/<pid>/stack` and `/proc/<pid>/wchan` show kernel state. `lsof -p` shows open files/sockets. Caveat: strace adds overhead — brief use only on production.

**[Hard] RSS vs VSZ vs PSS?**
VSZ = total virtual address space (misleadingly large). RSS = resident physical RAM but double-counts shared pages. PSS = proportionally splits shared pages — most accurate "real cost." For OOM, RSS is the practical number.

## Networking

**[Easy] Connection refused vs connection timed out?**
Refused = packet reached host, nothing listening (RST returned) — app/port issue. Timeout = no response — firewall drop, routing, or network policy. The distinction localizes the problem layer.

**[Medium] What happens in a DNS lookup inside Kubernetes?**
Pod's resolv.conf points to CoreDNS ClusterIP. CoreDNS (watches API for Services/Endpoints) resolves `*.svc.cluster.local` from memory, forwards external names upstream. `ndots:5` causes short names to try search domains first.

**[Medium] Walk through a TLS handshake.**
Client hello (ciphers, TLS version) → server hello + certificate → client verifies cert chain against trusted CAs → key exchange (ECDHE) establishes a shared session key → encrypted symmetric communication begins. TLS 1.3 cut this to one round trip.

**[Hard] L4 vs L7 load balancing — when each?**
L4 (TCP/UDP) routes on IP/port, fast, protocol-agnostic, no payload inspection (NLB). L7 (HTTP) routes on host/path/headers, enables TLS termination, path routing, rate limiting (ALB, Ingress). Use L4 for raw throughput/non-HTTP; L7 for HTTP routing and richer control.

## Kubernetes & containers

**[Easy] Pod vs container vs node?**
Container = a running image instance. Pod = one or more containers sharing network/storage namespace, the smallest deployable unit. Node = the VM/machine running pods.

**[Medium] How does a Service route to Pods?**
Virtual ClusterIP (no process listens on it). CoreDNS resolves name→ClusterIP. kube-proxy programs iptables/IPVS rules that DNAT the ClusterIP to a backend Pod IP from the EndpointSlice; conntrack tracks it for return traffic.

**[Medium] Liveness vs readiness vs startup probe?**
Readiness gates traffic (fail = removed from endpoints, not killed). Liveness gates restarts (fail = container killed/restarted). Startup disables the other two until a slow app finishes starting. Misconfigured readiness = silent traffic loss; misconfigured liveness = CrashLoopBackOff.

**[Hard] Walk through `kubectl apply` end to end.**
API server (authn → authz → admission → validate) → etcd → Deployment controller makes a ReplicaSet → ReplicaSet controller makes Pods (Pending) → scheduler filters/scores nodes, sets nodeName → kubelet pulls image via CRI, CNI assigns IP → readiness passes → endpoints controller updates EndpointSlice → kube-proxy programs routing.

**[Hard] QoS classes and eviction order?**
Guaranteed (requests==limits), Burstable (requests<limits), BestEffort (none). Under node memory pressure, kubelet evicts BestEffort first, then Burstable over requests, Guaranteed last. Set requests==limits for critical pods.

## CI/CD

**[Easy] CI vs CD?**
CI = automatically build and test on every change. CD = automatically deliver/deploy that tested artifact. CI catches breakage early; CD makes releases safe and frequent.

**[Medium] Where do GitHub Actions secrets live and how does the runner access them?**
Encrypted in repo/org secrets, injected as masked env vars into the job at runtime, scoped to that job. Modern cloud auth uses OIDC federation for short-lived tokens instead of stored long-lived credentials.

**[Hard] Push vs pull (GitOps) deployment — why pull?**
Push: CI holds cluster creds and applies changes (creds leave the pipeline = risk). Pull: in-cluster controller (ArgoCD) watches Git and syncs in — creds stay in-cluster, Git is source of truth, you get drift detection and revert-based rollback. Pull wins on security and auditability.

## Infrastructure as Code ⚠️ (your lighter area — know concepts, be honest on depth)

**[Easy] Why IaC over manual provisioning?**
Repeatable, version-controlled, reviewable, auditable, eliminates config drift and snowflake servers.

**[Medium] What is Terraform state and why does locking matter?**
State maps config to real resources so Terraform knows what exists. Two simultaneous applies can corrupt it — remote backends (Azure Storage blob lease, S3+DynamoDB) lock state so the second apply waits.

**[Hard] How do you manage multi-environment infra and prevent drift?**
Reusable modules called with per-env variables; remote state per environment; everything through Terraform (no console changes); `terraform plan` in CI to detect drift before apply.
*(Your honest add: "I've worked with modules and state during our CloudFormation→Terraform migration, modifying variables and configs; I'm deepening my module-authoring.")*

## Cloud — Azure-primary

**[Easy] What does Azure manage in AKS vs you?**
Azure manages the control plane (API server, etcd, scheduler) — free, invisible. You manage worker node pools, workloads, config; you get managed upgrades and Azure AD/ACR/Monitor integration.

**[Medium] Azure Monitor vs App Insights vs Log Analytics?**
Log Analytics = data store + query engine (KQL). App Insights = APM layer (requests, dependencies, exceptions) on top. Azure Monitor = umbrella tying metrics/logs/alerts together. App Insights data lands in Log Analytics.

**[Medium] How do pods get Azure credentials securely?**
Workload Identity — federates a K8s service account with an Azure AD identity for short-lived tokens, no stored secrets. (Older: pod-managed identities.)

**[Hard] Design HA for an AKS workload across failure domains.**
Multi-zone node pools, `topologySpreadConstraints` to spread replicas across zones, PodDisruptionBudgets for safe drains, multiple replicas, managed DB with zone redundancy, health probes + HPA, and ideally a secondary region for DR.

## Observability (your strength — deepest)

**[Easy] Metrics vs logs vs traces?**
Metrics = numeric time-series, cheap, for dashboards/alerts. Logs = discrete event detail. Traces = one request across services, for "where did it slow down." They correlate: metric → trace → log.

**[Medium] Why pull-based Prometheus?**
Scraping gives a free up/down signal (`up` metric), centralized scrape control, no need for services to know where to push. Downside: short-lived jobs may die before scrape → Pushgateway for those.

**[Medium] Four golden signals + PromQL?**
Latency (`histogram_quantile(0.99, ...)`), Traffic (`sum(rate(requests[5m]))`), Errors (`sum(rate(requests{status=~"5.."}[5m]))/sum(rate(requests[5m]))`), Saturation (usage/limit).

**[Hard] Why `rate()` before `sum()`, and how do you get p99?**
Counters reset on restart; `rate()` is reset-aware, raw `sum()` first turns a restart into a fake cliff. p99: `histogram_quantile(0.99, sum(rate(bucket[5m])) by (le))` — aggregate buckets first; you can't average percentiles.

**[Hard] Biggest Prometheus failure mode?**
Cardinality explosion — high-cardinality labels (user IDs, request IDs) create millions of series and OOM Prometheus. Control via relabeling, never labeling with unbounded values, monitoring `prometheus_tsdb_head_series`.

## Reliability concepts

**[Easy] SLI vs SLO vs SLA?**
SLI = measured indicator (% successful requests). SLO = internal target (99.5%). SLA = external contract with penalties (usually looser than the SLO).

**[Medium] What's an error budget and why useful?**
1 − SLO = allowed failure (0.5% = ~216 min/month). It's a shared currency: budget remaining = ship features; budget burned = focus on reliability. Removes the dev-vs-ops "how reliable" argument.

**[Medium] What is toil?**
Manual, repetitive, automatable, reactive work that scales with service size and adds no lasting value. SRE aims to cap and automate it. (Your disk-cleanup/image-update scripts are toil reduction.)

**[Hard] Multi-window burn-rate alerting — why better than thresholds?**
Alerts on budget consumption *rate*, not raw thresholds. Two windows (short confirms it's happening now, long confirms it's sustained) must both breach — kills flapping, fast recovery, urgency matched to severity.

## Incident management & on-call

**[Easy] What's a runbook?**
A documented procedure for handling a known scenario — steps to diagnose and remediate — so any on-call engineer can respond consistently without tribal knowledge.

**[Medium] What makes a good postmortem?**
Blameless (focus on systems, not people), clear timeline, root cause, impact, and concrete action items with owners. Goal: learn and prevent recurrence, not assign fault.

**[Hard] Fix-forward vs rollback — how do you decide?**
If users are impacted and cause isn't obvious — roll back first, stop the bleeding, investigate after (error budget burns while you debug). Minor issue with a known quick fix — fix forward. For bad deploys, rollback is usually the right first move.

## Scripting / automation

**[Easy] Why `set -euo pipefail` in Bash?**
Fail fast — exit on any error (`e`), undefined variable (`u`), or pipe failure (`pipefail`). Without it, ops scripts silently continue after errors, which is dangerous.

**[Medium] When Python over Bash?**
Bash for simple orchestration/glue. Python when parsing structured data (JSON), complex logic, error handling, or maintainability matter. (Your log-analyzer is Python because JSON parsing + aggregation is painful in Bash.)

**[Hard] Walk through a script that safely updates a K8s image with rollback.**
Validate args/namespace/deployment exist → capture current image → `kubectl set image` → `kubectl rollout status` with timeout → verify ready replicas → on failure `kubectl rollout undo`. (This is your image-update script — be ready to explain each safety check.)

---

## 1. "Walk me through your day-to-day at Cognizant."

> "My day breaks into three buckets. About 40% is observability — I start by reviewing our Grafana dashboards and any overnight alerts to check platform health, investigate whether alerts were real issues or tuning problems, maintain our 5-6 production dashboards, and tune Prometheus alert rules. About 30% is incident response — when a P1 or P2 comes in I do the initial investigation using App Insights logs, Kubernetes events, and Dynatrace traces; if it's an ops-level issue like a crash loop or image-pull problem I resolve it, and if it's an application bug I partner with the L3 engineering team and hand them a full diagnostic trail. The last 30% is automation and infrastructure — Bash and Python scripts for operational tasks, supporting our CI/CD pipelines, and contributing to our Terraform work. And I'm on alternate-weekend on-call as first responder."

*Why it works: structured, honest about the L3 partnership, leads with observability (your strength).*

---

## 2. "You mention ~1,500 pods and 86 deployments. Describe the platform architecture."

> "It's a healthcare platform running on Azure Kubernetes Service across multiple environments. The observability architecture has a few layers: Prometheus collects both Kubernetes-level metrics like pod health and CPU/memory, and application metrics from the services. Grafana is the visualization layer where I own dashboards for latency, error rates, and resource saturation. For deeper APM and distributed tracing we use Dynatrace, and for application logs and telemetry we use Azure Application Insights and Log Analytics, which integrate tightly with our Azure infra. Alerting runs through Alertmanager with severity-based routing. CI/CD is on GitHub Actions and Azure DevOps, and infrastructure is managed with Terraform — currently being migrated from CloudFormation. My deepest ownership is the observability layer and the operational reliability of the platform."

*Why it works: describes the system, then names your specific ownership scope honestly.*

---

## 3. "You say you 'own' the Prometheus and Grafana stack. What does owning it actually involve?"

> "Day to day it means I maintain our 5-6 production dashboards — building and updating panels for latency, error rates, pod health, and saturation. I author and tune the Prometheus alert rules, which is the bigger part of ownership: adjusting thresholds, adding `for` durations so transient spikes don't fire, and adding inhibition rules so we don't get alert storms. When we onboard a new service or a team needs visibility into something, I build the dashboard and the alerts for it. I don't manage the Prometheus infrastructure itself at the cluster level — that's a platform-team responsibility — but the dashboards, queries, and alerting logic are mine."

*Why it works: honest scoping — you own dashboards/alerts/queries, not the Prometheus server infra. Defensible if drilled.*

---

## 4. ⚠️ "You reduced MTTR by ~40%. How was that measured, and what did you personally do?"

> "That was a team-level improvement tracked through our ticketing system and Dynatrace over my time on the account — average resolution time for P2 incidents came down by roughly that much. My personal contribution was on the detection and triage side: I tuned our alerts to reduce noise so real signals weren't buried, built dashboards that gave incident responders the right context faster, and when I triaged incidents I handed the engineering teams a complete diagnostic trail — the metric trend, the relevant traces, the suspect component — instead of just 'service is down.' Engineering told me that context cut their investigation time significantly. So I wouldn't claim I single-handedly drove a 40% reduction — it was a team effort, and my piece was making detection and triage faster."

*Why it works: honest attribution. You don't overclaim, which is exactly what protects you when they drill.*

---

## 5. "Tell me about a specific P1 or P2 incident you handled end to end."

> "A backend service started getting OOMKilled every few hours — pods would run fine, then get killed and restart, and our error-rate dashboard spiked on each restart. I picked it up from the alert. First I checked Grafana and saw memory climbing on a steady slope regardless of traffic, which told me it was a leak, not load. I confirmed OOMKilled from the pod's last state — exit code 137. Then I correlated with App Insights logs and found one endpoint being hit repeatedly during the growth window. I handed the engineering team the memory trend, the trace data, and the suspect endpoint — they found an unbounded in-memory cache and fixed it within a day. Memory flattened out completely. What I took away was that handing off a full diagnostic trail, not just 'it's OOMing,' cut their investigation time roughly in half."

*Why it works: full STAR-R, shows technical depth and the partnership reality.*

---

## 6. "How do you tune a noisy alert? Give a real example."

> "When I moved into the DevOps role, on-call was getting paged 8-10 times a weekend and most of it wasn't actionable. I exported about 90 days of alert history and categorized each alert by whether it actually led to a human action or self-resolved. About 12 rules were causing 70% of the noise. For each, I looked at the underlying Prometheus rule — some had thresholds on raw counts instead of rates, some were missing a `for` clause so they fired on instantaneous spikes, and some were on infrastructure metrics that didn't actually impact users. I fixed thresholds, added `for` durations, added inhibition rules so downstream alerts didn't fire when an upstream one was active, and removed the non-actionable ones. Weekend pages dropped to 2-3, almost all actionable. My principle became: every alert should require a human action — if it doesn't, it's a dashboard metric, not an alert."

*Why it works: concrete, methodical, ends with a memorable principle.*

---

## 7. "You use Dynatrace. Walk me through diagnosing a performance issue with it."

> "When a service shows elevated latency, I'll start in Dynatrace by looking at the service's response-time trend and breaking it down — is it all requests or specific endpoints, and is the time being spent in the service itself or in a downstream call like a database or external dependency. The distributed traces are the key part — I can open a slow trace and see exactly which span is taking the time, which immediately tells me whether it's our code or a dependency. From there I'll correlate with the metrics — is there a saturation issue, a slow query, a dependency that's degraded. Then I package that up for the engineering team if it's a code-level fix."

*Why it works: shows you use it as an investigator (traces, breakdowns), not just a dashboard viewer.*

---

## 8. ⚠️ "You contributed to a CloudFormation-to-Terraform migration. What was your role?"

> "I joined the team while that migration was in progress, so my contribution was on the variable and environment-configuration side — modifying module variables and environment-specific configs for our resources, and reviewing plans. The senior engineers on the team owned the module design and the overall migration strategy. It was a great learning period for me — I got hands-on with how Terraform state works, why state locking matters, and how modules are structured to stay DRY across environments. I'd say I'm comfortable working within an existing Terraform codebase, and authoring modules from scratch is an area I'm actively deepening — which is part of why I built the infrastructure layer in my Athena project with Terraform."

*Why it works: completely honest about scope, frames it as growth, bridges to Athena. Will not crack under drilling.*

---

## 9. "How does your CI/CD pipeline work, and what's your role in it?"

> "Our pipelines run on GitHub Actions and Azure DevOps — they handle build, test, image build and push to ACR, and deployment across environments. My role is on the operational side: when a build or release fails, I troubleshoot it — whether it's a failing test, an image-pull issue, a misconfigured step, or a deployment that didn't roll out cleanly. I also support image promotion across environments and rollback procedures when a release causes a regression. The pipeline design and the workflow architecture are owned by senior engineers on the team. To get hands-on with building pipelines end-to-end, I built a full GitHub Actions pipeline in my Athena project — build, test, image push, manifest update, with approval gates — so I understand the design side as well as the operations side."

*Why it works: honest (you support, don't design at work), but shows you've done the design side in Athena.*

---

## 10. "Tell me about Athena — why you built it and the architecture."

> "Athena is an observability platform I built from scratch. The motivation was honest: at work I operate an observability stack that was set up before me — I tune and maintain it — but I hadn't designed one end to end, and that gap came up in an SRE interview. So I built one. It instruments three microservices in different languages — Python, Node, and Go — each emitting metrics, logs, and traces. Metrics go to Prometheus via ServiceMonitor CRDs. Logs go through Promtail to Loki. Traces go through an OpenTelemetry Collector to Tempo. Everything's unified in Grafana so I can pivot from a metric anomaly to a trace to the exact log line. I defined an SLO — 99.5% availability — with multi-window burn-rate alerting through Alertmanager. And the most valuable part was deliberately breaking it five ways — memory leaks, latency injection, dependency failures, error spikes, probe misconfigurations — and documenting the full root-cause investigation for each. It's all on my GitHub."

*Why it works: honest motivation, complete architecture, ends on the failure-simulation hook that's memorable.*

---

## 11. "In Athena, why Loki over ELK? Why Tempo? Why OpenTelemetry?"

> "Loki indexes only metadata labels — namespace, pod, container — not the full log content. ELK indexes every word, which is powerful for ad-hoc search but expensive at scale. Loki's assumption is that you query logs by context you already know — which service, which pod — usually because you arrived from a dashboard or trace. That makes it much cheaper and simpler to operate, which is the right trade-off for most operational logging. Tempo applies the same idea to traces — store cheaply, retrieve by trace ID, no expensive attribute indexing. And OpenTelemetry was deliberate because it decouples instrumentation from the backend — I instrument once with vendor-neutral SDKs, and I could swap Tempo for Datadog or anything else without touching application code. Vendor lock-in through the SDK is basically over now that every major vendor accepts OTLP."

*Why it works: real design reasoning with trade-offs — proves you didn't just follow a tutorial.*

---

## 12. "Show me a PromQL query you've written and explain it."

> "A common one is the error rate for a service: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`. The `rate` gives me the per-second rate of 5xx responses over a 5-minute window, I `sum` across all instances, and divide by the total request rate to get an error ratio. One important detail — I always apply `rate` before `sum`, never the other way around, because counters reset to zero when a pod restarts, and `rate` is reset-aware. If you sum the raw counters first, a single restart looks like a massive drop and gives you garbage. For latency I use `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` — aggregating the buckets first, because you can't average percentiles across instances."

*Why it works: shows real PromQL fluency plus two senior-level gotchas (rate-before-sum, can't-average-percentiles).*

---

## 13. "You're on alternate-weekend on-call. Walk me through your on-call process."

> "When I get paged, the first thing is to stabilize — is there an immediate mitigation or rollback that stops user impact, even before I fully understand the cause. Then I gather signal — check the dashboards, look at what changed recently like a deploy or config push, pull the relevant logs and traces. If there's a runbook for the scenario, I follow it. If it's within my scope — an ops-level issue — I resolve it. If it's beyond my scope, like an application bug, I escalate to L3 with full context rather than thrashing on it alone. And afterward I contribute to the post-incident review and update the runbook so the next person has it. The mindset I try to keep is: systematic beats fast, and escalating with good context is a strength, not a weakness."

*Why it works: calm, systematic, honest about escalation, mature mindset.*

---

## 14. "A pod is in CrashLoopBackOff. How do you debug it?"

> "First I'll `kubectl describe pod` to read the events and the container's last state — that tells me the exit code and reason. Then `kubectl logs --previous` to see the crashed container's logs, since the current one may have already restarted. From there it's usually one of a few causes: an application error on startup, a failed liveness probe, a missing config or secret, OOMKilled which shows as exit 137, or a bad image. I had a real case where a service went into CrashLoopBackOff right after a release — the events showed liveness probe failures, and it turned out a config change had made startup slower than the probe's initial delay, so the kubelet was killing it before it finished starting. The fix was increasing initialDelaySeconds and adding a startupProbe. So I always check whether it's the app actually crashing or a probe killing a healthy-but-slow container."

*Why it works: clear method + a real story that shows depth (probe-vs-startup distinction).*

---

## 15. "Why are you leaving Cognizant after 4 years?"

> "I've grown a lot there — from intern to DevOps Engineer, operating a real production platform at scale, and I'm grateful for that foundation. But the nature of the role is mostly operating systems that other teams designed. I want to go deeper on the engineering and building side of reliability — designing observability, building platforms, writing the automation, not just running it. Building Athena confirmed that's the work I find most engaging. So I'm looking for a product-focused team or a strong SRE org where that kind of hands-on building is the core of the role, and where I can keep growing toward senior reliability engineering."

*Why it works: positive, honest, running toward something, ties to Athena.*

---

## 16. ⚠️ "You're Azure-strong but list AWS. How comfortable are you with AWS?"

> "I want to be straight about that — my production experience is on Azure, specifically AKS, App Insights, and Azure Monitor, which is what I work with daily. My AWS experience is at the project level — I've worked with the core services like EC2, S3, VPC, IAM, and EKS in personal projects, so I understand the fundamentals and the mapping between Azure and AWS concepts is pretty direct. I'm confident I could ramp up on AWS quickly in a production context, but I wouldn't claim deep production AWS experience today. The reliability and Kubernetes concepts transfer directly — it's mostly learning the AWS-specific service names and tooling."

*Why it works: honest, doesn't bluff, frames transferability without overclaiming. This honesty actually builds trust.*

---

## 17. "How do you decide what deserves an alert versus a dashboard metric?"

> "My rule is simple: every alert should require a human action. If something fires and the on-call engineer looks at it and there's nothing for them to do — it self-resolves, or it's just informational — then it shouldn't be an alert, it should be a dashboard metric you look at when investigating. Alerts are expensive: they wake people up, they cause fatigue, and fatigue means real signals get missed. So I'm aggressive about keeping alerts actionable and tied to user impact or SLO burn, and pushing everything else to dashboards. That's exactly the cleanup I did at Cognizant — cutting our weekend pages from 8-10 to 2-3 by removing non-actionable alerts."

*Why it works: clear philosophy + ties back to real work.*

---

## 18. "Healthcare is regulated. How did that affect how you worked?"

> "It made me much more careful about production changes. In healthcare there's real sensitivity around data and availability, so there was strong emphasis on change management — changes going through proper approval, clear audit trails, and caution about anything touching production. It shaped good habits: documenting what I did and why, being conservative about production changes, and understanding that reliability isn't just a technical goal but sometimes a compliance one. I think that discipline around change management and audit awareness is something that transfers well to any environment where reliability genuinely matters, like finance."

*Why it works: turns domain into a transferable strength, relevant for GCC/finance targets.*

---

## 19. "What's the hardest production problem you've debugged, and what did you learn?"

> "Probably the OOMKilled memory leak I mentioned — what made it hard wasn't the eventual root cause, it was that nothing was obviously broken. There was no full outage because we had replicas, just intermittent error spikes every few hours, so it was easy to dismiss as noise. The breakthrough was recognizing the pattern — memory climbing on a steady slope regardless of traffic — which told me it was accumulation, not load. The lesson that stuck with me is that the hardest incidents are often the slow, intermittent ones that don't trip an obvious alarm, and that the speed of resolution depends heavily on how clearly you can hand off context to whoever owns the fix. That's actually what pushed me to start building diagnostic templates so anyone on the team could investigate the same way."

*Why it works: shows depth, pattern recognition, and a growth-oriented reflection.*

---

## 20. "What do you want to learn or grow into next?"

> "Two things. First, the building and design side of reliability — I'm strong on operating and observing, and I want to get deeper on designing observability platforms, writing infrastructure as code from scratch, and platform engineering. Building Athena was a deliberate step in that direction. Second, scale — I've operated a real platform, but I want to work somewhere with bigger scale and stronger SRE practices so I can learn things like SLO-driven engineering, capacity planning, and reliability at a level I haven't been exposed to yet. Honestly that's a big part of why I'm looking to move to a product-focused team."

*Why it works: self-aware about gaps, shows ambition, ties to why you're interviewing.*

---

# 1. KUBERNETES

**Q: What happens when you run `kubectl apply`? (asked constantly)**
kubectl sends the manifest to the API server, which authenticates, authorizes (RBAC), runs admission controllers, validates, and writes the object to etcd. etcd notifies the API server, which streams a watch event. The Deployment controller sees it and creates a ReplicaSet; the ReplicaSet controller creates Pod objects (still Pending, no node). The scheduler watches for unscheduled Pods, filters and scores nodes, and writes `nodeName` back. The kubelet on that node sees the Pod assigned to it, calls containerd via CRI to pull the image and start the container, and the CNI plugin assigns the IP. Once readiness passes, the endpoints controller adds the Pod IP to the EndpointSlice and kube-proxy programs iptables so traffic routes to it.

**Q: Liveness vs readiness vs startup probe?**
Readiness gates *traffic* — fail and the pod is removed from Service endpoints but not killed. Liveness gates *restarts* — fail and the kubelet kills and restarts the container. Startup probe is for slow-starting apps — it disables liveness/readiness until the app is up, so slow startup doesn't trigger a restart loop. Misconfigured readiness = silent traffic loss; misconfigured liveness = CrashLoopBackOff.

**Q: How does a Service route traffic to Pods?**
A Service gets a virtual ClusterIP — no process listens on it. CoreDNS resolves the Service name to that ClusterIP. kube-proxy on every node programs iptables (or IPVS) rules that DNAT the ClusterIP to one of the backend Pod IPs, picked from the EndpointSlice. The kernel does the rewrite; conntrack tracks it so return packets are rewritten back. So: DNS → ClusterIP → iptables DNAT → Pod IP.

**Q: Difference between Deployment, ReplicaSet, StatefulSet, DaemonSet?**
Deployment manages stateless apps via ReplicaSets and handles rolling updates. ReplicaSet just ensures N pods exist (you rarely create it directly). StatefulSet is for stateful apps needing stable network identity and ordered, persistent storage (pod-0, pod-1...). DaemonSet runs one pod per node — used for agents like log collectors, node-exporter, kube-proxy.

**Q: What's CrashLoopBackOff and how do you debug it?**
The container keeps crashing and Kubernetes backs off between restarts with exponential delay. Debug: `kubectl describe pod` for events and last state, `kubectl logs --previous` for the crashed container's logs. Common causes: app error on startup, failed liveness probe, missing config/secret, OOMKilled (exit 137), or a bad image. You isolate by reading why the previous container exited.

**Q: What's the difference between resource requests and limits?**
Requests are what the scheduler uses to place the pod — guaranteed minimum. Limits are the hard cap — exceed the memory limit and the container is OOMKilled; exceed CPU and it's throttled (not killed). The request/limit ratio determines the pod's QoS class (Guaranteed, Burstable, BestEffort), which affects eviction order under node pressure.

**Q: How does HPA work?**
Horizontal Pod Autoscaler watches a metric (CPU, memory, or custom), compares it against a target, and scales replica count. The formula is roughly: `desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)`. It polls the metrics-server (or custom metrics adapter) every ~15 seconds. It needs resource requests set to compute CPU/memory percentages.

**Q: What is etcd and what happens if it goes down?**
etcd is the distributed key-value store holding all cluster state, using Raft consensus (needs an odd number for quorum). If etcd is down, you can't make changes — no new pods, no edits — but existing workloads keep running because the kubelet has cached its assignments. It's the single source of truth, and only the API server talks to it.

**Q: Namespace vs label — when to use which?**
Namespaces are hard boundaries for isolation, RBAC, and resource quotas (e.g., dev/staging/prod, or per-team). Labels are flexible tags for selection and grouping within or across namespaces (e.g., `app=checkout`, `tier=backend`). You can't query across namespaces easily, but labels work anywhere.

---

# 2. DOCKER

**Q: What's the difference between an image and a container?**
An image is a read-only template — layered filesystem plus metadata. A container is a running (or stopped) instance of an image with a writable layer on top. Image is the class, container is the object.

**Q: What is a multi-stage build and why use it?**
A Dockerfile with multiple `FROM` stages. You build/compile in a heavy stage with all the build tools, then copy only the final artifact into a slim runtime stage. Result: much smaller final image, smaller attack surface, no build tools shipped to production. Classic example — compile a Go binary in a `golang` stage, copy it into a `scratch` or `alpine` final image.

**Q: How do Docker layers and caching work?**
Each instruction creates a layer. Docker caches layers and reuses them if the instruction and its inputs haven't changed. The key optimization: order your Dockerfile so things that change rarely (installing dependencies) come *before* things that change often (copying source code). Otherwise every code change busts the dependency cache and reinstalls everything.

**Q: CMD vs ENTRYPOINT?**
ENTRYPOINT is the fixed executable that always runs. CMD provides default arguments that can be overridden at `docker run`. Common pattern: ENTRYPOINT is the binary, CMD is the default flags.

**Q: Why is running as root in a container a problem?**
If the container is compromised and there's a container-escape vulnerability, root in the container can become root on the host. Best practice: create a non-root user in the Dockerfile and run as it (`USER appuser`). Many security policies (and Kubernetes admission controllers) reject root containers.

**Q: How do you reduce image size?**
Multi-stage builds, slim base images (alpine, distroless), combining RUN commands to reduce layers, cleaning package manager caches in the same layer, and using `.dockerignore` to avoid copying junk into the build context.

---

# 3. CI/CD (GitHub Actions / Azure DevOps)

**Q: Walk me through a typical CI/CD pipeline.**
On push/PR: checkout code → run tests and linting → build the Docker image → scan it (Trivy for vulnerabilities) → push to a registry (ACR) with a versioned tag → update the deployment manifest with the new tag → deploy (or, in GitOps, a tool like ArgoCD detects the manifest change and syncs it). Gates like manual approval sit before production.

**Q: Where do secrets live in a GitHub Actions pipeline?**
In GitHub repository/organization secrets (encrypted). The runner gets them injected as environment variables at runtime, scoped to the job. They're masked in logs. For cloud auth, the modern approach is OIDC federation — the runner gets a short-lived token from the cloud provider instead of storing long-lived credentials.

**Q: Push vs pull-based deployment (and why GitOps)?**
Push: CI has cluster credentials and pushes changes to the cluster (e.g., `kubectl apply` from the pipeline). Pull (GitOps): a controller inside the cluster (ArgoCD) watches Git and pulls changes in. Pull is preferred because cluster credentials never leave the cluster (more secure), Git becomes the single source of truth, and you get drift detection and easy rollback (revert the commit).

**Q: How do you do a rollback?**
Kubernetes: `kubectl rollout undo deployment/x` reverts to the previous ReplicaSet. GitOps: revert the Git commit and ArgoCD syncs back. Image-based: redeploy the previous image tag. The cleanest is GitOps revert because it's auditable.

**Q: Blue-green vs canary deployment?**
Blue-green: run two full environments (blue = current, green = new), test green, then switch all traffic at once. Instant rollback (switch back), but needs double resources. Canary: gradually shift a percentage of traffic to the new version (5% → 25% → 100%), watching metrics at each step. Lower blast radius, catches problems early, but slower and needs good observability to automate.

**Q: What's an image tagging strategy?**
Avoid `latest` in production — it's not immutable and you can't tell what's running. Use immutable tags: semantic versions (`v1.2.3`), git SHA (`sha-a1b2c3`), or build number. SHA-based tags guarantee you know exactly which commit is deployed.

---

# 4. TERRAFORM

**Q: What is Terraform state and why does it matter?**
State is Terraform's record of what it has created and the mapping between your config and real resources. Terraform compares desired config against state to compute what to change. Without it, Terraform wouldn't know what already exists. It can contain sensitive data, so it should be stored securely (remote backend), never in Git.

**Q: What happens if two engineers run `terraform apply` at the same time?**
Without locking, they can corrupt the state or create conflicting changes. The fix is state locking — a remote backend (S3 + DynamoDB, or Azure Storage with blob lease) locks the state during an operation so the second apply waits. This is the #1 reason to use a remote backend with locking.

**Q: `terraform plan` vs `apply` vs `refresh`?**
`plan` shows what changes *would* happen (dry run, diff between config and state). `apply` executes those changes. `refresh` updates the state to match real-world resources without changing infrastructure (now folded into plan/apply by default in newer versions).

**Q: What is a Terraform module?**
A reusable, parameterized package of resources. Instead of copying the same VNet/cluster config across dev/staging/prod, you write a module once and call it with different variables per environment. Promotes consistency and DRY infrastructure.

**Q: What is drift and how do you detect it?**
Drift is when the real infrastructure differs from what's in state — usually because someone made a manual change in the console. `terraform plan` detects it by comparing real state against config. Best practice is to avoid manual changes entirely and make everything go through Terraform.

**Q: How do you manage secrets in Terraform?**
Never hardcode them. Use variables marked `sensitive`, pull from a secret manager (Key Vault, Vault, AWS Secrets Manager) at apply time, and ensure the state backend is encrypted since state can contain secrets in plaintext.

*(Honest note for you: Terraform is your lighter area. Know these conceptually — state, locking, plan/apply, modules, drift — and if pushed deeper, be honest: "I've modified module variables and worked with state in our CloudFormation-to-Terraform migration; I'm deepening my module-authoring skills." Honesty beats bluffing here.)*

---

# 5. OBSERVABILITY (your strongest area — go deep)

**Q: Difference between metrics, logs, and traces?**
Metrics are numeric time-series, cheap to store, good for dashboards and alerts ("error rate is 5%"). Logs are discrete event records, good for detail ("here's the exact error message"). Traces follow a single request across services, good for "where in the chain did it slow down?" The three correlate — a metric anomaly leads you to a trace leads you to the log line.

**Q: Why does Prometheus use a pull model?**
Prometheus scrapes targets rather than receiving pushed data. Benefits: you get a free up/down signal (the `up` metric — if a scrape fails, the target is down), centralized control of scrape frequency, and no need for every service to know where to push. The downside is short-lived jobs may die before being scraped — solved with the Pushgateway for those specific cases.

**Q: What's the RED method? The USE method?**
RED (for services/requests): Rate, Errors, Duration. USE (for resources): Utilization, Saturation, Errors. RED tells you how your service is behaving from the user's view; USE tells you about the health of underlying resources like CPU, memory, disk.

**Q: What's an SLI, SLO, and error budget?**
SLI = the measured indicator (e.g., % of successful requests). SLO = the target for that SLI (e.g., 99.5% over 30 days). Error budget = 1 minus the SLO (0.5%) — the allowed amount of failure. The error budget gives you a data-driven way to balance reliability vs feature velocity: budget left = ship features; budget burned = focus on reliability.

**Q: What's burn-rate alerting and why is it better than threshold alerting?**
Instead of "alert if error rate > 1%," burn-rate alerts on how fast you're consuming the error budget. Multi-window burn-rate requires both a short and a long window to confirm before firing — so a momentary spike doesn't page anyone, but a sustained problem pages fast. It aligns alerts with real reliability risk rather than arbitrary thresholds, dramatically reducing false pages.

**Q: Loki vs ELK?**
Loki indexes only metadata labels (namespace, pod), not log content — much cheaper, lower operational burden, and works on the assumption you query logs by context you already know. ELK indexes full text — powerful for ad-hoc search and analytics, but expensive at scale. Loki for cost-efficient operational logs; ELK when you need deep full-text search or SIEM.

**Q: What does a Prometheus histogram give you?**
Buckets of observations (e.g., request durations in <0.1s, <0.5s, <1s buckets). You use `histogram_quantile()` to compute percentiles like p99 from the buckets. Important: percentiles are estimated from bucket boundaries, and you can't average percentiles across instances — you aggregate the buckets first, then compute the quantile.

**Q: What's the difference between Counter, Gauge, Histogram, Summary?**
Counter only goes up (total requests) — use `rate()` on it. Gauge goes up and down (memory usage, queue depth). Histogram buckets observations for percentile calculation server-side. Summary computes quantiles client-side (can't aggregate across instances). For latency, prefer Histogram because it aggregates.

**Q: How do you reduce alert fatigue?**
Tune thresholds to real user impact, add `for` durations so transient spikes don't fire, use burn-rate instead of static thresholds, add inhibition rules (suppress symptom alerts when a root-cause alert is firing), group related alerts, and remove alerts that don't require human action — those are dashboard metrics, not alerts.

---

# 6. LINUX & TROUBLESHOOTING

**Q: A server is slow. How do you start debugging?**
Start broad with the USE method. `top`/`htop` for CPU and memory, `load average` (is it CPU-bound?). `free -h` for memory. `df -h` and `du` for disk. `iostat` for disk I/O. `ss -s` for network connections. Then narrow — which process? `ps`, `pidstat`. The discipline is top-down: system → subsystem → process → cause.

**Q: How do you find what's using a port?**
`ss -tlnp | grep :8080` (or `netstat -tlnp`) shows the listening process and PID. `ss` is the modern replacement for `netstat` — faster and the kernel's preferred tool.

**Q: Disk shows full with `df` but `du` doesn't add up. Why?**
A process is holding open a deleted file — the space isn't freed until the process closes the handle. Find it with `lsof | grep deleted`. The fix is to restart or signal the process (or truncate the file). This is exactly why my cleanup script truncates large active logs instead of deleting them.

**Q: How do you check why a process was killed?**
`dmesg | grep -i kill` or check `/var/log/messages` for OOM killer activity. In Kubernetes, `kubectl describe pod` shows OOMKilled with exit code 137 (128 + SIGKILL 9).

**Q: What's the difference between RSS and VSZ?**
VSZ is virtual memory size — everything the process *could* address, including unused. RSS is resident set size — actual physical RAM in use. RSS is what matters for "is this process using too much memory."

**Q: How do you debug DNS resolution issues?**
`nslookup` or `dig` to test resolution. Check `/etc/resolv.conf` for the configured nameserver. In Kubernetes, exec into a pod and check it points to CoreDNS, then test `nslookup kubernetes.default`. If external DNS fails but internal works, it's likely CoreDNS upstream config or a network policy.

**Q: A pod can't reach an external service. How do you debug?**
Exec into the pod (or a netshoot debug pod). Test DNS (`nslookup`), test connectivity (`curl`/`nc`), check if it's a network policy blocking egress, check if the node itself can reach it (rule out cluster vs node issue), and check the service's own health. Isolate layer by layer.

---

# 7. CLOUD / AZURE (your platform)

**Q: What is AKS and what does Azure manage vs you?**
Azure Kubernetes Service is managed Kubernetes. Azure manages the control plane (API server, etcd, scheduler, controller-manager) — you don't see or pay for those nodes. You manage the worker node pools, your workloads, and configuration. You get managed upgrades, scaling, and integration with Azure AD, ACR, and Azure Monitor.

**Q: How do pods get Azure credentials securely?**
Workload Identity (the modern way) — federates a Kubernetes service account with an Azure AD identity, so pods get short-lived tokens with no stored secrets. The older way was pod-managed identities. The principle: no long-lived credentials in the cluster.

**Q: What's the difference between Azure Monitor, App Insights, and Log Analytics?**
Log Analytics is the underlying data store and query engine (KQL). Application Insights is the APM layer on top — request tracing, dependencies, exceptions for your apps. Azure Monitor is the umbrella platform tying metrics, logs, and alerts together. App Insights data lands in Log Analytics workspaces.

---

Kubernetes internals it is. This is the area that separates "I operate Kubernetes" from "I understand Kubernetes" — and it's the most common place SRE interviews go deep. I'm going to give you 25 advanced questions across the control plane, networking, scheduling, storage, and failure modes.

These are harder than the basics. Read each, try to answer out loud before checking, and pay special attention to the "follow-up" notes — that's where interviewers actually probe.

---

# CONTROL PLANE & RECONCILIATION

**Q1: Walk me through the reconciliation loop. What does "level-triggered" mean and why does Kubernetes use it?**

Controllers continuously compare desired state (in etcd) against actual state and take action to close the gap. Level-triggered means controllers react to the *current state*, not to the *event* that caused it — they look at "what is true now," not "what changed." This is why Kubernetes is self-healing: if a controller misses an event (crash, network blip), it still reconciles correctly on the next loop because it re-reads the full current state. Edge-triggered systems break if they miss an event; level-triggered systems recover. *Follow-up: "What if a controller crashes mid-reconcile?" → It re-reads state on restart and continues; reconciliation is idempotent by design.*

**Q2: How do controllers actually learn about changes without polling etcd constantly?**

They use the watch mechanism through the API server. A controller opens a long-lived HTTP connection (a "watch") to the API server, which streams events whenever a relevant object changes. Internally, client-go uses an "informer" — it does an initial LIST to build a local cache, then WATCHes for deltas, keeping the cache in sync. Controllers read from this local cache, not from etcd directly, which massively reduces API server load. *Follow-up: "What's a resync period?" → Informers periodically re-list everything as a safety net against missed events.*

**Q3: The API server is the only component that talks to etcd. Why is that architecturally important?**

It centralizes authentication, authorization, admission control, validation, and audit logging in one place. etcd has no concept of RBAC or admission — if components talked to it directly, every security control would have to be reimplemented everywhere. It also means you can swap or shard the datastore behind the API server without touching any other component. *Follow-up: "Could you run Kubernetes on a different datastore?" → Yes — k3s uses SQLite/others via a shim called kine.*

**Q4: What are the phases of admission control, and what's the difference between mutating and validating admission?**

After authn/authz, the request hits admission controllers in two phases: mutating admission runs first (can modify the object — e.g., inject sidecars, set defaults, add labels), then validating admission runs (can only accept or reject — e.g., enforce policy, reject root containers). Order matters: mutate then validate, so validation sees the final mutated object. Tools like Kyverno and OPA Gatekeeper plug in here via webhooks. *Follow-up: "Why mutating before validating?" → So policies validate the final state after all mutations, not an intermediate one.*

**Q5: What happens to a running cluster if etcd loses quorum?**

etcd uses Raft, which needs a majority (quorum) to accept writes. Lose quorum (e.g., 2 of 3 members down) and etcd goes read-only — no writes. The API server can't persist changes, so no new pods, no edits, no scaling. But existing workloads keep running because kubelets have cached their pod specs and keep them alive. Recovery means restoring quorum (bring members back) or restoring from an etcd snapshot. *Follow-up: "Why an odd number of etcd members?" → To form a clear majority and tolerate failures efficiently — 3 tolerates 1 failure, 5 tolerates 2; an even number gives no fault-tolerance benefit over the odd number below it.*

---

# SCHEDULING

**Q6: Explain the scheduler's two phases in detail.**

Filtering (predicates): eliminate nodes that can't run the pod — insufficient CPU/memory vs requests, unschedulable taints not tolerated, failed nodeSelector/nodeAffinity, port conflicts, volume zone mismatches. Scoring (priorities): rank the surviving feasible nodes — least requested resources (spread load), affinity preferences, image locality (node already has the image), topology spread. Highest score wins; the scheduler writes `nodeName` to the pod. *Follow-up: "What if no node passes filtering?" → Pod stays Pending with a FailedScheduling event; if cluster autoscaler is present, it may provision a new node.*

**Q7: Difference between node affinity, pod affinity, and pod anti-affinity?**

Node affinity attracts a pod to nodes with certain labels (e.g., "run on GPU nodes"). Pod affinity attracts a pod to nodes where certain *other pods* run (e.g., "co-locate with the cache for low latency"). Pod anti-affinity repels — keep replicas apart (e.g., "don't put two replicas on the same node/zone"). Each can be `requiredDuringScheduling` (hard) or `preferredDuringScheduling` (soft). *Follow-up: "Why is pod anti-affinity expensive at scale?" → It evaluates against all matching pods cluster-wide, which is computationally costly; topologySpreadConstraints is the lighter modern alternative.*

**Q8: What are taints and tolerations, and how do they differ from affinity?**

Taints are applied to *nodes* and repel pods that don't tolerate them ("keep pods off unless they explicitly accept this"). Tolerations are on *pods* and let them schedule onto tainted nodes. It's the inverse of affinity: affinity is a pod *attracting* itself to nodes; taints are nodes *repelling* pods. Used for dedicated nodes (GPU, system), and the node controller auto-taints unhealthy nodes (`NotReady`) to evict pods. *Follow-up: "What are the three taint effects?" → NoSchedule (don't schedule new), PreferNoSchedule (try not to), NoExecute (evict existing pods that don't tolerate).*

**Q9: A pod is Pending. Walk me through every reason and how you'd diagnose.**

`kubectl describe pod` and read the events. Causes: insufficient resources (no node has enough CPU/memory for the requests), taints not tolerated, node affinity/selector matches nothing, unbound PVC (no available PersistentVolume or storage class), pod anti-affinity can't be satisfied, or the image can't be pulled (though that's usually ContainerCreating, not Pending). The event message almost always names the exact reason. *Follow-up: "Pending due to resources but `kubectl top nodes` shows free capacity?" → top shows *usage*, but the scheduler places on *requests*; nodes may be "full" on requested resources while actual usage is low.*

---

# NETWORKING (high-frequency for SRE)

**Q10: Explain the Kubernetes network model's core requirements.**

Three rules: every pod gets its own IP; pods can communicate with all other pods across nodes without NAT; and the IP a pod sees itself as is the same IP others use to reach it. No port mapping, no NAT between pods. The CNI plugin implements this — how it does so (overlay like VXLAN, or native routing like Azure CNI/AWS VPC CNI) varies, but the model is constant. *Follow-up: "Overlay vs native routing tradeoff?" → Overlay (Flannel VXLAN) works anywhere but adds encapsulation overhead; native (Azure CNI) gives real VNet IPs and better performance but consumes VNet IP space.*

**Q11: How does kube-proxy in iptables mode actually work, and what's the difference from IPVS mode?**

In iptables mode, kube-proxy programs a chain of iptables rules: traffic to a ClusterIP matches a rule that uses statistical probability to DNAT to one of the backend pod IPs. The problem: iptables rules are evaluated sequentially, so with thousands of services, rule processing becomes O(n) and slow. IPVS mode uses the kernel's IP Virtual Server with hash tables — O(1) lookup, better at scale, and more load-balancing algorithms (round-robin, least-connection). *Follow-up: "Is kube-proxy in the data path?" → No — it programs the rules ahead of time; the kernel handles each packet. kube-proxy isn't a proxy in modern modes despite the name.*

**Q12: Trace a DNS lookup from inside a pod, end to end.**

The pod's `/etc/resolv.conf` (injected by the kubelet) points its nameserver to the CoreDNS ClusterIP. A lookup for `svc.ns.svc.cluster.local` goes to CoreDNS. CoreDNS watches the API server for Services and Endpoints and answers from its in-memory data — returning the Service's ClusterIP. The `ndots:5` setting and search domains mean short names get suffixed and tried in order. CoreDNS forwards anything outside `cluster.local` to the upstream resolver. *Follow-up: "Why is ndots:5 a performance concern?" → A name with fewer than 5 dots gets each search domain appended and queried first, causing multiple failed lookups before the real one — a known latency source; FQDNs with a trailing dot skip this.*

**Q13: What's the difference between a NodePort, ClusterIP, and LoadBalancer Service at the implementation level?**

ClusterIP: virtual IP reachable only inside the cluster, implemented as kube-proxy iptables/IPVS rules. NodePort: same as ClusterIP plus kube-proxy opens a port (30000–32767) on every node that forwards to the service. LoadBalancer: same as NodePort plus the cloud controller provisions an external cloud load balancer pointing at those node ports. They're layered — each builds on the previous. *Follow-up: "Why is NodePort rarely used directly?" → Limited port range, no TLS, no L7 routing, exposes node IPs; an Ingress + single LoadBalancer is almost always better.*

**Q14: How does an Ingress controller differ from a Service of type LoadBalancer?**

A LoadBalancer Service is L4 (TCP) — one cloud LB per service, expensive and no HTTP awareness. An Ingress controller (nginx, etc.) is L7 (HTTP) — it runs as pods, reads Ingress resources, and routes by host/path to many backend services, all behind a *single* LoadBalancer. So you pay for one cloud LB and route everything through the Ingress with TLS termination, path routing, and rate limiting. *Follow-up: "What's the Gateway API?" → The newer, more expressive successor to Ingress, separating infrastructure (Gateway) from routing (HTTPRoute) with richer L4/L7 capabilities.*

**Q15: A Service has endpoints but traffic still fails intermittently. What are the likely causes?**

Possibilities: a pod is Ready but the app isn't truly serving (readiness probe too lenient); kube-proxy reconciliation lag after an endpoint change; a terminating pod still in the endpoint list briefly (no preStop hook / graceful shutdown, so it gets traffic mid-shutdown); conntrack table full dropping connections; or one backend pod is unhealthy but passing readiness. *Follow-up: "How do you avoid traffic to terminating pods?" → readiness probe + a preStop hook with a sleep, so the pod is removed from endpoints before it stops accepting connections.*

---

# STORAGE

**Q16: Explain the PV / PVC / StorageClass relationship.**

A PersistentVolume (PV) is a piece of actual storage. A PersistentVolumeClaim (PVC) is a request for storage by a pod. A StorageClass defines *how* to dynamically provision PVs (which provisioner, disk type, parameters). Flow: pod references a PVC → PVC references a StorageClass → the provisioner creates a PV and binds it to the PVC. Static provisioning means an admin pre-creates PVs; dynamic means the StorageClass creates them on demand. *Follow-up: "What's a reclaim policy?" → Delete (PV and underlying disk deleted when PVC is deleted) vs Retain (kept for manual recovery).*

**Q17: Why can't a typical cloud block volume be mounted to pods on different nodes simultaneously, and how does that interact with Deployments?**

Cloud block storage (Azure Disk, EBS) is ReadWriteOnce — mountable by one node at a time. If a Deployment with such a volume tries to schedule pods on multiple nodes, only one can mount it; others fail. Also, during a rolling update, the new pod can't mount the disk until the old pod releases it, causing a stall. For multi-node shared access you need ReadWriteMany storage (Azure Files, NFS). This is a classic reason StatefulSets (one volume per pod) suit stateful workloads better than Deployments. *Follow-up: "Access modes?" → RWO (one node), ROX (many nodes read-only), RWX (many nodes read-write).*

**Q18: How does a StatefulSet differ from a Deployment in pod and storage management?**

StatefulSet gives stable, ordered identities: pods are named `name-0`, `name-1`, created/scaled/deleted in order, each with a stable network identity (via a headless Service) and its *own* persistent volume that follows it across rescheduling. A Deployment treats pods as interchangeable cattle with shared or no persistent identity. Use StatefulSet for databases, Kafka, anything needing stable identity or per-pod storage. *Follow-up: "What's a headless Service?" → `clusterIP: None` — DNS returns individual pod IPs instead of a single VIP, giving each StatefulSet pod a stable DNS name.*

---

# FAILURE MODES & OPERATIONS

**Q19: What's the full lifecycle of a node going NotReady, and what happens to its pods?**

The kubelet stops sending heartbeats (node lease updates). After `node-monitor-grace-period` (~40s), the node controller marks it NotReady and applies a `NotReady:NoExecute` taint. Pods without a matching toleration get evicted after `tolerationSeconds` (default 300s) — so pods aren't immediately rescheduled, giving the node time to recover. After that, pods are deleted and recreated elsewhere (if managed by a controller). *Follow-up: "Why the 5-minute delay?" → To avoid mass rescheduling churn for transient network blips; a flapping node shouldn't trigger a stampede.*

**Q20: Explain the pod termination sequence when you delete a pod.**

The pod is marked Terminating and removed from Service endpoints (so it stops receiving new traffic). The kubelet runs the preStop hook if defined, then sends SIGTERM to the container. It waits up to `terminationGracePeriodSeconds` (default 30s) for graceful shutdown. If the container hasn't exited, it sends SIGKILL. The endpoint removal and SIGTERM happen roughly concurrently, which is why you often add a preStop sleep — to ensure in-flight requests drain before the process dies. *Follow-up: "Why might you still get errors during termination?" → Endpoint removal propagates asynchronously to kube-proxy on all nodes; a brief window exists where traffic still routes to the terminating pod, which the preStop sleep covers.*

**Q21: What's a PodDisruptionBudget and when does it actually protect you?**

A PDB sets the minimum available (or max unavailable) pods during *voluntary* disruptions — node drains, cluster upgrades, autoscaler scale-down. It does NOT protect against involuntary disruptions (node crash, OOM). So during a `kubectl drain` for an upgrade, the PDB ensures you don't take down too many replicas at once. *Follow-up: "Can a PDB block a node drain forever?" → Yes — if honoring the PDB is impossible (e.g., minAvailable equals replica count), the drain blocks, which is a real operational gotcha.*

**Q22: You see a pod stuck in Terminating forever. Why, and how do you handle it?**

Common causes: a finalizer on the object waiting for cleanup that never completes; the kubelet on the node is down so it can't confirm deletion; a volume that won't unmount; or a process ignoring SIGTERM with no SIGKILL escalation reaching it. Diagnose with `kubectl describe`. If it's a stuck finalizer, you can patch it out (carefully). If the node is dead, you may force-delete (`--grace-period=0 --force`), but understand that just removes it from the API — the actual container may still run if the node comes back. *Follow-up: "Risk of force-delete on StatefulSet?" → Two pods with the same identity could run simultaneously (split-brain) if the node wasn't truly dead — dangerous for databases.*

**Q23: Explain QoS classes and pod eviction order under node memory pressure.**

Three classes: Guaranteed (requests == limits for all resources), Burstable (requests < limits), BestEffort (no requests/limits). Under node memory pressure, the kubelet evicts in order: BestEffort first, then Burstable exceeding requests, Guaranteed last. The OOM killer also uses an oom_score influenced by QoS. So setting proper requests/limits isn't just scheduling — it determines who dies first under pressure. *Follow-up: "How to make a critical pod survive pressure?" → Give it Guaranteed QoS (requests == limits) and/or a high priorityClass.*

**Q24: What's the difference between PriorityClass-based preemption and QoS eviction?**

Preemption is scheduler-driven: a high-priority pending pod can evict lower-priority running pods to make room for scheduling. QoS eviction is kubelet-driven under node resource pressure. They're different mechanisms: preemption is about *fitting* a pod that can't schedule; eviction is about *relieving* pressure on an already-full node. *Follow-up: "Can preemption violate a PDB?" → The scheduler tries to respect PDBs during preemption but will violate them if necessary to schedule a higher-priority pod.*

**Q25: A Deployment rollout is stuck — `kubectl rollout status` hangs. Walk through the diagnosis.**

Check `kubectl rollout status` and `kubectl get rs` to see old vs new ReplicaSet replica counts. Likely causes: new pods failing readiness (so the rollout won't progress past maxUnavailable), image pull errors on the new image, insufficient cluster resources to schedule the surge pods, a CrashLoopBackOff in the new version, or a PDB blocking the old pods from terminating. `kubectl describe` the new ReplicaSet's pods for events. The rollout is gated on new pods becoming Ready, so it's almost always a readiness or scheduling problem with the new version. *Follow-up: "What are maxSurge and maxUnavailable?" → maxSurge = how many extra pods above desired count during rollout; maxUnavailable = how many below. Tuning these controls rollout speed vs availability.*

---
All three. Good — networking, observability, and Linux debugging are the trio that SRE interviews lean on hardest, and they reinforce each other (most real incidents touch all three). This is a big set, so I'm giving you the most interview-frequent, highest-leverage questions in each, with the follow-ups that actually trip people up.

Drill by active recall: cover the answer, say it out loud, then check.

---

# PART A — KUBERNETES NETWORKING (CNI / kube-proxy / CoreDNS)

**A1: What does a CNI plugin actually do when a pod starts?**
When the kubelet creates a pod, it calls the CNI plugin via the runtime. The plugin: allocates an IP from the cluster CIDR (IPAM), creates a veth pair connecting the pod's network namespace to the node's, configures the pod-side interface and routes, and programs the node so traffic to/from that pod IP works. On pod deletion it reverses this and returns the IP. *Follow-up: "Who calls the CNI — kubelet or runtime?" → The container runtime (containerd) invokes the CNI plugin, orchestrated by the kubelet.*

**A2: Overlay vs native (underlay) networking — explain the real tradeoff.**
Overlay (Flannel VXLAN, Calico with IPIP/VXLAN) encapsulates pod packets inside node-to-node packets — works on any network because the underlying network only sees node IPs, but adds encapsulation overhead and ~50 bytes per packet (MTU concerns). Native/underlay (Azure CNI, AWS VPC CNI, Calico BGP) gives pods real routable IPs from the VNet/VPC — better performance, real IPs visible to the network, but consumes IP address space and depends on the cloud network. *Follow-up: "Why might Azure CNI exhaust IPs?" → Every pod gets a real VNet IP, and nodes pre-allocate a block of IPs per node, so large clusters can drain the subnet fast.*

**A3: What is the MTU problem in overlay networks?**
Encapsulation (VXLAN adds ~50 bytes) means the inner packet must be smaller than the physical MTU, or packets get fragmented or dropped. If pod MTU isn't lowered to account for the overlay header, you get mysterious failures — small packets work, large ones (like a big HTTP response) hang or fail. Classic symptom: TLS handshake works, then the connection stalls on the first large payload. *Follow-up: "How do you diagnose it?" → ping with `-M do` (don't fragment) and increasing packet sizes to find where it breaks.*

**A4: kube-proxy iptables mode — walk through the actual rule chain.**
Traffic to a ClusterIP hits the `KUBE-SERVICES` chain, which matches the service and jumps to a per-service chain (`KUBE-SVC-xxx`). That chain uses the `statistic` module with random probability to pick one of the backend endpoint chains (`KUBE-SEP-xxx`), which DNATs the destination to a real pod IP. conntrack records the mapping so replies are un-NATed. *Follow-up: "Why is iptables mode O(n)?" → Rules are evaluated sequentially; thousands of services mean thousands of rules to traverse per connection setup, increasing latency and rule-sync time.*

**A5: IPVS mode — what changes and why is it better at scale?**
IPVS uses kernel hash tables instead of sequential iptables rules, giving O(1) service lookup regardless of service count. It also offers real load-balancing algorithms (round-robin, least-connection, source-hash) versus iptables' random selection. It still uses a few iptables rules for things like masquerading, but the load-balancing decision is in IPVS. *Follow-up: "Does IPVS eliminate iptables entirely?" → No — it still needs iptables/ipset for packet marking, masquerade, and a few cases, but the per-service routing is IPVS.*

**A6: What is conntrack and how does it cause production incidents?**
conntrack is the kernel's connection tracking table that remembers NAT mappings so return traffic is rewritten correctly. The incident: the table has a max size (`nf_conntrack_max`). Under high connection churn, it fills up, and new connections get dropped — you see `nf_conntrack: table full, dropping packet` in dmesg, and intermittent connection failures that look like the app's fault but aren't. *Follow-up: "Fix?" → Raise `nf_conntrack_max`, reduce connection churn (connection pooling, keepalive), or for certain traffic use `NOTRACK` rules.*

**A7: Trace a CoreDNS lookup end to end, including ndots.**
Pod's `/etc/resolv.conf` has `nameserver <CoreDNS ClusterIP>`, `search <ns>.svc.cluster.local svc.cluster.local cluster.local`, and `options ndots:5`. A query for `payments` (0 dots) — because 0 < 5, the resolver appends each search domain and tries them in order before trying `payments` as-is. CoreDNS, which watches the API for Services/Endpoints, answers `svc.cluster.local` names from memory; anything else it forwards upstream. *Follow-up: "Why does ndots:5 hurt external lookups?" → Calling `api.stripe.com` (2 dots, < 5) triggers `api.stripe.com.ns.svc.cluster.local` etc. first — multiple failed queries before the real one. Fix: trailing dot (`api.stripe.com.`) to force FQDN.*

**A8: CoreDNS is slow / DNS latency is high cluster-wide. How do you debug and fix?**
Check CoreDNS pod CPU/memory and replica count (it's a Deployment, often under-provisioned). Look at the ndots:5 amplification (every short name = multiple queries). Check for conntrack exhaustion on UDP port 53. Solutions: scale CoreDNS replicas, enable `autopath` plugin, deploy NodeLocal DNSCache (a per-node DNS cache that cuts cross-node DNS traffic and conntrack pressure dramatically), and set ndots lower or use FQDNs in apps. *Follow-up: "What's NodeLocal DNSCache?" → A DaemonSet running a DNS cache on each node, so pods hit a local cache over TCP to CoreDNS, reducing latency and UDP conntrack load.*

**A9: What is a NetworkPolicy and what enforces it?**
A NetworkPolicy defines allowed ingress/egress for pods by label selector — default-deny once a policy selects a pod. Crucially, the API object does nothing by itself; the CNI plugin must support and enforce it (Calico, Cilium do; Flannel alone does not). So you can apply a NetworkPolicy on a cluster whose CNI ignores it, and it silently has no effect. *Follow-up: "Common gotcha?" → Applying policies but the CNI doesn't enforce them, giving false security; or forgetting that a policy selecting a pod flips it to default-deny for that direction.*

**A10: How does east-west pod-to-pod traffic differ on the same node vs across nodes?**
Same node: traffic goes pod veth → node bridge/routing → destination pod veth, never leaving the host — microsecond latency, no encapsulation. Across nodes: it goes out the node's network — either encapsulated (overlay) or routed natively (underlay) to the destination node, then to the pod. The CNI handles the cross-node path. *Follow-up: "Why does this matter for performance?" → Topology-aware routing / topologySpreadConstraints can keep chatty services on the same node/zone to cut latency and cross-zone egress cost.*

---

# PART B — OBSERVABILITY + PromQL (your strongest area, so go deep)

**B1: Explain `rate()` vs `irate()` vs `increase()`.**
`rate()` = per-second average rate over the whole range window (smoothed, good for alerting/graphing). `irate()` = instantaneous rate using only the last two data points (spiky, good for fast-moving graphs, bad for alerting). `increase()` = total increase over the window (= rate × window seconds), good for "how many X in the last hour." All three only work on counters. *Follow-up: "Why never alert on irate?" → It's too noisy — based on two samples, it swings wildly; rate over a longer window is stable.*

**B2: Why must you `rate()` before `sum()`, never `sum()` before `rate()`?**
Counters reset to zero on pod restart. `rate()` is reset-aware — it detects the drop and corrects. If you `sum()` raw counters first, a single pod restart makes the summed counter appear to drop massively, and then `rate()` on that sum sees a huge negative-then-positive jump, producing garbage. Always `sum(rate(metric[5m]))`, never `rate(sum(metric)[5m])`. *Follow-up: "What about aggregation labels?" → `sum(rate(...)) by (label)` aggregates after the per-series rate is computed correctly.*

**B3: How do you compute p99 latency, and what's the catch with averaging percentiles?**
`histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`. The catch: you cannot average percentiles across instances — p99 of (p99_a, p99_b) is mathematically meaningless. You must aggregate the raw histogram *buckets* first (sum by `le`), then compute the quantile once. *Follow-up: "Histogram vs Summary for this?" → Histogram computes quantiles server-side from buckets and is aggregatable across instances; Summary computes client-side and can't be aggregated — so for distributed services, use Histogram.*

**B4: What's cardinality and why is it the #1 cause of Prometheus blowing up?**
Cardinality = number of unique time series, driven by label combinations. Each unique combination of label values is a separate series stored in memory. Putting high-cardinality values in labels (user ID, request ID, email, full URL with IDs) creates millions of series, exhausting memory and crashing Prometheus. *Follow-up: "How do you control it?" → Never label with unbounded values; use relabeling to drop dangerous labels; monitor `prometheus_tsdb_head_series`; aggregate or move high-cardinality data to logs/traces, not metrics.*

**B5: Explain the four golden signals and map them to PromQL.**
Latency: `histogram_quantile(0.99, ...)`. Traffic: `sum(rate(http_requests_total[5m]))`. Errors: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`. Saturation: resource usage vs capacity, e.g., `container_memory_working_set_bytes / kube_pod_container_resource_limits`. *Follow-up: "Golden signals vs RED vs USE?" → Golden signals (Google) = latency/traffic/errors/saturation. RED (Weave) = rate/errors/duration, for request-driven services. USE (Gregg) = utilization/saturation/errors, for resources. RED ≈ the request-facing subset of golden signals.*

**B6: What does the `up` metric tell you and why is it special?**
`up` is auto-generated by Prometheus per scrape target: 1 if the scrape succeeded, 0 if it failed. It's the foundation of the pull model's value — you get a free liveness signal for every target without the target doing anything. Alert `up == 0 for 5m` catches a down service. *Follow-up: "What does `absent()` add?" → `up` is 0 when a known target fails; `absent(up{job="x"})` fires when the target *disappears entirely* (e.g., the whole job vanished), which `up == 0` can't catch because there's no series.*

**B7: Explain multi-window, multi-burn-rate alerting in detail.**
You alert on how fast the error budget burns relative to the SLO. Burn rate = (error rate) / (1 − SLO). For 99.5% SLO, burning at 14.4× exhausts the 30-day budget in ~2 days. Multi-window means a fast (short) and a slow (long) window must *both* breach before firing — the short window ensures it's happening now, the long window confirms it's sustained, eliminating flapping. Multiple burn-rate tiers (fast page, slow ticket) match urgency to severity. *Follow-up: "Why two windows per alert?" → The long window prevents firing on a brief spike; the short window ensures the alert clears quickly once resolved (fast recovery).*

**B8: Recording rules — what are they and when do you use them?**
A recording rule pre-computes an expensive or frequently-used query at scrape interval and stores the result as a new metric. Use them for: dashboard queries that are slow (heavy aggregations), SLI computations referenced by multiple alerts, and any query evaluated often. They trade a little storage for big query-time savings. *Follow-up: "Naming convention?" → `level:metric:operation`, e.g., `sli:checkout_availability:ratio_rate5m` — indicates aggregation level and what it represents.*

**B9: Loki's LogQL — how does it differ from PromQL conceptually?**
LogQL has two parts: a log stream selector (`{app="checkout"}`) using labels, then a filter/pipeline (`|= "error" | json | status="500"`). Because Loki only indexes labels, the selector uses the index (fast), but content filters scan the matched streams (slower the more you match). You can also generate metrics *from* logs (`rate({app="x"} |= "error" [5m])`). *Follow-up: "Why keep label cardinality low in Loki too?" → Same as Prometheus — each label combo is a stream; high cardinality (e.g., labeling by request ID) destroys Loki performance.*

**B10: How do metrics, logs, and traces correlate in practice during an incident?**
A metric alert fires (error rate spike). You pivot to the trace for a failing request to see *which service/span* failed and where the latency went. From the trace's span you jump to the exact logs for that trace ID to read the error detail. Metrics tell you *something's wrong and where roughly*, traces tell you *which hop*, logs tell you *exactly what*. Exemplars (trace IDs attached to metric samples) and shared trace IDs in logs make this jump seamless in Grafana. *Follow-up: "What's an exemplar?" → A sample trace ID attached to a metric data point, letting you click from a latency spike directly to an example slow trace.*

---

# PART C — LINUX PRODUCTION DEBUGGING

**C1: A server has high load average but low CPU usage. What's happening?**
Load average counts processes in runnable *and uninterruptible sleep* (D state) — usually blocked on I/O. High load + low CPU = I/O wait or blocked processes, not CPU saturation. Check `top` for `wa` (I/O wait), `iostat -x` for disk saturation, and `ps aux | awk '$8 ~ /D/'` for processes stuck in uninterruptible sleep. *Follow-up: "Why can't you kill a D-state process?" → It's in an uninterruptible kernel operation (often disk/NFS I/O); it won't respond to signals until the I/O completes or times out.*

**C2: `df` says disk full, `du` says there's space. Explain.**
A process has an open file handle to a file that was deleted. The directory entry is gone (so `du`, which walks the tree, doesn't count it) but the inode and its blocks aren't freed until the last handle closes — so `df`, which reads the filesystem's free-block count, still shows it used. Find it: `lsof +L1` or `lsof | grep deleted`. Fix: restart/signal the holding process, or truncate via `/proc/<pid>/fd/<n>`. *Follow-up: "Why truncate not delete in log cleanup?" → Deleting a log a process has open frees nothing; truncating the open file frees the blocks immediately.*

**C3: How do you find which process is using a port, and which is eating CPU/memory?**
Port: `ss -tlnp | grep :8080` (shows PID/process). CPU: `top` / `pidstat 1` (per-process CPU over time). Memory: `top` sorted by RES, or `ps aux --sort=-rss | head`. For a deeper memory breakdown: `/proc/<pid>/status` (VmRSS) or `pmap`. *Follow-up: "ss vs netstat?" → ss reads kernel socket data directly via netlink — faster and the modern replacement; netstat parses /proc and is slower/deprecated.*

**C4: RSS vs VSZ vs PSS — what's the difference and which matters?**
VSZ = total virtual address space the process *could* use (includes unmapped, shared libs, reserved) — usually large and misleading. RSS = resident physical RAM, but counts shared pages fully in every process sharing them (double-counts). PSS = proportional set size, splits shared pages fairly across sharers — the most accurate for "how much RAM does this really cost." For OOM concerns, RSS is the practical number. *Follow-up: "Why can summing RSS exceed total RAM?" → Shared libraries are counted in full for each process; PSS corrects this.*

**C5: How do you investigate why a process was OOM-killed?**
`dmesg -T | grep -i oom` or `journalctl -k | grep -i oom` shows the OOM killer's victim, its RSS, and the oom_score. The kernel kills the process with the highest oom_score (influenced by memory use and oom_score_adj). In Kubernetes, a container exceeding its memory limit gets cgroup-OOM-killed (exit 137); a node under global pressure triggers kubelet eviction. *Follow-up: "Cgroup OOM vs system OOM?" → Cgroup OOM kills within one container's limit (container restarts); system OOM is node-wide memory exhaustion (can kill anything, more dangerous).*

**C6: A process is hung. How do you find out what it's stuck on without killing it?**
`strace -p <pid>` shows the syscall it's blocked in (e.g., stuck in `read()` on a socket = waiting on network; `futex` = lock contention). `cat /proc/<pid>/stack` shows the kernel stack. `lsof -p <pid>` shows open files/sockets. `cat /proc/<pid>/wchan` shows the kernel function it's sleeping in. *Follow-up: "strace caveat in production?" → strace pauses the process on every syscall and adds significant overhead — use briefly and carefully on a hot production process.*

**C7: How do you debug "connection refused" vs "connection timed out"?**
Connection *refused* = the packet reached the host but nothing is listening on that port (RST returned) — service down, wrong port, or not bound. Connection *timed out* = no response at all — firewall dropping packets, wrong IP, routing issue, or network policy. The distinction tells you where to look: refused = the target host/app layer; timeout = the network/firewall layer. *Follow-up: "Tools?" → `curl -v`, `nc -zv host port`, `telnet`; `traceroute`/`mtr` for routing; check security groups/NetworkPolicies for timeouts.*

**C8: Walk through debugging high latency to an external API from a pod.**
Layer by layer: exec into the pod (or netshoot). DNS first — `time nslookup api.example.com` (is resolution slow? ndots amplification?). Then connectivity/latency — `curl -w "@curl-format" -o /dev/null -s` to break down DNS vs connect vs TLS vs TTFB. Check if it's the pod, the node (`curl` from the node), or the network. Check conntrack table fullness, MTU issues for large payloads, and the external service's own health. *Follow-up: "curl timing breakdown fields?" → `time_namelookup`, `time_connect`, `time_appconnect` (TLS), `time_starttransfer` (TTFB), `time_total` — isolates exactly which phase is slow.*

**C9: What is `dmesg` and what production issues does it reveal?**
`dmesg` is the kernel ring buffer — kernel-level events. It reveals: OOM kills, conntrack table full, disk I/O errors / SCSI errors (failing disk), network interface flaps, segfaults, filesystem errors (read-only remount), and TCP/network drops. It's the first place to look for issues that aren't in application logs because they're below the app layer. *Follow-up: "Where do these go long-term?" → `journalctl -k` (systemd) or `/var/log/dmesg` / `/var/log/kern.log`, since the ring buffer is finite and overwrites.*

**C10: How do you debug a "too many open files" error?**
The process hit its file-descriptor limit (sockets count as FDs). Check current limit: `ulimit -n` or `/proc/<pid>/limits`. Count open FDs: `ls /proc/<pid>/fd | wc -l`. Causes: FD leak (not closing connections/files), or legitimately high concurrency with a low limit. Fix: raise the limit (`LimitNOFILE` in systemd, `ulimit`, or pod securityContext), and fix the leak if it's unbounded growth. *Follow-up: "How does this manifest in a service?" → New connections fail, `accept()` returns EMFILE, the service appears hung or rejects clients while the process is still running.*

---
