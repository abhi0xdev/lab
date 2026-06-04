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
