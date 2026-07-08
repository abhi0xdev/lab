# EPAM SRE / DevOps Interview Prep — 100 Questions with Answers
**Target:** SRE/DevOps role, ~4–5 YOE, financial-domain client
**Stack focus:** DevOps, CI/CD (Jenkins/CloudBees), Kubernetes, Observability (Dynatrace/Splunk/Prometheus/Grafana), Cloud (AWS/Azure/GCP), SRE (SLI/SLO/error budgets/toil), Python & PowerShell scripting, Incident Management & RCA, Security.

> **How to use this:** Read the answer, then say it out loud in your own words in ~30–60 seconds. In a financial-domain interview, always tie answers back to **availability, latency, compliance/audit, security, and zero data loss**. Where you can, give a concrete example from your own experience ("In my last role we…").

---

## Section 1 — DevOps & SRE Fundamentals (Q1–12)

**Q1. What is the difference between DevOps and SRE?**
DevOps is a culture and set of practices that break the wall between Dev and Ops so software ships faster and more reliably — it's a philosophy. SRE is a concrete implementation of that philosophy, pioneered at Google, that applies software engineering to operations. The one-liner: "SRE is what happens when you ask a software engineer to design an operations team." DevOps says *reduce silos, accept failure as normal, measure everything*; SRE gives you the *specific tools* to do it — SLIs, SLOs, error budgets, toil limits, and blameless postmortems.

**Q2. What are the core principles of SRE?**
Embracing risk (100% reliability is the wrong target), setting SLOs backed by error budgets, eliminating toil through automation, monitoring/observability, release engineering with automated CI/CD, simplicity, and blameless postmortems. Google caps operational/toil work at ~50% so engineers spend the rest on engineering that reduces future toil.

**Q3. What is toil and how do you manage it?**
Toil is manual, repetitive, automatable, tactical work that scales linearly with service growth and has no lasting value — e.g., manually restarting a pod, running the same runbook every deploy. You manage it by measuring it (track hours spent), keeping it under a threshold (Google's ~50% cap), and prioritizing a "toil backlog" so the most painful/repetitive items get automated first. Reducing toil frees engineers for reliability work.

**Q4. Explain SLI, SLO, and SLA.**
- **SLI (Indicator):** a quantitative measure of service behavior, e.g., request latency, availability %, error rate.
- **SLO (Objective):** the internal target for an SLI, e.g., "99.9% of requests succeed over 30 days."
- **SLA (Agreement):** the external, contractual promise to customers with financial/legal consequences if breached. SLA is usually looser than SLO so you have internal headroom. In finance, SLAs are strict and audited, so you keep SLOs tighter than SLAs.

**Q5. What are the Four Golden Signals?**
Latency (time to serve a request — separate success vs error latency), Traffic (demand, e.g., requests/sec), Errors (rate of failed requests), and Saturation (how "full" the service is — CPU/memory/IO/queue depth). Monitor these four and you catch most user-facing problems. (RED = Rate, Errors, Duration for request-driven services; USE = Utilization, Saturation, Errors for resources.)

**Q6. What is an error budget and how do you use it?**
An error budget is `100% − SLO`. If your SLO is 99.9% availability, your budget is 0.1% of unreliability over the window. As long as budget remains, teams can ship features/take risks. When the budget is exhausted, you freeze risky releases and focus on reliability until it recovers. It turns reliability-vs-velocity from an argument into a data-driven decision — this is exactly the "partner with product owners to manage error budget" bullet in the JD.

**Q7. How do you calculate an error budget for 99.9% over 30 days?**
30 days ≈ 43,200 minutes. Allowed downtime = 0.1% × 43,200 ≈ **43.2 minutes/month**. For 99.99% ("four nines") it's ~4.32 min/month; for 99% it's ~7.2 hours/month. Know these numbers cold — interviewers love this.

**Q8. What SRE metrics / DORA metrics do you track?**
The four DORA metrics: **Deployment Frequency**, **Lead Time for Change** (commit → prod), **Change Failure Rate** (% deploys causing incidents), and **MTTR** (Mean Time To Restore). First two measure velocity, last two measure stability. Elite teams deploy on-demand, lead time under an hour, CFR 0–15%, MTTR under an hour.

**Q9. What is MTTR and how do you improve it?**
MTTR = Mean Time To Restore/Recover — average time from incident detection to service restoration. Improve it with better observability (fast detection), clear runbooks, automated rollback, feature flags to disable bad code, on-call training, and blameless postmortems that feed fixes back in. Related: MTTD (detect), MTTA (acknowledge), MTBF (between failures).

**Q10. What makes a good SLO?**
It should be user-centric (reflect what users actually experience), measurable via an SLI, achievable, and have a defined time window. Don't set 100% — it's impossible and every 9 costs exponentially more. Start from current performance, negotiate with product owners, and iterate.

**Q11. How do you decide what to alert on?**
Alert on symptoms, not causes — alert when users are affected (SLO burn, golden-signal breaches), not on every CPU spike. Every alert should be actionable and urgent; if there's nothing to do, it's a dashboard/ticket, not a page. Use multi-window, multi-burn-rate alerts on error budget: fast burn = page now, slow burn = ticket. Aim to reduce alert fatigue.

**Q12. What is a blameless postmortem and why does it matter?**
A postmortem documents an incident's timeline, impact, root cause, and action items without blaming individuals — the assumption is people act reasonably with the info they had, so failures are system/process failures. Blamelessness encourages honesty, which surfaces the real causes and prevents recurrence. In finance it also creates an audit trail regulators may want.

---

## Section 2 — CI/CD (Jenkins, CloudBees) (Q13–24)

**Q13. What is CI/CD?**
Continuous Integration = developers merge to a shared branch frequently, each merge triggering an automated build + test so integration bugs surface early. Continuous Delivery = every change that passes the pipeline is *ready* to deploy (deploy is a manual gate). Continuous Deployment = every passing change deploys to prod *automatically*. In finance you often do Continuous *Delivery* with a manual approval gate for change-control/compliance.

**Q14. Walk me through a typical CI/CD pipeline.**
Source (commit/PR triggers webhook) → Build (compile, package, containerize) → Test (unit, integration, SAST/security scans, dependency/vuln scan) → Artifact (push image to registry with immutable tag) → Deploy to staging → Automated acceptance/smoke tests → Approval gate → Deploy to prod (blue-green/canary) → Post-deploy verification & monitoring. Each stage fails fast and is auditable.

**Q15. What is Jenkins and what is a Jenkinsfile?**
Jenkins is an open-source automation server that orchestrates CI/CD. A Jenkinsfile is a text file (checked into the repo — "pipeline as code") defining the pipeline using declarative or scripted Groovy syntax. Storing it in the repo gives version control, review, and auditability.

**Q16. Declarative vs Scripted pipeline in Jenkins?**
Declarative is a structured, opinionated syntax (`pipeline { agent … stages { … } }`) — easier to read, has built-in validation and `post` blocks; preferred for most cases. Scripted is full Groovy — more flexible/powerful for complex logic but harder to maintain. You can drop into scripted inside declarative using `script { }`.

**Q17. What is CloudBees and how does it relate to Jenkins?**
CloudBees is the enterprise platform built around Jenkins (CloudBees CI). It adds enterprise features: high-availability controllers, RBAC/SSO, operations center to manage many controllers, security hardening, compliance/audit features, and support. For a regulated financial client this matters — governance, access control, and auditability out of the box.

**Q18. How do you secure secrets in a Jenkins pipeline?**
Never hardcode. Use the Jenkins Credentials plugin (`withCredentials`), or better, integrate an external secrets manager — HashiCorp Vault, AWS Secrets Manager, Azure Key Vault. Inject at runtime, mask in logs, scope credentials to folders/jobs, rotate regularly, and audit access. In finance, secrets management is a compliance requirement.

**Q19. How do you make a Jenkins pipeline scalable/resilient?**
Use a controller-agent architecture — the controller orchestrates, ephemeral agents run builds (Kubernetes pod agents that spin up per build and disappear). This isolates builds, scales horizontally, and avoids "snowflake" agents. Add HA controllers (CloudBees), shared libraries for reuse (DRY), and store config as code.

**Q20. What are Jenkins Shared Libraries?**
Reusable Groovy code stored in a separate repo and loaded into pipelines, so common steps (build, notify, deploy) are written once and shared across teams. Improves consistency, reduces duplication, and centralizes governance — a single place to enforce security scans or approval gates.

**Q21. How do you implement a quality gate in the pipeline?**
Fail the build unless criteria pass: unit test coverage threshold, SonarQube quality gate (bugs, code smells, vulnerabilities), SAST/DAST results, dependency vulnerability scan (Snyk/Trivy/OWASP Dependency-Check), and license checks. The pipeline stops and reports so bad code never reaches prod.

**Q22. Blue-Green vs Canary vs Rolling deployment?**
- **Rolling:** replace instances gradually; no extra full environment; slower rollback.
- **Blue-Green:** two identical environments; switch traffic all at once; instant rollback by switching back; needs double capacity briefly.
- **Canary:** release to a small % of users, watch metrics, then ramp up; catches issues with minimal blast radius. In finance, canary + automated rollback on SLO breach is ideal for reducing change failure impact.

**Q23. How would you achieve zero-downtime deployment?**
Blue-green or canary, load balancer draining connections gracefully, readiness/liveness probes in Kubernetes, backward-compatible database migrations (expand-contract pattern), feature flags to decouple deploy from release, and automated rollback triggered by health/SLO checks.

**Q24. What is GitOps?**
A model where Git is the single source of truth for both app and infrastructure declarative config. A controller (Argo CD / Flux) continuously reconciles the cluster's actual state to match Git. Benefits: full audit trail (every change is a commit/PR), easy rollback (git revert), and drift detection — very attractive for regulated finance environments.

---

## Section 3 — Kubernetes (Q25–42)

**Q25. What is Kubernetes and why use it?**
An open-source container orchestration platform that automates deployment, scaling, healing, and networking of containerized apps. You use it for self-healing (restarts failed containers), horizontal scaling, rolling updates/rollbacks, service discovery, and declarative config — key for running high-availability, fault-tolerant distributed systems.

**Q26. Explain the Kubernetes architecture.**
**Control plane:** API Server (front door, all comms go through it), etcd (distributed key-value store = cluster state), Scheduler (assigns pods to nodes), Controller Manager (reconciliation loops), Cloud Controller Manager. **Worker nodes:** kubelet (agent that runs pods), kube-proxy (networking/rules), container runtime (containerd). Desired state in etcd, controllers reconcile actual → desired.

**Q27. Pod vs Deployment vs ReplicaSet vs StatefulSet?**
- **Pod:** smallest deployable unit, one or more containers sharing network/storage.
- **ReplicaSet:** ensures N identical pod replicas.
- **Deployment:** manages ReplicaSets, gives rolling updates/rollbacks — use for stateless apps.
- **StatefulSet:** for stateful apps needing stable identity/ordering/persistent storage (databases, Kafka). Also DaemonSet (one pod per node, e.g., log/metric agents) and Job/CronJob (batch/scheduled).

**Q28. What are liveness, readiness, and startup probes?**
- **Liveness:** is the container alive? Fail → kubelet restarts it.
- **Readiness:** is it ready for traffic? Fail → removed from Service endpoints (no restart).
- **Startup:** for slow-starting apps; disables the other probes until the app has started, preventing premature restarts. Correct probes are essential for zero-downtime deploys.

**Q29. How does a Service work? Types of Services?**
A Service gives a stable virtual IP/DNS name and load-balances across a dynamic set of pods (selected by labels). Types: **ClusterIP** (internal only, default), **NodePort** (exposes a port on every node), **LoadBalancer** (provisions a cloud LB), and **ExternalName** (DNS CNAME). For HTTP routing across services you typically add an **Ingress** controller.

**Q30. ConfigMap vs Secret?**
Both inject config into pods. ConfigMap = non-sensitive config (env vars, files). Secret = sensitive data (passwords, tokens), base64-encoded (not encrypted by default!). Enable encryption at rest for etcd, use RBAC to restrict access, and integrate an external secrets manager/CSI driver for real security in finance.

**Q31. How does Kubernetes autoscaling work?**
- **HPA (Horizontal Pod Autoscaler):** scales pod *count* based on CPU/memory/custom metrics.
- **VPA (Vertical Pod Autoscaler):** adjusts pod CPU/memory *requests/limits*.
- **Cluster Autoscaler / Karpenter:** adds/removes *nodes* when pods can't be scheduled. HPA is the most common; combine with Cluster Autoscaler for full elasticity.

**Q32. Requests vs Limits?**
**Requests** = guaranteed resources used for scheduling. **Limits** = hard cap. If a container exceeds its memory limit it's OOMKilled; exceeding CPU limit gets throttled. Set both to protect neighbors ("noisy neighbor") and define QoS class (Guaranteed/Burstable/BestEffort). Getting these right prevents cascading failures.

**Q33. How do you troubleshoot a pod stuck in CrashLoopBackOff?**
`kubectl describe pod` (events, exit codes, OOMKilled?), `kubectl logs <pod> --previous` (logs from the crashed container), check probes (misconfigured liveness restarts a healthy app), check resource limits, config/secret issues, missing dependencies, and image problems. Reproduce locally if needed. Common causes: app config error, failed dependency, bad probe, OOM.

**Q34. Pod stuck in Pending — what do you check?**
Usually a scheduling problem: insufficient node resources (CPU/memory requests too high), no node matches nodeSelector/affinity/taints-tolerations, PVC can't bind, or image pull issues. `kubectl describe pod` shows the scheduler's reason. Fix by adding capacity (Cluster Autoscaler), adjusting requests, or fixing scheduling constraints.

**Q35. What is a namespace and why use it?**
A virtual cluster partition for isolating resources — separate teams/environments, apply resource quotas, RBAC boundaries, and network policies per namespace. In a multi-tenant financial platform, namespaces isolate applications and enforce governance.

**Q36. How do you handle persistent storage in Kubernetes?**
PersistentVolume (PV = actual storage), PersistentVolumeClaim (PVC = a request for storage by a pod), and StorageClass (dynamic provisioning). Pods reference PVCs. StatefulSets use volumeClaimTemplates for per-pod stable storage. Use appropriate access modes (RWO/RWX) and backup/snapshot policies.

**Q37. What is a Network Policy?**
A firewall for pods — controls which pods/namespaces can talk to each other (ingress/egress) by labels. Default is all-allow; apply a default-deny then explicitly allow. Critical for financial workloads to enforce least-privilege network segmentation and meet compliance.

**Q38. How does a rolling update / rollback work in Kubernetes?**
Deployment creates a new ReplicaSet, gradually scaling it up while scaling the old one down, controlled by `maxSurge`/`maxUnavailable`. Readiness probes gate traffic. `kubectl rollout status` monitors, `kubectl rollout undo` rolls back to the previous ReplicaSet. Revision history enables rollback to any prior version.

**Q39. What is Helm?**
A package manager for Kubernetes. A Helm "chart" templates K8s manifests with values files, so you parameterize per-environment (dev/stage/prod) and version/release your app. `helm install/upgrade/rollback` manage releases. Reduces YAML duplication and enables consistent, repeatable deploys.

**Q40. How do you secure a Kubernetes cluster?**
RBAC least-privilege, network policies, Pod Security Standards/admission controllers (OPA Gatekeeper/Kyverno), image scanning + signed images, secrets encryption at rest + external secrets manager, disable anonymous API access, audit logging, run containers as non-root/read-only FS, and keep the cluster patched. Defense in depth — essential in finance.

**Q41. What are taints and tolerations, and node affinity?**
**Taints** repel pods from a node unless the pod has a matching **toleration** — used to reserve nodes (e.g., GPU or PCI-compliant nodes). **Node affinity** attracts pods to nodes with certain labels. **Pod (anti-)affinity** co-locates or spreads pods (e.g., spread replicas across zones for HA).

**Q42. How do you achieve high availability in Kubernetes?**
Multiple control-plane nodes + etcd quorum (odd number, 3/5), worker nodes across multiple availability zones, pod anti-affinity + PodDisruptionBudgets so replicas aren't all evicted at once, HPA + Cluster Autoscaler, health probes, and multi-region for DR. PDBs are key during node drains/upgrades.

---

## Section 4 — Observability: Dynatrace, Splunk, Prometheus, Grafana (Q43–56)

**Q43. What is observability and how is it different from monitoring?**
Monitoring tells you *whether* something is wrong (predefined dashboards/alerts on known failure modes). Observability lets you ask *why* — explore a system's internal state from its outputs to debug problems you didn't predict. Observability is built on the three pillars: metrics, logs, and traces.

**Q44. What are the three pillars of observability?**
**Metrics** (numeric time-series — cheap, good for alerting/trends), **Logs** (discrete timestamped events — rich detail for debugging), **Traces** (the path of a request across services with timing — essential in microservices to find the slow/failing hop). Correlating all three is how you go from "something's wrong" to root cause fast.

**Q45. What is Dynatrace and what makes it different?**
Dynatrace is a full-stack, AI-powered observability platform. Its OneAgent auto-discovers and instruments the whole stack (infra, services, apps, real-user monitoring) with minimal manual work. Smartscape maps dependencies automatically, and its Davis AI engine does automatic anomaly detection and root-cause analysis — it correlates events and tells you the probable cause, reducing MTTR. Strong fit for large enterprises.

**Q46. What is a "problem" in Dynatrace and how does Davis help?**
Dynatrace groups related anomalies/events into a single "Problem" rather than flooding you with alerts, then Davis AI analyzes the topology and event timeline to surface the root-cause entity. This cuts alert noise and speeds RCA — directly supports the MTTR and RCA responsibilities in the JD.

**Q47. What is Splunk and what is it used for?**
Splunk is a platform for searching, analyzing, and visualizing machine-generated data — primarily logs. You ingest logs from everywhere, index them, and query with SPL (Search Processing Language) for troubleshooting, dashboards, alerting, and security (SIEM). In finance it's heavily used for audit, compliance, and security analytics.

**Q48. What is SPL? Give a basic example.**
Splunk Search Processing Language. Example: `index=app_logs status=500 | stats count by uri | sort -count` — searches error logs, counts 500s per endpoint, sorts descending. The pipe passes results between commands like Unix pipes. Common commands: `stats`, `timechart`, `eval`, `where`, `rex`, `transaction`.

**Q49. What is Prometheus and how does it work?**
An open-source metrics monitoring/alerting system. It **pulls** (scrapes) metrics over HTTP from instrumented targets/exporters at intervals, stores them in a time-series DB with labels, and you query with PromQL. Service discovery (e.g., Kubernetes) finds targets automatically. Alertmanager handles alert routing/dedup/silencing. Pull model + labels + PromQL is the core.

**Q50. Pull vs Push monitoring — trade-offs?**
Prometheus pulls: easier to detect a down target (scrape fails), central control of scrape config, simpler targets. Push (e.g., StatsD, Pushgateway) suits short-lived/batch jobs that die before a scrape. Prometheus offers a Pushgateway for exactly those ephemeral cases. Most K8s monitoring is pull-based.

**Q51. What is PromQL? Give an example.**
Prometheus Query Language. Example — request error rate over 5 min:
`sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`.
`rate()` computes per-second average over the window; you aggregate with `sum`. Know `rate` vs `irate`, `increase`, `histogram_quantile` for latency percentiles.

**Q52. How do you measure latency percentiles and why not use averages?**
Averages hide outliers — a good average can still mean many users have terrible latency. Use percentiles (p50/p95/p99) which show the tail experience. In Prometheus with a histogram: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`. SLOs are almost always set on p95/p99, not averages.

**Q53. What is Grafana and how does it fit with Prometheus?**
Grafana is a visualization/dashboarding tool that queries many data sources (Prometheus, Splunk, Loki, cloud, SQL). Prometheus stores/queries metrics; Grafana displays them and can alert. Common stack: Prometheus (metrics) + Loki (logs) + Tempo (traces) + Grafana (single pane of glass). You'd build golden-signal dashboards and SLO burn-rate panels here.

**Q54. How would you set up SLO monitoring end to end?**
Define SLIs (e.g., availability = good requests / total). Instrument the app (Prometheus histograms/counters, or Dynatrace service metrics). Compute SLO compliance and error-budget burn over rolling windows. Build a Grafana/Dynatrace dashboard showing budget remaining. Configure multi-burn-rate alerts (page on fast burn, ticket on slow burn). Review in ops meetings with product owners.

**Q55. What is distributed tracing and OpenTelemetry?**
Distributed tracing follows one request across many microservices, assigning a trace ID and spans per service with timing, so you see exactly where latency/errors occur. OpenTelemetry (OTel) is the vendor-neutral CNCF standard (APIs/SDKs/collector) for generating and exporting metrics, logs, and traces — so you're not locked into one vendor. Dynatrace, Grafana Tempo, etc., ingest OTel data.

**Q56. How do you reduce alert fatigue?**
Alert only on user-impacting symptoms, use SLO burn-rate alerts instead of raw thresholds, deduplicate/group related alerts (Alertmanager, Dynatrace Problems), set proper severities (page vs ticket), add reasonable durations to avoid flapping, tune noisy alerts in postmortems, and route to the right team. Every page must be actionable.

---

## Section 5 — Cloud (AWS / Azure / GCP) (Q57–66)

**Q57. Explain IaaS vs PaaS vs SaaS.**
IaaS = raw infra (VMs, storage, network) you manage — EC2, Azure VM, GCE. PaaS = managed platform, you deploy code and the provider runs the runtime — App Service, App Engine, Elastic Beanstalk. SaaS = fully managed software you just consume — Gmail, Salesforce. More managed = less control but less ops burden.

**Q58. What is a VPC and how do you design a secure network?**
A Virtual Private Cloud is your isolated network in the cloud. Design: public subnets (load balancers, NAT), private subnets (app/DB, no direct internet), route tables, security groups (stateful, instance-level) + NACLs (stateless, subnet-level), NAT gateway for outbound-only from private subnets, and VPN/Direct Connect/ExpressRoute for on-prem connectivity. In finance, keep data tiers private and segment heavily.

**Q59. How do you achieve high availability and DR in the cloud?**
HA: deploy across multiple Availability Zones behind a load balancer, auto-scaling groups, managed multi-AZ databases. DR: multi-region with a strategy chosen by RTO/RPO — backup & restore (cheapest, slowest), pilot light, warm standby, or active-active (fastest, priciest). Finance usually needs low RTO/RPO, so warm standby or active-active.

**Q60. What are RTO and RPO?**
**RTO (Recovery Time Objective):** max acceptable downtime to restore service. **RPO (Recovery Point Objective):** max acceptable data loss measured in time (how far back your last usable backup is). Finance often demands near-zero RPO (synchronous replication) and low RTO — this drives DR architecture cost.

**Q61. IAM best practices?**
Least privilege, no root for daily use (MFA on root), use roles/managed identities instead of long-lived keys, temporary credentials (STS), rotate secrets, group-based permissions, permission boundaries/SCPs, and audit with CloudTrail/Azure Monitor/Cloud Audit Logs. Regularly review with access analyzers. Critical for financial compliance (SOX, PCI-DSS).

**Q62. What is Infrastructure as Code? Terraform vs CloudFormation?**
IaC = managing infra through machine-readable declarative config, versioned in Git — repeatable, auditable, no manual clicking. Terraform is cloud-agnostic (HCL, multi-provider, state file). CloudFormation is AWS-native (YAML/JSON, managed state). Terraform's portability wins in multi-cloud; CloudFormation integrates deeply with AWS. Both give you plan/preview before apply.

**Q63. What is Terraform state and why does it matter?**
State is Terraform's record mapping config to real resources. It must be stored remotely (S3 + DynamoDB lock, Terraform Cloud) for team use, with **state locking** to prevent concurrent corruption. Never edit by hand; use `terraform import` to bring in existing resources. Protect it — it can contain secrets.

**Q64. How do you manage costs in the cloud?**
Right-size instances, use auto-scaling to match demand, spot/preemptible for fault-tolerant workloads, reserved/savings plans for steady baseline, delete unused resources, set budgets/alerts, tag resources for cost allocation, and use cost dashboards (Cost Explorer / Azure Cost Management). FinOps culture: make cost visible to teams.

**Q65. Containers vs Serverless — when to use which?**
Containers (K8s/ECS) give control, portability, long-running services, and predictable cost at scale. Serverless (Lambda/Functions/Cloud Functions) is event-driven, auto-scales to zero, no server management, pay-per-use — great for spiky/irregular workloads and glue code, but has cold starts and time/resource limits. Choose by workload pattern and operational appetite.

**Q66. What managed observability/security services do cloud providers offer?**
AWS: CloudWatch (metrics/logs), X-Ray (tracing), CloudTrail (audit), GuardDuty (threat detection), Config (compliance). Azure: Monitor/App Insights, Sentinel (SIEM), Defender. GCP: Cloud Monitoring/Logging, Cloud Trace, Security Command Center. In finance you combine these with Splunk/Dynatrace for a unified, compliant view.

---

## Section 6 — Incident Management, RCA & Troubleshooting (Q67–74)

**Q67. Walk me through your incident management process.**
Detect (alert/monitor) → Triage & declare severity → Assemble responders, assign an Incident Commander → Communicate (status page/stakeholders on a cadence) → Mitigate first (rollback/failover/feature-flag — restore service before finding root cause) → Resolve → Blameless postmortem with action items tracked to closure. Mitigate before you diagnose.

**Q68. How do you approach root cause analysis?**
Gather the timeline and data (logs/metrics/traces/deploys), reproduce if possible, use techniques like the **5 Whys** and fishbone (Ishikawa) diagrams to go past symptoms, correlate with recent changes (deploys are the #1 cause), identify the root cause and contributing factors, then define preventive actions. Distinguish trigger vs true root cause.

**Q69. A service is slow in production. How do you troubleshoot?**
Check golden signals first (latency/traffic/errors/saturation). Is it all requests or a subset? Recent deploy? Use tracing to find the slow hop (DB? downstream API?). Check resource saturation (CPU/mem/connection pools/thread pools), DB slow queries/locks, cache hit rate, network. Correlate with a change. Mitigate (scale/rollback) then RCA. Narrow scope systematically.

**Q70. High CPU on a Linux host — how do you diagnose?**
`top`/`htop` to find the process, `pidstat`/`ps` for threads, `vmstat`/`mpstat` for system-wide, check load average vs core count, `strace`/`perf` for syscall/CPU profiling, look at whether it's user vs system vs iowait. Correlate with app logs and recent deploys. For a JVM app, check GC activity. Then decide: bug, under-provisioning, or runaway process.

**Q71. What are the common Linux commands you use for troubleshooting?**
`top/htop` (CPU/mem), `df -h`/`du` (disk), `free -m` (memory), `netstat/ss` (connections/ports), `iostat/vmstat` (IO/system), `journalctl`/`tail -f` (logs), `ps aux`, `lsof` (open files), `dig/nslookup/curl` (network/DNS), `grep/awk/sed` (parsing). Financial systems run on Linux — be fluent here.

**Q72. Disk is full on a production server — what do you do?**
`df -h` to confirm, `du -sh /*` drilling down to find the offender (often runaway logs). Immediate mitigation: clear/rotate large logs, remove temp files, expand the volume. Then fix root cause: set up log rotation (logrotate), ship logs off-box (to Splunk), add disk alerting before it fills. Don't delete blindly — verify it's safe.

**Q73. What is a cascading failure and how do you prevent it?**
One component's failure overloads others until the whole system collapses (e.g., a slow dependency exhausts thread pools, retries amplify load). Prevent with timeouts, circuit breakers, bulkheads (resource isolation), rate limiting/load shedding, exponential backoff with jitter on retries, and graceful degradation. Design for partial failure.

**Q74. What is chaos engineering?**
Deliberately injecting failures (killing pods, adding latency, taking down a zone) in a controlled way to verify the system's resilience and find weaknesses before they cause real outages — e.g., Netflix's Chaos Monkey. You start with a hypothesis, small blast radius, and run in staging then prod with guardrails. Builds confidence in HA/DR claims.

---

## Section 7 — Security & Vulnerability Management (Q75–79)

**Q75. What is DevSecOps / "shift left" security?**
Integrating security throughout the pipeline rather than as a final gate — "shifting left" means catching issues early/cheaply. In practice: SAST on commit, dependency/SCA scanning, container image scanning, DAST on staging, IaC scanning (tfsec/Checkov), secrets scanning, and policy-as-code. Security becomes everyone's responsibility, automated in CI/CD.

**Q76. How do you perform vulnerability assessment in a pipeline?**
Layered scanning: SAST (SonarQube/Semgrep) for source code, SCA (Snyk/OWASP Dependency-Check) for third-party libs/CVEs, container scanning (Trivy/Clair) for image OS + package vulns, DAST (OWASP ZAP) against the running app, and IaC scanning. Gate the build on severity thresholds, track remediation SLAs, and rescan continuously since new CVEs appear daily.

**Q77. How do you manage and prioritize vulnerabilities (risk management)?**
Score with CVSS, but prioritize by actual risk: exploitability (is there a known exploit — EPSS/KEV?), exposure (internet-facing?), and business/data sensitivity. Track in a register with owners and SLAs (e.g., critical fixed in X days). Patch, mitigate (WAF/compensating control), or accept with sign-off. Report progress to management — a JD responsibility.

**Q78. How do you secure the software supply chain?**
Pin/verify dependencies, use a private artifact registry, scan images, sign artifacts (Sigstore/cosign), generate an SBOM, enforce provenance (SLSA framework), and restrict who can push to registries. Post-SolarWinds/Log4Shell, this is a top priority in finance.

**Q79. How do you handle secrets across the whole stack?**
Central secrets manager (Vault/AWS Secrets Manager/Azure Key Vault), dynamic short-lived credentials where possible, encryption at rest and in transit, no secrets in code/images/logs (secret scanning in CI), automatic rotation, and least-privilege access with full audit logging. Compliance frameworks require demonstrable secret governance.

---

## Section 8 — Python Coding & Scripting (Q80–95)

> EPAM commonly asks you to *write and explain* code, plus Python fundamentals. Practice typing these, and be ready to explain time complexity.

**Q80. List vs Tuple vs Set vs Dict — differences?**
List: ordered, mutable, allows duplicates `[]`. Tuple: ordered, **immutable**, hashable `()`. Set: unordered, mutable, **unique** elements, fast membership `{}`. Dict: key-value pairs, keys unique/hashable, ordered since 3.7 `{k:v}`. Choose set for dedup/membership (O(1) lookup), dict for lookups by key, tuple when data shouldn't change.

**Q81. What are Python decorators? Write one.**
A decorator wraps a function to add behavior without changing its code — used for logging, timing, auth, retries.
```python
import time, functools
def timing(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time()-start:.3f}s")
        return result
    return wrapper

@timing
def work(n):
    return sum(range(n))
```
`functools.wraps` preserves the original function's name/docstring.

**Q82. Write a decorator that retries a function on exception (very common for SRE).**
```python
import time, functools
def retry(times=3, delay=1, backoff=2, exceptions=(Exception,)):
    def deco(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            wait = delay
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == times:
                        raise
                    print(f"Attempt {attempt} failed: {e}; retrying in {wait}s")
                    time.sleep(wait)
                    wait *= backoff
        return wrapper
    return deco
```
Exponential backoff with retries is a core resilience pattern.

**Q83. `*args` and `**kwargs`?**
`*args` collects extra positional arguments into a tuple; `**kwargs` collects extra keyword arguments into a dict. They let functions accept a variable number of arguments and are used to forward arguments (e.g., in decorators: `func(*args, **kwargs)`).

**Q84. List comprehension — give examples.**
```python
squares = [x*x for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
matrix_flat = [v for row in matrix for v in row]     # nested
d = {k: len(k) for k in ["a", "bb", "ccc"]}          # dict comp
```
More concise/faster than manual loops; don't over-nest or readability suffers.

**Q85. Shallow vs deep copy?**
Shallow copy (`copy.copy` / `list[:]`) copies the outer object but shares nested references — mutating a nested object affects both. Deep copy (`copy.deepcopy`) recursively copies everything, fully independent. Matters when you have nested/mutable structures.

**Q86. What are generators and why use them?**
A generator (using `yield`) produces items lazily one at a time instead of building the whole list in memory — memory-efficient for large/streaming data (like reading a huge log file). It's an iterator; state is preserved between `next()` calls.
```python
def read_large_file(path):
    with open(path) as f:
        for line in f:
            yield line.strip()
```

**Q87. Write code to count error occurrences in a log file.**
```python
from collections import Counter
def count_errors(path):
    counts = Counter()
    with open(path) as f:
        for line in f:
            if "ERROR" in line:
                # e.g., group by status code or message keyword
                counts["ERROR"] += 1
    return counts
```
Using a generator-style read keeps memory low for multi-GB logs — mention this.

**Q88. Parse a log line and extract fields (regex).**
```python
import re
LOG = r'(?P<ip>\d+\.\d+\.\d+\.\d+) .* "(?P<method>\w+) (?P<path>\S+).*" (?P<status>\d{3})'
def parse(line):
    m = re.search(LOG, line)
    return m.groupdict() if m else None
# parse('127.0.0.1 - - "GET /api HTTP/1.1" 200') -> {'ip':..,'method':'GET',...}
```
Named groups make extraction readable. Know `search` vs `match` vs `findall`.

**Q89. Reverse a string and check if it's a palindrome.**
```python
def reverse(s): return s[::-1]
def is_palindrome(s):
    s = ''.join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]
```
`s[::-1]` is slice with step -1. O(n) time.

**Q90. Find duplicates in a list / count word frequency.**
```python
from collections import Counter
def duplicates(items):
    c = Counter(items)
    return [k for k, v in c.items() if v > 1]

def word_freq(text):
    return Counter(text.lower().split())
```
`Counter` is the idiomatic, O(n) tool for frequency problems.

**Q91. FizzBuzz (classic warm-up).**
```python
for i in range(1, 101):
    if i % 15 == 0: print("FizzBuzz")
    elif i % 3 == 0: print("Fizz")
    elif i % 5 == 0: print("Buzz")
    else: print(i)
```
Check `% 15` first (or you'll never reach it).

**Q92. Two Sum — return indices of two numbers adding to target.**
```python
def two_sum(nums, target):
    seen = {}                      # value -> index
    for i, n in enumerate(nums):
        if target - n in seen:
            return [seen[target - n], i]
        seen[n] = i
    return []
```
Hash map = O(n) time, O(n) space vs the naive O(n²) double loop. Explain the trade-off.

**Q93. Find the most frequent element / top-K.**
```python
from collections import Counter
def top_k(items, k):
    return [item for item, _ in Counter(items).most_common(k)]
```
`most_common(k)` is O(n log k) with a heap internally. Mention `heapq` if asked to implement it manually.

**Q94. Make an HTTP health check with error handling (SRE-flavored).**
```python
import requests
def health_check(url, timeout=5):
    try:
        r = requests.get(url, timeout=timeout)
        r.raise_for_status()
        return {"url": url, "status": r.status_code, "ok": True,
                "latency_ms": r.elapsed.total_seconds() * 1000}
    except requests.exceptions.Timeout:
        return {"url": url, "ok": False, "error": "timeout"}
    except requests.exceptions.RequestException as e:
        return {"url": url, "ok": False, "error": str(e)}
```
Always set a timeout (a missing timeout can hang forever — a real prod incident cause) and catch specific exceptions.

**Q95. Read a JSON/YAML config and handle missing keys safely.**
```python
import json
def load_config(path):
    with open(path) as f:
        cfg = json.load(f)
    # .get with defaults avoids KeyError crashes
    return {
        "timeout": cfg.get("timeout", 30),
        "retries": cfg.get("retries", 3),
        "endpoint": cfg["endpoint"],   # required -> let it raise if absent
    }
```
Use `dict.get(key, default)` for optional config; fail loud on truly required values. Wrap file/JSON errors with clear messages.

---

## Section 9 — PowerShell & Git (Q96–98)

**Q96. When would you use PowerShell, and what's a basic example?**
PowerShell shines in Windows/Azure automation — managing Windows servers, Active Directory, Azure resources (Az module), and it's object-oriented (cmdlets return objects, not just text). Example — restart a service and log it:
```powershell
$svc = "MyAppService"
Restart-Service -Name $svc -Force
Get-Service -Name $svc | Select-Object Name, Status |
    Out-File -Append "C:\logs\restart.log"
```
Cmdlets follow Verb-Noun (`Get-`, `Set-`, `Restart-`), and the pipeline passes objects.

**Q97. Git: difference between merge and rebase; how do you resolve conflicts?**
`merge` combines branches and creates a merge commit, preserving history as-is (non-destructive). `rebase` replays your commits on top of another branch for a linear, cleaner history but rewrites commit hashes — never rebase shared/public branches. Resolve conflicts by editing the conflicting files, `git add`, then continue (`git rebase --continue` or commit the merge). Use `git rebase -i` to squash/clean commits before a PR.

**Q98. What Git branching strategy would you use for a financial client?**
Trunk-based development with short-lived feature branches + feature flags supports high deploy frequency and low lead time (DORA). GitFlow (develop/release/hotfix branches) suits stricter, scheduled releases with change control — common in regulated finance. The key is: protected main, mandatory PR reviews, automated checks as merge gates, and a clear hotfix path — all auditable.

---

## Section 10 — Behavioral / Financial-Domain (Q99–100)

**Q99. Tell me about a time you handled a production incident. (Use STAR)**
Structure it: **Situation** (payment/trading service was returning elevated 5xx during business hours), **Task** (I was on-call, needed to restore service fast while protecting data integrity), **Action** (checked golden signals in Dynatrace, correlated the spike with a deploy 20 min earlier, declared a Sev-2, communicated to stakeholders, rolled back via the pipeline while preserving in-flight transactions, verified recovery on dashboards), **Result** (restored in ~15 min, wrote a blameless postmortem, added a canary stage + automated rollback on SLO breach so it couldn't recur). Emphasize communication, mitigation-first, and prevention. In finance, stress zero data loss and stakeholder comms.

**Q100. Why do reliability and compliance matter more in the financial domain, and how does that change your approach?**
Financial systems handle money and sensitive PII, are regulated (SOX, PCI-DSS, GDPR, regional rules), and downtime/errors have direct financial, legal, and reputational cost. So: tighter SLOs and stricter SLAs, near-zero RPO/RTO with tested DR, everything auditable (GitOps, change control, immutable logs to Splunk), strong access control and secrets governance, security shifted left, and change management with approval gates even at high deploy velocity. My approach balances DevOps velocity with the guardrails regulators require — using error budgets and automation so we move fast *safely*.

---

## Quick-Reference Cheat Sheet

**Availability numbers (per 30 days):** 99% → ~7.2h | 99.9% → ~43m | 99.99% → ~4.3m | 99.999% → ~26s

**DORA metrics:** Deployment Frequency · Lead Time for Change · Change Failure Rate · MTTR

**Golden Signals:** Latency · Traffic · Errors · Saturation | **RED:** Rate/Errors/Duration | **USE:** Utilization/Saturation/Errors

**Three pillars:** Metrics · Logs · Traces

**MTT-x:** MTTD (detect) · MTTA (acknowledge) · MTTR (restore) · MTBF (between failures)

**Incident flow:** Detect → Triage/Severity → IC + responders → Communicate → **Mitigate first** → Resolve → Blameless postmortem

**Deploy strategies:** Rolling · Blue-Green (instant switch) · Canary (small blast radius)

**Kubernetes debug first commands:** `kubectl describe pod`, `kubectl logs --previous`, `kubectl get events`, `kubectl top`

---

### Final tips for the EPAM interview
- Speak in **trade-offs**, not absolutes — seniors reason about cost/benefit.
- Always add a **financial-domain lens**: availability, latency, audit, security, zero data loss.
- For coding: **think aloud**, state the brute-force approach, then optimize, and mention **time/space complexity**.
- Have **2–3 STAR stories** ready (an incident, an automation you built to kill toil, a reliability improvement with before/after metrics).
- If you don't know something, say how you'd **find out** — that's an SRE trait.
