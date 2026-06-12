# SRE / DevOps Interview Q&A Bank
### Read-aloud study guide — deduplicated & category-ordered

**How to use this:** Cover the answer, say it out loud in your own words, then check. ⭐ = high-frequency / likely-asked. Don't memorize word-for-word — own the *shape* and the *why*. The honest-framing notes (marked ⚠️) protect you when an interviewer drills past your real depth.

---

## TABLE OF CONTENTS

1. About You & Your Resume
2. Linux & Troubleshooting
3. Networking (general + TLS + HTTP)
4. Kubernetes — Core Objects & YAML
5. Kubernetes — Internals & Control Plane
6. Kubernetes — Networking (CNI / kube-proxy / CoreDNS)
7. Kubernetes — Scenarios
8. Docker & Containers
9. CI/CD, GitHub Actions, Helm, GitOps, ArgoCD
10. Terraform / IaC
11. Cloud — Azure
12. Observability (Metrics, Logs, Traces, Prometheus, PromQL)
13. Reliability — SLI/SLO/Error Budgets, Resilience, Incidents
14. System Design / Scenario Frameworks
15. Behavioral (STAR)
16. Bash Scripting
17. Python (for Ops)
18. One-Page Cheat Sheet

---

# 1. ABOUT YOU & YOUR RESUME

### ⭐ "Tell me about yourself" (short intro)
I'm Abhinandan, a DevOps and SRE engineer with about 4 years on a healthcare platform running on Azure Kubernetes. Most of my work is reliability — I look after our Prometheus and Grafana stack for an AKS environment with around 1,500 pods, I'm on the P1/P2 on-call rotation, and I do a lot of root-cause work with Dynatrace and Azure Monitor. I also support our CI/CD pipelines and automate operational tasks with Bash and Python. Outside work I built my own observability platform from scratch — Prometheus, Loki, Tempo, OpenTelemetry — to get hands-on building one end to end. It's on my GitHub. I'm AZ-400 certified and looking for an SRE or DevOps role where I can go deeper on reliability at scale.

### ⭐ "Walk me through your day-to-day."
Three buckets. ~40% observability — reviewing Grafana dashboards and overnight alerts, investigating whether alerts were real or tuning problems, maintaining 5-6 dashboards (latency, error rates, pod health, saturation), tuning Prometheus alert rules. ~30% incident response — initial investigation with App Insights logs, Kubernetes events, Dynatrace traces; if it's an ops-level issue (crash loop, image-pull, node resource) I resolve it, if it's an app bug I partner with L3 and hand them a full diagnostic trail. ~30% automation/infra — Bash and Python for operational tasks, supporting CI/CD when builds/releases fail, contributing to Terraform. And I'm on alternate-weekend on-call as first responder.

### ⭐ "Describe the platform architecture." (~1,500 pods, 86 deployments)
Healthcare platform on AKS across multiple environments, with separate platform, infra, dev, and QA teams. Prometheus collects Kubernetes-level metrics (pod health, CPU/memory) and application metrics. Grafana is the visualization layer where I own dashboards. Dynatrace for APM and distributed tracing. Azure Application Insights and Log Analytics for app logs (tight Azure integration). Alertmanager with severity-based routing. CI/CD on GitHub Actions and Azure DevOps; infra moving from CloudFormation to Terraform. My deepest ownership is the observability layer and operational reliability.

### "What does 'owning the Prometheus/Grafana stack' actually involve?"
I maintain 5-6 production dashboards (building/updating panels), and I author and tune the Prometheus alert rules — adjusting thresholds, adding `for` durations, adding inhibition rules. When a new service is onboarded I build its dashboard and alerts. I don't manage the Prometheus cluster infrastructure itself — that's the platform team — but the dashboards, queries, and alerting logic are mine.

### ⚠️ "You reduced MTTR by ~40%. How was it measured and what did you do?"
That was a team-level improvement tracked through our ticketing system and Dynatrace — average P2 resolution time came down by roughly that. My personal contribution was on detection and triage: I tuned alerts to reduce noise so real signals weren't buried, built dashboards that gave responders context faster, and handed engineering a complete diagnostic trail instead of just "service is down." I wouldn't claim I single-handedly drove 40% — it was a team effort and my piece was making detection and triage faster.

### "Tell me about a P1/P2 incident you handled end to end." (OOMKilled)
A backend service was getting OOMKilled every few hours — pods ran fine then got killed and restarted, error rate spiked each time. I checked Grafana and saw memory climbing on a steady slope regardless of traffic — a leak, not load. Confirmed OOMKilled from the pod's last state (exit 137). Correlated with App Insights logs and found one endpoint hit repeatedly during the growth window. Handed engineering the memory trend, trace data, and suspect endpoint — they found an unbounded in-memory cache and fixed it in a day. The lesson: handing off a full diagnostic trail, not just "it's OOMing," cut their investigation time roughly in half.

### "How do you tune a noisy alert? Real example." (alert cleanup)
On-call was getting paged 8-10 times a weekend, mostly non-actionable. I exported ~90 days of alert history and categorized each by whether it led to a human action. About 12 rules caused 70% of the noise. For each I looked at the Prometheus rule — some had thresholds on raw counts instead of rates, some missed a `for` clause so they fired on instant spikes, some were on infra metrics that didn't impact users. I fixed thresholds, added `for` durations, added inhibition rules, removed non-actionable ones. Weekend pages dropped to 2-3, almost all actionable. My principle: every alert should require a human action — if not, it's a dashboard metric.

### "Walk me through diagnosing a performance issue in Dynatrace."
I start with the service's response-time trend and break it down — all requests or specific endpoints, and is the time in the service itself or a downstream call. The distributed traces are key — I open a slow trace and see exactly which span is slow, which tells me whether it's our code or a dependency. Then I correlate with metrics (saturation, slow query, degraded dependency) and package it for engineering if it's a code-level fix.

### ⚠️ "What was your role in the CloudFormation→Terraform migration?"
I joined while it was in progress, so my contribution was on the variable and environment-config side — modifying module variables and env-specific configs, reviewing plans. Senior engineers owned the module design and migration strategy. It was a great learning period — I got hands-on with how state works, why locking matters, how modules stay DRY across environments. I'm comfortable working within an existing Terraform codebase; authoring complex modules from scratch is something I'm actively deepening, which is part of why I built the infra layer in my Athena project with Terraform.

### "How does your CI/CD pipeline work and what's your role?"
Pipelines run on GitHub Actions and Azure DevOps — build, test, image build/push to ACR, deployment across environments. My role is operational: troubleshooting build/release failures, image promotion, rollbacks. The pipeline design is owned by senior engineers. To get hands-on with building pipelines end-to-end, I built a full GitHub Actions pipeline in my Athena project — build, test, image push, manifest update, with approval gates.

### ⭐ "Tell me about Athena — why and the architecture."
Athena is an observability platform I built from scratch. The motivation was honest: at work I operate a stack set up before me, but I hadn't designed one end to end, and that gap came up in an SRE interview. So I built one. Three microservices in different languages (Python, Node, Go), each emitting metrics, logs, traces. Metrics → Prometheus via ServiceMonitor CRDs. Logs → Promtail → Loki. Traces → OpenTelemetry Collector → Tempo. Unified in Grafana so I can pivot metric → trace → log. I defined a 99.5% availability SLO with multi-window burn-rate alerting through Alertmanager. The most valuable part: I deliberately broke it five ways — memory leak, latency injection, dependency failure, error spike, probe misconfig — and documented the full RCA for each.

### "Why Loki over ELK? Why Tempo? Why OpenTelemetry?" (Athena design)
Loki indexes only metadata labels (namespace, pod), not full content — much cheaper and simpler, on the assumption you query logs by context you already know. ELK indexes every word — powerful for ad-hoc search but expensive at scale. Tempo applies the same idea to traces — store cheaply, retrieve by trace ID. OpenTelemetry decouples instrumentation from backend — instrument once with vendor-neutral SDKs, swap Tempo for Datadog without touching app code. SDK-level vendor lock-in is over now that every vendor accepts OTLP.

### ⭐ "Why are you leaving Cognizant?"
I've grown a lot — intern to DevOps Engineer, operating a real platform at scale. But the role is mostly operating systems other teams designed. I want to go deeper on the engineering and building side of reliability — designing observability, building platforms, writing automation, not just running it. Building Athena confirmed that's the work I find most engaging. I'm looking for a product-focused team or strong SRE org where that's the core of the role.

### ⚠️ "You're Azure-strong but list AWS. How comfortable are you with AWS?"
Straight answer: my production experience is Azure — AKS, App Insights, Azure Monitor, daily. My AWS is project-level — EC2, S3, VPC, IAM, EKS in personal projects, so I understand the fundamentals and the Azure↔AWS mapping is direct. I could ramp on AWS quickly in production, but I wouldn't claim deep production AWS today. The reliability and Kubernetes concepts transfer directly — it's mostly learning AWS-specific service names.

### "Healthcare is regulated — how did that affect your work?"
It made me careful about production changes — strong emphasis on change management, proper approval, audit trails, caution touching production. It built good habits: documenting what I did and why, conservative production changes, understanding reliability is sometimes a compliance goal. That discipline transfers well to any environment where reliability matters, like finance.

### "Hardest production problem you've debugged?"
The OOMKilled leak — what made it hard wasn't the root cause, it was that nothing was obviously broken. No full outage (we had replicas), just intermittent error spikes, easy to dismiss as noise. The breakthrough was recognizing the pattern — memory climbing regardless of traffic = accumulation, not load. The lesson: the hardest incidents are the slow, intermittent ones that don't trip an obvious alarm, and resolution speed depends on how clearly you hand off context.

### "What do you want to grow into next?"
Two things. The building/design side of reliability — designing observability platforms, IaC from scratch, platform engineering (Athena was a step toward this). And scale — working somewhere with bigger scale and stronger SRE practices so I can learn SLO-driven engineering and capacity planning at a level I haven't been exposed to.

### "How do you decide alert vs dashboard metric?"
Every alert should require a human action. If it fires and there's nothing for on-call to do — self-resolves, or just informational — it's a dashboard metric, not an alert. Alerts are expensive: they cause fatigue, and fatigue means real signals get missed. I keep alerts actionable and tied to user impact or SLO burn, push everything else to dashboards. That's exactly the cleanup I did — 8-10 weekend pages down to 2-3.

---

# 2. LINUX & TROUBLESHOOTING

### Process vs thread
A process is independent with its own memory space. A thread is a lighter unit inside a process, sharing its memory. Threads are cheap and share data; processes are isolated and safer.

### `chmod 755`
Three digits — owner/group/others; read=4, write=2, execute=1. 755 = owner rwx (7), group r-x (5), others r-x (5). 755 for executables/dirs, 644 for regular files.

### Hard link vs soft (symbolic) link
Hard link = another name for the same inode (same data); deleting the original doesn't remove the data. Soft link = a pointer to a path; if the original is deleted, the link breaks.

### `/etc`, `/var`, `/tmp`, `/proc`
`/etc` = config. `/var` = variable data (logs, caches). `/tmp` = temporary (often cleared on reboot). `/proc` = virtual filesystem exposing kernel/process info. `/proc/<pid>/` is a debugging goldmine; `/var` filling up is a common incident.

### ⭐ High load average but low CPU usage — what's happening?
Load average counts runnable AND uninterruptible-sleep (D-state, usually I/O-blocked) processes. High load + low CPU = I/O wait, not CPU saturation. Check `top`'s `wa` column, `iostat -x`, and `ps aux | awk '$8 ~ /D/'` for stuck processes. *(A D-state process can't be killed — it's in an uninterruptible kernel operation, won't respond to signals until the I/O completes.)*

### ⭐ `df` says full but `du` doesn't add up — why?
A process holds an open handle to a deleted file. The directory entry is gone (so `du` doesn't count it), but blocks aren't freed until the last handle closes (so `df` still shows them used). Find: `lsof +L1` or `lsof | grep deleted`. Fix: restart/signal the process, or truncate the open fd. *(This is why log cleanup truncates instead of deletes — deleting a file a process has open frees nothing.)*

### Find what's using a port / eating memory
Port: `ss -tlnp | grep :8080` (shows PID/process). Memory: `top` sorted by RES, or `ps aux --sort=-rss | head`. CPU: `top` / `pidstat 1`. `ss` is the modern, faster replacement for `netstat`.

### `kill` vs `kill -9`
`kill` sends SIGTERM (15) — graceful, asks the process to clean up and exit. `kill -9` sends SIGKILL — forceful, immediate, no cleanup. Try SIGTERM first; SIGKILL can leave open files/locks in a bad state. *(Maps to Kubernetes pod termination — same SIGTERM-then-SIGKILL pattern.)*

### Zombie / defunct process
A process that finished but whose parent hasn't read its exit status, so it stays in the process table. Consumes no resources except a table entry. Many zombies = parent not reaping children.

### Cron vs systemd timers
Cron (`crontab -e`) = classic time-based scheduler. Systemd timers = more powerful, integrated with systemd units, better logging via journald, can handle dependencies and missed runs (`Persistent=true`).

### ⭐ RSS vs VSZ vs PSS — which matters for OOM?
VSZ = total virtual address space the process could use (misleadingly large). RSS = resident physical RAM, but counts shared pages (libc) fully in every process. PSS = proportional set size, splits shared pages fairly — most accurate "real cost." For OOM, RSS is the practical number the kernel acts on. *(Summing RSS across processes can exceed total RAM because shared libraries are double-counted; PSS corrects this.)*

### ⭐ How to investigate why a process was OOM-killed
`dmesg -T | grep -i oom` or `journalctl -k | grep -i oom` shows the victim, its memory use, and oom_score. The kernel kills the highest oom_score process. In containers, exceeding the cgroup memory limit = cgroup-OOM (exit 137, container restarts); node-wide exhaustion = system OOM (can kill anything). *(In Kubernetes, `kubectl describe pod` shows OOMKilled with exit 137 = 128 + SIGKILL 9.)*

### A process is hung — find what it's blocked on without killing it
`strace -p <pid>` shows the blocking syscall (`read` on a socket = network wait, `futex` = lock contention, `fsync` = disk). `cat /proc/<pid>/stack` shows the kernel stack; `/proc/<pid>/wchan` the sleeping function. `lsof -p <pid>` shows open files/sockets. *(Caveat: strace pauses the process on every syscall — heavy on production, use briefly.)*

### File descriptor / "too many open files"
An FD is a kernel handle to an open file or socket. Each process has a limit (`ulimit -n`). The error = the process hit its limit, usually from an FD leak or high concurrency with a low limit. Check: `ls /proc/<pid>/fd | wc -l`. Fix: raise the limit (`LimitNOFILE` in systemd) and fix the leak. *(Symptom: new connections fail, the service seems hung while still "running.")*

### cgroup vs namespace (and containers)
Namespaces isolate *what a process can see* (own PID space, network, mounts, users) — container isolation. cgroups limit *what a process can use* (CPU, memory, I/O) — resource limits. Together they're the kernel primitives that *make* containers; Docker/containerd orchestrate them.

### A server is slow — how do you start?
USE method, top-down. `top`/`htop` (CPU, load), `free -h` (memory), `df -h`/`du` (disk), `iostat` (disk I/O), `ss -s` (network). Then narrow to the process with `ps`/`pidstat`. Discipline: system → subsystem → process → cause.

### `dmesg` — what does it reveal?
The kernel ring buffer. Reveals OOM kills, conntrack table full, disk/SCSI errors (failing disk), NIC flaps, segfaults, filesystem errors (read-only remount), network drops. First place to look for issues below the app layer. Long-term: `journalctl -k` or `/var/log/kern.log` (the ring buffer overwrites).

---

# 3. NETWORKING (general + TLS + HTTP)

### What happens when you type a URL and hit enter
DNS resolves the domain to an IP → TCP 3-way handshake → TLS handshake if HTTPS → HTTP request → server responds → browser renders.

### TCP vs UDP
TCP = connection-oriented, reliable, ordered (handshake, acks, retransmission) — HTTP, databases. UDP = connectionless, fast, no guarantees — DNS, streaming, VoIP. Explains why DNS (UDP) behaves differently from HTTP (TCP).

### Public vs private IP
Public = globally routable on the internet. Private (10.x, 172.16-31.x, 192.168.x) = internal, not routable, reach out via NAT.

### ⭐ DNS record types
A (name→IPv4), AAAA (name→IPv6), CNAME (alias), MX (mail), TXT (verification/SPF), NS (nameservers). DNS issues cause a huge share of incidents.

### How DNS resolution works (general)
A resolver queries the hierarchy: root → TLD (.com) → authoritative nameservers, which return the record. Cached per TTL. In cloud you also have private DNS zones for internal resolution within a VNet.

### ⭐ TCP 3-way handshake
SYN (client→server "I want to connect") → SYN-ACK (server→client "ok, and back at you") → ACK (client→server "confirmed"). Connection established.

### ⭐ Connection refused vs connection timed out
Refused = packet reached the host and was actively rejected (RST) — nothing listening, service down, wrong port → app/host layer. Timed out = no response — firewall/NSG silently dropping, wrong IP, routing → network/firewall layer. Tells you which layer to investigate. *(Tools: `curl -v`, `nc -zv host port`, `traceroute`/`mtr` for routing.)*

### ⭐ Walk through a TLS handshake
Client hello (ciphers, TLS version) → server hello + certificate → client verifies the cert chain against trusted CAs → key exchange (ECDHE) creates a shared session key → encrypted communication. TLS 1.3 cut this to one round trip.

### L4 vs L7 load balancing
L4 routes on IP/port — fast, protocol-agnostic, no payload inspection (Azure LB, AWS NLB). L7 routes on HTTP host/path/headers — TLS termination, path routing, rate limiting (App Gateway, ALB, Ingress). L4 for raw throughput/non-HTTP; L7 for HTTP routing and richer control.

### NAT
Network Address Translation maps private IPs to a public IP for internet access. A NAT gateway lets many private resources share one public IP outbound, tracking connections so responses return. How private subnets reach the internet.

### HTTP status code categories
2xx success, 3xx redirect, 4xx client error (400 bad request, 401 unauthorized, 403 forbidden, 404 not found, 429 rate-limited), 5xx server error.

### ⭐ 502 vs 503 vs 504 — what each suggests
502 Bad Gateway = proxy/LB got an invalid response from upstream (upstream crashed or returned garbage). 503 Service Unavailable = no healthy backends / overloaded. 504 Gateway Timeout = upstream took too long (slow backend, exhausted connection pool). In an incident, the specific code narrows the cause fast.

### ⭐ conntrack — how it causes incidents
The kernel's connection-tracking table that remembers NAT mappings so return traffic routes correctly. It has a max size (`nf_conntrack_max`); under high connection churn it fills and new connections get dropped — `dmesg` shows "nf_conntrack: table full," and you see intermittent failures that look like app bugs. Fix: raise `nf_conntrack_max`, reduce churn (connection pooling, keepalive), reduce DNS churn with NodeLocal DNSCache.

### MTU — how it causes mysterious failures
MTU = max packet size a link carries (~1500 bytes). In overlay networks (VXLAN adds ~50 bytes), if pod MTU isn't lowered, large packets get fragmented/dropped while small ones work. Classic symptom: TLS handshake works (small packets), then the connection hangs on the first large payload. Diagnose: `ping -M do -s <size>` to find where it breaks.

### HTTP keep-alive / connection pooling
Reuses a TCP connection for multiple requests instead of opening a new one each time. Avoids repeated handshakes (lower latency), reduces conntrack churn, prevents port/FD exhaustion. Connection pooling does the same for backend calls.

### Forward proxy vs reverse proxy
Forward proxy sits in front of *clients* (outbound — e.g., corporate filtering). Reverse proxy sits in front of *servers* (inbound — nginx/LB terminating TLS, routing, caching).

### ⭐ Debug high latency to an external API
Break it down by phase with `curl -w`: `time_namelookup` (DNS slow?), `time_connect` (TCP/network), `time_appconnect` (TLS), `time_starttransfer` (TTFB — server processing), `time_total`. Isolates exactly which phase is slow. Then check pod vs node (`curl` from the node), the path (`mtr`/`traceroute`), conntrack, MTU, and the external service's health.

---

# 4. KUBERNETES — CORE OBJECTS & YAML

### ⭐ What is a Pod and why not run containers directly?
The smallest deployable unit — one or more containers sharing a network namespace (same IP, localhost), storage, and lifecycle. You don't run bare containers because K8s schedules, heals, and networks at the Pod level. Multiple containers per Pod = tightly-coupled sidecars.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: checkout
spec:
  containers:
  - name: checkout
    image: myacr.azurecr.io/checkout:v1.2.3
    ports:
    - containerPort: 8080
    resources:
      requests: {cpu: 100m, memory: 128Mi}   # scheduler places on this
      limits:   {cpu: 500m, memory: 256Mi}   # exceed mem = OOMKilled; CPU = throttled
```

### ⭐ What does a Deployment do and what's underneath?
Manages stateless apps. Creates a ReplicaSet → which creates Pods. Handles rolling updates (new ReplicaSet alongside old), rollbacks, scaling. Change the pod template → new ReplicaSet → gradual transition.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  replicas: 3
  selector:
    matchLabels:
      app: checkout          # MUST match template labels
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # extra pods above desired during update
      maxUnavailable: 0      # 0 = zero-downtime, needs surge capacity
  template:
    metadata:
      labels:
        app: checkout
    spec:
      containers:
      - name: checkout
        image: myacr.azurecr.io/checkout:v1.2.3
        ports:
        - containerPort: 8080
        readinessProbe:       # gates traffic
          httpGet: {path: /healthz, port: 8080}
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:        # gates restart
          httpGet: {path: /healthz, port: 8080}
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits:   {cpu: 500m, memory: 256Mi}
```
**Key points:** selector must match template labels (common error). `maxUnavailable: 0` = zero-downtime but needs surge capacity. Probes: readiness gates traffic, liveness gates restart.

### Deployment vs ReplicaSet vs StatefulSet vs DaemonSet
Deployment = stateless apps via ReplicaSets + rolling updates. ReplicaSet = just ensures N pods (rarely created directly). StatefulSet = stateful apps needing stable identity + ordered persistent storage. DaemonSet = one pod per node (log collectors, node-exporter, kube-proxy).

### ⭐ What is a Service and the types?
A stable virtual ClusterIP load-balancing to Pods selected by labels (Pods are ephemeral; the Service is stable). Types: ClusterIP (internal), NodePort (port on every node), LoadBalancer (cloud LB), ExternalName (CNAME). Routes: DNS → ClusterIP → kube-proxy iptables DNAT → Pod IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout
spec:
  type: ClusterIP
  selector:
    app: checkout            # routes to pods with this label
  ports:
  - port: 80                 # the Service's port (clients hit this)
    targetPort: 8080         # the container's port
```
**Gotcha:** `port` = what clients hit on the Service; `targetPort` = the container's actual port.

### ⭐ ConfigMap vs Secret
ConfigMap = non-sensitive config. Secret = sensitive data, base64-**encoded** (NOT encrypted by default — common misconception). Both consumed as env vars or mounted files.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0    # base64 — NOT encrypted, just encoded
```
**Gotcha:** Secrets are base64-encoded, not encrypted — anyone with API access can decode. For real security: enable encryption-at-rest in etcd, RBAC to restrict access, and ideally an external store (Key Vault via CSI driver, Vault).

### ⭐ StatefulSet vs Deployment
StatefulSet gives stable, ordered identity: pods named `name-0`, `name-1`, created/scaled in order, each with a stable DNS name (via headless Service) and its **own** persistent volume that follows it across reschedules. Use for databases, Kafka, Zookeeper. Deployments treat pods as interchangeable.

```yaml
  volumeClaimTemplates:        # each pod gets its OWN PVC (data-0, data-1...)
  - metadata: {name: data}
    spec:
      accessModes: ["ReadWriteOnce"]
      resources: {requests: {storage: 10Gi}}
```
Headless Service (`clusterIP: None`) gives each pod a stable DNS name like `postgres-0.postgres-headless`.

### Job vs CronJob
Job = runs a Pod to completion (batch — migration, backup), tracks success/retries. CronJob = runs Jobs on a schedule.

### ⭐ Requests vs limits + QoS classes
Requests = guaranteed minimum the scheduler reserves (placement). Limits = hard cap (exceed memory → OOMKilled exit 137; exceed CPU → throttled, not killed). Ratio sets QoS: Guaranteed (requests==limits), Burstable (requests<limits), BestEffort (none) — which sets eviction order under pressure.

### ⭐ Liveness vs readiness vs startup probe
Readiness gates *traffic* (fail = removed from endpoints, not killed). Liveness gates *restart* (fail = killed/restarted). Startup = grace period for slow starters, disables the others until ready. Misconfigured readiness = silent traffic loss; misconfigured liveness = CrashLoopBackOff. *(Real story: liveness initialDelay too short for slow startup → kubelet killed it before ready → fixed with higher initialDelaySeconds + startupProbe.)*

```yaml
startupProbe:
  httpGet: {path: /healthz, port: 8080}
  failureThreshold: 30        # allows 30 × periodSeconds for startup
  periodSeconds: 10
```

### nodeSelector vs affinity vs taints vs topology spread
nodeSelector = exact label match. nodeAffinity = richer (required/preferred). podAffinity/anti-affinity = schedule relative to other pods. Taints (on nodes) + tolerations (on pods) = repel/allow. topologySpreadConstraints = distribute replicas across zones/nodes (the modern, lighter way to spread across AZs for HA).

### PodDisruptionBudget
Sets minimum available pods during *voluntary* disruptions (drains, upgrades, autoscaler scale-down). Does NOT protect against involuntary (node crash).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: checkout}
```

### ⭐ HPA — how it works
Scales replicas on a metric vs target. Formula: `desired = ceil(current × currentMetric / targetMetric)`. Needs resource requests set (to compute CPU %). Polls metrics-server ~15s.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: checkout}
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 70}
```

### ⭐ Ingress vs Service (LoadBalancer)
A LoadBalancer Service is L4, one cloud LB per service — expensive. An Ingress is L7 HTTP routing (host/path) to many services behind a *single* LB, with TLS termination. An Ingress controller (nginx) runs as pods implementing the rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  ingressClassName: nginx
  tls:
  - {hosts: [app.example.com], secretName: app-tls}
  rules:
  - host: app.example.com
    http:
      paths:
      - {path: /checkout, pathType: Prefix, backend: {service: {name: checkout, port: {number: 80}}}}
      - {path: /orders,   pathType: Prefix, backend: {service: {name: orders,   port: {number: 80}}}}
```

### ⭐ NetworkPolicy + the critical gotcha
Defines allowed ingress/egress by label selector. **Gotcha: the CNI must support it** (Calico, Cilium do; Flannel alone doesn't) — apply on an unsupporting CNI and it silently does nothing. Once a policy selects a pod, that pod becomes default-deny for that direction.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  podSelector:
    matchLabels: {app: checkout}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {app: api-gateway}}   # only gateway reaches checkout
    ports: [{protocol: TCP, port: 8080}]
```

### ⭐ PV / PVC / StorageClass
StorageClass = *how* to provision (provisioner, disk type). PVC = a request for storage. PV = the actual storage. Dynamic: PVC → StorageClass → PV created automatically. Access modes: ReadWriteOnce (one node — Azure Disk), ReadWriteMany (many nodes — Azure Files/NFS). RWO is why a Deployment with a cloud disk can't easily run pods across nodes. Reclaim policy: Delete vs Retain.

### ⭐ RBAC — Role, ClusterRole, RoleBinding
Role (namespace) / ClusterRole (cluster-wide) define *permissions*. RoleBinding / ClusterRoleBinding *attach* them to a user, group, or ServiceAccount. Least privilege — minimum verbs/resources. ServiceAccounts are how pods authenticate to the API.

```yaml
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]      # read-only
```

### Namespace vs label
Namespaces = hard boundaries for isolation, RBAC, quotas (dev/staging/prod, per-team). Labels = flexible tags for selection within/across namespaces. Can't query across namespaces easily; labels work anywhere.

---

# 5. KUBERNETES — INTERNALS & CONTROL PLANE

### ⭐ What happens when you run `kubectl apply`? (full lifecycle)
kubectl → API server (authenticate → authorize via RBAC → admission controllers → validate) → writes to etcd. etcd notifies the API server, which streams a watch event. The Deployment controller creates a ReplicaSet; the ReplicaSet controller creates Pods (Pending, no node). The scheduler filters and scores nodes, writes `nodeName`. The kubelet on that node calls containerd via CRI to pull the image and start the container; the CNI assigns the IP. Once readiness passes, the endpoints controller adds the Pod IP to the EndpointSlice and kube-proxy programs iptables so traffic routes.

### Reconciliation loop / level-triggered
Controllers continuously compare desired state (etcd) vs actual and close the gap. Level-triggered = they react to *current state*, not the *event* — "what is true now," not "what changed." This is why K8s self-heals: a missed event still reconciles correctly on the next loop because it re-reads full state. *(Controller crashes mid-reconcile → re-reads on restart; reconciliation is idempotent.)*

### How controllers learn about changes (watch / informer)
The watch mechanism through the API server — a long-lived connection streaming events. client-go uses an "informer": initial LIST to build a local cache, then WATCH for deltas. Controllers read from the local cache, not etcd directly, reducing API server load. *(Resync period = periodic re-list as a safety net.)*

### Why is the API server the only thing that talks to etcd?
It centralizes authn, authz, admission, validation, and audit in one place. etcd has no RBAC or admission — direct access would mean reimplementing security everywhere. It also lets you swap/shard the datastore behind the API server. *(k3s uses SQLite via a shim called kine.)*

### Mutating vs validating admission
After authn/authz, two phases: mutating runs first (can modify — inject sidecars, set defaults), then validating (accept or reject only — enforce policy, reject root containers). Mutate-then-validate so validation sees the final object. Kyverno / OPA Gatekeeper plug in here.

### ⭐ What if etcd loses quorum?
etcd uses Raft, needs a majority to accept writes. Lose quorum → read-only, no writes → no new pods, no edits, no scaling. But existing workloads keep running (kubelets cached their specs). Recovery: restore quorum or restore from snapshot. *(Odd number of members for a clear majority — 3 tolerates 1 failure, 5 tolerates 2.)*

### Scheduler's two phases
Filtering (predicates): eliminate nodes that can't run the pod — insufficient CPU/memory vs requests, untolerated taints, failed affinity/selector, port conflicts, volume zone mismatch. Scoring (priorities): rank survivors — least requested, affinity preferences, image locality, topology spread. Highest score wins; writes `nodeName`. *(No node passes → Pending + FailedScheduling event; cluster autoscaler may add a node.)*

### Node affinity vs pod affinity vs anti-affinity
Node affinity = attract pod to nodes with certain labels. Pod affinity = attract to nodes where certain *other pods* run (co-locate). Pod anti-affinity = repel (keep replicas apart). Each can be required (hard) or preferred (soft). *(Anti-affinity is expensive at scale — evaluates against all matching pods cluster-wide; topologySpreadConstraints is lighter.)*

### Taints and tolerations vs affinity
Taints on *nodes* repel pods that don't tolerate them. Tolerations on *pods* let them schedule onto tainted nodes. Inverse of affinity (pod attracting itself). Effects: NoSchedule, PreferNoSchedule, NoExecute (evict existing). The node controller auto-taints NotReady nodes.

### ⭐ Node goes NotReady — full lifecycle
kubelet stops heartbeating → node controller marks NotReady after ~40s, applies NotReady:NoExecute taint → pods without toleration evicted after tolerationSeconds (default 300s) → recreated elsewhere by their controller. The 5-min delay avoids mass rescheduling on transient blips.

### ⭐ Pod termination sequence
Pod marked Terminating, removed from Service endpoints (stops new traffic). kubelet runs preStop hook, sends SIGTERM, waits up to terminationGracePeriodSeconds (default 30s), then SIGKILL. Endpoint removal + SIGTERM happen ~concurrently — that's why you add a preStop sleep, so in-flight requests drain before the process dies. *(Errors can still happen because endpoint removal propagates asynchronously to kube-proxy; the preStop sleep covers the gap.)*

### Pod stuck Terminating forever
Causes: a finalizer waiting on cleanup that never completes; kubelet on the node is down; volume won't unmount; process ignoring SIGTERM. `kubectl describe`. Stuck finalizer → patch it out carefully. Dead node → force-delete (`--grace-period=0 --force`) but it just removes it from the API. *(Dangerous for StatefulSets — split-brain if the node wasn't truly dead.)*

### ⭐ QoS classes + eviction order under memory pressure
Guaranteed (requests==limits), Burstable (requests<limits), BestEffort (none). kubelet evicts BestEffort first, then Burstable over requests, Guaranteed last. Set requests==limits + a high priorityClass for a critical pod to survive pressure.

### Preemption vs QoS eviction
Preemption is scheduler-driven: a high-priority pending pod evicts lower-priority running pods to make room. QoS eviction is kubelet-driven under node resource pressure. Preemption = *fitting* a pod that can't schedule; eviction = *relieving* pressure on a full node. *(Preemption tries to respect PDBs but will violate them if necessary.)*

---

# 6. KUBERNETES — NETWORKING (CNI / kube-proxy / CoreDNS)

### The K8s network model (core requirements)
Three rules: every pod gets its own IP; pods communicate across nodes without NAT; the IP a pod sees itself as is the same others use to reach it. The CNI implements this — overlay (VXLAN) or native routing (Azure CNI) — but the model is constant.

### What a CNI plugin does when a pod starts
Allocates an IP from the cluster CIDR (IPAM), creates a veth pair connecting the pod's namespace to the node's, configures the interface and routes, programs the node. On deletion it reverses and returns the IP. *(The container runtime, containerd, invokes the CNI, orchestrated by the kubelet.)*

### Overlay vs native (underlay) networking
Overlay (Flannel VXLAN, Calico IPIP) encapsulates pod packets inside node-to-node packets — works anywhere, adds ~50 bytes overhead (MTU concerns). Native (Azure CNI, AWS VPC CNI, Calico BGP) gives pods real routable IPs from the VNet — better performance, but consumes IP space. *(Azure CNI exhausts IPs because each pod gets a real VNet IP and nodes pre-allocate blocks.)*

### MTU problem in overlay networks
VXLAN's ~50-byte header means the inner packet must be smaller than physical MTU, or packets fragment/drop. If pod MTU isn't lowered: small packets work, large ones hang. Classic symptom — TLS handshake works, connection stalls on first large payload. Diagnose: `ping -M do -s <size>`.

### ⭐ kube-proxy iptables mode — the rule chain
Traffic to a ClusterIP hits `KUBE-SERVICES` → per-service chain `KUBE-SVC-xxx` → uses the `statistic` module with random probability to pick a backend `KUBE-SEP-xxx` → DNATs to a real pod IP. conntrack records the mapping so replies are un-NATed. *(O(n) because rules are evaluated sequentially — thousands of services = thousands of rules per connection.)*

### ⭐ IPVS mode — why better at scale
Kernel hash tables instead of sequential iptables — O(1) service lookup regardless of count. Real LB algorithms (round-robin, least-connection). Still uses some iptables for masquerade/marking, but the routing decision is IPVS. *(kube-proxy isn't in the data path — it programs rules ahead of time; the kernel handles each packet.)*

### ⭐ Trace a CoreDNS lookup (with ndots)
Pod's `/etc/resolv.conf` has `nameserver <CoreDNS ClusterIP>`, search domains, `options ndots:5`. A short name (< 5 dots) gets each search domain appended and tried first. CoreDNS (watches the API for Services/Endpoints) answers `*.svc.cluster.local` from memory, forwards external names upstream. *(ndots:5 hurts external lookups — `api.stripe.com` triggers `api.stripe.com.ns.svc.cluster.local` first; fix with a trailing dot to force FQDN.)*

### CoreDNS slow / high DNS latency — debug
Check CoreDNS pod CPU/memory and replica count (often under-provisioned). The ndots:5 amplification. conntrack exhaustion on UDP 53. Fix: scale CoreDNS, enable `autopath`, deploy **NodeLocal DNSCache** (a per-node DNS cache that cuts cross-node DNS and conntrack pressure), use FQDNs.

### NodePort vs ClusterIP vs LoadBalancer (implementation)
ClusterIP = virtual IP, internal only, kube-proxy rules. NodePort = ClusterIP + a port (30000-32767) on every node forwarding to the service. LoadBalancer = NodePort + a cloud LB pointing at those node ports. Layered. *(NodePort rarely used directly — limited port range, no TLS, no L7; Ingress + one LB is better.)*

### NetworkPolicy + what enforces it
Defines allowed ingress/egress by label selector — default-deny once a policy selects a pod. The CNI must support and enforce it (Calico, Cilium do; Flannel doesn't). Apply on an unsupporting CNI → silently no effect (false security).

### East-west traffic: same node vs across nodes
Same node: pod veth → node bridge → destination pod veth, never leaves the host (microsecond latency, no encapsulation). Across nodes: out the node's network, encapsulated (overlay) or routed natively (underlay). *(topologySpreadConstraints can keep chatty services on the same zone to cut latency and cross-zone egress cost.)*

---

# 7. KUBERNETES — SCENARIOS

### ⭐ Pod in CrashLoopBackOff — debug
`kubectl describe pod` (events + last state + exit code) → `kubectl logs --previous` (the crashed container). Causes: app error on startup, failed liveness probe, missing config/secret, OOMKilled (137), bad image. *(Real story: liveness initialDelay too short for slow startup → killed before ready → fix with startupProbe.)* Narrate the investigation, don't just list commands.

### ⭐ Pod is Pending — why?
`kubectl describe pod` → read FailedScheduling. Causes: insufficient resources (no node has enough for *requests*), untolerated taints, affinity/selector matches nothing, unbound PVC, anti-affinity unsatisfiable. *(Gotcha: "insufficient resources" is based on requests, not usage — nodes can be "full" on requests while CPU usage is low.)*

### ⭐ Service has endpoints but traffic intermittently fails
A pod Ready but app not truly serving (readiness too lenient); kube-proxy reconciliation lag after endpoint changes; a terminating pod still in endpoints getting traffic (fix: preStop hook + graceful shutdown); conntrack table full. Check `kubectl get endpoints`, the readiness probe, termination handling.

### ⭐ Deployment rollout stuck — `kubectl rollout status` hangs
`kubectl get rs` (old vs new counts), `kubectl describe` the new RS's pods. Causes: new pods failing readiness (won't progress past maxUnavailable), image pull errors, insufficient resources for surge pods, CrashLoopBackOff in the new version, PDB blocking old pods. Rollout gates on new pods becoming Ready. *(maxSurge = extra above desired; maxUnavailable = below.)*

### Pod Running but receives no traffic
Check the READY column — Running but 0/1 Ready = readiness failing → not added to endpoints → no traffic. `kubectl get endpoints <svc>` shows `<none>`. `kubectl describe pod` shows the readiness failure. The "readiness misconfig = silent traffic loss" pattern.

### Memory grows, pods OOMKilled every few hours
Memory climbing on a steady slope regardless of traffic = leak, not load. Confirm OOMKilled via exit 137 in last state. Correlate logs/traces to find the leaking endpoint. Short-term: raise limit / add replicas; real fix: the code leak. Prevention: alert at 80% of limit *before* the OOMKill.

### Zero-downtime deployment — how?
RollingUpdate with `maxUnavailable: 0`, `maxSurge: 1`; proper readiness probes; preStop hook + graceful shutdown (drain in-flight); PDB; multiple replicas. For higher safety, blue-green or canary.

### Pod can't reach an external service / another pod
Exec in (or a netshoot debug pod). Test DNS (`nslookup`), connectivity (`nc -zv`), check NetworkPolicy blocking egress, check pod vs node level, check the target's health. Isolate layer by layer: DNS → connectivity → policy → target.

### Cloud-specific: pod can't reach Azure SQL
From inside the pod: test DNS (`nslookup` the DB hostname), connectivity (`nc -zv host 1433`). DNS fails → check private DNS zone / private endpoint. Connect times out → NSG rules on the subnet, the DB firewall (is the AKS subnet allowed?), private endpoint config. Refused → DB-side. Also check NetworkPolicy egress. Isolate pod / node / DNS / path / DB firewall.

---

# 8. DOCKER & CONTAINERS

### ⭐ Image vs container
Image = read-only template (layered filesystem + metadata). Container = a running/stopped instance with a thin writable layer on top. Image = class, container = object.

### ⭐ Container vs VM
A VM virtualizes hardware, runs a full guest OS with its own kernel (GBs, slow boot). A container virtualizes the OS — shares the *host kernel*, isolates the process with namespaces + cgroups (MBs, ms startup). Containers are isolated processes, not mini-machines.

### ⭐ What makes a container isolated (kernel primitives)
**Namespaces** isolate what a process can see (PID, network, mount, user). **cgroups** limit what it can use (CPU, memory, I/O). Docker/containerd orchestrate these kernel features.

### Docker daemon, client, registry
Client (`docker` CLI) sends commands. Daemon (`dockerd`) does the work — build, run, manage. Registry (Docker Hub, ACR) stores images. `docker build` runs server-side on the daemon.

### `docker stop` vs `docker kill`
`stop` = SIGTERM, wait (10s grace), then SIGKILL — graceful. `kill` = SIGKILL immediately. Prefer `stop`. Same SIGTERM-then-SIGKILL pattern as Kubernetes pod termination.

### ⭐ How image layers work
Each Dockerfile instruction (RUN, COPY, ADD) creates a read-only layer, stacked via a union filesystem (overlay2). Containers add a writable layer. Layers are cached and shared — a shared base is stored once.

### ⭐ Build caching + Dockerfile optimization
Docker reuses a cached layer if the instruction and its inputs are unchanged; once busted, every later layer rebuilds. Put rarely-changing instructions first (install dependencies), frequently-changing last (copy source). Classic: copy `package.json` + `npm install` *before* copying source, so a code change doesn't bust the dependency cache.

### Dangling image / cleanup
A dangling image = untagged leftover layer (`<none>:<none>`). `docker image prune` removes them; `docker system prune -a` removes unused images, containers, networks, build cache.

### ⭐ Common Dockerfile instructions
FROM (base), WORKDIR, COPY (copies files), ADD (COPY + URLs + tarball extraction — prefer COPY), RUN (build-time, creates a layer), ENV, EXPOSE (documents the port, doesn't publish), CMD (default runtime args, overridable), ENTRYPOINT (fixed runtime executable), ARG (build-time variable).

### ⭐ CMD vs ENTRYPOINT
ENTRYPOINT = the fixed executable that always runs. CMD = default args, overridable at `docker run`. Pattern: `ENTRYPOINT ["python","app.py"]` + `CMD ["--port=8080"]`. CMD-only = the whole thing is overridable.

### COPY vs ADD
Both copy files. ADD also extracts tarballs and fetches URLs — surprising, a security risk. Use COPY for plain copies; ADD only when you need tarball extraction.

### RUN vs CMD/ENTRYPOINT (build-time vs runtime)
RUN executes at *build* and bakes into a layer (installing packages). CMD/ENTRYPOINT define what runs when the *container starts*.

### ⭐ Multi-stage build
Multiple FROM stages: build in a heavy stage with all tools, `COPY --from=builder` only the final artifact into a slim runtime stage. Smaller image, smaller attack surface, no build tools shipped, faster pulls.

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```

### ⭐ Reduce image size
Multi-stage; slim/distroless/alpine base; combine RUN with `&&` (fewer layers); clean package caches in the same layer (`apt-get install ... && rm -rf /var/lib/apt/lists/*`); `.dockerignore`; remove build deps. *(Diagnose with `docker history <image>` for per-layer size.)*

### `.dockerignore`
Like `.gitignore` for the build context — excludes `.git`, `node_modules`, secrets, large test data. Without it, `COPY . .` drags everything in, bloating the image and slowing builds.

### ⭐ Why not run as root + how to fix
A container-escape vuln means root-in-container can become root-on-host. Fix: create and switch to a non-root user (`RUN adduser ...` + `USER appuser`). Many policies/admission controllers reject root containers.

### Secrets in a Docker build
Never COPY secrets in or put them in ENV (they persist in layers + `docker history`). Use BuildKit secret mounts (`--mount=type=secret`) or inject at runtime via env / mounted files.

### scratch vs alpine vs ubuntu vs distroless
scratch = empty, no OS (only static binaries like Go) — smallest, most secure. alpine = ~5MB, package manager, musl libc (occasional quirks). ubuntu/debian = full but large (~70MB+). distroless = minimal but with libc — a popular middle ground.

### ⭐ Docker network drivers
bridge (default — private internal network, NAT to outside, each container an internal IP), host (shares the host's network namespace — fast, port conflicts), none (no networking), overlay (spans multiple hosts — Swarm), macvlan (own MAC, appears as a physical device).

### ⭐ Containers on the same bridge — how they communicate
On a *user-defined* bridge, Docker provides automatic DNS — containers resolve each other by name (`app` reaches `db` at `http://db:5432`). On the *default* bridge this DNS doesn't work. *(This is why Compose works — it creates a user-defined network with name-based DNS.)*

### ⭐ EXPOSE vs -p vs --expose
EXPOSE (Dockerfile) = documentation only, doesn't publish. `-p 8080:80` (publish) at `docker run` actually maps host port 8080 to container port 80 — what genuinely exposes a container.

### How a container reaches the internet / how inbound reaches it
Outbound: on bridge, NAT'd through the host's IP. Inbound: only with `-p host:container`, which sets an iptables DNAT rule on the host forwarding the host port to the container.

### `host.docker.internal`
A special DNS name resolving to the host machine from inside a container — for reaching a service running on the host.

### Docker networking vs Kubernetes networking
Docker's bridge is single-host. K8s needs cross-host pod-to-pod with every pod routable, so it uses CNI plugins (not Docker's bridge). Docker = the runtime; CNI = the cluster network.

### ⭐ Volumes vs bind mounts vs tmpfs
Volume = Docker-managed (`/var/lib/docker/volumes/`), the preferred way to persist — portable, decoupled from the host path. Bind mount = maps a host directory in — tight host coupling, good for local dev. tmpfs = in-memory, never on disk — sensitive/temp data.

### Why container data disappears / why databases are tricky
Data in the writable layer is destroyed when the container is removed — mount a volume to persist. Databases need durable state, careful volume management, backup/restore, and don't love being killed/rescheduled — in production, managed databases (RDS, Azure SQL) are usually preferred.

### ⭐ What is Docker Compose and when?
Define and run multi-container apps with one YAML (`docker-compose.yml`) and one command (`docker compose up`). For *local dev/test* — not production orchestration (that's Kubernetes).

```yaml
services:
  web:
    build: ./web
    ports: ["8080:80"]
    environment: [DB_HOST=db]
    depends_on: [db]
  db:
    image: postgres:16
    environment: [POSTGRES_PASSWORD=secret]
    volumes: [dbdata:/var/lib/postgresql/data]
volumes:
  dbdata:
```

### How Compose services communicate
Compose creates a default user-defined bridge network — services reach each other by *service name* via Docker DNS (`web` connects to `db` at hostname `db`). No IP hardcoding.

### `depends_on` — the catch
Controls *startup order* (start db before web) but NOT *readiness* — only that it's *started*. So web may start before Postgres accepts connections. Fix: a healthcheck with `condition: service_healthy`, or retry logic.

### Compose vs Kubernetes
Compose: local dev, single host, simple. Kubernetes: production, multi-node, scaling, self-healing, rolling updates, service discovery, HA. Compose doesn't do multi-node scheduling or self-healing.

### ⭐ Scenario: container exits immediately
`docker ps -a` (Exited) → `docker logs` → exit code (`docker inspect`). Causes: main process crashed on startup (bad config, missing env/dependency), wrong CMD/ENTRYPOINT, or the foreground process finished. Key insight: a container runs only as long as PID 1; if it exits or runs in the background, the container stops.

### ⭐ Scenario: container OOM-killed
Exceeded its memory limit → OOMKilled (exit 137). Check `docker stats` (live memory), `docker inspect` (limit + OOMKilled flag), app logs. Causes: leak, limit too low, spike. Maps exactly to Kubernetes OOMKilled.

### Scenario: image too big (1.2GB)
Multi-stage build; slim/alpine/distroless base; combine RUN layers + clean caches; `.dockerignore`; remove unneeded deps. `docker history` shows per-layer size.

### Scenario: build went from 2 min to 10 min
Cache busted early — a `COPY . .` or frequently-changing file before `RUN npm install` re-runs the expensive install every build. Fix: reorder so dependencies install before source. `docker build` output ("Using cache" vs rebuilding) shows where it breaks.

### Scenario: works on my machine, fails in production
Different architecture (ARM Mac → x86, need multi-arch/`--platform`); different env/config; missing volumes/secrets; network/DNS differences; `latest` tag pulled a different version; resource limits in prod (OOM/throttle) not present locally.

### Scenario: debug a container with no shell (distroless)
`docker logs`, `docker inspect`, `docker stats`. For deeper debugging, attach an ephemeral container sharing the target's namespaces (`docker run --pid container:<target> --network container:<target> nicolaka/netshoot`). In Kubernetes this is `kubectl debug` (ephemeral containers).

---

# 9. CI/CD, GITHUB ACTIONS, HELM, GITOPS, ARGOCD

## GitHub Actions

### ⭐ Core building blocks
A CI/CD platform in GitHub. **Workflow** (YAML in `.github/workflows/`, triggered by events) → **jobs** (run on runners, parallel by default) → **steps** (commands or actions). An **action** = a reusable unit. A **runner** = the machine that executes a job.

### Runner: GitHub-hosted vs self-hosted
GitHub-hosted = a fresh VM per job (clean, managed, on GitHub's infra). Self-hosted = your own machines — for private networks (a private AKS cluster), custom hardware, more control. Self-hosted needs careful securing; never on public repos.

### Common triggers
`push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), `release`, `workflow_call` (reusable). Filter by branch, path, tag.

### ⭐ Secrets in GitHub Actions + OIDC
Encrypted GitHub Secrets (repo/env/org), injected as masked env vars at runtime, scoped to the job. For cloud auth, the modern best practice is **OIDC federation** — the runner requests a short-lived token from the cloud provider instead of storing long-lived credentials.

### ⭐ Why OIDC beats stored credentials
Stored credentials are long-lived — if leaked, valid until manually rotated. OIDC issues a short-lived, scoped token per run, no stored secret. If a run is compromised, the token expires in minutes. Eliminates the leaked-long-lived-credential risk.

### Environments + protection rules
GitHub Environments (dev/staging/prod) can have required reviewers (manual approval gate before prod), wait timers, env-scoped secrets.

### Passing data between jobs/steps
Within a job: shared filesystem + step outputs. Between jobs: `needs:` + job `outputs`, or artifacts (`upload-artifact`/`download-artifact`). Jobs are isolated (different runners).

### ⭐ GitHub Actions security risks + mitigations
Untrusted action code → pin to a full commit SHA, not a moving tag. `pull_request_target` → runs with secrets on fork PRs, dangerous, avoid. Script injection → untrusted input (PR titles) in `run:` can execute code; sanitize / use env vars. Over-privileged `GITHUB_TOKEN` → set `permissions:` to least-privilege. Self-hosted runner compromise → isolate, never on public repos.

### Reusable workflow vs composite action
Reusable workflow (`workflow_call`) = one workflow calls another (share whole pipelines). Composite action = bundle multiple steps into one action. Reusable = higher-level (jobs); composite = lower-level (steps).

## Helm

### ⭐ What is Helm and why use it?
The package manager for Kubernetes. A **chart** = a package of templated manifests. Raw K8s YAML is static and duplicated; Helm templates the manifests and injects environment-specific **values**, so one chart deploys everywhere with different `values.yaml`. It also versions releases and enables rollback.

### Core concepts: chart, values, release, template
Chart = the package (templates + defaults + metadata). Templates = manifests with Go placeholders (`{{ .Values.replicaCount }}`). Values = inputs (`values.yaml`), overridable per env. Release = a deployed instance of a chart (versioned history).

### Chart structure
`Chart.yaml` (metadata), `values.yaml` (defaults), `templates/` (manifests), `templates/_helpers.tpl` (reusable snippets), `charts/` (subchart dependencies).

### Override values per environment
Separate files (`helm install -f values-prod.yaml`), `--set key=value`, or layered values. Later files/flags override earlier.

### ⭐ Helm rollback
Helm tracks release history (`helm history <release>`). `helm rollback <release> <revision>` re-applies a previous revision's manifests — a big advantage over raw `kubectl apply`.

### Helm hooks
Annotations running resources at lifecycle points — pre-install, post-install, pre-upgrade, post-delete — e.g., a DB migration Job before an upgrade.

### Helm vs Kustomize
Helm = templating + values + packaging + release/rollback (Go templating can get complex). Kustomize = template-free overlays, patch a base per environment (simpler, built into kubectl, no packaging/release). Helm for packaging/distribution; Kustomize for simpler overlays.

### Validate a Helm chart
`helm lint` (syntax/best-practice), `helm template` (render locally without deploying), `--dry-run` (server-side validation), `helm test` (run test hooks against a release).

## GitOps & ArgoCD

### ⭐ What is GitOps?
An operational model where **Git is the single source of truth** for declarative infra and app state. You describe desired state in Git; an in-cluster controller continuously reconciles the cluster to match. No `kubectl apply` from a pipeline — you commit to Git, the controller pulls. Changes via PR (reviewed, audited, revertable).

### ⭐ Push vs Pull CD — why pull (GitOps)?
Push = CI holds cluster credentials and pushes (`kubectl apply` from CI). Pull = an in-cluster controller watches Git and pulls in. Pull is preferred: (1) security — cluster creds never leave the cluster; (2) source of truth — Git *is* the cluster, with drift detection; (3) auditability — every change is a reviewed commit; (4) easy rollback — revert the commit.

### Reconciliation loop in GitOps
The controller continuously compares desired (Git) vs actual (cluster). On drift it reports OutOfSync and (with auto-sync) re-applies Git's state. Same level-triggered, desired-vs-actual model as K8s controllers.

### ⭐ What is ArgoCD and how does it work?
A GitOps CD controller running *inside* the cluster. You define an **Application** pointing to a Git repo path (manifests, Helm, or Kustomize) and a target cluster/namespace. ArgoCD watches the repo, compares desired vs live, shows sync status, applies changes manually or via auto-sync. UI/CLI shows health + sync status.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: https://github.com/org/manifests
    targetRevision: main
    path: apps/checkout
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true          # delete resources removed from Git
      selfHeal: true       # revert manual cluster changes back to Git
```

### ⭐ prune, selfHeal, auto-sync
Auto-sync = automatically apply Git changes. prune = delete cluster resources removed from Git (cluster matches Git exactly). selfHeal = revert manual cluster changes back to Git (enforces Git as truth).

### App-of-Apps pattern
A parent ArgoCD Application pointing to a Git path containing *other* Application definitions — one app bootstraps many. Manages many apps/environments from one root.

### Sync waves and hooks in ArgoCD
Sync waves order resource application (database before the app needing it) via annotations. Resource hooks (PreSync, Sync, PostSync) run at sync phases — like a migration Job before the new version syncs.

### ⭐ ArgoCD rollback
Revert the Git commit (the GitOps way — ArgoCD syncs back, fully audited), or roll back to a previous synced revision via ArgoCD history. Git-revert is preferred — keeps Git as source of truth.

### ArgoCD vs Flux
Both reconcile Git → cluster. ArgoCD = rich UI, application-centric (good visibility). Flux = lightweight, CLI/GitOps-toolkit, composable. Choice is preference/ecosystem.

## The Full Pipeline & Security

### ⭐ Walk me through a complete CI/CD + GitOps pipeline
Developer pushes → **GitHub Actions (CI)**: checkout → tests → SAST scan → build image → **Trivy** scan → push to **ACR** with an immutable tag (git SHA) → update the image tag in a Helm values file in a separate *manifests* repo (or open a PR). → **ArgoCD (CD)** detects the manifests change → compares desired vs live → syncs into AKS. Rollback = revert the commit. Key point: **CI builds + pushes to a registry + updates Git; CD pulls from Git into the cluster** — CI never touches the cluster directly. That's the GitOps security win.

### Why separate app repo from manifests repo
Separation of concerns: app repo triggers builds, manifests repo is what ArgoCD watches. A config change doesn't require an app rebuild, deployment history is a clean Git log, and you control who changes *what's deployed* separately from who changes *code*.

### Typical CI/CD pipeline (general)
On push/PR: checkout → tests + lint → build image → scan (Trivy) → push to registry with a versioned tag → update the manifest → deploy (or ArgoCD syncs). Manual approval gates before production.

### Rollback methods
Kubernetes: `kubectl rollout undo deployment/x`. GitOps: revert the commit, ArgoCD syncs back. Image-based: redeploy the previous tag. Cleanest = GitOps revert (auditable).

### Blue-green vs canary
Blue-green = two full environments, test green, switch all traffic at once (instant rollback, double resources). Canary = gradually shift traffic % (5→25→100), watching metrics (low blast radius, slower, needs good observability).

### Image tagging strategy
Avoid `latest` in production (not immutable, can't tell what's running). Use immutable tags — semver (`v1.2.3`), git SHA, or build number. SHA guarantees you know the exact commit deployed.

### CI vs CD
CI = automatically build and test on every change. CD = automatically deliver/deploy the tested artifact.

### ⭐ Security checks in a CI/CD pipeline
SAST (scan source — CodeQL, SonarQube), SCA (scan dependencies for CVEs — Dependabot, Snyk), image scanning (Trivy, Grype, Defender — *before* push), secret scanning (gitleaks, GitHub secret scanning), IaC scanning (tfsec, Checkov, kube-linter), image signing (Cosign/Sigstore, verify at deploy). "Shift-left security."

### ⭐ Secure the supply chain (code → cluster)
Pin Actions to SHAs; least-privilege GITHUB_TOKEN + OIDC; scan code/deps/images in CI; sign images and verify at admission (Cosign + Kyverno/OPA Gatekeeper); immutable tags; private registry (ACR) with RBAC; admission policies rejecting unsigned/untrusted/root images. Trusted code → scanned → signed → only verified artifacts run.

### Testing in the pipeline
Unit (fast, every commit), integration, `helm lint`/`template`/`--dry-run`, `kube-linter`/`kubeval` for K8s YAML, smoke tests post-deploy, ArgoCD health checks, and canary analysis for progressive delivery.

### How GitOps improves security
Cluster creds never leave the cluster (CI has no cluster access); every change is a reviewed, audited commit (no ad-hoc kubectl to prod); drift is auto-corrected (selfHeal); access control via Git (who can merge controls what deploys). Turns "who has kubectl access to prod" into "who can merge to Git."

### Azure DevOps vs GitHub Actions
Both do CI/CD. Azure DevOps = full suite (Repos, Pipelines, Boards, Artifacts), strong Azure integration, common in enterprises. GitHub Actions = CI/CD in GitHub, huge marketplace. Many orgs are shifting toward Actions.

### What is ACR?
Azure Container Registry — managed private Docker registry. Build → push to ACR → AKS pulls (via Managed Identity, no stored creds). Also image scanning (Defender) and geo-replication.

---

# 10. TERRAFORM / IaC

### ⭐ What is Terraform and what problem does it solve?
An IaC tool — declare desired infra in HCL, Terraform makes the real world match. Solves manual, error-prone provisioning: infra becomes version-controlled, reviewable, repeatable, auditable, no drift or snowflake servers.

### ⭐ Declarative vs imperative
Terraform is declarative — describe the *desired end state*, Terraform figures out *how* (create/update/delete to reach it). Imperative (bash) describes the *steps*. Declarative = idempotent, converges to the same state without you tracking what exists.

### Terraform vs Ansible vs CloudFormation
Terraform = cloud-agnostic provisioning, declarative, state-based. CloudFormation = AWS-only provisioning. Ansible = config management (configuring existing servers), procedural. Terraform provisions infra; Ansible configures it. Often used together.

### Core workflow
`terraform init` (providers + backend) → `plan` (preview the diff) → `apply` (execute) → `destroy`.

### ⭐ plan vs apply vs refresh
`plan` = dry run, shows what *would* change (config vs state vs reality), no changes. `apply` = executes, updates state. `refresh` = updates state to match reality without changing infra (now folded into plan/apply).

### ⭐ What is Terraform state and why critical?
State (`terraform.tfstate`) is Terraform's record of what it provisioned — the mapping between config and real resources. Terraform compares config → state → reality to decide changes. Without it, it wouldn't know what exists. Contains sensitive data — store securely, never in Git.

### ⭐ Why store state remotely?
Local state breaks team collaboration and risks loss. Git is worse — state often holds *secrets in plaintext*, so committing it is a leak. A remote backend (Azure Storage, S3, Terraform Cloud) gives a shared source of truth, encryption at rest, and state locking.

### ⭐ Two engineers run `apply` at once — what happens?
Without locking, they corrupt state or make conflicting changes (race). Fix: state locking — a remote backend acquires an exclusive lock (Azure Storage blob lease, S3 + DynamoDB); the second apply waits. The #1 reason to use a remote backend with locking.

### `terraform import`
Brings *existing* infra under Terraform management by adding it to state (without recreating) — for brownfield adoption. You still write the matching config.

### ⭐ Terraform module
A reusable, parameterized package of resources — a folder of `.tf` called with different inputs. Instead of copy-pasting VNet/AKS config across environments, write a module once and call with per-env variables. DRY, consistent. *(⚠️ Honest add: "I've worked within existing modules — variables, env configs — during our migration. Authoring complex modules from scratch is something I'm building.")*

### Variables, outputs, locals
Variables = inputs to a module. Outputs = values a module exposes (the VNet ID). Locals = computed/reused values within a config (DRY within a file).

### Manage multiple environments
Separate state files per env, shared modules with per-env variable files (`dev.tfvars`, `prod.tfvars`), or a directory structure. Workspaces exist but are discouraged for prod separation (shared config, easy to apply to the wrong one).

### count vs for_each
Both create multiple instances. `count` uses an integer index — remove a middle item and everything after re-indexes/recreates. `for_each` uses a map/set with stable keys — add/remove one without disturbing others. Prefer `for_each` when items change.

### ⭐ Drift — detect and handle
Drift = real infra differs from state, usually from a manual console change. `terraform plan` detects it (refreshes reality, shows the diff). Handle: `apply` to revert reality to config, or update the config to match. Prevent: all changes through Terraform, no manual edits.

### Dependencies between resources
Mostly implicit — if B references A's attribute, Terraform infers A first and builds a dependency graph. `depends_on` for explicit dependencies with no reference. Creates/destroys in order, parallelizing where it can.

### `lifecycle` block
`create_before_destroy` (avoid downtime), `prevent_destroy` (guard a prod DB), `ignore_changes` (ignore drift on attributes managed elsewhere).

### In-place update vs forced replacement
Some changes apply in-place (a tag), others destroy + recreate (immutable properties). `plan` shows `~` (update) vs `-/+` (replace). Danger: a small change can replace a critical resource — read the plan carefully.

### Provisioners — why discouraged
`local-exec`/`remote-exec` run scripts during apply. A last resort — not declarative, don't track state, break idempotency (a failed provisioner taints the resource). Prefer cloud-init, Ansible, or baked images.

### ⭐ Secrets in Terraform
Never hardcode. Mark variables `sensitive = true`. Pull from a secret manager (Key Vault, Vault) at apply via data sources. State contains secrets in plaintext, so the backend must be encrypted and access-controlled.

### data source vs resource
`resource` creates/manages infra. `data` reads existing info (an existing VNet, a Key Vault secret) without managing it.

### `.terraform.lock.hcl`
The dependency lock file — pins provider versions for reproducible runs across the team and CI. Commit it.

### Scenario: manual console change — next `apply`?
`plan` refreshes reality, detects drift, wants to revert the manual change. `apply` overwrites it. Manual changes get clobbered — all changes through Terraform. If intentional, update the config (or `ignore_changes`) first.

### Scenario: plan wants to destroy + recreate the prod DB (you only changed a tag)
Stop — don't apply blindly. Read the plan: `-/+` = replacement. Find why a tag triggers replacement. Use `prevent_destroy` as a guardrail and investigate first. Never apply a plan that destroys prod data without understanding it.

### Scenario: state out of sync with reality
Exists in cloud but not state → `terraform import`. In state but deleted in cloud → `plan` recreates it (or `terraform state rm` to forget). `terraform state` subcommands (`mv`, `rm`, `list`) for surgery.

### Scenario: apply failed halfway through
Some resources created (in state), others not — state reflects what succeeded. Investigate the error (quota, permission, dependency), fix, re-run — Terraform is idempotent, picks up where it left off. A resource that failed mid-creation may be tainted and recreated.

---

# 11. CLOUD — AZURE

### Azure compute options
VMs (full control), VM Scale Sets (auto-scaling identical VMs), App Service (managed PaaS web apps), Functions (serverless event-driven), AKS (managed Kubernetes), Container Instances (single containers, no orchestration).

### ⭐ What is AKS and what does Azure manage vs you?
Managed Kubernetes. **Azure manages the control plane** (API server, etcd, scheduler, controller-manager) — free, invisible. **You manage** worker node pools, workloads, config. You get managed upgrades, scaling, and integration with Azure AD, ACR, Azure Monitor, networking.

### Node pools
Groups of worker nodes with the same config. System node pool runs critical system pods (CoreDNS, metrics-server); user node pools run workloads. Multiple user pools with different VM sizes + taints/nodeSelectors for placement. Each scales independently.

### ⭐ Autoscaling in AKS — the two layers
HPA scales *pods* (CPU/memory/custom). Cluster Autoscaler scales *nodes* — adds nodes when pods can't schedule, removes underutilized ones. They work together: HPA adds pods → no room → Cluster Autoscaler adds nodes. KEDA = event-driven pod scaling (queue depth).

### ⭐ How AKS pods get Azure credentials securely
**Workload Identity** — federates a K8s ServiceAccount with an Azure AD identity, so pods get short-lived tokens with *no stored secrets*. (Older: pod-managed identities, deprecated.)

### ⭐ Azure CNI vs kubenet
Azure CNI = every pod gets a real VNet IP — directly routable, better performance, VNet integration, but consumes lots of IP space (nodes pre-allocate blocks → can exhaust the subnet). kubenet = pods get IPs from an overlay range, NAT through the node — conserves IPs but adds a hop, routing limits. CNI when you have IP space; kubenet to conserve.

### AKS upgrades — safely
Azure manages the control-plane upgrade. For nodes, a rolling upgrade — cordon + drain (respecting PDBs), upgrade, bring back, next. Best practice: control plane first, then node pools; PDBs + multiple replicas so draining doesn't cause outages; test in non-prod.

### ⭐ VNet, subnet, NSG
VNet = isolated private network. Subnet = a segment (separate tiers). NSG = stateful firewall rules on subnets/NICs by source/dest/port. Stateful = allowing inbound auto-allows return traffic. NSG misconfig is a common "can't reach the service" cause.

### Public vs private subnet
Public = has a route to the internet gateway (directly reachable / can reach the internet). Private = no direct route, reaches out via NAT gateway (outbound only). DBs/app servers private; load balancers/bastion public.

### NAT gateway
Lets private-subnet resources make *outbound* connections without being reachable *inbound*. Source NAT — translates private→public for outbound, tracks the connection so responses return.

### NSG / security group — stateful or stateless?
NSGs (Azure) and security groups (AWS) are **stateful** — allow inbound, return traffic is auto-allowed. Network ACLs (AWS) are stateless — must allow both directions explicitly.

### ⭐ Azure Load Balancer vs App Gateway vs Front Door vs Traffic Manager
Azure LB = L4, regional, raw TCP/UDP. App Gateway = L7, regional, HTTP routing + WAF + TLS termination. Front Door = L7, *global*, edge routing across regions + CDN + global failover + WAF. Traffic Manager = DNS-based global routing. Pick by layer (L4/L7) and scope (regional/global).

### How LBs avoid unhealthy instances
Health checks (probes) — periodically ping a configured endpoint (`/healthz`). Fail the threshold → removed from rotation until it passes again. Automatic failover at the LB layer.

### Traffic from the internet to a pod in AKS
Internet → Azure LB (or App Gateway via AGIC) → Ingress controller pods → Service (ClusterIP) → pod. One LB for the ingress controller; the ingress does L7 host/path routing.

### Private Endpoint vs Service Endpoint
Both secure access to Azure PaaS (SQL, Storage) from your VNet. Private Endpoint = the service gets a private IP *inside* your VNet (traffic never touches the internet — most secure). Service Endpoint = extends VNet identity over the Azure backbone but the service keeps a public IP. Private Endpoint is the modern choice.

### ExpressRoute vs VPN Gateway
Both connect on-prem to Azure. VPN Gateway = encrypted tunnel over the public internet (cheaper, variable). ExpressRoute = private dedicated connection via a provider (bypasses the internet, consistent, costlier, enterprise).

### ⭐ Azure AD (Entra ID) + RBAC
Azure AD = the identity provider (users, groups, service principals, auth — OIDC, SAML, MFA). Azure RBAC = what an identity can do via role assignments (Owner, Contributor, Reader, custom) scoped to subscription/RG/resource. Identity (who) + RBAC (what).

### ⭐ Service Principal vs Managed Identity
Both are non-human identities. Service Principal = you create and rotate the credentials. Managed Identity = Azure manages the credentials, no secrets to store/rotate — preferred. System-assigned (tied to one resource) vs user-assigned (standalone, shareable). Maps to AKS Workload Identity.

### ⭐ Azure Key Vault + AKS
Managed service for secrets/keys/certs, RBAC-controlled, audited. AKS pods access it via the **Secrets Store CSI driver** (mounts Key Vault secrets as files) + Workload Identity (no stored creds).

### Least privilege in Azure
Built-in or custom roles with minimal permissions, scope as narrowly as possible (resource > RG > subscription), Managed Identities over stored creds, PIM for just-in-time elevation, audit with Azure AD logs.

### Azure Storage types
Within a Storage Account: Blob (object — files, backups, logs), Files (SMB/NFS — can be RWX for AKS), Queue (messages), Table (NoSQL key-value). Plus Managed Disks (block storage for VM/AKS).

### ⭐ AKS PV: Azure Disk vs Azure Files
Azure Disk = block, **ReadWriteOnce** (one node) — single-pod stateful (a database). Azure Files = SMB/NFS, **ReadWriteMany** (multiple nodes) — shared access. RWO is why a Deployment sharing one disk across nodes fails — Files for shared, Disk for single-pod.

### Azure SQL vs Cosmos DB vs managed Postgres/MySQL
Azure SQL = managed SQL Server (relational, ACID). Cosmos DB = global NoSQL, multi-model, low-latency, tunable consistency. Managed Postgres/MySQL = managed open-source relational. By data model, scale/distribution, existing stack.

### Event Hubs vs Service Bus
Service Bus = enterprise messaging (queues, topics, ordering, transactions) — reliable command/message delivery. Event Hubs = high-throughput event streaming (millions/sec, like Kafka) — telemetry, logs, pipelines.

### ⭐ Azure Monitor vs App Insights vs Log Analytics
Log Analytics = the data store + query engine (KQL). App Insights = the APM layer (request tracing, dependencies, exceptions); its data lands in Log Analytics. Azure Monitor = the umbrella tying metrics, logs, alerts, dashboards together.

### ⭐ What is KQL?
Kusto Query Language — the query language for Log Analytics and App Insights (e.g., `requests | where resultCode >= 500 | summarize count() by bin(timestamp, 5m)`). The Azure equivalent of PromQL/LogQL.

### Alerting in Azure Monitor
Alert rules on metrics or log (KQL) queries, with conditions/thresholds, routed through **Action Groups** (email, SMS, webhook, PagerDuty, runbook). Metric alerts (fast) or log alerts (KQL-based, flexible).

### Azure monitoring + AKS
Container Insights collects AKS metrics/logs into Log Analytics. You can also run managed Azure Monitor for Prometheus + Azure Managed Grafana — closer to your actual stack.

### ARM vs Bicep vs Terraform
ARM = Azure-native JSON IaC (verbose). Bicep = a cleaner DSL compiling to ARM (Azure-native, readable). Terraform = cloud-agnostic, multi-cloud, big ecosystem. Azure-only → Bicep gaining traction; multi-cloud/existing → Terraform.

### ⭐ AZ vs Region — designing for failure
Region = a geographic area. AZ = a physically separate datacenter within a region (independent power/cooling/network). Across AZs survives a single-datacenter failure (low latency); across regions survives a whole-region outage (DR, higher latency/complexity). For AKS: multi-zone node pools + topologySpreadConstraints.

### ⭐ Design an AKS workload to survive AZ failure
Multi-zone node pool, topologySpreadConstraints to spread replicas across zones, enough replicas with headroom to absorb a lost zone, a zone-redundant managed DB, PDBs, health probes + autoscaling. Losing a zone degrades capacity, not availability.

### RTO and RPO
RTO = max acceptable downtime. RPO = max acceptable data loss (in time). Tight RPO → synchronous replication / frequent backups; tight RTO → warm/hot standby over cold restore. In Azure: geo-redundant storage, Azure SQL active geo-replication, multi-region AKS + Front Door/Traffic Manager failover. Looser = cheaper.

### Azure HA out of the box
Availability Zones, zone-redundant services (SQL, Storage, LB), VM Scale Sets with health-based replacement, higher SLAs for multi-AZ, Front Door/Traffic Manager for global failover, managed services with built-in replication.

---

# 12. OBSERVABILITY (Metrics, Logs, Traces, Prometheus, PromQL)

### ⭐ Observability vs monitoring
Monitoring tells you *whether* a system works (predefined dashboards/alerts on known failure modes). Observability is asking *arbitrary new questions* about internal state from external outputs, including unpredicted failures. Monitoring = known-unknowns; observability = unknown-unknowns.

### ⭐ The three pillars
Metrics = numeric time-series, cheap, dashboards/alerts/trends. Logs = discrete event records, detail. Traces = follow a request across services, where time was spent. They correlate: a metric says something's wrong, a trace says which hop, a log says exactly what.

### When to use each pillar
Metrics: alerting, dashboards, trends, SLOs (always-on, cheap). Logs: detailed debugging, audit, exact errors (higher cost, query when investigating). Traces: latency analysis, bottlenecks in distributed flows, dependency mapping. The skill is using them together.

### ⭐ How the three pillars correlate in an incident
A metric alert fires (error spike) → pivot to a trace of a failing request (which span/hop failed, where latency went) → jump to logs for that trace ID (the actual error). Metrics = where roughly; traces = which hop; logs = exactly what. Exemplars and shared trace IDs make the jump seamless.

### Exemplar
A sample trace ID attached to a metric data point — click from a latency spike directly to an example slow trace. The bridge from metrics to traces.

### RED, USE, Four Golden Signals
RED (request-driven services): Rate, Errors, Duration. USE (resources): Utilization, Saturation, Errors. Four Golden Signals (Google): Latency, Traffic, Errors, Saturation. RED ≈ the request-facing subset of golden signals; USE is for resources.

### ⭐ Why Prometheus pull model + consequence
Scrapes targets, not pushed data. Benefits: a free up/down signal (`up` metric), centralized scrape control, targets don't need to know where to push. Consequence: short-lived jobs may finish before being scraped → Pushgateway for those.

### ⭐ Counter vs Gauge vs Histogram vs Summary
Counter = only increases (total requests), use `rate()`. Gauge = up and down (memory, queue depth). Histogram = buckets observations for server-side percentiles. Summary = client-side quantiles, can't aggregate across instances. For latency, prefer Histogram (aggregatable).

### ⭐ rate() vs irate() vs increase()
`rate()` = per-second average over the window (smoothed — alerting/graphing). `irate()` = instantaneous from the last two points (spiky — fast graphs, never alert on it). `increase()` = total increase over the window (= rate × seconds — "how many X in the last hour"). All only on counters.

### ⭐ Why rate() before sum(), never sum() before rate()
Counters reset on pod restart. `rate()` is reset-aware. `sum()` raw counters first → a restart makes the sum plummet → `rate()` on that = garbage. Always `sum(rate(metric[5m]))`, never `rate(sum(metric)[5m])`. Aggregate with `sum(rate(...)) by (label)`.

### ⭐ p99 latency + the percentile catch
`histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`. You **cannot average percentiles** across instances — aggregate the raw buckets first (sum by `le`), then compute the quantile once.

### ⭐ Cardinality — the #1 Prometheus killer
Number of unique time-series, driven by label-value combinations. High-cardinality labels (user ID, request ID, full URL, email) create millions of series → exhausts memory → crashes Prometheus. Control: never label with unbounded values, relabel to drop dangerous labels, monitor `prometheus_tsdb_head_series`, push high-cardinality data to logs/traces.

### Recording rules
Pre-compute an expensive/frequent query at scrape interval, store as a new metric. For slow dashboard queries, SLI computations used by multiple alerts, frequently-evaluated queries. Naming: `level:metric:operation` (e.g., `sli:checkout_availability:ratio_rate5m`).

### How Prometheus discovers what to scrape in K8s
ServiceMonitor (and PodMonitor) CRDs — declare which services/pods, interval, path; Prometheus auto-discovers matching targets. Plus kubernetes_sd service discovery. *(You wrote ServiceMonitors in Athena.)*

### The `up` metric + absent()
`up` is auto-generated per scrape target: 1 = scrape succeeded, 0 = failed. The pull model's free liveness signal. Alert `up == 0 for 5m`. `absent(up{job="x"})` fires when the target *disappears entirely* (no series), which `up == 0` can't catch.

### Four golden signals → PromQL
Latency: `histogram_quantile(0.99, ...)`. Traffic: `sum(rate(http_requests_total[5m]))`. Errors: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`. Saturation: `container_memory_working_set_bytes / kube_pod_container_resource_limits`.

### ⭐ Loki vs ELK
Loki indexes only metadata labels (namespace, pod), not content — ~10× cheaper, simpler, on the assumption you query by context you already know. ELK indexes full text — powerful for ad-hoc search/analytics but expensive at scale. Loki for cost-efficient operational logs; ELK for deep full-text search / SIEM.

### LogQL vs PromQL
LogQL: a stream selector by labels (`{app="checkout"}`, uses the index, fast) + a filter pipeline (`|= "error" | json | status="500"`, scans matched streams). Can generate metrics from logs (`rate({app="x"} |= "error" [5m])`). Keep label cardinality low — each combo is a stream.

### Good structured log
JSON, consistent fields: timestamp, level, service, **trace ID** (for correlation), clear message, context — but no secrets/PII. Query by field instead of regex-scraping.

### ⭐ Distributed tracing — why essential for microservices
A trace follows one request across services, each step a **span** with timing — shows the full path and where time went. Metrics/logs alone can't tell you *which hop* failed/slowed; only traces stitch it together.

### Span vs trace
Trace = the whole journey of one request. Span = one unit of work within it (a service's processing, a downstream call) with start time, duration, parent-child links. Spans nest into the call tree.

### ⭐ What is OpenTelemetry and why it matters
The vendor-neutral instrumentation standard — SDKs for metrics/logs/traces + a Collector. Decouples instrumentation from backend: instrument once, send to Tempo/Jaeger/Datadog/Honeycomb without changing app code. Killed SDK-level vendor lock-in; every vendor accepts OTLP.

### Tempo vs Jaeger
Both store traces. Tempo = Loki's philosophy (store cheaply, retrieve by ID, no attribute indexing) — cheaper, Grafana-integrated. Jaeger = indexes spans for richer search — more flexible, costlier.

### Sampling
Tracing every request is expensive. Head-based = decide at the start (keep 10% — simple, may miss rare errors). Tail-based = decide after the trace completes (keep all errors + slow, sample the rest — smarter, more infra). Tail-based for keeping errors is the senior answer.

### Prometheus histogram — what it gives you
Buckets of observations (durations in <0.1s, <0.5s, <1s). `histogram_quantile()` computes percentiles from buckets. Percentiles are estimated from bucket boundaries; can't average percentiles — aggregate buckets first, then compute.

---

# 13. RELIABILITY — SLI/SLO/Error Budgets, Resilience, Incidents

### ⭐ SLI vs SLO vs SLA
SLI = the measured number (% successful requests, % under 300ms). SLO = the internal target (99.5% over 30 days). SLA = the external contract with penalties (usually looser than the SLO for a safety margin). SLI = what you measure, SLO = your goal, SLA = the promise to customers.

### ⭐ Error budget + how it changes how teams work
Error budget = 1 − SLO. For 99.5%, that's 0.5% (~216 min/month). A shared currency: budget remaining → ship features fast; budget burned → freeze risky releases, focus on reliability. Turns "how reliable?" from an argument into a data-driven decision.

### ⭐ What's a good SLI?
Reflects the *user's experience*, not internal mechanics. Base on user-journey outcomes: availability (% successful), latency (% under a threshold), correctness, freshness. Measure closest to the user (LB/gateway). Avoid SLIs the user doesn't care about (CPU — a cause, not a symptom).

### ⭐ Multi-window, multi-burn-rate alerting
Alert on how fast you burn the error budget. Burn rate = error rate ÷ (1 − SLO). At 14.4× burn, a 30-day budget exhausts in ~2 days (page now). Multi-window = a short *and* a long window must both breach — short confirms it's happening now, long confirms it's sustained, eliminating flapping. Tiers (fast page, slow ticket) match urgency. Better than raw thresholds because it aligns alerts with real reliability risk.

### Cause vs symptom alerting
Symptom-based fires on user-facing impact ("error rate high," "latency exceeds SLO") — actionable, user-relevant. Cause-based fires on internal conditions ("CPU 90%") which may not affect users. Page on symptoms, diagnose with causes.

### ⭐ Reduce alert fatigue
Every alert must require a human action (else it's a dashboard metric). Add `for` durations. Use burn-rate over raw thresholds. Add inhibition rules (suppress symptoms when a root-cause alert fires). Group related alerts. Remove non-actionable ones. *(Real story: 8-10 weekend pages → 2-3.)*

### Alertmanager
Prometheus fires alerts; Alertmanager handles them — routing (by severity/team), grouping (bundle related), inhibition (suppress when a higher-priority fires), silencing (maintenance), deduplication. Delivers to receivers (PagerDuty, Slack, webhook).

### Resilience patterns (prevent cascading failures)
Timeouts (don't wait forever), retries with backoff + jitter (recover from transient without thundering-herd), circuit breakers (stop calling a failing dependency, fail fast), bulkheads (isolate resource pools), load shedding (drop excess load to protect core), graceful degradation (reduced functionality over failing). Contain blast radius.

### Chaos engineering + observability
Deliberately injecting controlled failures (kill a pod, inject latency, simulate a zone outage) to validate resilience and find unknown failure modes — with a hypothesis tied to an SLO, blast-radius limits, abort conditions. Observability is essential — you can't run chaos safely without the signal to see impact and confirm the hypothesis. *(⚠️ Honest: "I understand the principles and I'd run controlled experiments in my Athena cluster.")*

### MTTD vs MTTR vs MTBF
MTTD = mean time to detect (observability drives this down). MTTR = mean time to restore/repair. MTBF = mean time between failures. Good observability improves MTTD and MTTR.

### What is toil?
Manual, repetitive, automatable, reactive work that scales with service size and adds no lasting value. SRE caps and automates it. *(Your disk-cleanup/image-update scripts are toil reduction.)*

### Runbook
A documented procedure for a known scenario — steps to diagnose and remediate — so any on-call engineer responds consistently without tribal knowledge.

### ⭐ Good postmortem
Blameless (systems and contributing factors, not individuals), clear timeline, impact (users/duration/budget consumed), root cause(s), what went well/poorly, concrete action items with owners and deadlines — then verify they're done. Learn and prevent, not blame.

### ⭐ Fix-forward vs rollback
Users impacted + cause unclear → roll back first, stop the bleeding, investigate after (the budget burns while you debug). Minor issue + known quick fix → fix forward. For bad deploys, rollback is usually the right first move.

### On-call process
Stabilize first (any immediate mitigation/rollback) → gather signal (dashboards, what changed, logs, traces) → follow the runbook → resolve if in scope, escalate with full context if not → contribute to the postmortem and update the runbook. Systematic beats fast; escalating with good context is a strength.

---

# 14. SYSTEM DESIGN / SCENARIO FRAMEWORKS

*Interviewers grade your thinking process — narrate your reasoning, don't recite a solution.*

### ⭐ "A service is returning 5xx errors. Walk me through debugging."
1. **Scope impact first** — how many users, all requests or a %, when did it start, *what changed* (deploy, config, traffic spike). The "what changed" question solves most incidents.
2. **Check dashboards** — golden signals; spike or ramp; all endpoints or one.
3. **Localize the layer** — LB/ingress, the service, or a downstream dependency; check `up` and per-service error rates.
4. **Use traces** — which span is erroring; is the time in the service or a downstream call (us or a dependency?).
5. **Read logs** — from the trace, jump to logs for that trace ID; the actual error.
6. **Mitigate then fix** — correlates with a deploy → roll back first; a dependency → check its health, circuit-break.
7. **Root cause + postmortem after.**
*Emphasize:* "First question is always *what changed*. And I roll back before deep-debugging if users are impacted — the budget burns while I investigate."

### "Design a highly available web service."
1. Redundancy at every layer (replicas across AZs, no SPOF).
2. Load balancing with health checks.
3. Stateless app tier (push state to a shared store/cache).
4. Data tier HA (multi-AZ managed DB, read replicas).
5. Autoscaling with failover headroom.
6. Graceful degradation (timeouts, retries, circuit breakers).
7. Observability + SLOs (define them upfront).
8. DR (backups, secondary region with RPO/RTO).
*Emphasize:* "HA = removing single points of failure at every layer — compute, network, data. And I'd define an SLO so 'highly available' has a number."

### "Design a monitoring/observability system for microservices." *(your strength + Athena)*
1. Three pillars — Prometheus (metrics), Loki (logs), Tempo via OpenTelemetry (traces) — unified in Grafana.
2. Standardize on OpenTelemetry (vendor-neutral, cross-language).
3. Golden signals per service; RED for request-driven.
4. SLI/SLO per critical service, track error budgets.
5. Multi-window burn-rate alerts; route by severity through Alertmanager; inhibition.
6. Correlation — exemplars + shared trace IDs for metric → trace → log.
7. Cost/scale — control cardinality, object storage for retention, downsampling.
*Emphasize:* "I built exactly this in Athena — and the part most people skip is alerting on SLOs with burn rate, not raw thresholds, plus controlling cardinality."

### "Latency is high but error rate is normal. Investigate."
1. Confirm/quantify — which percentile (p50 or just p99 tail?), which endpoints, started when.
2. Rule out load — traffic spike? saturation (CPU throttling, memory, connection pool)?
3. Use traces — where is the time, in our code or a downstream call?
4. Downstream → check that dependency (slow query, lock contention, undersized pool).
5. In-service → CPU throttling, GC pauses, a slow code path.
6. Check saturation — `container_cpu_cfs_throttled`, memory, connection pool metrics.
*Emphasize:* "Latency-without-errors usually means saturation or a slow dependency, not a bug. Traces tell me in seconds whether it's our code or downstream."

### "An alert pages repeatedly but self-resolves each time."
1. Acknowledge the pattern; don't blindly silence.
2. Assess quality — if it self-resolves and needs no action, it's a bad alert (a dashboard metric).
3. Fix the alert — add `for`, switch to burn-rate/sustained, raise the threshold.
4. But investigate the underlying flapping — is something actually degrading and recovering (a pod OOMing every few hours)? Noisy *and* hiding a real problem.
5. Document, improve the runbook.
*Emphasize:* "Every alert must require a human action. A self-resolving page is either a tuning problem or masking a real recurring issue — I check both. Did exactly this: 8-10 → 2-3 pages."

---

# 15. BEHAVIORAL (STAR)

*STAR = Situation, Task, Action, Result. Fill in your specifics; keep it honest.*

### "A difficult production incident you handled." *(technical depth + composure)*
The OOMKilled service — pods dying every few hours, error spikes → I was on-call, needed to find why memory grew → Grafana memory slope = leak not load → exit 137 → log correlation to an endpoint → handed L3 a full diagnostic trail → fixed in a day, memory flattened. *Reflection:* a complete diagnostic trail cut their investigation time in half.

### "A time you improved a process / reduced toil." *(ownership)*
On-call paged 8-10x/weekend, mostly non-actionable → I took ownership of the alert rules → exported 90 days, categorized by actionability, 12 rules = 70% of noise, fixed thresholds/`for`/inhibition → pages dropped to 2-3, almost all actionable. *Reflection:* every alert should require a human action.

### "A time you disagreed with a teammate/senior." *(communication)*
A proposed alert threshold/config change I thought was wrong → needed to raise it without overstepping → brought data (historical pattern/impact), proposed an alternative, discussed rather than insisted → reached a better decision together (or disagreed-and-committed). *Reflection:* bring data, not opinions.

### "A time you failed / made a mistake." *(humility — pick a real, recoverable one)*
A change that didn't go as planned → what you were trying to do → caught it, owned it, fixed it → recovered, no lasting damage. *Reflection:* the specific lesson + what you now do differently.

### "A time you learned something new quickly." *(learning agility — Athena)*
Got SRE interview feedback that I lacked hands-on observability building → needed to close the gap fast → built Athena (full LGTM stack, OTel, SLO alerting) in ~2 weeks, learning by doing → now have hands-on depth + a portfolio. *Reflection:* the fastest way I learn is building the real thing and breaking it.

### "A time you took ownership outside your role." *(initiative)*
The alert cleanup, or building the health-check/disk-cleanup automation that wasn't assigned but reduced team toil.

### "A high-pressure situation." *(composure)*
A P1 during on-call — CrashLoopBackOff during a release → stayed systematic: checked events, found liveness failures, identified slow startup vs probe timing, recommended the fix → resolved, documented in a runbook. *Reflection:* under pressure, systematic beats fast; escalate rather than thrash alone.

### "Why are you leaving Cognizant?" *(motivation)*
Grown a lot, but the role is mostly operating systems other teams designed. I want the engineering/building side of reliability — designing observability, building platforms, writing automation. Building Athena confirmed it. Looking for a product-focused team where that's the core.

### "Why this company / role?" *(genuine interest — research first)*
Tie their stack/domain to your strengths: large observability practice / healthcare domain (you have it) / strong SRE culture — exactly the direction you're building toward.

### "A time you handled an on-call escalation well." *(on-call maturity)*
An incident where you triaged correctly → gathered signal, attempted ops-level fixes, escalated to L3 with full context when it was an app bug → faster resolution because of the context. *Reflection:* knowing when to escalate with good context is a strength.

---

# 16. BASH SCRIPTING

### Shebang line
`#!/bin/bash` (or `#!/usr/bin/env bash`) tells the system which interpreter runs the script. Without it, it runs in whatever shell invoked it; Bash-specific syntax breaks under plain `sh`.

### Single vs double quotes
Single `'...'` = literal, no expansion. Double `"..."` = allows `$var`/`$(cmd)` but preserves spaces. Use double quotes around variables; single for literal strings.

### ⭐ Why quote variables — `"$var"`
Unquoted, a variable with spaces or empty causes word-splitting/globbing. `rm $file` where `file="my file.txt"` deletes two files. Empty `[ $x = "y" ]` → `[ = "y" ]`, a syntax error. The #1 Bash bug.

### Command-line arguments
`$1`, `$2` (positional), `$0` (script name), `$#` (count), `$@` (all args), `$*` (all as one string).

### `>` vs `>>` vs `2>`
`>` overwrites stdout, `>>` appends, `2>` redirects stderr, `&>` / `2>&1` both. `2>&1` = send stderr where stdout goes (order matters).

### ⭐ `set -euo pipefail` — why in every production script
`-e` exit on any error, `-u` error on undefined variables (catches typos), `-o pipefail` a pipeline fails if any command fails. Without these, Bash silently continues after errors — dangerous (e.g., a failed `cd` then `rm -rf *` in the wrong place). The #1 "do you write safe scripts" signal.

### `[ ]` vs `[[ ]]`
`[ ]` = POSIX test (works everywhere, needs careful quoting). `[[ ]]` = a Bash keyword (safer, no word-splitting, supports `&&`/`||`, regex `=~`). Prefer `[[ ]]`.

### Check if a command succeeded
`$?` (exit code — 0 success, non-zero failure), or `if command; then ...`. `&&` runs next on success, `||` on failure.

### Loop over files/lines safely
Files: `for f in *.log; do ...; done`. Lines: `while IFS= read -r line; do ...; done < file` (`IFS=` preserves whitespace, `-r` prevents backslash mangling). The naive `for line in $(cat file)` breaks on spaces.

### Command substitution
`result=$(command)`. Use `$(...)` not backticks — nests cleanly, more readable.

### Default values
`${var:-default}` (use default if unset/empty), `${var:=default}` (also assign), `${var:?error}` (exit if unset). E.g., `NAMESPACE="${1:-default}"`.

### ⭐ Error handling + cleanup (trap)
`trap 'rm -f "$tmpfile"' EXIT` runs cleanup on any exit. `trap 'echo "failed at $LINENO"' ERR` catches errors. With `set -e`, ensures temp files/locks are cleaned up even when the script dies.

### ⭐ Prevent concurrent runs (flock)
`exec 200>/var/lock/myscript.lock; flock -n 200 || exit 1` — exclusive lock on an FD; if another instance holds it, `-n` exits immediately. Critical for cron jobs so a slow run doesn't overlap the next.

### Debug a Bash script
`bash -x script.sh` (or `set -x`) prints each command with expanded variables. `set -v` prints lines as read. Combine with `set -euo pipefail`.

### `./script.sh` vs `bash script.sh` vs `source script.sh`
First two run in a *new subshell* — variables don't affect your current shell. `source` (`. script.sh`) runs in the *current shell* — can change your environment (sourcing config/env files).

### Process large files / when NOT to use Bash
Streaming tools — `grep`, `awk`, `sed` — process line-by-line without loading into memory; `awk` for columns/aggregation. Switch to Python for structured data (JSON), complex logic, maintainability.

### `xargs`
Builds and runs commands from stdin (`find . -name "*.log" | xargs rm`), batching arguments efficiently. Use `xargs -0` with `find -print0` for filenames with spaces.

### Config/secrets in a script
Source a config file for non-secrets. For secrets, never hardcode — read from env vars at runtime or a secret manager. Don't echo secrets (they end up in logs/`set -x`); `set +x` around sensitive sections.

---

# 17. PYTHON (for Ops)

### List vs tuple vs dict vs set
List `[]` = ordered, mutable, duplicates. Tuple `()` = ordered, immutable (fixed records, dict keys). Dict `{}` = key-value, fast lookup. Set = unordered, unique, fast membership. "Count unique errors" → set/dict; "fast lookup" → dict.

### Read a file safely
`with open("file") as f: for line in f: ...`. The `with` auto-closes even on error; iterating line-by-line streams instead of loading all into memory. `f.read()` on a huge log is a memory mistake.

### `==` vs `is`
`==` compares values. `is` compares identity (same object). Use `==` for values; `is` only for `None`/`True`/`False`.

### ⭐ Error handling
`try/except` with *specific* exceptions, not bare `except:` (hides bugs, catches KeyboardInterrupt). `finally:` for always-run cleanup, `else:` for no-exception code.

### ⭐ Parse a log file + count error types
```python
from collections import Counter
errors = Counter()
with open("app.log") as f:
    for line in f:
        if "ERROR" in line:
            errors[extract_type(line)] += 1
for err, count in errors.most_common(10):
    print(count, err)
```
The canonical ops Python task — `Counter.most_common()`.

### Parse JSON
`import json`. `json.loads(string)` → dict; `json.load(file)`; `json.dumps(obj)` → string. Why you reach for Python over Bash for structured data.

### ⭐ HTTP API call
```python
import requests
resp = requests.get(url, headers={"Authorization": f"Bearer {token}"}, timeout=10)
resp.raise_for_status()      # raises on 4xx/5xx
data = resp.json()
```
Always set a `timeout` (a hung server hangs your script forever) and check the status.

### Run a shell command from Python
```python
result = subprocess.run(["kubectl","get","pods"], capture_output=True, text=True, check=True, timeout=30)
```
Args as a *list* (no shell injection), `check=True` (raise on failure), `text=True`, a `timeout`.

### List comprehension
`[x*2 for x in items if x > 0]`. Readable for simple transforms/filters; use a regular loop for complex logic.

### append vs extend + mutable default trap
`append(x)` adds one item; `extend([a,b])` adds each. Trap: `def f(items=[])` — the default list is created once and shared across calls. Use `def f(items=None): items = items or []`.

### List vs generator (large data)
A list holds everything in memory; a generator (`yield`, `(x for x in ...)`) produces items lazily, near-zero memory. For huge logs / streaming, a generator avoids OOM.

### ⭐ Idempotent / safe-to-rerun automation
Check state before acting (resource exists? skip). Wrap each operation in try/except so one failure doesn't abort everything. Log what was done. Retries with backoff for external calls. Make operations reversible or checkpoint so a re-run resumes, not duplicates.

### Retries with backoff
```python
for attempt in range(max_retries):
    try:
        return call_api()
    except TransientError:
        if attempt == max_retries - 1: raise
        time.sleep(2 ** attempt)   # exponential backoff
```
Add jitter in production to avoid thundering-herd.

### Context managers
The `with` statement manages setup/teardown automatically. Write your own with `@contextmanager`. Ensures cleanup (close connections, release locks) even on error — the Python equivalent of Bash's `trap`.

### Secrets/config in Python
Never hardcode. `os.environ["TOKEN"]` (required, fail-fast) or `os.getenv("KEY", default)` (optional). Keep config separate; don't log secrets.

### When NOT to use Python (use Bash)
Simple command orchestration / file ops / glue → Bash is lighter. Python for structured data, complex logic, error handling, API calls, maintainability. Judgment about the right tool is a senior signal.

---

# 18. ONE-PAGE CHEAT SHEET

**Linux** — Load avg = runnable + D-state (I/O); high load + low CPU = I/O wait. `df` full but `du` not = deleted file held open (`lsof +L1`). Port: `ss -tlnp`. OOM: `dmesg | grep -i oom`, container OOM = exit 137. RSS = real RAM (double-counts shared), VSZ = virtual (misleading), PSS = fair share. Hung: `strace -p`, `/proc/<pid>/stack`.

**Networking** — Refused = nothing listening (RST); timeout = dropped (firewall/route). TLS: hello → cert → verify → key exchange → encrypted; 1.3 = 1-RTT. L4 = IP/port; L7 = HTTP host/path. 502 = bad upstream response; 503 = no capacity; 504 = upstream too slow. conntrack full → dropped packets (`dmesg`); raise `nf_conntrack_max`. ndots:5 → short names try search domains first.

**Kubernetes** — Service = virtual ClusterIP; kube-proxy iptables/IPVS DNAT to Pod IP; conntrack tracks return. iptables O(n), IPVS O(1). Probes: readiness=traffic, liveness=restart, startup=slow-start grace. `kubectl apply`: API server (authn/authz/admission/validate) → etcd → Deployment ctrl → ReplicaSet ctrl → scheduler → kubelet (CRI pull, CNI IP) → endpoints → kube-proxy. QoS eviction: BestEffort → Burstable → Guaranteed. CrashLoop: `describe` → `logs --previous`. etcd lose quorum = read-only, existing pods keep running.

**Docker** — Image = template (layers); container = instance + writable layer. Multi-stage = build heavy, ship slim. Dockerfile order: rarely-changing first (cache). Don't run as root; slim/distroless; `.dockerignore`. EXPOSE = docs only; `-p` actually publishes.

**CI/CD & GitOps** — Pipeline: test → build → scan → push → deploy/sync. Secrets: encrypted store, OIDC for cloud (short-lived). Push vs pull: pull (GitOps/ArgoCD) keeps creds in-cluster, Git = source of truth, drift detection + selfHeal. Image tags: immutable (SHA/semver), never `latest` in prod. Helm = templating + values + release/rollback. Blue-green = instant switch; canary = gradual %.

**Terraform** — State = config↔real mapping; remote backend + locking prevents concurrent corruption. Modules = reusable per-env. Drift = real ≠ state; `plan` detects. Read the plan: `~` = update, `-/+` = replace. Secrets: `sensitive` vars, Key Vault, encrypt state.

**Azure** — AKS: Azure manages the control plane (free); you manage node pools + workloads. Log Analytics = store/KQL; App Insights = APM on top; Azure Monitor = umbrella. Workload Identity = short-lived AAD tokens, no stored secrets. Azure CNI = real VNet IPs (can exhaust); kubenet = overlay, conserves IPs. Disk = RWO (single-pod); Files = RWX (shared). AZ vs Region; multi-zone node pools + topologySpreadConstraints for AZ failure.

**Observability** — Metrics (dashboards/alerts), logs (detail), traces (cross-service latency) — correlate. Pull model → free `up` signal. Golden signals: latency, traffic, errors, saturation. RED: rate, errors, duration. `rate()` before `sum()` (reset-aware). p99 = `histogram_quantile(0.99, sum(rate(bucket[5m])) by (le))` — can't avg percentiles. Cardinality = #1 killer; never label with unbounded values. Loki indexes labels not content (cheap); ELK full-text (expensive). OpenTelemetry decouples instrumentation from backend.

**Reliability** — SLI = measured; SLO = target; SLA = contract w/ penalties. Error budget = 1 − SLO; spend on features when healthy, reliability when burning. Multi-window burn-rate: short (now) + long (sustained) both breach. Toil = manual/repetitive/automatable. Fix-forward vs rollback: users impacted + cause unclear → roll back first. Postmortem = blameless, timeline, root cause, action items with owners. Page on symptoms, diagnose with causes.

**Scripting** — Bash: `set -euo pipefail` = fail fast; quote `"$var"`; `trap` for cleanup; `flock` for locking; truncate (not delete) open logs. Python over Bash for JSON/logic/maintainability; `try/except` specific exceptions; `Counter.most_common()` for log analysis; always set a `timeout` on API calls; idempotent + retries with backoff for automation.

---

*End of bank. Drill by active recall — cover, say it out loud, check. The ⭐ and ⚠️ items are the highest-leverage. Recognition ≠ delivery: practice these spoken, not just read.*
