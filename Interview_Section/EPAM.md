# EPAM SRE / DevOps Interview Prep — 250 Q&A
### Financial-domain client · ~4 YOE · Python (must) · PowerShell (nice) · SRE, CI/CD, K8s, Cloud, Observability (Dynatrace/Splunk/Grafana/Prometheus)

> How to use this: These are interview-ready answers — say them in your own words, add a short "in my last project I…" example wherever you can. For finance clients, always tie answers back to **availability, auditability, security, and change control**. Interviewers value structured, honest answers over buzzword dumps.

---

## SECTION 1 — SRE Fundamentals (SLI / SLO / SLA, Error Budget, Toil, Golden Signals)

**Q1. What is SRE and how is it different from DevOps?**
SRE (Site Reliability Engineering) is a discipline that applies software-engineering practices to operations problems, with reliability treated as a measurable feature of the product. DevOps is a broader culture/philosophy about breaking silos between Dev and Ops and shipping faster with feedback loops. I usually say: "DevOps says *what* to achieve (collaboration, automation, fast feedback); SRE is one *prescriptive implementation* of DevOps — it gives you concrete tools like SLOs, error budgets, and toil limits to actually run reliable systems." SRE quantifies reliability so it can be traded off against velocity.

**Q2. Define SLI, SLO, and SLA with an example.**
- **SLI (Service Level Indicator)** — a quantitative measure of service behavior, e.g. the ratio of successful HTTP requests to total requests over a window.
- **SLO (Service Level Objective)** — the target for an SLI, e.g. 99.9% of requests succeed over 28 days. This is an internal goal.
- **SLA (Service Level Agreement)** — a contractual promise to the customer with consequences (credits/penalties) if breached, e.g. 99.5% uptime or we refund. SLAs are usually looser than SLOs so we have buffer. For a banking client, an SLA breach can mean real financial penalties, so the SLO is deliberately stricter.

**Q3. Give an example SLI formula.**
Availability SLI = (good events / valid events) × 100. For an API: `(count of requests with status < 500 and latency < 300ms) / (total valid requests) × 100`. Always define "good" precisely and exclude invalid traffic (e.g. 4xx from bad clients) so the number reflects our service, not the caller.

**Q4. What is an error budget and why is it powerful?**
Error budget = 100% − SLO. If the SLO is 99.9% over 30 days, the budget is 0.1% ≈ ~43 minutes of allowed unreliability. It's powerful because it turns reliability into a shared currency: as long as budget remains, teams can ship features fast; when the budget is exhausted, the policy freezes risky changes and forces focus on reliability. It removes the emotional Dev-vs-Ops argument and replaces it with data.

**Q5. What happens when the error budget is exhausted?**
We trigger the agreed **error budget policy**: typically a feature freeze — only reliability fixes, bug fixes, and hardening are deployed until the budget recovers. We also do a review to find what burned it (bad deploy, dependency failure, capacity). The policy must be agreed with product owners *in advance* so it isn't negotiated during a crisis.

**Q6. What are the four Golden Signals?**
**Latency, Traffic, Errors, Saturation** (from the Google SRE book).
- Latency — how long requests take (track successful vs failed separately).
- Traffic — demand on the system (req/s, transactions/s).
- Errors — rate of failed requests.
- Saturation — how "full" the system is (CPU, memory, I/O, queue depth) — the leading indicator of trouble.
For finance workloads I add business signals too (e.g. failed-payment rate).

**Q7. What is the RED method and USE method?**
- **RED** (for request-driven services): **R**ate, **E**rrors, **D**uration.
- **USE** (for resources): **U**tilization, **S**aturation, **E**rrors.
RED is service-centric, USE is resource-centric; golden signals combine both ideas. I use RED for microservices dashboards and USE for node/infra dashboards.

**Q8. What is toil? Give examples.**
Toil is manual, repetitive, automatable, tactical work that scales linearly with service size and has no lasting value — e.g. manually restarting a hung service, hand-applying a config change to 20 servers, manually clearing a full disk, repetitive access approvals. Google recommends keeping toil under ~50% of an SRE's time. The rest goes to engineering that reduces future toil.

**Q9. How do you manage/reduce toil?**
First **measure** it (track ticket categories and time spent), then **prioritize** the highest-frequency × highest-cost items into a toil backlog, then **automate** them (scripts, runbooks-as-code, self-healing, auto-remediation). I also eliminate root causes (fix the disk-fill instead of automating the cleanup) and push repetitive requests to self-service. Report toil % to management so it stays visible.

**Q10. What SRE metrics do you track? (DORA + reliability)**
The four **DORA metrics**: Deployment Frequency, Lead Time for Change, Change Failure Rate, and MTTR (Mean Time to Restore). Plus reliability metrics: SLO attainment, error-budget burn rate, MTBF, MTTD (detect), MTTA (acknowledge). For finance I also track change-approval cycle time because of governance overhead.

**Q11. Define MTTR, MTTD, MTTA, MTBF.**
- **MTTD** — Mean Time to Detect (incident start → alert).
- **MTTA** — Mean Time to Acknowledge (alert → on-call picks it up).
- **MTTR** — Mean Time to Restore/Repair (detection → service restored).
- **MTBF** — Mean Time Between Failures (reliability of the system between incidents).
Lower MTTD/MTTA/MTTR and higher MTBF = healthier operations.

**Q12. What is an error budget burn rate and why alert on it?**
Burn rate = how fast you're consuming the error budget relative to "normal." A burn rate of 1 means you'll exactly exhaust the budget by the SLO window's end; a burn rate of 10 means you'll exhaust it 10× faster. Multi-window, multi-burn-rate alerts (e.g. fast burn: 14.4× over 1h, slow burn: 3× over 6h) catch both sudden outages and slow degradations while avoiding alert fatigue.

**Q13. How do you choose a good SLO target? Why not 100%?**
Base it on user expectations and business need, not a round number. 100% is the wrong target — it's impossible, infinitely expensive, and leaves zero room to ship changes. Every "nine" you add costs exponentially more. I start from current baseline performance and what users actually notice, set a realistic SLO (e.g. 99.9%), then tighten over time if the business demands it.

**Q14. What is the difference between reliability and availability?**
Availability is the fraction of time the service is usable (uptime). Reliability is broader — the service does the *right thing correctly* (correct results, acceptable latency, data integrity) over time. A bank API can be "up" (availability) but returning wrong balances — that's available but not reliable. Finance cares deeply about correctness, not just uptime.

**Q15. What are leading vs lagging indicators in reliability?**
Lagging indicators tell you something already happened (error rate, MTTR, past outages). Leading indicators warn before failure (saturation trends, growing queue depth, rising latency percentiles, error-budget burn rate). Good monitoring emphasizes leading indicators so you act before customers are impacted.

**Q16. Why measure latency at percentiles (p50/p95/p99) instead of averages?**
Averages hide tail latency — a few very slow requests get masked by many fast ones. In finance, the slow 1% might be high-value transactions or a failing dependency. p95/p99 expose the worst user experience. I set SLOs on percentiles (e.g. p99 < 500ms) and alert when the tail degrades even if the average looks fine.

**Q17. What is a blameless postmortem and why is it important?**
A postmortem documents an incident's timeline, impact, root cause, and action items **without blaming individuals**, focusing on systemic causes ("the deploy pipeline allowed an unreviewed config change") rather than "person X made a mistake." It's important because blame kills honesty — engineers hide details to protect themselves. Blameless culture surfaces the real weaknesses so you actually fix them. Every serious incident should produce one with owned, tracked action items.

**Q18. How do you decide what to monitor for a new service?**
Start from the user journey and define SLIs on it (availability, latency, correctness), then add the golden signals, then dependency health, then resource saturation. I follow "symptom-based alerting" — alert on user-facing symptoms (page failing), and keep cause-based signals as diagnostics on dashboards rather than pages. Avoid alerting on everything; alert on what breaks the SLO.

**Q19. What is the difference between symptom-based and cause-based alerting?**
Symptom-based alerts fire on user-visible impact ("checkout error rate > 2%"). Cause-based alerts fire on internal conditions ("CPU > 90%"). Symptom-based is preferred for paging because it's tied to actual user pain; high CPU may be harmless. Cause-based signals are useful for diagnosis and capacity planning but shouldn't wake someone at 3 AM unless they predict SLO breach.

**Q20. How do you handle alert fatigue?**
Reduce noise by: alerting only on SLO-threatening symptoms, using multi-window burn-rate alerts, deleting or tuning alerts that are consistently ignored, grouping/deduplicating related alerts, adding proper severity levels, and ensuring every page is actionable and has a runbook. I periodically review the "alerts that fired vs. alerts that mattered" ratio and prune aggressively.

**Q21. What is capacity planning and how do you approach it?**
Capacity planning ensures resources meet future demand at the target SLO. I forecast demand from historical growth + known events (e.g. month-end batch, salary-day traffic in banking), load-test to find per-instance limits, add headroom for failures (N+1 or N+2), and validate with saturation metrics. In cloud I combine autoscaling for elasticity with reserved capacity for baseline cost control.

**Q22. What is a runbook and what makes a good one?**
A runbook is a documented procedure for handling a known operational situation (alert, task, incident). A good one is concise, tested, actionable, has clear steps and expected outcomes, links to dashboards, states when to escalate, and is version-controlled. The best runbooks evolve into automation ("runbook automation"), so the human step disappears.

**Q23. What is chaos engineering and would you use it in finance?**
Chaos engineering deliberately injects failures (kill a pod, add latency, drop a dependency) to verify the system's resilience and that alerts/recovery work. Yes, but carefully in finance — start in non-prod, use "game days" with tight blast-radius controls, ensure rollback and no real transaction/data risk, and get change approval. It validates your fault tolerance before a real outage does.

**Q24. What is the difference between proactive and reactive SRE work?**
Reactive = responding to incidents/pages/tickets. Proactive = engineering to prevent them: automation, SLO tuning, capacity planning, hardening, chaos testing, reducing toil. Mature SRE shifts the ratio toward proactive. A team drowning in reactive work needs to invest in automation to break the cycle — that's the core SRE value proposition.

**Q25. How do you enforce SLAs with application teams that aren't cooperating? (from the JD)**
I lead with data, not authority: publish the SLIs/SLOs and current attainment, show the error-budget burn and its business impact, and make the error-budget policy an agreement co-owned with product owners. When budget is exhausted, the policy — not me — enforces the freeze. I partner rather than police: help them add observability, fix the top reliability issues, and celebrate improvements. Escalate with evidence only when needed.

---

## SECTION 2 — DevOps & CI/CD (Jenkins, CloudBees, Pipelines)

**Q26. What is CI/CD and why does it matter for reliability?**
CI (Continuous Integration) merges code frequently with automated build + test on every commit, catching integration issues early. CD (Continuous Delivery/Deployment) automates releasing that code to environments — Delivery = always-releasable with a manual gate; Deployment = fully automatic to prod. It matters for reliability because small, frequent, automated, tested changes are far safer than big manual releases, and rollback is easier. This directly improves DORA metrics.

**Q27. Continuous Delivery vs Continuous Deployment?**
Both automate the pipeline up to production. Continuous **Delivery** stops at a manual approval before prod (common in finance for change governance). Continuous **Deployment** goes all the way to prod automatically with no human gate. Most regulated financial clients use Continuous Delivery with an approval + audit trail.

**Q28. Walk me through a typical Jenkins CI/CD pipeline.**
Trigger (webhook on push/PR) → checkout → build/compile → unit tests → static analysis (SonarQube) + security scan (SAST/dependency scan) → package artifact → publish to artifact repo (Nexus/Artifactory) → deploy to dev/QA → integration/smoke tests → manual approval gate → deploy to staging → prod deploy (blue-green/canary) → post-deploy verification/monitoring. Each stage fails fast and notifies.

**Q29. What is a Jenkinsfile and declarative vs scripted pipeline?**
A Jenkinsfile defines the pipeline as code, stored in the repo (pipeline-as-code). **Declarative** uses a structured `pipeline { stages { … } }` syntax — easier, opinionated, better error handling; preferred. **Scripted** uses Groovy with `node { … }` — more flexible/programmatic but harder to maintain. I default to declarative and drop into `script { }` blocks only when I need Groovy logic.

**Q30. What is CloudBees and how does it relate to Jenkins?**
CloudBees is the enterprise platform built on/around Jenkins (CloudBees CI, formerly CloudBees Jenkins). It adds enterprise features: high-availability controllers, role-based access control (RBAC), centralized management of many Jenkins controllers via Operations Center, security/compliance, pipeline templates, and support. Enterprises (especially finance) use it for governance, scalability, and audited access that open-source Jenkins lacks out of the box.

**Q31. How do you secure a Jenkins/CloudBees pipeline?**
Use RBAC (least privilege per folder/job), store secrets in a credentials store or Vault (never in Jenkinsfiles), restrict who can approve prod deploys, run builds on ephemeral agents, scan dependencies (SCA) and code (SAST) in-pipeline, sign artifacts, enforce PR reviews before merge, audit-log all actions, and keep Jenkins/plugins patched. In finance, segregation of duties (the person who codes ≠ the person who approves prod) is often mandatory.

**Q32. What is a blue-green deployment?**
Two identical prod environments: Blue (live) and Green (idle). Deploy the new version to Green, test it, then switch the load balancer/router from Blue to Green. If something breaks, switch back instantly — near-zero-downtime, fast rollback. Cost is doubled infrastructure during the switch. Great for finance where downtime and rollback speed are critical.

**Q33. What is a canary deployment?**
Release the new version to a small subset of users/traffic (e.g. 5%), monitor SLIs/errors, then progressively increase (25% → 50% → 100%) if healthy, or roll back if metrics degrade. It limits blast radius and validates against real traffic. Often automated with metric-based promotion/rollback (e.g. Argo Rollouts, Flagger).

**Q34. Blue-green vs canary vs rolling — when to use which?**
- **Rolling** — replace instances gradually (default in K8s Deployments); simple, no extra infra, but harder to roll back mid-way.
- **Blue-green** — instant switch and instant rollback; needs double infra; good for all-or-nothing releases and DB-compatible changes.
- **Canary** — gradual, metric-driven, smallest blast radius; best for high-risk changes and validating on real traffic.
Choose based on risk tolerance, cost, and rollback needs.

**Q35. How do you roll back a bad deployment safely?**
Prefer automated rollback triggered by failing health/SLI checks. Mechanisms: switch back in blue-green, `kubectl rollout undo` for K8s, redeploy the previous immutable artifact/image tag, or revert via GitOps by reverting the Git commit. Key rule: keep deployments immutable and versioned so "the previous known-good" is always deployable. Handle DB migrations with backward-compatible (expand/contract) changes so app rollback doesn't break the schema.

**Q36. What is Infrastructure as Code (IaC)?**
Managing infrastructure (servers, networks, load balancers, K8s clusters) through machine-readable definition files in version control, rather than manual clicking. Tools: Terraform, CloudFormation, ARM/Bicep, Pulumi, Ansible. Benefits: reproducibility, versioning, peer review, drift detection, auditability, and disaster recovery. Every environment can be rebuilt identically — critical for finance audit and DR.

**Q37. Terraform vs Ansible — difference?**
Terraform is **declarative provisioning** — it manages the lifecycle of infrastructure (create/update/destroy) and tracks state. Ansible is primarily **configuration management/procedural** — it configures existing servers (install packages, set config) and does orchestration. Common pattern: Terraform provisions the infra, Ansible configures it. Terraform is idempotent via state; Ansible is idempotent via modules.

**Q38. What is Terraform state and why does it matter?**
State is Terraform's record of what real resources map to your config. It's how Terraform knows what to change/destroy. It must be stored remotely (e.g. S3 + DynamoDB lock, Terraform Cloud) with locking to prevent concurrent corruption, and secured because it can contain secrets. Never edit it by hand; use `terraform import`, `state mv`, etc. State drift (manual changes) is detected by `terraform plan`.

**Q39. What is idempotency and why does it matter in automation?**
Idempotency means running an operation multiple times produces the same end state as running it once. It matters because automation/retries must be safe — re-running a deploy or config script shouldn't create duplicates or errors. IaC and config tools are built around it. I design my own scripts to be idempotent (check-before-act) so they're safe to re-run after partial failures.

**Q40. What is GitOps?**
GitOps uses Git as the single source of truth for both application and infrastructure declarative state. A controller (Argo CD, Flux) continuously reconciles the actual cluster state to match Git. You deploy by merging a PR; rollback by reverting a commit. Benefits: full audit trail, easy rollback, drift correction, and least-privilege (the controller pulls, humans don't push to prod). Excellent fit for finance auditability.

**Q41. What are the benefits of trunk-based development vs long-lived branches?**
Trunk-based = short-lived branches merged to main frequently (daily) behind feature flags. It reduces merge hell, enables true CI, and keeps main always releasable. Long-lived feature branches drift, cause painful merges, and delay integration feedback. Feature flags let you merge incomplete work safely and decouple deploy from release.

**Q42. What is a feature flag / feature toggle?**
A runtime switch that turns functionality on/off without redeploying. Uses: dark launches, canary of features to a user segment, kill switches for risky features, A/B testing. In finance, a kill switch to instantly disable a misbehaving feature is a big safety win. Manage flags carefully — stale flags become tech debt.

**Q43. What is an artifact repository and why use one?**
A store for built binaries/images (Nexus, Artifactory, container registries). Benefits: immutable versioned artifacts, "build once, deploy everywhere" (same artifact promoted through environments), dependency caching/proxying, vulnerability scanning, and access control/audit. You never rebuild between environments — the tested artifact is exactly what reaches prod.

**Q44. How do you manage secrets in CI/CD?**
Never hardcode. Use a secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) or the CI credential store; inject at runtime as short-lived env vars; use dynamic/rotating secrets where possible; scope by least privilege; mask them in logs; and scan repos for leaked secrets (git-secrets, truffleHog). In K8s prefer external secrets operators over plain Secrets, and enable encryption at rest.

**Q45. What is "shift left" in DevOps?**
Moving quality and security checks earlier in the lifecycle — running tests, SAST, dependency scanning, IaC scanning, and linting at commit/PR time rather than late in QA or prod. Catching issues early is cheaper and faster. "Shift-left security" (DevSecOps) embeds security into the pipeline instead of a late gate.

**Q46. What is DevSecOps?**
Integrating security into every stage of the DevOps pipeline rather than bolting it on at the end: SAST, DAST, SCA (dependency scanning), container image scanning, IaC scanning, secrets scanning, and policy-as-code — all automated in CI/CD. Security becomes a shared responsibility. Essential in finance due to regulatory and data-sensitivity requirements.

**Q47. How do you handle database schema changes in a CI/CD pipeline?**
Use versioned migration tools (Flyway, Liquibase) run as a pipeline stage. Apply the **expand/contract** pattern: make additive, backward-compatible changes first (add column), deploy code that works with both, then contract (remove old) later — so app and schema can be rolled back independently. Always back up before migration and test on a prod-like dataset. Never do destructive changes in the same release as code that depends on them.

**Q48. What causes a slow CI pipeline and how do you speed it up?**
Common causes: serial stages, no caching, huge test suites, big Docker images, cold agents. Fixes: parallelize independent stages/tests, cache dependencies and Docker layers, use test splitting/sharding, run only affected tests, use lightweight/ephemeral agents with pre-warmed images, fail fast on cheap checks first, and separate fast unit tests (every commit) from slow integration tests (later stages).

**Q49. What is a quality gate?**
An automated pass/fail checkpoint in the pipeline based on defined criteria — e.g. test coverage ≥ 80%, zero critical SonarQube issues, no high/critical CVEs, all tests pass. If the gate fails, the pipeline stops. It enforces standards objectively and prevents low-quality or insecure code from progressing.

**Q50. How do you design a CI/CD pipeline for a regulated financial application?**
Add governance without killing velocity: enforce PR review + segregation of duties, mandatory security scans (SAST/SCA/container/IaC) as quality gates, immutable versioned artifacts, full audit logging of who approved/deployed what and when, a manual change-approval gate for prod (tied to change management/CAB), automated rollback, environment parity, and evidence capture for auditors. GitOps helps because every change is an auditable Git commit.

---

## SECTION 3 — Kubernetes & Containers

**Q51. What is a container and how is it different from a VM?**
A container packages an app with its dependencies and runs as an isolated process sharing the host OS kernel — lightweight, fast to start, small footprint. A VM virtualizes hardware and runs a full guest OS — heavier, slower to boot, stronger isolation. Containers give consistency ("works the same everywhere") and density; VMs give stronger isolation. Kubernetes orchestrates containers.

**Q52. What problem does Kubernetes solve?**
It orchestrates containers at scale: scheduling, self-healing (restart failed containers), horizontal scaling, service discovery, load balancing, rolling updates/rollbacks, config/secret management, and storage orchestration. Instead of manually managing containers across many hosts, you declare desired state and K8s continuously reconciles the actual state to match it.

**Q53. Explain the main Kubernetes objects.**
- **Pod** — smallest deployable unit; one or more tightly-coupled containers sharing network/storage.
- **ReplicaSet** — maintains a desired number of pod replicas.
- **Deployment** — manages ReplicaSets, enables rolling updates/rollbacks.
- **Service** — stable network endpoint + load balancing to pods (ClusterIP/NodePort/LoadBalancer).
- **ConfigMap/Secret** — externalized config and sensitive data.
- **Ingress** — HTTP(S) routing into the cluster.
- **Namespace** — logical isolation/partitioning.
- **StatefulSet** — for stateful apps needing stable identity/storage.
- **DaemonSet** — one pod per node (e.g. log/monitoring agents).

**Q54. What is the Kubernetes control plane vs worker nodes?**
**Control plane**: kube-apiserver (front door / API), etcd (cluster state store), scheduler (assigns pods to nodes), controller-manager (reconciliation loops), cloud-controller-manager. **Worker nodes** run: kubelet (agent that manages pods), kube-proxy (networking/routing), and the container runtime (containerd/CRI-O). The control plane decides; nodes execute.

**Q55. What is etcd and why does it matter?**
etcd is the distributed key-value store holding all cluster state (the "source of truth"). If etcd is lost or corrupted, the cluster loses its state. So it must be backed up regularly, run as an odd-numbered quorum (3/5) for HA, secured with TLS, and its access tightly controlled. Restoring etcd is central to cluster disaster recovery.

**Q56. How does a Deployment do a rolling update?**
It creates a new ReplicaSet with the new version and gradually shifts pods from old to new, controlled by `maxSurge` (how many extra pods above desired) and `maxUnavailable` (how many can be down). It waits for new pods to become Ready (readiness probe) before continuing, giving zero-downtime updates. `kubectl rollout undo` reverts to the previous ReplicaSet.

**Q57. What are liveness, readiness, and startup probes?**
- **Liveness** — is the container alive? If it fails, K8s restarts the container (fixes deadlocks/hangs).
- **Readiness** — is it ready to serve traffic? If it fails, the pod is removed from the Service endpoints but not restarted (prevents sending traffic to a warming-up or overloaded pod).
- **Startup** — for slow-starting apps; delays liveness/readiness until startup completes so slow boots aren't killed.
Misconfigured probes are a very common cause of crash loops and traffic blackholes.

**Q58. What is a CrashLoopBackOff and how do you debug it?**
The container keeps crashing and K8s restarts it with increasing backoff. Debug: `kubectl describe pod` (events, last state, exit code), `kubectl logs <pod> --previous` (crash logs), check the exit code (137 = OOMKilled, 1 = app error), verify config/secrets/env, check liveness probe settings, and resource limits. Common causes: bad config, missing dependency, failing migration, OOM, or an over-aggressive liveness probe.

**Q59. How does Kubernetes networking work at a high level?**
Every pod gets its own IP; pods communicate directly without NAT (flat network via a CNI plugin like Calico/Cilium). **Services** provide a stable virtual IP + DNS name and load-balance across pod IPs. **kube-proxy** (or eBPF in Cilium) implements the routing. **Ingress** handles external HTTP(S) routing. **NetworkPolicies** restrict which pods can talk to which — important for finance segmentation/zero-trust.

**Q60. Requests vs limits for CPU and memory?**
**Requests** = guaranteed resources used for scheduling (the pod won't be placed on a node lacking them). **Limits** = the hard ceiling. Exceeding a CPU limit gets throttled; exceeding a memory limit gets the container OOMKilled. Setting them correctly prevents noisy-neighbor problems and node exhaustion. QoS classes (Guaranteed/Burstable/BestEffort) derive from how you set them, affecting eviction order.

**Q61. What is HPA vs VPA vs Cluster Autoscaler?**
- **HPA (Horizontal Pod Autoscaler)** — adds/removes pod replicas based on metrics (CPU, memory, custom).
- **VPA (Vertical Pod Autoscaler)** — adjusts pods' CPU/memory requests/limits.
- **Cluster Autoscaler** — adds/removes *nodes* when pods can't be scheduled or nodes are underused.
They combine: HPA scales pods, Cluster Autoscaler scales the nodes to fit them. KEDA extends HPA with event-driven scaling (e.g. queue length).

**Q62. How do you manage configuration and secrets in K8s?**
Non-sensitive config in **ConfigMaps**, sensitive data in **Secrets** (base64-encoded, so enable encryption-at-rest and RBAC — they're not encrypted by default). Better: use an external secrets manager (Vault, AWS/Azure) via the External Secrets Operator or CSI Secrets Store driver, so secrets never live in etcd as plain values. Mount as env vars or files; rotate regularly.

**Q63. What is a StatefulSet and when do you use it?**
For stateful workloads needing stable, unique network identities and persistent per-pod storage — e.g. databases, Kafka, Zookeeper. Unlike Deployments, pods get ordered, stable names (`app-0`, `app-1`), stable storage (via PVCs), and ordered rollout/scaling. Use it when identity and storage must persist across restarts.

**Q64. Explain PV, PVC, and StorageClass.**
- **PV (PersistentVolume)** — a piece of actual storage in the cluster.
- **PVC (PersistentVolumeClaim)** — a pod's request for storage (size, access mode).
- **StorageClass** — defines how storage is dynamically provisioned (backend, type, reclaim policy).
A PVC binds to a PV (dynamically provisioned via StorageClass). This decouples pods from the underlying storage technology.

**Q65. What is a Helm chart?**
Helm is the K8s package manager; a chart is a templated, versioned bundle of manifests with configurable `values.yaml`. It lets you parameterize deployments across environments, version releases, and roll back (`helm rollback`). Benefits: reuse, consistency, and templating. Alternatives: Kustomize (overlay-based, template-free), often used with GitOps.

**Q66. Helm vs Kustomize?**
Helm uses Go templating + values for packaging and distribution (good for reusable, parameterized apps and third-party charts). Kustomize uses declarative overlays/patches on plain YAML (no templating) for environment-specific customization. Many teams use both: Helm for packaging, Kustomize for per-environment overlays. GitOps tools support both.

**Q67. How do you secure a Kubernetes cluster? (important for finance)**
RBAC with least privilege; NetworkPolicies for pod-to-pod segmentation; Pod Security Standards/admission controls (no privileged/root containers); scan images and use trusted registries; encrypt etcd at rest + TLS everywhere; secrets in an external manager; disable anonymous API access; audit logging; keep nodes/K8s patched; use namespaces + resource quotas for isolation; and admission policy engines (OPA/Gatekeeper, Kyverno) to enforce rules. Runtime security (Falco) detects anomalies.

**Q68. What is a namespace and how do you isolate workloads?**
A namespace is a virtual cluster partition for logical isolation (dev/qa/prod, or per-team). You isolate with: RBAC per namespace, ResourceQuotas/LimitRanges to cap resource use, NetworkPolicies to restrict cross-namespace traffic, and separate secrets/configs. Note: namespaces are logical, not hard security boundaries — for strong isolation, use separate clusters or node pools.

**Q69. How do you troubleshoot a pod stuck in Pending?**
`kubectl describe pod` and read events. Common causes: insufficient node resources (no node satisfies requests), no node matching nodeSelector/affinity/taints-tolerations, unbound PVC (storage not available), or image pull issues aren't Pending (that's ImagePullBackOff). Fix by adjusting requests, adding nodes (autoscaler), fixing affinity/taints, or provisioning storage.

**Q70. What is a service mesh (Istio/Linkerd) and do you need one?**
A service mesh adds a sidecar proxy to each pod to handle service-to-service traffic: mTLS encryption, fine-grained traffic control (canary/traffic splitting), retries/timeouts/circuit breaking, and rich telemetry — without changing app code. Useful at scale/microservices and for zero-trust mTLS (attractive in finance). But it adds complexity and overhead, so adopt it only when the observability/security/traffic-control needs justify it.

**Q71. What are taints and tolerations, and node affinity?**
**Taints** repel pods from a node unless the pod has a matching **toleration** (e.g. dedicate nodes to certain workloads or keep pods off control-plane nodes). **Node affinity** attracts pods to nodes with certain labels (e.g. GPU nodes, specific zones). Together they control placement — useful for compliance (keep sensitive workloads on dedicated, hardened nodes).

**Q72. How do you achieve high availability in Kubernetes?**
Multiple control-plane nodes with etcd quorum; worker nodes across multiple availability zones; multiple replicas per Deployment with pod anti-affinity + PodDisruptionBudgets so replicas don't all land on one node or all get evicted at once; readiness probes; HPA + cluster autoscaler; and multi-region/DR for the highest tiers. In finance, active-active or active-passive across regions may be required.

**Q73. What is a PodDisruptionBudget (PDB)?**
A PDB limits how many pods of an application can be voluntarily disrupted at once (e.g. during node drains/upgrades), by setting `minAvailable` or `maxUnavailable`. It protects availability during maintenance — e.g. "always keep at least 2 of 3 replicas running." It applies to voluntary disruptions, not crashes.

**Q74. What is the difference between a Deployment and a DaemonSet?**
A **Deployment** runs a desired number of replicas scheduled anywhere. A **DaemonSet** runs exactly one pod on every (or selected) node — used for node-level agents like log collectors (Fluent Bit), monitoring agents (node-exporter, Dynatrace OneAgent), or CNI components. When you add a node, the DaemonSet pod is automatically placed on it.

**Q75. How would you debug high latency in a Kubernetes microservice?**
Layered approach: check the service's own metrics/traces (which span is slow — app, DB, downstream?), check pod resource saturation (CPU throttling from limits, memory pressure), check readiness/scaling (are enough replicas up? HPA thrashing?), check network (DNS latency, service mesh overhead, cross-AZ hops), check dependencies (slow DB/external API), and use distributed tracing (Dynatrace/Jaeger) to pinpoint. CPU throttling due to low limits is a very common hidden culprit.

---

## SECTION 4 — Observability & Monitoring (Dynatrace, Splunk, Grafana, Prometheus)

**Q76. Monitoring vs Observability — what's the difference?**
Monitoring tells you *whether* something is wrong against predefined metrics/thresholds ("is CPU high?"). Observability lets you ask *why* and explore unknown-unknowns by querying rich telemetry — you can investigate problems you didn't anticipate. Monitoring is a subset; observability is built on the **three pillars: metrics, logs, and traces**. Modern systems (microservices) need observability because you can't predefine every failure mode.

**Q77. What are the three pillars of observability?**
- **Metrics** — numeric time-series (rates, latencies, saturation); cheap, great for dashboards/alerts/trends.
- **Logs** — timestamped event records; detailed context for debugging.
- **Traces** — the path of a single request across services with timing per span; essential for microservice latency/root-cause.
Together they answer "what's happening, why, and where." Increasingly unified via OpenTelemetry.

**Q78. What is OpenTelemetry (OTel)?**
An open, vendor-neutral standard and SDK/collector for generating and exporting metrics, logs, and traces. It decouples instrumentation from the backend — instrument once, ship to Dynatrace, Grafana, Splunk, etc. It prevents vendor lock-in and standardizes telemetry. Dynatrace, Grafana, and Splunk all ingest OTel data.

**Q79. What is Dynatrace and what makes it distinctive?**
Dynatrace is a full-stack observability + APM platform. Distinctive features: the **OneAgent** auto-instruments hosts/apps with minimal config; **Smartscape** auto-maps topology/dependencies; **PurePath** captures end-to-end distributed traces; and **Davis AI** does automatic anomaly detection and root-cause analysis, correlating problems and reducing alert noise. Its auto-discovery and AI RCA are big time-savers in complex environments.

**Q80. What is Davis AI in Dynatrace?**
Davis is Dynatrace's causation-based AI engine. Instead of flooding you with alerts, it correlates events across the auto-mapped topology (Smartscape) to identify a single root-cause "Problem" and the affected entities, with severity and impact. It reduces noise and MTTD/MTTR by pointing at the actual cause rather than symptoms. I still validate its conclusion, but it's a strong starting point.

**Q81. What is a PurePath in Dynatrace?**
PurePath is Dynatrace's distributed-tracing technology that captures the full transaction path of a single request across all tiers (web → service → DB), with method-level timing and code-level detail. It's how you pinpoint exactly which service/method/DB call is slow in a request, making it invaluable for latency root-cause in microservices.

**Q82. How do you set up an SLO in Dynatrace?**
Define the SLI via a metric expression (e.g. availability from service success rate, or latency from request duration), set the target and evaluation window, and Dynatrace tracks attainment and error-budget burn on an SLO dashboard. You can alert on burn rate. I tie the SLI to a management zone or service so it reflects the specific application the finance team owns.

**Q83. What is Splunk and what is it best at?**
Splunk is a platform for searching, analyzing, and visualizing machine-generated data — primarily logs and events. It ingests, indexes, and lets you query with **SPL (Search Processing Language)**. Best at: centralized log aggregation, security analytics (SIEM via Splunk Enterprise Security), compliance/audit, and ad-hoc investigation across huge log volumes. In finance it's heavily used for audit trails and security monitoring.

**Q84. What is SPL and give a simple example.**
SPL (Search Processing Language) is Splunk's query language using pipes. Example: find error rate by host in the last hour —
`index=app_logs sourcetype=api status>=500 | stats count by host | sort -count`.
Another: `index=payments "transaction failed" | timechart span=5m count`. You pipe results through commands (`stats`, `timechart`, `eval`, `where`, `transaction`) like a Unix pipeline.

**Q85. What are index, sourcetype, and source in Splunk?**
- **index** — where data is stored (a logical bucket, e.g. `payments`, `security`); controls access and retention.
- **sourcetype** — the data format/type (e.g. `access_combined`, `syslog`); drives field extraction/parsing.
- **source** — the origin file/input (e.g. `/var/log/app.log`).
Good index/sourcetype design controls performance, access (RBAC), and cost.

**Q86. How do you create an alert in Splunk?**
Save a search as a scheduled or real-time alert with a trigger condition (e.g. `count > 10` over 5 minutes), set throttling to avoid duplicates, and define actions (email, webhook, ServiceNow ticket, PagerDuty page, or run a script). Example: alert if failed-login events exceed a threshold — feeding both security and reliability workflows.

**Q87. Prometheus — what is it and its data model?**
Prometheus is an open-source metrics monitoring system that **pulls** (scrapes) time-series metrics from targets over HTTP, stores them locally, and queries with **PromQL**. Data model: metric name + key/value labels + timestamped value (e.g. `http_requests_total{method="GET",status="200"}`). It's the de-facto standard for K8s/cloud-native metrics, with service discovery and Alertmanager for alerting.

**Q88. What are the four Prometheus metric types?**
- **Counter** — monotonically increasing (requests total, errors total); use `rate()`.
- **Gauge** — goes up/down (memory usage, queue length, temperature).
- **Histogram** — samples into buckets (request durations) → enables percentile estimation via `histogram_quantile`.
- **Summary** — client-side calculated quantiles.
Histograms are preferred for latency SLIs because you can aggregate across instances.

**Q89. Give a couple of useful PromQL examples.**
- Request rate: `rate(http_requests_total[5m])`
- Error ratio: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
- p99 latency: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`
- CPU per pod: `sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)`

**Q90. How does Prometheus alerting work (Alertmanager)?**
Prometheus evaluates alerting rules (PromQL expressions with a `for:` duration) and fires alerts to **Alertmanager**, which handles grouping, deduplication, silencing, inhibition (suppress lower alerts when a higher one fires), and routing to receivers (email, Slack, PagerDuty, Opsgenie). This separation keeps alert logic in Prometheus and notification/routing logic in Alertmanager.

**Q91. What is Grafana and how does it relate to Prometheus?**
Grafana is a visualization/dashboarding tool that connects to many data sources (Prometheus, Loki, Splunk, CloudWatch, Elasticsearch, SQL). Prometheus stores metrics; Grafana visualizes them and can also alert. Common stack: Prometheus (metrics) + Loki (logs) + Tempo (traces) + Grafana (single pane of glass). I build SLO dashboards, golden-signal panels, and burn-rate visualizations in Grafana.

**Q92. What is Loki?**
Loki is Grafana's log aggregation system, designed to be cost-efficient by indexing only labels (not full log content) — "like Prometheus but for logs." It pairs with Promtail/agents to ship logs and integrates tightly with Grafana, letting you jump from a metric spike to the related logs and traces (correlated observability).

**Q93. How do you correlate metrics, logs, and traces during an incident?**
Start from the symptom metric (error/latency spike on a dashboard), drill into the trace to find the failing service/span, then pivot to that service's logs at that timestamp for the error detail. Tools like Dynatrace do this automatically (PurePath + logs), and the Grafana LGTM stack (Loki/Grafana/Tempo/Mimir) links them via shared labels/trace IDs. Consistent trace IDs across telemetry are the key.

**Q94. What is cardinality and why is it a problem in metrics?**
Cardinality = number of unique label-combination time series. High-cardinality labels (user ID, request ID, full URLs) explode the number of series, blowing up memory/storage and slowing queries in Prometheus. Rule: labels should be low-cardinality and bounded (status, method, service) — put high-cardinality data in logs/traces, not metric labels.

**Q95. How do you monitor an application end to end?**
Layered: **infrastructure** (node CPU/mem/disk/network via node-exporter/OneAgent), **container/orchestration** (K8s pod health, restarts, resource use), **application** (golden signals, business metrics, APM traces), **logs** (centralized in Splunk/Loki), **synthetic monitoring** (proactive uptime/journey checks from outside), and **real-user monitoring (RUM)** for actual user experience. Define SLIs/SLOs on top and alert on burn rate.

**Q96. What is synthetic vs real-user monitoring (RUM)?**
**Synthetic** proactively runs scripted checks (uptime, login journey) from outside on a schedule — catches problems before users, works even with no traffic, good for SLA verification. **RUM** captures actual user sessions/performance from the browser/app — reflects real experience and geography. Use both: synthetic for early detection/SLA, RUM for true user impact. Dynatrace supports both.

**Q97. How do you reduce logging costs while keeping observability?**
Sample high-volume/low-value logs, set appropriate retention tiers (hot/warm/cold/archive), drop noisy debug logs in prod, structure logs (JSON) so you can query without storing everything at full fidelity, use metrics for high-frequency signals instead of logs, and route only security/audit logs to long-retention indexes. In Splunk, index design and retention policies control cost significantly.

**Q98. What is structured logging and why is it better?**
Structured logging emits logs as machine-parseable key/value (usually JSON) instead of free text — e.g. `{"level":"error","trace_id":"abc","user":"x","msg":"payment failed"}`. Benefits: reliable field extraction, easy filtering/aggregation, correlation via trace IDs, and consistency across services. It makes log queries in Splunk/Loki far more powerful than regex on free text.

**Q99. What is a dashboard vs an alert — when to use each?**
Dashboards are for *investigation and situational awareness* — humans look at them when they want to. Alerts are for *notification of conditions needing action* — they come to you. Rule: not everything on a dashboard should alert; alert only on actionable, SLO-relevant conditions. Dashboards support the alert during triage.

**Q100. How do you design an SLO dashboard for a stakeholder/management audience?**
Keep it business-focused: show SLO attainment (green/amber/red) per service, error-budget remaining and burn trend, the four golden signals summarized, DORA metrics, and current incidents. Avoid raw infra noise. For management/finance clients I add "impact in business terms" (e.g. % of transactions affected) and trend over the reporting period, since the JD calls for periodic reporting to management and the customer.

---

## SECTION 5 — Python Scripting (MUST-HAVE)

**Q101. Why is Python popular for SRE/DevOps automation?**
Readable, cross-platform, huge standard library and ecosystem (requests, boto3, kubernetes, paramiko, psutil), great for glue code, API automation, log parsing, and tooling. It integrates with cloud SDKs and CI/CD, and is fast to write and maintain. For SRE it's ideal for auto-remediation scripts, health checks, report generation, and interacting with monitoring/cloud APIs.

**Q102. List vs tuple vs set vs dict?**
- **List** — ordered, mutable, allows duplicates: `[1,2,2]`.
- **Tuple** — ordered, immutable: `(1,2)`; hashable, usable as dict keys.
- **Set** — unordered, unique elements, fast membership: `{1,2}`.
- **Dict** — key/value map, ordered (3.7+), fast lookup: `{"a":1}`.
Choose set for dedup/membership, dict for lookups, tuple for fixed records, list for ordered mutable sequences.

**Q103. What is a list comprehension? Example.**
A concise way to build a list. `squares = [x*x for x in range(10) if x % 2 == 0]`. It's more readable and often faster than a loop+append. Also dict/set comprehensions: `{k: v for k, v in items}`. Don't overuse for complex logic — readability wins.

**Q104. Mutable vs immutable types — why does it matter?**
Immutable: int, float, str, tuple, frozenset — can't change in place. Mutable: list, dict, set. It matters for: default-argument pitfalls (mutable defaults persist across calls), passing objects to functions (mutable ones can be modified by the callee), and using objects as dict keys (must be hashable/immutable). A classic bug: `def f(x, items=[])` — the shared list accumulates across calls; use `items=None` then `items = items or []`.

**Q105. How do you read a large file line by line efficiently?**
Iterate over the file object (lazy, streams line by line) instead of `read()`/`readlines()` which load everything into memory:
```python
with open("app.log") as f:
    for line in f:
        if "ERROR" in line:
            print(line.strip())
```
The `with` block ensures the file closes automatically even on error.

**Q106. Write a script to parse a log file and count error types.**
```python
from collections import Counter
import re
counts = Counter()
with open("app.log") as f:
    for line in f:
        m = re.search(r"\b(ERROR|WARN|CRITICAL)\b", line)
        if m:
            counts[m.group(1)] += 1
for level, n in counts.most_common():
    print(f"{level}: {n}")
```
`Counter` makes tallying trivial; `most_common()` sorts by frequency.

**Q107. How do you make HTTP API calls in Python?**
Using the `requests` library:
```python
import requests
r = requests.get("https://api.example.com/health", timeout=5)
r.raise_for_status()          # raise on 4xx/5xx
data = r.json()
```
Always set a timeout (never hang forever), handle exceptions, and check status. For monitoring/cloud APIs this is the workhorse.

**Q108. How do you handle exceptions properly in Python?**
Catch specific exceptions, not bare `except:`; use `try/except/else/finally`; clean up in `finally` or use context managers; and log with context.
```python
try:
    resp = requests.get(url, timeout=5)
    resp.raise_for_status()
except requests.Timeout:
    log.error("timeout calling %s", url)
except requests.HTTPError as e:
    log.error("http error: %s", e)
```
Fail loudly, don't swallow errors silently — in automation a silent failure is dangerous.

**Q109. What is a decorator? Give a practical example.**
A function that wraps another to add behavior without changing it. Common SRE use: retry/timing/logging.
```python
import time, functools
def retry(times=3, delay=2):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw):
            for i in range(times):
                try:
                    return fn(*a, **kw)
                except Exception as e:
                    if i == times-1: raise
                    time.sleep(delay)
        return wrapper
    return deco
```
Apply with `@retry(3, 2)` above a flaky API call.

**Q110. What is a generator and why use it?**
A function using `yield` that produces values lazily, one at a time, without building the whole result in memory — ideal for streaming large datasets/logs.
```python
def read_errors(path):
    with open(path) as f:
        for line in f:
            if "ERROR" in line:
                yield line
```
Memory-efficient; you iterate results as they're produced.

**Q111. `*args` and `**kwargs`?**
`*args` captures extra positional arguments as a tuple; `**kwargs` captures extra keyword arguments as a dict. They let functions accept variable arguments and pass them through (common in decorators/wrappers): `def wrapper(*args, **kwargs): return fn(*args, **kwargs)`.

**Q112. How do you run a shell command from Python?**
Use `subprocess.run` (not the deprecated `os.system`):
```python
import subprocess
result = subprocess.run(
    ["kubectl", "get", "pods", "-n", "prod"],
    capture_output=True, text=True, timeout=30
)
if result.returncode != 0:
    raise RuntimeError(result.stderr)
print(result.stdout)
```
Pass args as a list (avoids shell-injection), set a timeout, and check `returncode`.

**Q113. How do you parse JSON and YAML?**
JSON: stdlib `json` — `json.loads(text)` / `json.dumps(obj, indent=2)`. YAML: `yaml.safe_load(text)` from PyYAML (use `safe_load`, never `load`, to avoid arbitrary code execution). Both are everywhere in DevOps for configs, API responses, and K8s/IaC manifests.

**Q114. Write a script to check disk usage and alert.**
```python
import shutil, sys
def check_disk(path="/", threshold=85):
    total, used, free = shutil.disk_usage(path)
    pct = used / total * 100
    if pct > threshold:
        print(f"ALERT: {path} at {pct:.1f}% (>{threshold}%)")
        return 1
    print(f"OK: {path} at {pct:.1f}%")
    return 0
sys.exit(check_disk())
```
Returning a non-zero exit code lets a monitoring/cron wrapper detect the alert.

**Q115. How do you manage Python dependencies and environments?**
Use virtual environments (`venv`) per project to isolate dependencies, pin versions in `requirements.txt` (or use Poetry/pip-tools for lockfiles), and never `pip install` globally on shared servers. In CI, install into a clean venv and cache. For reproducibility and security, pin exact versions and scan dependencies for CVEs.

**Q116. What is the difference between `==` and `is`?**
`==` compares **values** (equality). `is` compares **identity** (same object in memory). Use `==` for value checks and `is` only for singletons like `None` (`if x is None`). A common bug is using `is` to compare strings/numbers, which may work by chance (interning) but is incorrect.

**Q117. How do you write a health-check script for a web service?**
```python
import requests, sys
def health(url, expected=200, timeout=5):
    try:
        r = requests.get(url, timeout=timeout)
        ok = r.status_code == expected
        print(f"{url} -> {r.status_code} {'OK' if ok else 'FAIL'}")
        return 0 if ok else 1
    except requests.RequestException as e:
        print(f"{url} -> ERROR {e}")
        return 1
sys.exit(health("https://api.example.com/health"))
```
Wire it into a cron/K8s CronJob or an alerting pipeline via exit code.

**Q118. What is the GIL and does it matter for SRE scripts?**
The Global Interpreter Lock lets only one thread execute Python bytecode at a time, so CPU-bound multithreading doesn't scale on multiple cores. For SRE work it rarely matters because our scripts are mostly **I/O-bound** (API calls, disk, network) — threads or `asyncio` help there. For CPU-bound parallelism use `multiprocessing`. Most automation is I/O-bound, so it's a non-issue.

**Q119. How would you interact with Kubernetes or AWS from Python?**
Kubernetes: the official `kubernetes` client library (or shell out to `kubectl` via subprocess for simple cases). AWS: `boto3` (the AWS SDK) — e.g. list EC2 instances, read CloudWatch metrics, put objects in S3. Both use credentials from the environment/role. Example: `boto3.client("cloudwatch").get_metric_statistics(...)` to pull metrics for a report.

**Q120. How do you make a script idempotent and safe to re-run?**
Check current state before acting (e.g. "does this file/resource already exist?"), make operations additive/convergent rather than blindly creating, use atomic operations (write to temp file then rename), guard destructive actions with confirmation/dry-run flags, and log every action. This ensures a re-run after partial failure doesn't duplicate or corrupt anything — essential for automation and remediation scripts.

---

## SECTION 6 — PowerShell Scripting (Nice-to-have)

**Q121. What is PowerShell and how is it different from bash?**
PowerShell is an object-oriented shell and scripting language (cross-platform since PowerShell Core/7). Unlike bash, which passes **text** between commands, PowerShell passes **.NET objects** through the pipeline, so you can access properties directly without parsing strings. It's strong on Windows administration, Active Directory, Azure, and Microsoft 365 automation.

**Q122. What is a cmdlet?**
A cmdlet is a lightweight PowerShell command following a `Verb-Noun` naming convention (e.g. `Get-Process`, `Stop-Service`, `Get-ChildItem`). Cmdlets return objects, not text. Consistent verbs (`Get`, `Set`, `New`, `Remove`, `Start`, `Stop`) make commands discoverable — `Get-Command` and `Get-Help` help find and learn them.

**Q123. How does the PowerShell pipeline differ from bash piping?**
It pipes objects, so downstream commands access properties directly. Example: `Get-Process | Where-Object CPU -gt 100 | Sort-Object CPU -Descending | Select-Object Name, CPU`. No `awk`/`grep`/`cut` text parsing needed — you filter and select on real properties, which is more robust.

**Q124. How do you get and filter processes/services in PowerShell?**
```powershell
Get-Service | Where-Object {$_.Status -eq 'Running'}
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
Restart-Service -Name 'MyAppService'
```
`$_` is the current pipeline object. This is the bread-and-butter of Windows ops automation.

**Q125. How do you write a function in PowerShell?**
```powershell
function Test-Endpoint {
    param(
        [Parameter(Mandatory)] [string]$Url,
        [int]$Timeout = 5
    )
    try {
        $r = Invoke-WebRequest -Uri $Url -TimeoutSec $Timeout -UseBasicParsing
        Write-Output "$Url -> $($r.StatusCode)"
    } catch {
        Write-Error "Failed: $($_.Exception.Message)"
    }
}
```
`param()` with types and `Mandatory` gives validation; `$_` in `catch` holds the error.

**Q126. How do you make an HTTP request in PowerShell?**
`Invoke-RestMethod` (auto-parses JSON to objects) or `Invoke-WebRequest` (full response object).
```powershell
$data = Invoke-RestMethod -Uri "https://api.example.com/health" -TimeoutSec 5
$data.status
```
`Invoke-RestMethod` is preferred for REST APIs because it returns usable objects directly.

**Q127. How do you handle errors in PowerShell?**
Use `try/catch/finally` with `-ErrorAction Stop` to make non-terminating errors catchable:
```powershell
try {
    Get-Content "C:\missing.txt" -ErrorAction Stop
} catch {
    Write-Error "Could not read file: $($_.Exception.Message)"
}
```
`$ErrorActionPreference = 'Stop'` sets it globally. Without `Stop`, many cmdlet errors won't trigger `catch`.

**Q128. What is the PowerShell execution policy?**
A safety feature controlling whether scripts can run: `Restricted` (none), `RemoteSigned` (local scripts run; downloaded must be signed), `AllSigned`, `Unrestricted`, `Bypass`. Set via `Set-ExecutionPolicy RemoteSigned`. It's a guardrail, not a security boundary. In enterprises it's often enforced via Group Policy; `RemoteSigned` is a common balanced choice.

**Q129. How do you loop and process items in PowerShell?**
```powershell
foreach ($svc in Get-Service) {
    if ($svc.Status -eq 'Stopped') { Write-Output "$($svc.Name) is stopped" }
}
# pipeline form:
1..5 | ForEach-Object { Write-Output "Item $_" }
```
`foreach` statement vs `ForEach-Object` cmdlet (pipeline) — the cmdlet streams, the statement is faster for in-memory collections.

**Q130. How would you use PowerShell with Azure?**
The `Az` module: `Connect-AzAccount`, then cmdlets like `Get-AzVM`, `Start-AzVM`, `Get-AzResourceGroup`, `New-AzResource`. It's ideal for Azure automation, and pairs with Azure Automation Runbooks for scheduled ops. Example: stop non-prod VMs nightly to save cost — a classic toil-reduction script.

**Q131. How do you parse logs / event logs in PowerShell?**
```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 50 |
    Select-Object TimeCreated, Id, Message
# text logs:
Select-String -Path "C:\logs\app.log" -Pattern "ERROR" | Select-Object -First 20
```
`Get-WinEvent` for Windows event logs, `Select-String` (PowerShell's grep) for text.

**Q132. What is PowerShell Remoting?**
Run commands on remote machines via WS-Management (WinRM): `Invoke-Command -ComputerName srv01 -ScriptBlock { Get-Service }` or interactive `Enter-PSSession`. It enables fleet automation (run one command across many servers). Secure it with HTTPS/Kerberos and least-privilege. On Linux/cross-platform, PowerShell 7 supports SSH-based remoting.

**Q133. How do you schedule a PowerShell script?**
On Windows via Task Scheduler (or `Register-ScheduledJob`), in Azure via Automation Runbooks, or in CI/CD as a pipeline step. For SRE, scheduled scripts handle routine toil (cleanup, health checks, reports). Always add logging and exit codes so failures surface to monitoring.

**Q134. How do you securely handle credentials in PowerShell?**
Never hardcode. Use `Get-Credential` for interactive, `SecureString`, the Windows Credential Manager / `SecretManagement` module, or pull from Azure Key Vault (`Get-AzKeyVaultSecret`). In automation, use managed identities/service principals rather than stored passwords. Avoid plain-text secrets in scripts or logs — a hard rule in finance.

**Q135. Write a PowerShell script to check a service and restart if stopped (self-healing).**
```powershell
$name = 'MyAppService'
$svc = Get-Service -Name $name -ErrorAction SilentlyContinue
if ($svc.Status -ne 'Running') {
    Write-Output "$name down, restarting..."
    Start-Service -Name $name
    Start-Sleep 5
    if ((Get-Service $name).Status -eq 'Running') { Write-Output "Recovered" }
    else { Write-Error "Restart failed - escalating" }
}
```
This is a simple auto-remediation pattern that reduces toil and MTTR.

---

## SECTION 7 — Cloud (AWS / Azure / GCP)

**Q136. What are the main cloud service models?**
- **IaaS** — raw infrastructure (VMs, storage, network): EC2, Azure VMs. You manage OS up.
- **PaaS** — platform to deploy apps without managing servers: App Service, Elastic Beanstalk. Manage code + config.
- **SaaS** — fully managed software: Microsoft 365, Salesforce.
More management on your side with IaaS, least with SaaS. Serverless (FaaS) like Lambda is a further abstraction.

**Q137. What is the shared responsibility model?**
The cloud provider secures the **cloud itself** (physical, hypervisor, managed-service infra); the customer secures what's **in the cloud** (data, IAM, OS patching for IaaS, network config, encryption). The line shifts by service model — with SaaS the provider handles more. In finance, misunderstanding this line is a top cause of breaches (e.g. public S3 buckets).

**Q138. Explain regions and availability zones.**
A **region** is a geographic area (e.g. eu-west-1); an **availability zone (AZ)** is an isolated datacenter (independent power/network) within a region. Deploy across multiple AZs for high availability (survive a datacenter failure) and across regions for disaster recovery / data residency. Finance often mandates a specific region for data residency and multi-AZ + DR for resilience.

**Q139. What is IAM and least privilege in the cloud?**
IAM (Identity and Access Management) controls who can do what on which resources via users, roles, groups, and policies. **Least privilege** means granting only the permissions needed, nothing more. Use roles (not long-lived keys), scope policies tightly, enforce MFA, rotate credentials, and audit access. Critical for finance compliance and reducing breach blast radius.

**Q140. What is autoscaling and how does it work in the cloud?**
Automatically adjusts capacity to demand. AWS Auto Scaling Groups / Azure VM Scale Sets add/remove instances based on metrics (CPU, request count, custom) or schedules. Benefits: handle traffic spikes, save cost off-peak, and maintain SLOs. Combine with load balancers and health checks so unhealthy instances are replaced. For containers, this is HPA + cluster autoscaler.

**Q141. What is a load balancer and what types exist?**
Distributes traffic across healthy backends for availability and scale. Types: **Layer 4 (network)** — routes by IP/port, fast (AWS NLB); **Layer 7 (application)** — routes by HTTP path/host/headers, supports TLS termination, sticky sessions (AWS ALB, Azure App Gateway). It also does health checks, removing unhealthy targets. Essential for zero-downtime deploys and HA.

**Q142. How do you design for high availability and DR in the cloud?**
HA: multi-AZ deployment, redundant instances behind load balancers, autoscaling, health checks, managed multi-AZ databases. DR: cross-region replication/backups with a defined **RTO** (how fast you recover) and **RPO** (how much data loss is acceptable), plus a tested failover plan. Finance typically demands low RTO/RPO and periodic DR drills. Strategies range from backup-restore (cheap, slow) to active-active (expensive, instant).

**Q143. RTO vs RPO?**
- **RTO (Recovery Time Objective)** — maximum acceptable *downtime* before recovery.
- **RPO (Recovery Point Objective)** — maximum acceptable *data loss* (how far back your last good backup is).
Example: RTO 1h, RPO 5min means recover within an hour losing at most 5 minutes of data. They drive backup frequency and DR architecture cost. Finance usually wants both very low.

**Q144. What are the main compute options in AWS/Azure?**
AWS: EC2 (VMs), ECS/EKS (containers), Lambda (serverless), Fargate (serverless containers). Azure: Virtual Machines, AKS (K8s), Azure Functions (serverless), Container Apps/ACI. Choose by control vs. management trade-off: VMs for full control/legacy, containers for portability/microservices, serverless for event-driven/low-ops.

**Q145. What is serverless / FaaS and its trade-offs?**
Serverless (Lambda/Azure Functions) runs code on demand without managing servers; you pay per execution and it auto-scales. Pros: no infra management, cost-efficient for spiky/low workloads, fast to ship. Cons: cold starts, execution time/memory limits, vendor lock-in, harder debugging/observability, and not ideal for long-running or high-steady-throughput workloads.

**Q146. How do you manage cloud costs (FinOps)?**
Right-size instances, use autoscaling and scheduled shutdown of non-prod, buy reserved/savings plans or spot instances for suitable workloads, tag resources for cost allocation, set budgets/alerts, delete orphaned resources (unattached disks, idle load balancers), and review with cost tools (Cost Explorer, Azure Cost Management). SRE contributes by reducing over-provisioning and toil that wastes resources.

**Q147. What is a VPC / virtual network and subnetting?**
A VPC (AWS) / VNet (Azure) is an isolated private network in the cloud. You divide it into **subnets** — public (internet-facing via internet gateway) and private (no direct internet, egress via NAT). Security groups/NSGs and route tables control traffic. Finance architectures put databases/sensitive workloads in private subnets with tight network controls and no public exposure.

**Q148. Security group vs NACL (AWS)?**
- **Security Group** — stateful firewall at the instance/ENI level; return traffic auto-allowed; only allow rules.
- **NACL (Network ACL)** — stateless firewall at the subnet level; allow + deny rules; evaluate return traffic separately.
Use security groups as the primary control; NACLs as a coarse subnet-level guardrail. Defense in depth uses both.

**Q149. How do you handle encryption in the cloud?**
**At rest**: enable disk/volume/storage/DB encryption using KMS/Key Vault-managed keys (or customer-managed keys for stricter control). **In transit**: TLS everywhere. Manage keys in a KMS with rotation and access policies. In finance, customer-managed keys, key rotation, and encryption everywhere are usually mandatory for compliance (PCI-DSS, etc.).

**Q150. What cloud-native monitoring services exist and how do they fit with Dynatrace/Splunk?**
AWS CloudWatch, Azure Monitor, GCP Cloud Monitoring provide native metrics/logs/alarms. They're great for infra-level and cloud-service metrics, but for full-stack APM, distributed tracing, AI RCA, and cross-cloud correlation, teams layer Dynatrace/Splunk on top (often ingesting cloud metrics/logs into them). Strategy: native for cloud primitives, Dynatrace/Splunk for unified app observability and audit.

---

## SECTION 8 — Incident Management, Troubleshooting, RCA & On-call

**Q151. Walk me through the incident management lifecycle.**
Detect (monitoring/alert) → Triage & assess severity/impact → Declare incident + assemble responders → Communicate (stakeholders, status page) → Investigate & mitigate (restore service first, even with a workaround) → Resolve → Postmortem (RCA + action items). The priority order is **stabilize first, root-cause later** — restore user service before fully understanding the cause.

**Q152. What are incident severity levels?**
Typically Sev1 → Sev4. Sev1 = critical, full outage / major customer impact / data or financial loss — all hands, immediate. Sev2 = significant degradation. Sev3 = minor impact/workaround exists. Sev4 = low/cosmetic. In finance, anything affecting transactions, data integrity, or regulatory reporting is usually Sev1. Severity drives response time, escalation, and communication cadence.

**Q153. What roles exist during a major incident?**
**Incident Commander (IC)** — coordinates, makes decisions, owns the incident (doesn't do hands-on fixing). **Operations/Tech lead** — does the technical investigation/remediation. **Communications lead** — updates stakeholders/status page. **Scribe** — records the timeline. Separating command from hands-on work keeps a major incident organized and prevents chaos.

**Q154. Describe your systematic troubleshooting approach.**
1) Define the problem and its scope/impact (what changed, who's affected, since when). 2) Check recent changes (deploys, config, infra) — most incidents follow a change. 3) Look at the golden signals and narrow the layer (app/network/DB/infra). 4) Form a hypothesis, test it, isolate the cause. 5) Mitigate, verify recovery, document. I work top-down from the symptom and bisect to localize fast rather than guessing.

**Q155. What is Root Cause Analysis and the "5 Whys"?**
RCA identifies the underlying cause, not just the symptom, so the problem doesn't recur. **5 Whys**: repeatedly ask "why" to drill from symptom to root. E.g. Site down → app crashed → OOM → memory leak → no memory limit/alerting → no capacity testing in CI. The fix targets the deepest actionable cause. I pair it with a timeline and often "contributing factors" since complex outages rarely have a single cause.

**Q156. Give an example of a tough incident you resolved. (STAR)**
Structure it: **Situation** (e.g. intermittent 5xx on the payments API during peak), **Task** (restore service, find cause), **Action** (checked recent deploy, correlated the error spike in Dynatrace with a DB connection-pool exhaustion, scaled the pool + rolled back the change, added a saturation alert), **Result** (restored in ~20 min, wrote a blameless postmortem, added pool-saturation SLI so it's caught early — no recurrence). Emphasize data-driven diagnosis and the preventive follow-up.

**Q157. How do you handle an incident when you don't know the cause?**
Mitigate first: roll back the last change, fail over, scale up, or enable a kill switch to restore service even without full understanding. Communicate status. Then investigate methodically with telemetry. It's fine to restore via rollback and root-cause afterward — customers care about the service being back, not that you've solved the mystery live.

**Q158. What is on-call and how should a healthy rotation work?**
On-call engineers respond to production alerts outside business hours on a rotation. Healthy practices: sustainable rotation (enough people, reasonable frequency), primary + secondary/escalation, clear runbooks, actionable alerts only (no noise), fair compensation/time-off, handoff notes between shifts, and a feedback loop to reduce recurring pages. If someone is paged constantly, that's a signal to invest in reliability, not to endure it.

**Q159. What makes a good alert (so on-call isn't miserable)?**
It's actionable (a human must do something), urgent (needs attention now, not next morning), tied to user/SLO impact, has a clear runbook, and low false-positive rate. If an alert isn't actionable, it should be a dashboard item or ticket, not a page. I regularly audit which alerts fire vs. which mattered and delete/tune the noise.

**Q160. How do you communicate during an incident to non-technical stakeholders?**
Clear, calm, business-focused, and regular: what's impacted (in business terms), current status, what's being done, and next update time — no jargon, no blame, no speculation. For finance clients I tie impact to transactions/customers affected and regulatory considerations. Consistent cadence (e.g. every 30 min) builds trust even when there's no new fix yet.

**Q161. What is escalation and when do you escalate?**
Escalation brings in more help/authority when the current responders can't resolve within expected time or the impact grows. Escalate when: you're stuck past a time threshold, severity increases, you need a specialist/vendor, or a decision exceeds your authority (e.g. customer comms, data-loss decision). Escalating early isn't failure — delaying it worsens MTTR.

**Q162. How do you prevent incidents from recurring?**
Every serious incident gets a blameless postmortem with concrete, owned, tracked action items — and I make sure they actually get done (not just filed). Preventive measures: add the missing alert/SLI, fix the root cause, add tests/quality gates, harden the failing component, update runbooks, and sometimes automate the remediation. Track recurrence to prove the fix worked.

**Q163. How do you troubleshoot high CPU on a Linux server?**
`top`/`htop` to find the process, `pidstat`/`ps` for per-thread detail, check if it's user vs system vs I/O wait (`vmstat`, `iostat`), look at recent changes/deploys, check logs, and for app processes use profiling or thread dumps. Distinguish real load (scale/optimize) from a runaway process (bug/loop). Correlate with app metrics to see if it's traffic-driven or a regression.

**Q164. How do you troubleshoot a memory leak?**
Confirm via trend (steadily rising memory that never drops, eventual OOMKills), identify the process, then use language-specific tooling (heap dumps/profilers — e.g. jmap for Java, tracemalloc for Python). Short term: restart/scale and set memory limits + alerts to contain it. Long term: fix the leak in code. In K8s, a container repeatedly OOMKilled (exit 137) is the classic signature.

**Q165. How do you troubleshoot network connectivity issues?**
Layered: `ping` (reachability), `traceroute` (path/hops), `telnet`/`nc` or `curl` (port/service reachability), DNS (`dig`/`nslookup`), then firewall/security-group/NACL rules, and TLS/cert issues for HTTPS. In K8s add: service endpoints, NetworkPolicies, DNS (CoreDNS), and kube-proxy. Work up the stack from L3 connectivity to L7 application, isolating where it breaks.

**Q166. A deployment caused an outage — what do you do?**
Immediately roll back to the last known-good version (fastest path to recovery), confirm service restored, communicate. Then investigate why it wasn't caught: missing tests, no canary, bad config. Add the guardrail (canary/automated health-check gate, better tests) so the same class of change fails safely in the pipeline next time. Change is the #1 cause of incidents, so pipeline safety is the real fix.

**Q167. What is a "war room" and when do you use it?**
A dedicated real-time channel/bridge (video/Slack) where responders coordinate a major (Sev1/2) incident. Used to centralize communication, avoid duplicate effort, and let the IC coordinate. Keep it focused — only responders act; observers stay muted. A parallel stakeholder channel handles external comms so the war room isn't interrupted.

**Q168. How do you balance incident response with normal work?**
The on-call person owns incidents/interrupts so the rest of the team protects focus time for engineering. Post-incident, feed learnings into the backlog. If interrupts consistently overwhelm the team, that's the signal to prioritize reliability/automation work (reduce toil, fix noisy alerts) — otherwise you're stuck permanently firefighting.

**Q169. What is the difference between mitigation and resolution?**
**Mitigation** restores service / stops the bleeding, possibly with a temporary workaround (rollback, failover, scale-up, kill switch) — the immediate goal during an incident. **Resolution** fixes the underlying problem permanently. You mitigate first to end customer impact, then resolve properly afterward via the postmortem action items.

**Q170. How do you handle a data-integrity incident in finance (e.g. wrong balances)?**
Treat it as high severity. First contain: stop further corruption (pause the offending process/feature via kill switch), preserve evidence/logs for audit. Assess scope (which records/customers affected). Involve the right stakeholders early (compliance, product, possibly regulators). Restore correct data from a trusted source/backup with verification, reconcile, and document meticulously for audit. Data integrity in finance can carry regulatory obligations, so communication and evidence trail matter as much as the technical fix.

---

## SECTION 9 — Linux, Networking & System Fundamentals

**Q171. What happens when you type a URL and press Enter (end to end)?**
DNS resolves the hostname to an IP → TCP connection (3-way handshake) → TLS handshake for HTTPS → HTTP request sent → server (often via load balancer → app → DB) processes and responds → browser renders. For SRE this maps to layers to check when something's slow/broken: DNS, network, TLS, LB, app, DB. Knowing the chain lets you bisect a latency/failure problem quickly.

**Q172. Explain the TCP 3-way handshake.**
SYN (client → server, "let's connect") → SYN-ACK (server → client, "ok, and same to you") → ACK (client → server, "confirmed"). Then data flows. It establishes a reliable, ordered connection. Relevant when debugging connection failures — e.g. SYNs with no SYN-ACK usually mean a firewall/security-group is dropping the traffic or the service isn't listening.

**Q173. Difference between TCP and UDP?**
TCP is connection-oriented, reliable, ordered, with retransmission and flow control (web, DB, most apps). UDP is connectionless, best-effort, no guarantees, lower overhead (DNS, streaming, VoIP, some metrics like statsd). Choose TCP for correctness, UDP for low-latency where occasional loss is acceptable.

**Q174. What Linux commands do you use to diagnose performance?**
`top`/`htop` (CPU/mem/processes), `vmstat` (system-wide), `iostat` (disk I/O), `free -m` (memory), `df -h` / `du` (disk usage), `netstat`/`ss` (connections/ports), `sar` (historical), `dmesg` (kernel/OOM messages), `journalctl`/`tail -f` (logs), `strace`/`lsof` (deep debugging). I usually start with top → identify the constrained resource → drill in with the specific tool.

**Q175. How do you find what's using a port / a large file / high I/O?**
Port: `ss -tulpn | grep :8080` or `lsof -i :8080`. Large files: `du -ahx / | sort -rh | head` or `find / -size +500M`. High I/O: `iostat -x 1` then `iotop` to find the offending process. Full-disk incidents are common toil — a script combining these makes a good auto-diagnostic.

**Q176. What is a load average and how do you interpret it?**
The average number of processes running or waiting (runnable + uninterruptible I/O) over 1/5/15 minutes, shown in `uptime`/`top`. Interpret relative to CPU cores: load of 4 on a 4-core box ≈ fully utilized; higher means saturation/queuing. Rising 1-min above 15-min = load increasing. High load with low CPU often means I/O wait, not CPU.

**Q177. Explain Linux file permissions.**
Three sets — owner, group, others — each with read(4)/write(2)/execute(1). `chmod 750 file` = owner rwx, group r-x, others none. `chown user:group file` sets ownership. Principle of least privilege applies: don't `chmod 777`. In finance, tight permissions on config/secret files and audit of changes matter.

**Q178. What is a reverse proxy and how does it differ from a forward proxy?**
A **reverse proxy** sits in front of servers, receiving client requests and forwarding to backends (nginx, HAProxy, LBs) — provides TLS termination, load balancing, caching, and hides backend topology. A **forward proxy** sits in front of clients, forwarding their outbound requests (corporate egress proxy, filtering). Reverse = protects/serves servers; forward = controls/serves clients.

**Q179. How does DNS work and why does it cause outages?**
DNS translates names to IPs via a hierarchy (resolver → root → TLD → authoritative), with TTL-based caching. It causes outages because: expired/misconfigured records, TTLs too high (slow failover) or too low (load), resolver failures, or propagation delays. "It's always DNS" is a running joke because DNS issues are common, cascade widely, and are easy to overlook. Always check DNS early in connectivity incidents.

**Q180. What is the OSI model (briefly) and why does it matter for troubleshooting?**
7 layers: Physical, Data Link, Network (IP), Transport (TCP/UDP), Session, Presentation, Application (HTTP). It matters because troubleshooting is layered — isolate whether a problem is L3 (IP reachability/`ping`), L4 (port/TCP/`telnet`), or L7 (application/`curl`, logs). Knowing the layer narrows the tools and the likely owner of the fix.

---

## SECTION 10 — Security & Vulnerability Management

**Q181. How do you approach vulnerability management (per the JD)?**
Continuous cycle: **identify** (scan images, dependencies, IaC, hosts with tools like Trivy, Snyk, Nessus, SonarQube), **assess/prioritize** by severity (CVSS) + exploitability + exposure (internet-facing prod first), **remediate** (patch/upgrade/mitigate), and **verify/report**. Integrate scanning into CI/CD (shift-left) so vulns are caught before prod, and track SLAs for fixing criticals — especially strict in finance.

**Q182. What is CVE and CVSS?**
**CVE** — a unique public identifier for a known vulnerability (e.g. CVE-2021-44228, Log4Shell). **CVSS** — a 0–10 score rating severity (Critical/High/Medium/Low) based on exploitability and impact. You use CVSS plus context (is it exploitable in your environment? internet-facing?) to prioritize. A high CVSS on an isolated internal component may be lower real-world priority than a medium one on an exposed service.

**Q183. What is SAST, DAST, and SCA?**
- **SAST** — Static Application Security Testing; analyzes source code for flaws (early, in CI).
- **DAST** — Dynamic; tests the running app from outside (like an attacker).
- **SCA** — Software Composition Analysis; scans third-party/open-source dependencies for known CVEs.
Together they cover code, runtime, and dependencies. All belong in a DevSecOps pipeline.

**Q184. How do you secure secrets across the whole stack?**
Central secrets manager (Vault/AWS Secrets Manager/Key Vault), never in code/repos/images; inject at runtime; short-lived/rotating credentials; least-privilege access; encryption at rest + in transit; secret scanning in CI; and audit access. In K8s use external secrets operators. Rotation and audit trails are especially important for finance compliance.

**Q185. What is defense in depth?**
Layering multiple independent security controls so no single failure is catastrophic: network segmentation (VPC/subnets/NetworkPolicies), firewalls/security groups, IAM least privilege, encryption, WAF, host hardening, runtime detection, and monitoring/audit. If one layer is breached, others still protect. It's a core principle for regulated financial systems.

**Q186. How do you handle patching/OS hardening without breaking production?**
Automate patch management, test patches in lower environments first, roll out gradually (canary/rolling) with health checks and rollback ready, use immutable infrastructure (rebuild patched images rather than patching live servers), and schedule within change windows. Track patch SLAs for critical CVEs. Immutable/golden-image approaches make patching a redeploy, which is safer and auditable.

**Q187. What compliance/regulatory concerns matter in financial DevOps?**
Segregation of duties, change management/audit trails (who changed/approved/deployed what), data protection/encryption, access reviews, retention of logs for audit, and standards like PCI-DSS (card data), SOX (financial reporting controls), and GDPR/data-residency. Practically this shapes the pipeline: mandatory approvals, immutable audit logs, least-privilege access, and evidence capture for auditors.

**Q188. What is least privilege and how do you enforce it in practice?**
Grant only the minimum access needed for a role/service, nothing more. Enforce via scoped IAM roles/policies, RBAC in K8s/Jenkins, just-in-time/temporary elevated access, regular access reviews, and removing unused permissions. Use roles/service accounts over long-lived credentials. It limits blast radius if an account is compromised — a top control in finance.

**Q189. How do you secure a container image?**
Use minimal/distroless base images, scan for CVEs in CI (Trivy/Snyk), run as non-root, drop unnecessary capabilities, don't bake in secrets, pin/verify base image versions, sign images (cosign) and enforce signed-only deployment, and keep images small/updated. At runtime, use read-only filesystems and admission policies (OPA/Kyverno) to block risky configs.

**Q190. What would you do if a security vulnerability is found in production?**
Assess severity/exposure and exploitability, contain if actively exploited (isolate/WAF rule/kill switch), patch or mitigate on priority per the fix SLA, verify the fix, and document for audit. Communicate to security/compliance stakeholders. For a critical internet-facing CVE (like Log4Shell), it's an incident — treat it with the same urgency as an outage, because in finance it may be reportable.

---

## SECTION 11 — Behavioral, Soft Skills, Finance-Domain & EPAM-Specific

**Q191. Tell me about yourself. (opening question)**
Keep it ~90 seconds, tailored: "I'm an SRE/DevOps engineer with ~4 years of experience running production infrastructure. I've worked across CI/CD (Jenkins), Kubernetes, and cloud, with a focus on observability using Dynatrace/Splunk/Grafana. I define and track SLIs/SLOs and error budgets, automate toil with Python, and I'm part of an on-call rotation where I handle incidents and drive blameless RCAs. I enjoy the SRE mindset of making reliability measurable and using data to balance velocity with stability — which is why this role in a financial-domain team appeals to me." Then invite follow-up.

**Q192. Why do you want to work at EPAM / on a financial client?**
Show genuine, specific reasons: EPAM's engineering-led culture, exposure to enterprise-scale clients, and the chance to work on high-availability, compliance-heavy financial systems where reliability truly matters. Emphasize that finance's strict SLAs, auditability, and low tolerance for downtime make it a strong environment to grow SRE discipline. Avoid generic "big company" answers — tie it to the reliability/scale challenge.

**Q193. Describe a time you disagreed with a developer/team. (conflict)**
STAR + data-driven: e.g. a dev team wanted to ship a risky change with the error budget nearly exhausted. Situation/Task: I needed to hold the line without damaging the relationship. Action: I showed the burn-rate data and the agreed error-budget policy, proposed a canary + extra monitoring as a middle path, and offered to pair on hardening. Result: we deployed safely later, no incident, and the relationship improved because I came with data and a solution, not a "no." Emphasize partnership over policing — exactly what the JD wants.

**Q194. How do you handle working under pressure during an outage?**
Stay calm and systematic: prioritize restoring service over finding blame, communicate clearly at a steady cadence, follow runbooks, escalate early when needed, and rely on data rather than panic. After it's resolved, I decompress and channel it into a blameless postmortem. I mention that structure (IC roles, runbooks, dashboards) is what keeps pressure manageable — preparation beats heroics.

**Q195. How do you keep learning and adapt to new technologies? (the JD lists this)**
Concrete habits: hands-on labs/home-lab, following release notes and cloud/CNCF blogs, certifications (e.g. CKA, cloud associate), reading the Google SRE books, and learning from postmortems. I emphasize I learn fastest by building — when I hit a new tool at work I spin up a small project. Adaptability matters because the toolchain changes constantly; the underlying principles (reliability, automation, observability) stay stable.

**Q196. How do you prioritize when everything seems urgent?**
By impact and risk: customer-facing/SLO-threatening and security-critical items first, then things that reduce recurring toil (multiply future time saved), then improvements. I use error budgets and incident data to justify priorities objectively, and I communicate trade-offs to stakeholders rather than silently dropping things. For a toil backlog I weigh frequency × cost × risk.

**Q197. How do you report progress to management and customers? (JD requirement)**
Regular, audience-appropriate reporting: for management/customers I lead with outcomes and business impact — SLO attainment, incident summary and trends, DORA metrics, error-budget status, toil reduction, and risk/vulnerability status — in plain language with trends over the period. I keep the technical detail available but not front-and-center. Consistency and honesty (including bad news early) build trust with a client.

**Q198. What's special about running systems in the financial domain?**
Very low tolerance for downtime and data errors (money and trust are at stake), strict regulatory/compliance and audit requirements, mandatory change control and segregation of duties, strong security/encryption needs, data residency, and predictable load patterns (month-end, payroll, market hours) that drive capacity planning. It means SRE work leans heavily on auditability, correctness, security, and rigorous change management — not just uptime.

**Q199. Where do you see yourself / how do you drive a technology roadmap for an SRE team? (JD: drive roadmap discussions)**
I frame roadmap around maturing reliability: expanding SLO coverage, improving observability (unified tracing/OTel), automating remaining toil and remediation, hardening CI/CD (canary, automated rollback, security gates), and improving DORA metrics. I gather input from incident trends and team pain points, prioritize by impact, and socialize it with stakeholders. Personally I want to grow toward a senior/lead SRE role, deepening both hands-on skill and the ability to shape reliability strategy.

**Q200. Do you have any questions for us? (always prepare 2-3)**
Ask thoughtful ones: "What does the on-call rotation and incident culture look like — is it blameless?" "What's the current SLO/error-budget maturity for this client?" "What's the biggest reliability challenge the team faces right now?" "How is toil vs. project work balanced?" Asking about reliability culture and challenges signals you think like an SRE and are evaluating fit, not just seeking any job.

---

## Quick Prep Tips for the EPAM Interview

- **Rounds to expect:** typically an initial screen, one or two technical rounds (SRE/DevOps concepts + scenarios + light coding/scripting), sometimes a system-design/troubleshooting scenario, and a managerial/soft-skills + English-communication round (B2 requirement — practice speaking clearly).
- **Have 3-4 STAR stories ready:** a tough incident, a conflict/partnership with a dev team, an automation you built that cut toil, and a time you learned a new tech fast.
- **Be honest about depth:** for "nice-to-have" areas (PowerShell, deep SRE) it's fine to say "I've used the fundamentals and I'm actively deepening it" — EPAM values learning ability and honesty over bluffing.
- **Always connect answers to finance context:** availability SLAs, auditability, security/compliance, change control, data integrity.
- **Be ready to write/read simple code live:** a Python log parser, a health-check script, or reading a PromQL/SPL query.
- **Know your own resume cold:** every tool and project you list is fair game for deep-dive questions.
- **For scenario questions, think out loud** and use a structured layered approach (change first, then golden signals, then bisect by layer) — interviewers assess your reasoning, not just the final answer.

*Good luck! Practice saying these out loud so they sound natural, not memorized.*

---

## SECTION 12 — Terraform (Q201–225)

**Q201. What is Terraform and why use it?**
Terraform is an open-source Infrastructure-as-Code tool by HashiCorp that provisions and manages infrastructure declaratively using HCL (HashiCorp Configuration Language). You describe the desired end state and Terraform figures out the API calls to reach it. Benefits: cloud-agnostic (many providers), version-controlled/reviewable infra, reproducible environments, drift detection, and a clear plan-before-apply workflow. In finance it gives auditability — every infra change is a reviewed, versioned commit.

**Q202. Explain the core Terraform workflow.**
`terraform init` (download providers/modules, configure backend) → `terraform plan` (preview the diff between desired and current state — what will be created/changed/destroyed) → `terraform apply` (execute the plan) → `terraform destroy` (tear down). The plan step is the safety gate — you review exactly what will change before touching real infrastructure, which is essential in production and regulated environments.

**Q203. What is Terraform state and why is it critical?**
State (`terraform.tfstate`) is Terraform's record mapping your configuration to real-world resources. It's how Terraform knows what already exists, what to change, and what to destroy. It's critical because losing or corrupting it means Terraform loses track of your infrastructure. It can also contain sensitive values, so it must be stored securely (remote backend), encrypted, and access-controlled. Never edit it by hand.

**Q204. What is a remote backend and why use one?**
A backend defines where state is stored. A **remote backend** (S3+DynamoDB, Azure Storage, Terraform Cloud, GCS) stores state centrally instead of on someone's laptop. Benefits: team collaboration (shared state), **state locking** (prevents concurrent applies corrupting state), encryption at rest, versioning/backup, and keeping secrets off local machines. For any team — especially finance — remote backend with locking is mandatory.

**Q205. What is state locking and why does it matter?**
State locking prevents two people (or pipelines) from running `apply` simultaneously, which could corrupt state or cause conflicting changes. Backends like S3+DynamoDB or Terraform Cloud acquire a lock during operations and release it after. If a lock gets stuck (e.g. a crashed run), you can `terraform force-unlock` — carefully. It's the safeguard that makes concurrent team use safe.

**Q206. What is a Terraform provider?**
A provider is a plugin that lets Terraform interact with a specific platform's API — AWS, Azure, GCP, Kubernetes, GitHub, Datadog, etc. You declare it in a `provider` block with config (region, credentials). Providers expose resources and data sources. You pin provider versions in `required_providers` for reproducibility, since provider updates can introduce breaking changes.

**Q207. Resource vs data source?**
A **resource** (`resource "aws_instance" "web"`) is infrastructure Terraform creates and manages the lifecycle of. A **data source** (`data "aws_ami" "latest"`) reads existing information Terraform does *not* manage — e.g. look up an existing AMI, VPC, or secret to reference it. Data sources let you consume pre-existing or externally-managed resources without owning them.

**Q208. What is a Terraform module?**
A module is a reusable, parameterized package of Terraform config (a folder of `.tf` files). The root config is itself a module; you call child modules with `module "vpc" { source = "..." variables... }`. Modules promote reuse, consistency, and DRY infra (e.g. a standard "secure-vpc" or "eks-cluster" module reused across teams). They can come from local paths, the Terraform Registry, or Git.

**Q209. How do you pass and manage variables in Terraform?**
Declare with `variable "name" { type, default, description }`, reference as `var.name`. Set values via `terraform.tfvars`, `-var`/`-var-file` flags, environment variables (`TF_VAR_name`), or defaults. Use `validation` blocks for constraints and `sensitive = true` to hide secrets in output. Outputs (`output` blocks) expose values (e.g. an endpoint) to callers or other modules.

**Q210. How do you manage secrets in Terraform?**
Never hardcode secrets in `.tf` or commit `.tfvars` with secrets. Pull them at runtime from a secrets manager (Vault, AWS Secrets Manager, Azure Key Vault) via data sources, mark variables/outputs `sensitive = true`, use environment variables or CI secret stores for credentials, and remember secrets can still land in state — so encrypt and lock down the backend. In finance, customer-managed keys and no plaintext secrets are usually required.

**Q211. What do `count` and `for_each` do, and which is better?**
Both create multiple instances of a resource. `count = 3` creates indexed copies (`[0],[1],[2]`) — simple but fragile: removing a middle item re-indexes and can destroy/recreate resources. `for_each` iterates over a map/set, keying resources by a stable string — safer for lists that change, because adding/removing one item doesn't disturb the others. Prefer `for_each` for anything that will change over time.

**Q212. What is drift and how do you detect/handle it?**
Drift is when real infrastructure differs from what state/config says — usually from manual (out-of-band) changes in the console. `terraform plan` detects it by comparing state to reality and showing the diff. Handle it by either re-applying to restore the declared state, or updating the code to match the intended change. Best practice: prevent drift by disallowing manual changes and doing everything through Terraform (enforced via IAM/policy).

**Q213. `terraform import` — what is it for?**
It brings an existing, manually-created resource under Terraform management by adding it to state (you still write the matching config). Useful when adopting Terraform for infrastructure that already exists. In newer Terraform you can use `import` blocks to do it declaratively and even generate config. It does not create anything — it just tells Terraform "this real resource maps to this config."

**Q214. What are `terraform taint` / `-replace` and when do you use them?**
They force Terraform to destroy and recreate a specific resource on the next apply — useful when a resource is in a bad state but its config hasn't changed (e.g. a corrupted VM). `taint` is deprecated in favor of `terraform apply -replace="aws_instance.web"`, which is more explicit and shows the effect in the plan first.

**Q215. Explain `depends_on` and implicit vs explicit dependencies.**
Terraform builds a dependency graph automatically (**implicit**) when one resource references another's attribute (`subnet_id = aws_subnet.a.id`), and orders creation accordingly. `depends_on` is an **explicit** dependency for cases where there's no attribute reference but an ordering requirement still exists (e.g. an app that needs an IAM policy applied first). Use it sparingly — rely on implicit dependencies where possible.

**Q216. What are lifecycle meta-arguments (`create_before_destroy`, `prevent_destroy`, `ignore_changes`)?**
- `create_before_destroy` — create the replacement before destroying the old one (zero-downtime replacements).
- `prevent_destroy` — guard rail that blocks accidental deletion of critical resources (e.g. a prod database).
- `ignore_changes` — ignore drift on specific attributes managed outside Terraform (e.g. an autoscaling desired count changed by the autoscaler).
These control how Terraform handles resource replacement and change detection.

**Q217. How do you manage multiple environments (dev/qa/prod) in Terraform?**
Common approaches: (1) separate directories per environment with shared modules and per-env `.tfvars`/backends — clearest isolation, preferred for prod safety; (2) Terraform **workspaces** — same config, separate state per workspace (lighter, but easy to apply to the wrong env by mistake). For finance I favor separate state/backends per environment with strict access separation, so a dev change can never touch prod state.

**Q218. What are Terraform workspaces and their limitation?**
Workspaces let one configuration maintain multiple independent state files (`default`, `dev`, `prod`) via `terraform workspace new/select`. Limitation: they share the same code and backend, so it's easy to accidentally run `apply` against the wrong workspace, and they don't isolate credentials/blast radius well. They're fine for small variations but not a strong substitute for fully separated prod environments.

**Q219. How do you test and validate Terraform code?**
`terraform validate` (syntax/config), `terraform fmt` (formatting), `terraform plan` (preview), plus policy-as-code (Sentinel, OPA/Conftest) to enforce rules ("no public S3 buckets", "encryption required"), security scanning (tfsec, Checkov, Terrascan), and integration testing with Terratest or the native `terraform test` framework. In CI, run fmt/validate/scan/plan on every PR and gate merges on them.

**Q220. How do you integrate Terraform into a CI/CD pipeline?**
Typical flow: on PR → `init` → `fmt -check` → `validate` → security scan (tfsec/Checkov) → `plan` (post the plan output to the PR for review). On merge to main → `apply` (often with a manual approval gate for prod). Use remote state with locking, store credentials in the CI secret store (or OIDC/short-lived roles), and require the plan to be reviewed before apply. This gives an auditable, gated infra change process.

**Q221. What is the difference between Terraform and CloudFormation/ARM?**
Terraform is cloud-agnostic (one tool/language across AWS, Azure, GCP, and many SaaS providers) with external state and a large module ecosystem. CloudFormation (AWS) and ARM/Bicep (Azure) are cloud-native, provider-managed (no state file to manage yourself), and deeply integrated with their platform. Terraform wins on multi-cloud and flexibility; native tools win on tight single-cloud integration and managed state.

**Q222. What are provisioners and why avoid them?**
Provisioners (`local-exec`, `remote-exec`) run scripts as part of resource creation. HashiCorp recommends them as a **last resort** because they're not declarative, run only at create/destroy time, aren't tracked in state, and make runs non-idempotent and fragile. Prefer purpose-built tooling: cloud-init/user_data for bootstrapping, or a config-management step (Ansible) after provisioning.

**Q223. How do you refactor resources without destroying them (moved blocks)?**
Use `moved` blocks (or historically `terraform state mv`) to tell Terraform that a resource's address changed (e.g. you moved it into a module or renamed it) so it updates state instead of destroying and recreating. `moved` blocks are declarative, live in code, and are safer/reviewable in a PR. This is key when restructuring code around live production resources.

**Q224. What happens if state and real infrastructure get out of sync / state is lost?**
If state is lost, Terraform no longer knows about existing resources and may try to recreate them. Recovery: restore from backend versioning/backup, or rebuild state via `terraform import` for each resource. If state is merely out of sync (drift), `terraform plan` shows the difference and you reconcile via apply or code update. This is exactly why remote backends with versioning and locking are non-negotiable.

**Q225. How do you handle a large Terraform codebase / blast radius?**
Split infrastructure into smaller, independently-applied stacks (network, data, compute, app) each with its own state, so a change to one doesn't risk everything. Share via modules and reference across stacks using remote state data sources or explicit inputs. Benefits: smaller plans, faster applies, reduced blast radius, and clearer ownership. Tools like Terragrunt help keep DRY config across many stacks/environments.

---

## SECTION 13 — Git & GitHub (incl. GitHub Actions) (Q226–250)

**Q226. What is Git and how is it different from GitHub?**
Git is a distributed version-control system that tracks changes to code locally, with full history on every clone. GitHub is a cloud platform built around Git that adds collaboration (pull requests, code review, issues), hosting, access control, and automation (GitHub Actions). Git is the tool; GitHub is the hosted service and collaboration layer on top of it. (Alternatives: GitLab, Bitbucket.)

**Q227. Explain the basic Git workflow.**
Edit files (working directory) → `git add` moves changes to the **staging area** → `git commit` records a snapshot in the **local repository** → `git push` uploads commits to the **remote**. `git pull` (= fetch + merge) brings remote changes down. The three areas (working, staging, committed) let you craft precise commits. You branch off main, commit work, push, and open a PR to merge back.

**Q228. What is the difference between `git merge` and `git rebase`?**
`merge` combines branches by creating a merge commit, preserving the true history (branchy graph). `rebase` replays your commits on top of another branch, producing a linear history but *rewriting* commit hashes. Merge is safe and non-destructive; rebase is cleaner but should never be done on shared/public branches (it rewrites history others may have). Common rule: rebase local feature branches, merge into main.

**Q229. What is a pull request (PR) and why is it central to GitHub workflow?**
A PR proposes merging one branch into another, providing a place for code review, discussion, automated checks (CI, security scans), and approval before merge. It's central because it enforces quality and collaboration: nothing reaches main without review and passing checks. In finance, PRs give the audit trail and segregation of duties (author ≠ approver) that compliance requires.

**Q230. What are the common Git branching strategies?**
- **GitHub Flow** — simple: branch off main, PR, merge, deploy; main always deployable. Great with CI/CD.
- **Git Flow** — main + develop + feature/release/hotfix branches; structured but heavier, suits scheduled releases.
- **Trunk-based development** — very short-lived branches merged to main daily behind feature flags; best for high-frequency CI/CD.
Modern DevOps favors GitHub Flow / trunk-based for velocity; regulated release cadences sometimes still use Git Flow.

**Q231. How do you resolve a merge conflict?**
A conflict happens when two branches change the same lines. Git marks the conflict (`<<<<<<<`, `=======`, `>>>>>>>`); you open the file, choose/combine the correct content, remove the markers, then `git add` the resolved files and complete the merge/commit. Tools/IDEs help visualize it. Prevent conflicts by merging main frequently and keeping branches short-lived and small.

**Q232. What is `git rebase -i` (interactive rebase) used for?**
Interactive rebase lets you rewrite a series of local commits — reorder, `squash` (combine), `reword` messages, `edit`, or `drop` commits — to clean up history before pushing/opening a PR. E.g. squash five "wip" commits into one meaningful commit. Only do it on unpushed/private branches, since it rewrites history.

**Q233. Difference between `git reset`, `git revert`, and `git checkout/restore`?**
- `git revert` — creates a *new* commit that undoes a previous one; safe for shared history (doesn't rewrite).
- `git reset` — moves the branch pointer back (`--soft` keeps changes staged, `--mixed` unstages, `--hard` discards); rewrites history, use on local only.
- `git restore`/`checkout` — discards working-directory changes or switches branches.
Rule: use `revert` to undo on shared branches, `reset` for local cleanup.

**Q234. What is `git cherry-pick`?**
It applies a specific commit from one branch onto another without merging the whole branch. Common use: apply a hotfix commit from a release branch onto main (or vice versa), or pull one needed change from a feature branch. It copies the change as a new commit on the target branch.

**Q235. What is `git stash`?**
`git stash` temporarily shelves uncommitted changes so you can switch context (e.g. jump to a hotfix) with a clean working directory, then `git stash pop` to restore them. Useful when you're mid-work and need to quickly do something else without committing half-finished code.

**Q236. What is `.gitignore` and why does it matter for security?**
`.gitignore` specifies files/patterns Git should not track (build artifacts, `node_modules`, `.tfstate`, `.env`, secrets, credentials). It matters for security because it prevents accidentally committing sensitive files. But note: it only ignores *untracked* files — already-committed secrets stay in history and must be purged (e.g. `git filter-repo`) and rotated. Combine with secret-scanning to catch leaks.

**Q237. What are GitHub branch protection rules?**
Rules on important branches (main) that enforce quality/governance: require PR reviews (e.g. 2 approvals), require status checks (CI/tests/scans) to pass, require up-to-date branches, restrict who can push, require signed commits, and disallow force-push/deletion. In finance they enforce segregation of duties and an auditable change process — no one merges unreviewed code to main.

**Q238. What are CODEOWNERS files?**
A `CODEOWNERS` file maps paths/files to responsible reviewers/teams. When a PR touches those paths, the owners are automatically requested (and can be *required*) as reviewers. It ensures the right experts review changes to critical areas (e.g. security config, payment modules) and supports compliance by enforcing mandatory review by designated owners.

**Q239. What is GitHub Actions?**
GitHub's built-in CI/CD and automation platform. You define **workflows** in YAML (`.github/workflows/`) triggered by events (push, PR, schedule, manual). Workflows contain **jobs** (run on runners) made of **steps** that run commands or reusable **actions**. It integrates natively with the repo — no separate CI server — and is a common alternative/complement to Jenkins for build/test/deploy and automation.

**Q240. Explain the structure of a GitHub Actions workflow.**
```yaml
name: CI
on: [push, pull_request]      # events that trigger it
jobs:
  build:
    runs-on: ubuntu-latest    # runner
    steps:
      - uses: actions/checkout@v4      # a reusable action
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements.txt
      - run: pytest                    # a shell command step
```
Key parts: `on` (triggers), `jobs` (run in parallel by default), `runs-on` (runner), `steps` (`uses` an action or `run` a command).

**Q241. Jobs vs steps, and how do you control job order?**
A **step** is a single task within a job (run a command or use an action); steps in a job run sequentially on the same runner sharing the workspace. **Jobs** run in parallel by default on separate runners. Use `needs:` to make one job depend on another (`deploy` needs `test`), creating a pipeline order. Data passes between jobs via outputs or artifacts, since they don't share a filesystem.

**Q242. How do you manage secrets in GitHub Actions?**
Store them in GitHub encrypted **Secrets** (repo, environment, or org level), reference as `${{ secrets.NAME }}` — never hardcode. Use **environment protection rules** with required reviewers for prod secrets, scope secrets to environments, and prefer **OIDC** to get short-lived cloud credentials instead of storing long-lived cloud keys. Secrets are masked in logs. This is critical for finance to avoid leaked credentials.

**Q243. What is OIDC in GitHub Actions and why prefer it?**
OpenID Connect lets a workflow request a short-lived token from your cloud provider (AWS/Azure/GCP) by establishing trust, so you don't store long-lived cloud access keys as secrets. The credentials are ephemeral and scoped, dramatically reducing the risk and blast radius of leaked keys. It's the recommended, more secure way to authenticate CI/CD to the cloud — and auditors love it.

**Q244. What are GitHub Actions runners (GitHub-hosted vs self-hosted)?**
**GitHub-hosted** runners are managed, ephemeral VMs GitHub provides (fresh environment each run, no maintenance). **Self-hosted** runners are machines you manage — used when you need specific hardware, private-network access, custom software, or compliance/data-residency control. Finance clients often use self-hosted runners inside their network so builds never leave their controlled environment; secure and isolate them (ephemeral, least privilege).

**Q245. How do you cache dependencies and use artifacts in Actions?**
Use `actions/cache` to persist dependencies (pip, npm, Docker layers) across runs keyed on a hash of the lockfile — speeds up builds. Use `actions/upload-artifact` / `download-artifact` to pass build outputs (binaries, test reports) between jobs or store them for later. Caching improves lead time; artifacts enable multi-job pipelines and "build once, deploy many."

**Q246. What is a matrix build in GitHub Actions?**
A matrix runs the same job across multiple parameter combinations in parallel — e.g. test against Python 3.10/3.11/3.12 on Linux/Windows. Defined via `strategy: matrix:`. It gives broad compatibility coverage efficiently. `fail-fast` controls whether one failing combination cancels the rest.

**Q247. How do you deploy safely to production with GitHub Actions?**
Use **environments** with protection rules: required reviewers (manual approval gate for prod), wait timers, and environment-scoped secrets. Combine with branch protection, run tests/security scans as required checks, deploy via canary/blue-green, and use OIDC for short-lived creds. The approval gate + environment scoping + audit log gives the controlled, auditable prod deploy finance needs.

**Q248. What are reusable workflows and composite actions?**
- **Reusable workflows** — a whole workflow called by others (`uses: org/repo/.github/workflows/x.yml@ref`), so many repos share one standardized pipeline (DRY, central governance).
- **Composite actions** — bundle multiple steps into one custom action for reuse.
Both reduce duplication and let a platform team enforce consistent, secure pipelines across many repositories.

**Q249. Jenkins vs GitHub Actions — when would you choose each?**
GitHub Actions: native to GitHub, no server to maintain, great for repo-scoped CI/CD, fast to set up, large marketplace. Jenkins/CloudBees: self-hosted control, mature plugin ecosystem, complex enterprise pipelines, existing on-prem/legacy integrations, and strong enterprise governance (CloudBees). Many enterprises use both — Actions for lightweight repo automation, Jenkins/CloudBees for centralized, heavily-governed enterprise pipelines. Choose by control needs, existing investment, and governance requirements.

**Q250. How does GitHub support security and compliance (relevant to finance)?**
Features: branch protection + required reviews (segregation of duties, audit trail), CODEOWNERS (mandatory expert review), **Dependabot** (dependency vuln alerts + auto-update PRs), **code scanning** (CodeQL SAST), **secret scanning** with push protection (blocks committing secrets), signed commits, environment protection rules, fine-grained permissions, and org **audit logs**. Together they build a secure, auditable software supply chain — exactly what a regulated financial client expects.
