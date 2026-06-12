---

# PART 1 — GITHUB ACTIONS

## Basics

**⭐ Q: What is GitHub Actions and what are its core building blocks?**
A CI/CD platform built into GitHub. Building blocks: a **workflow** (a YAML file in `.github/workflows/` triggered by events), made of **jobs** (run on runners, parallel by default), made of **steps** (individual commands or actions). An **action** is a reusable unit of code (from the marketplace or your own). A **runner** is the machine that executes a job. *Why: foundational vocabulary; you need to read/write these.*

```yaml
name: build-and-deploy
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest        # the runner
    steps:
    - uses: actions/checkout@v4   # an action
    - name: Build image           # a step
      run: docker build -t app:${{ github.sha }} .
```

**⭐ Q: What's a runner — GitHub-hosted vs self-hosted?**
The machine that executes jobs. **GitHub-hosted** = GitHub provisions a fresh VM per job (clean, managed, but limited and runs on GitHub's infra). **Self-hosted** = your own machines (you manage them) — used when you need access to private networks (e.g., a private AKS cluster), custom hardware, or more control. Self-hosted runners must be secured carefully since they persist and can access your infra. *Why: tests practical knowledge; self-hosted security is a senior point.*

**Q: What are common triggers (`on:`)?**
`push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), `release`, `workflow_call` (reusable workflows). Filters by branch, path, tag. *Why: tests workflow design.*

## Intermediate

**⭐ Q: Where do secrets live in GitHub Actions and how does a job access them?**
In encrypted **GitHub Secrets** (repo, environment, or org level). The runner gets them injected as masked environment variables at runtime, scoped to the job, and masked in logs. For cloud auth, the modern best practice is **OIDC federation** — the runner requests a short-lived token from the cloud provider (Azure/AWS) instead of storing long-lived credentials as secrets. *Why: a guaranteed CI/CD security question; OIDC > stored secrets is the senior answer.*

**⭐ Q: Why is OIDC better than storing cloud credentials as secrets?**
Stored credentials (a service principal secret) are long-lived — if leaked, they're valid until rotated, and rotation is manual toil. OIDC issues a **short-lived token** per workflow run, scoped to specific permissions, with no secret stored anywhere. If a run is compromised, the token expires in minutes. It eliminates the "leaked long-lived credential" risk entirely. *Why: directly relevant to securing pipelines; a strong security signal.*

**Q: What are environments and protection rules?**
GitHub **Environments** (dev/staging/prod) can have protection rules: required reviewers (manual approval gate before deploying to prod), wait timers, and environment-scoped secrets. This is how you add an approval gate before production. *Why: ties to your resume's "approval gates"; tests deployment safety.*

**Q: How do you pass data between jobs and steps?**
Within a job, steps share the filesystem and can use step outputs. Between jobs, use `needs:` (job dependency) and job `outputs`, or **artifacts** (`actions/upload-artifact` / `download-artifact`) to pass files. Jobs are isolated by default (different runners), so you must explicitly pass data. *Why: tests workflow structure understanding.*

**Q: What's a matrix build?**
Run the same job across multiple configurations in parallel (e.g., test against Node 18, 20, 22, or multiple OSes) using a `strategy.matrix`. *Why: efficiency/testing question.*

## Advanced

**⭐ Q: What are the security risks in GitHub Actions and how do you mitigate them?**
- **Untrusted action code** — pin actions to a full commit SHA (not a moving tag like `@v4`) so a compromised tag can't inject malicious code.
- **`pull_request_target`** — runs with secrets access on PRs from forks; dangerous, can leak secrets. Avoid or handle carefully.
- **Script injection** — untrusted input (PR titles, branch names) interpolated into `run:` can execute arbitrary code; sanitize or use env vars instead of direct interpolation.
- **Over-privileged `GITHUB_TOKEN`** — set `permissions:` to least-privilege (default to read-only, grant write only where needed).
- **Self-hosted runner compromise** — don't use self-hosted runners on public repos; isolate them.
*Why: a senior CI/CD security question; pinning to SHA and least-privilege tokens are the key mitigations.*

**Q: What's a reusable workflow vs a composite action?**
A **reusable workflow** (`workflow_call`) lets one workflow call another — for sharing whole pipelines across repos. A **composite action** bundles multiple steps into one reusable action. Both reduce duplication; reusable workflows are higher-level (jobs), composite actions are lower-level (steps). *Why: tests how you scale CI/CD across an org.*

**Q: How do you make pipelines faster?**
Cache dependencies (`actions/cache` for npm/pip/Docker layers), use matrix parallelism, split into parallel jobs, use `paths:` filters to skip unaffected runs, and use Docker layer caching for builds. *Why: optimization question.*

---

# PART 2 — HELM

## Basics

**⭐ Q: What is Helm and why do we use it?**
Helm is the package manager for Kubernetes. A **chart** is a package of templated Kubernetes manifests. The problem it solves: raw K8s YAML is static and duplicated — you'd copy-paste the same Deployment/Service/Ingress across dev/staging/prod, changing only a few values. Helm **templates** the manifests and injects environment-specific **values**, so you maintain one chart and deploy it everywhere with different `values.yaml`. It also versions releases and enables rollback. *Why: the "why Helm" question — templating + values + release management is the answer.*

**⭐ Q: What are the core Helm concepts — chart, values, release, template?**
- **Chart** — the package (templates + default values + metadata).
- **Templates** — K8s manifests with Go templating placeholders (`{{ .Values.replicaCount }}`).
- **Values** — the inputs (`values.yaml`) that fill the templates; overridable per environment.
- **Release** — a deployed instance of a chart in a cluster (Helm tracks its version/history).
*Why: foundational Helm vocabulary.*

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}    # injected from values.yaml
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```
```yaml
# values.yaml
replicaCount: 3
image:
  repository: myacr.azurecr.io/app
  tag: v1.2.3
```

## Intermediate / Advanced

**Q: What's the structure of a Helm chart?**
`Chart.yaml` (metadata — name, version), `values.yaml` (default values), `templates/` (the templated manifests), `templates/_helpers.tpl` (reusable template snippets), `charts/` (dependency subcharts). *Why: tests hands-on chart knowledge.*

**Q: How do you override values per environment?**
Multiple ways: separate values files (`helm install -f values-prod.yaml`), `--set key=value` on the command line, or layered values (base + environment overlay). Later files/flags override earlier ones. *Why: the practical multi-env pattern.*

**⭐ Q: How does Helm rollback work?**
Helm tracks the history of each release (`helm history <release>`). `helm rollback <release> <revision>` reverts to a previous version — it re-applies that revision's manifests. This is one of Helm's big advantages over raw `kubectl apply`. *Why: ties to rollback strategy on your resume.*

**Q: What are Helm hooks?**
Annotations that run resources at specific points in a release lifecycle — `pre-install`, `post-install`, `pre-upgrade`, `post-delete`, etc. Used for things like running a database migration Job before an upgrade. *Why: advanced chart knowledge.*

**Q: Helm vs Kustomize?**
Helm = templating with values + package management + release/rollback (powerful but Go templating can get complex). Kustomize = template-free overlays — you patch a base manifest per environment (simpler, built into kubectl, no templating language, but no packaging/release management). Helm for packaging/distribution and complex parameterization; Kustomize for simpler environment overlays. Some teams use both (Kustomize to render, or Helm + post-render). *Why: a common "which and why" question.*

**Q: How do you test/validate a Helm chart?**
`helm lint` (syntax/best-practice checks), `helm template` (render templates locally to see the output YAML without deploying), `--dry-run` (server-side validation), and `helm test` (run test hooks against a deployed release). *Why: testing question you specifically asked about — these are the chart validation tools.*

---

# PART 3 — GITOPS & ARGOCD

## GitOps concepts

**⭐ Q: What is GitOps?**
An operational model where **Git is the single source of truth** for declarative infrastructure and application state. You describe the desired state in Git; an in-cluster controller continuously **reconciles** the actual cluster state to match Git. You don't `kubectl apply` from a pipeline — you commit to Git, and the controller pulls and applies. Changes happen via pull request (reviewed, audited, revertable). *Why: the core definition; "Git as source of truth + reconciliation" is the answer.*

**⭐ Q: Push vs Pull CD — and why GitOps (pull) is preferred?**
**Push** — the CI pipeline has cluster credentials and pushes changes in (`kubectl apply` from CI). **Pull** — a controller *inside* the cluster watches Git and pulls changes in. Pull (GitOps) is preferred because: (1) **security** — cluster credentials never leave the cluster; CI doesn't need cluster access; (2) **source of truth** — what's in Git *is* what's in the cluster, with drift detection; (3) **auditability** — every change is a Git commit, reviewed via PR; (4) **easy rollback** — revert the commit. *Why: THE GitOps question; this is on your resume (ArgoCD). Lean on your project experience.*

**⭐ Q: What's the reconciliation loop in GitOps?**
The controller (ArgoCD) continuously compares the desired state in Git against the actual cluster state. If they differ (drift — someone made a manual change, or Git was updated), it reports the app as **OutOfSync** and (if auto-sync is on) re-applies Git's state to correct it. It's the same level-triggered, desired-vs-actual model as Kubernetes controllers themselves. *Why: ties GitOps to the reconciliation concept from the K8s internals deep-dive.*

## ArgoCD

**⭐ Q: What is ArgoCD and how does it work?**
A GitOps continuous-delivery controller that runs *inside* the cluster. You define an **Application** pointing to a Git repo path (containing manifests, Helm charts, or Kustomize) and a target cluster/namespace. ArgoCD watches the repo, compares desired (Git) vs live (cluster) state, shows sync status, and applies changes — manually or via auto-sync. It gives a UI/CLI showing health and sync status of every app. *Why: you used ArgoCD in your project — explain it from real experience.*

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout
  namespace: argocd
spec:
  project: default
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

**⭐ Q: What do `prune`, `selfHeal`, and auto-sync do?**
**Auto-sync** — automatically apply Git changes to the cluster. **prune** — delete cluster resources that were removed from Git (keeps cluster matching Git exactly). **selfHeal** — if someone makes a manual change in the cluster, ArgoCD reverts it back to match Git (enforces Git as truth). *Why: these are the practical knobs; selfHeal enforcing Git-as-truth is the key concept.*

**Q: What's the App-of-Apps pattern?**
A parent ArgoCD Application that points to a Git path containing *other* Application definitions — so one app bootstraps many. Used to manage many apps/environments declaratively from a single root. *Why: tests scaling ArgoCD across many apps — an advanced pattern.*

**Q: What are sync waves and hooks in ArgoCD?**
**Sync waves** order resource application (e.g., apply the database before the app that needs it) using annotations. **Resource hooks** (PreSync, Sync, PostSync) run things at sync phases — like a migration Job before the new version syncs. *Why: tests ordered, safe deployments.*

**⭐ Q: How does ArgoCD handle rollback?**
Two ways: revert the Git commit (the GitOps way — ArgoCD syncs back to the previous state, fully audited), or use ArgoCD's history to roll back to a previous synced revision directly. The Git-revert approach is preferred because it keeps Git as the source of truth and is auditable. *Why: ties to rollback strategy; the "revert the commit" answer shows you get GitOps.*

**Q: ArgoCD vs Flux?**
Both are GitOps controllers. ArgoCD has a rich UI and is application-centric (good visibility, popular for app delivery). Flux is more lightweight, CLI/GitOps-toolkit-based, and composable. Both reconcile Git → cluster; choice is largely preference/ecosystem. *Why: comparison question.*

---

# PART 4 — THE FULL PIPELINE: HOW IT ALL FITS

**⭐ Q: Walk me through a complete CI/CD + GitOps pipeline using these tools.**
> Developer pushes code → **GitHub Actions** (CI) runs: checkout → tests → SAST scan → build Docker image → **Trivy scan** the image → push to **ACR** with an immutable tag (git SHA) → update the image tag in a **Helm values file** in a separate *manifests* Git repo (or open a PR). → **ArgoCD** (CD) detects the change in the manifests repo → compares desired vs live → syncs the new version into AKS. Rollback = revert the Git commit. 

The key architectural point: **CI builds and pushes to a registry + updates Git; CD (ArgoCD) pulls from Git into the cluster.** CI never touches the cluster directly — that's the security win of GitOps. *Why: this is the synthesis question; being able to narrate the whole flow with the CI/CD split is a strong signal. Ties to your resume and Athena.*

**Q: Why separate the app code repo from the manifests/config repo?**
Separation of concerns: the app repo triggers builds; the manifests repo is what ArgoCD watches for deployment. It means a config/deployment change doesn't require an app rebuild, the deployment history is a clean Git log, and you can control who can change *what's deployed* separately from who changes *code*. *Why: a GitOps best-practice question.*

---

# PART 5 — SECURITY & TESTING (you specifically asked)

**⭐ Q: What security checks belong in a CI/CD pipeline?**
- **SAST** (Static Application Security Testing) — scan source code for vulnerabilities (e.g., CodeQL, SonarQube).
- **SCA** (Software Composition Analysis) — scan dependencies for known CVEs (Dependabot, Snyk).
- **Image scanning** — scan the built container for vulnerabilities (**Trivy**, Grype, Defender for Containers) *before* pushing.
- **Secret scanning** — detect committed secrets (GitGuardian, GitHub secret scanning, gitleaks).
- **IaC scanning** — scan Terraform/manifests for misconfigurations (tfsec, Checkov, kube-linter).
- **Image signing** — sign images (Cosign/Sigstore) and verify at deploy so only trusted images run.
*Why: "shift-left security" — a strong DevSecOps signal; Trivy + SAST + secret scanning are the core ones to name.*

**⭐ Q: How do you secure the supply chain from code to cluster?**
Pin GitHub Actions to commit SHAs; least-privilege `GITHUB_TOKEN` and OIDC for cloud auth (no stored long-lived secrets); scan code, dependencies, and images in CI; sign images and verify signatures at admission (Cosign + a policy engine like Kyverno/OPA Gatekeeper); use immutable image tags; private registry (ACR) with RBAC; and in the cluster, admission policies that reject unsigned/untrusted/root images. End to end: trusted code → scanned → signed → only verified artifacts run. *Why: a senior supply-chain-security question; ties CI security to cluster admission control.*

**Q: What testing belongs in the pipeline?**
Unit tests (fast, every commit), integration tests (services together), `helm lint` + `helm template` + `--dry-run` (validate manifests), `kube-linter`/`kubeval` (validate K8s YAML), smoke tests post-deploy (is the service actually up?), and in GitOps, ArgoCD health checks confirm the deployed app is healthy. For progressive delivery, canary analysis automatically tests the new version against metrics before full rollout. *Why: the testing question you asked; covers code → manifest → post-deploy validation.*

**Q: How does GitOps improve security specifically?**
Cluster credentials never leave the cluster (CI has no cluster access); every change is a reviewed, audited Git commit (no ad-hoc `kubectl` to prod); drift is detected and auto-corrected (selfHeal), so manual tampering is reverted; and access control is enforced via Git (who can merge to the manifests repo controls what deploys). It turns "who has kubectl access to prod" into "who can merge to Git." *Why: connects GitOps to security — a sophisticated answer.*

---

# PART 1 — FOUNDATIONS

## Basics

**⭐ Q: What is observability, and how is it different from monitoring?**
Monitoring tells you *whether* a system is working (predefined dashboards/alerts on known failure modes — "is CPU high?"). Observability is the ability to ask *arbitrary new questions* about your system's internal state from its external outputs, including failures you didn't predict ("why is *this specific* user's request slow?"). Monitoring answers known-unknowns; observability helps with unknown-unknowns. *Why: the classic opener; the "known vs unknown unknowns" framing signals maturity.*

**⭐ Q: What are the three pillars of observability?**
**Metrics** — numeric time-series, cheap to store, great for dashboards/alerts/trends ("error rate is 5%"). **Logs** — discrete timestamped event records, detailed ("here's the exact exception"). **Traces** — follow a single request across services, showing where time was spent ("the checkout call spent 4s in the payments service"). They correlate: a metric tells you something's wrong, a trace tells you which hop, a log tells you exactly what. *Why: foundational; every SRE interview expects this.*

**Q: When do you use each pillar?**
Metrics: alerting, dashboards, trends, SLOs (always-on, low cost). Logs: detailed debugging, audit, exact error messages (higher cost, query when investigating). Traces: latency analysis, finding bottlenecks in distributed request flows, understanding service dependencies. The skill is using them *together* during an incident. *Why: tests practical judgment, not just definitions.*

## Intermediate

**⭐ Q: Explain how the three pillars correlate during an incident.**
A metric alert fires (error rate spike). You pivot to a **trace** of a failing request to see *which service/span* failed and where latency went. From the trace's span, you jump to the **logs** for that trace ID to read the actual error. Metrics tell you *something's wrong and roughly where*; traces tell you *which hop*; logs tell you *exactly what*. Exemplars (trace IDs attached to metric samples) and shared trace IDs in logs make this jump seamless in Grafana. *Why: this is the SRE workflow — demonstrating you think across pillars, not in silos, is a strong signal.*

**Q: What's an exemplar?**
A sample trace ID attached to a metric data point — so when you see a latency spike on a graph, you can click directly to an *example trace* that contributed to it. It's the bridge from metrics to traces. *Why: shows modern observability depth; ties pillars together.*

**Q: What are the RED, USE, and Four Golden Signals methods?**
**RED** (for request-driven services): Rate, Errors, Duration. **USE** (for resources): Utilization, Saturation, Errors. **Four Golden Signals** (Google): Latency, Traffic, Errors, Saturation. RED ≈ the request-facing subset of golden signals; USE is for the underlying resources (CPU, memory, disk). *Why: these frameworks structure *what* to measure — interviewers want to hear you have a method, not random metrics.*

---

# PART 2 — METRICS & PROMETHEUS

**⭐ Q: Why does Prometheus use a pull model, and what's the consequence?**
Prometheus *scrapes* targets rather than receiving pushed data. Benefits: a free up/down signal (the `up` metric — if a scrape fails, the target is down), centralized control of scrape intervals, and targets don't need to know where to push. Consequence: short-lived jobs (CronJobs, batch) may finish before being scraped — solved with the **Pushgateway** for those specific cases. *Why: a guaranteed Prometheus question; the pull-vs-push reasoning is core.*

**⭐ Q: What's the difference between Counter, Gauge, Histogram, and Summary?**
**Counter** — only increases (total requests); use `rate()` on it. **Gauge** — goes up and down (memory usage, queue depth, temperature). **Histogram** — buckets observations (request durations into <0.1s, <0.5s, <1s) for server-side percentile calculation. **Summary** — computes quantiles client-side. For latency, prefer Histogram because it's aggregatable across instances; Summary can't be aggregated. *Why: metric-type choice is commonly tested; the histogram-vs-summary aggregation point is the depth signal.*

**⭐ Q: Why must you `rate()` before `sum()`, never the reverse?**
Counters reset to zero on pod restart. `rate()` is reset-aware — it detects the drop and corrects for it. If you `sum()` raw counters first, a single restart makes the summed value appear to plummet, and `rate()` on that produces garbage. Always `sum(rate(metric[5m]))`. *Why: the single most common PromQL mistake interviewers probe.*

**⭐ Q: How do you compute p99 latency, and what's the catch with percentiles?**
`histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`. The catch: you **cannot average percentiles** across instances — p99 of (p99_a, p99_b) is meaningless. You must aggregate the raw histogram *buckets* first (sum by `le`), then compute the quantile once. *Why: tests real PromQL fluency; the "can't average percentiles" rule separates beginners from people who've done it.*

**⭐ Q: What is cardinality and why is it the #1 way Prometheus blows up?**
Cardinality = the number of unique time-series, driven by label-value combinations. Each unique combination is a separate series stored in memory. Putting high-cardinality values in labels (user IDs, request IDs, full URLs with IDs, emails) creates millions of series and exhausts memory, crashing Prometheus. Control it: never label with unbounded values, use relabeling to drop dangerous labels, monitor `prometheus_tsdb_head_series`, and push high-cardinality data to logs/traces instead. *Why: the most important operational Prometheus lesson; a guaranteed senior-level question.*

**Q: What are recording rules and when do you use them?**
Recording rules pre-compute expensive or frequently-used queries at scrape interval and store the result as a new metric. Use for slow dashboard queries, SLI computations referenced by multiple alerts, and anything evaluated often. Naming convention: `level:metric:operation` (e.g., `sli:checkout_availability:ratio_rate5m`). *Why: tests whether you know how to make a metrics system performant at scale.*

**Q: How does Prometheus discover what to scrape in Kubernetes?**
Via the Prometheus Operator's **ServiceMonitor** (and PodMonitor) CRDs — you declare which services/pods to scrape with which interval and path, and Prometheus auto-discovers matching targets. Plus standard kubernetes_sd service discovery. *Why: ties to your Athena project (you wrote ServiceMonitors) and real AKS Prometheus setup.*

---

# PART 3 — LOGS

**⭐ Q: Loki vs ELK (Elasticsearch) — the architectural difference and tradeoff?**
**Loki** indexes only metadata *labels* (namespace, pod, container), not log content. The assumption: you query logs by context you already know (which service/pod), usually arriving from a dashboard or trace. Result: ~10× cheaper storage, far simpler to operate. **ELK** indexes the full text of every log — powerful for ad-hoc search and analytics, but expensive at scale and operationally heavy. Loki for cost-efficient operational logs; ELK when you need deep full-text search, complex analytics, or SIEM. *Why: a guaranteed log question; the "index labels not content" insight is the differentiator.*

**Q: What is LogQL and how does it differ from PromQL?**
Loki's query language. Two parts: a **stream selector** using labels (`{app="checkout"}`) which uses the index (fast), then a **filter pipeline** (`|= "error" | json | status="500"`) which scans the matched streams (slower the more you match). You can also generate metrics *from* logs: `rate({app="x"} |= "error" [5m])`. *Why: shows you can actually query Loki, not just describe it.*

**Q: Why does label cardinality matter in Loki too?**
Same problem as Prometheus — each unique label combination is a separate *stream*. High-cardinality labels (request ID, user ID) explode the number of streams and destroy Loki's performance. Keep labels low-cardinality; put high-cardinality data in the log *content* (which isn't indexed), not in labels. *Why: tests that you understand the design principle deeply, not just the tool.*

**Q: What makes a good structured log?**
JSON-formatted (machine-parseable), with consistent fields: timestamp, level, service, **trace ID** (for correlation), a clear message, and relevant context — but *not* secrets or PII. Structured logs let you query by field instead of regex-scraping text. *Why: tests log hygiene; the trace ID for correlation is the key SRE detail.*

---

# PART 4 — TRACES

**⭐ Q: What is distributed tracing and why is it essential for microservices?**
A trace follows a single request as it flows through multiple services, recording each step as a **span** with timing. It shows the full request path and where time was spent — answering "which of the 8 services in this request is slow?" In microservices, metrics and logs alone can't tell you *which hop* failed or slowed; only traces stitch the request together end-to-end. *Why: core to SRE in distributed systems; ties to your dependency-failure incident.*

**Q: What's a span vs a trace?**
A **trace** is the whole journey of one request. A **span** is one unit of work within it (one service's processing, or one downstream call), with a start time, duration, and parent-child relationships. Spans nest to form the trace, showing the call tree and where latency accumulates. *Why: basic tracing vocabulary.*

**⭐ Q: What is OpenTelemetry and why does it matter?**
OpenTelemetry (OTel) is the vendor-neutral standard for instrumentation — SDKs to emit metrics, logs, and traces, plus a Collector to receive, process, and export them. It **decouples instrumentation from backend**: instrument once with OTel SDKs, then send to Tempo, Jaeger, Datadog, Honeycomb — any backend — without changing application code. It killed SDK-level vendor lock-in; every major vendor now accepts OTLP (the OTel protocol). *Why: THE modern observability shift; a guaranteed question for current SRE roles. Ties to your Athena project.*

**Q: Tempo vs Jaeger?**
Both store traces. **Tempo** follows Loki's philosophy — store cheaply, retrieve by trace ID, no expensive attribute indexing; you find traces via exemplars from metrics or IDs from logs. **Jaeger** indexes spans for richer free-form search — more flexible but more costly. Tempo for cost-efficiency and Grafana integration; Jaeger when you need deep span search. *Why: tracing backend choice; ties to Athena.*

**Q: What is sampling and why do you need it?**
Tracing every request at high volume is expensive (storage, overhead). Sampling keeps a subset. **Head-based** sampling decides at the start (e.g., keep 10%) — simple but may miss rare errors. **Tail-based** sampling decides after the trace completes (e.g., keep all errors and slow traces, sample the rest) — smarter, keeps the interesting traces, but needs more infrastructure. *Why: a practical scaling question; tail-based for keeping errors is the senior answer.*

---

# PART 5 — SLI / SLO / ERROR BUDGETS (the heart of SRE)

**⭐ Q: Define SLI, SLO, and SLA.**
**SLI** (Indicator) = the measured number — e.g., % of successful requests, or % of requests under 300ms. **SLO** (Objective) = the internal target for that SLI — e.g., 99.5% over 30 days. **SLA** (Agreement) = the external contract with customers, with penalties if breached — usually set *looser* than your SLO so you have a safety margin. SLI = what you measure, SLO = your goal, SLA = the promise to customers. *Why: the absolute core of SRE; you must nail this distinction.*

**⭐ Q: What is an error budget and how does it change how teams work?**
Error budget = 1 − SLO. For a 99.5% SLO, the budget is 0.5% — about 216 minutes of allowed failure per month. It's a *shared currency* between dev and ops: if there's budget remaining, you can take risks and ship features fast; if the budget is burned, you freeze risky releases and focus on reliability. It turns "how reliable should we be?" from an argument into a data-driven decision. *Why: the error budget is *the* SRE concept; "shared currency / data-driven decision" is the framing that impresses.*

**⭐ Q: What's a good SLI? How do you choose one?**
A good SLI reflects the *user's experience*, not internal mechanics. Base it on user-journey outcomes: availability (% successful requests), latency (% of requests under a threshold), correctness, freshness. Measure at the point closest to the user (the load balancer or gateway, not deep internal services). Avoid SLIs the user doesn't care about (like CPU — that's a cause, not a user-facing symptom). *Why: tests whether you understand SLOs are about *users*, not infrastructure.*

**⭐ Q: Explain multi-window, multi-burn-rate alerting and why it beats threshold alerting.**
Instead of alerting on a raw threshold ("error rate > 1%"), you alert on **burn rate** — how fast you're consuming the error budget relative to the SLO. Burn rate = error rate ÷ (1 − SLO). At 14.4× burn, you'd exhaust a 30-day budget in ~2 days (page now). **Multi-window** means a short *and* a long window must both breach before firing — the short confirms it's happening now, the long confirms it's sustained, which eliminates flapping and gives fast recovery. Multiple burn-rate tiers map urgency to severity (fast burn → page, slow burn → ticket). *Why: the most advanced alerting concept; straight from the Google SRE Workbook. This is what separates strong SRE candidates. Ties to your Athena project.*

**Q: What's the difference between alerting on causes vs symptoms?**
Symptom-based alerting fires on user-facing impact ("error rate is high," "latency exceeds SLO") — what the *user* feels. Cause-based fires on internal conditions ("CPU is 90%") which may or may not affect users. SRE favors **symptom-based** alerts because they're actionable and user-relevant; cause metrics belong on dashboards for diagnosis. The rule: page on symptoms, diagnose with causes. *Why: a core SRE alerting philosophy; that JD you shared specifically said "symptom-based alerting."*

---

# PART 6 — ALERTING & ON-CALL QUALITY

**⭐ Q: How do you reduce alert fatigue / improve on-call signal quality?**
Make every alert actionable (the rule: *if an alert doesn't require a human action, it's a dashboard metric, not an alert*). Use `for` durations so transient spikes don't fire. Switch from raw thresholds to burn-rate/SLO-based alerts. Add inhibition rules (suppress symptom alerts when a root-cause alert is firing). Group related alerts. Remove or downgrade non-actionable alerts. *Why: directly your real experience — the noisy-alert cleanup (8-10 → 2-3 pages). Lead with that story.*

**Q: What is Alertmanager and what does it do?**
Prometheus fires alerts; Alertmanager handles them — **routing** (by severity/team), **grouping** (bundle related alerts into one notification), **inhibition** (suppress alerts when a higher-priority one is active), **silencing** (mute during maintenance), and **deduplication**. It delivers to receivers (PagerDuty, Slack, email, webhook). *Why: tests the alerting pipeline beyond just firing.*

---

# PART 7 — RESILIENCE & ADVANCED (the JD-level topics)

**Q: What reliability patterns prevent cascading failures?**
Timeouts (don't wait forever), retries with backoff *and jitter* (recover from transient failures without thundering-herd), circuit breakers (stop calling a failing dependency, fail fast, let it recover), bulkheads (isolate resource pools so one failure can't exhaust everything), load shedding (drop excess load to protect core function), graceful degradation (serve reduced functionality rather than failing). *Why: the resilience patterns from that SRE JD; ties observability to system design.*

**Q: What is chaos engineering and how does it relate to observability?**
Deliberately injecting controlled failures (kill a pod, inject latency, simulate a zone outage) to validate resilience assumptions and uncover unknown failure modes — with a hypothesis tied to an SLO ("a regional dependency failure should degrade gracefully without breaching the availability SLO"), blast-radius limits, and abort conditions. Observability is *essential* — you can't run chaos safely without the metrics/traces/logs to see the impact and confirm the hypothesis. *Why: that JD wanted chaos engineering; honest framing — "I understand the principles and I'd run controlled experiments in my Athena cluster" (cross-ref the chaos suggestion).*

**Q: What's the difference between MTTD, MTTR, MTBF?**
MTTD (Mean Time To Detect) — how long until you *notice* a problem (observability drives this down). MTTR (Mean Time To Restore/Repair) — how long to *fix* it. MTBF (Mean Time Between Failures) — reliability of the system between incidents. Good observability improves MTTD and MTTR. *Why: reliability metrics vocabulary; ties to your "reduced MTTR" framing.*

**Q: What goes into a good postmortem?**
Blameless (focus on systems and contributing factors, not individuals), a clear timeline, the impact (users/duration/SLO budget consumed), root cause(s) including contributing factors, what went well/poorly in the response, and concrete action items with owners and deadlines — then verify they're done. The goal is learning and prevention, not blame. *Why: incident management is core SRE; "blameless" and "action items with owners" are the key signals.*

---

# PART 1 — COMPUTE & AKS (your core)

## Basics

**Q: What are the main Azure compute options and when each?**
VMs (full control, you manage the OS — lift-and-shift, custom workloads). VM Scale Sets (auto-scaling identical VMs behind a load balancer). App Service (managed PaaS for web apps — no infra management). Functions (serverless, event-driven, pay-per-execution). AKS (managed Kubernetes — container orchestration at scale). Container Instances (ACI — single containers, no orchestration, burst workloads). *Why: tests whether you can match workload to service — a design question.*

**⭐ Q: What is AKS and what does Azure manage vs you?**
Azure Kubernetes Service is managed Kubernetes. **Azure manages the control plane** — API server, etcd, scheduler, controller-manager — for free, and you don't see or SSH into those nodes. **You manage** the worker node pools, your workloads, and configuration. You get managed upgrades, scaling, and tight integration with Azure AD, ACR, Azure Monitor, and Azure networking. *Why: THE most important Azure answer for you — and it's your daily platform.*

**⭐ Q: What are node pools in AKS?**
Groups of worker nodes with the same config. A **system node pool** runs critical system pods (CoreDNS, metrics-server); **user node pools** run your workloads. You can have multiple user pools with different VM sizes (e.g., a GPU pool, a memory-optimized pool) and use taints/nodeSelectors to place workloads. Each pool can scale independently. *Why: real AKS operations knowledge; shows you understand cluster structure.*

## Intermediate / Advanced

**⭐ Q: How does autoscaling work in AKS — the two layers?**
Two distinct layers: **HPA (Horizontal Pod Autoscaler)** scales *pods* based on CPU/memory/custom metrics. **Cluster Autoscaler** scales *nodes* — when pods can't schedule due to insufficient resources, it adds nodes; when nodes are underutilized, it removes them. They work together: HPA adds pods → if no room, Cluster Autoscaler adds nodes. There's also KEDA for event-driven pod scaling (queue depth, etc.). *Why: tests understanding that pod scaling and node scaling are separate — a common confusion.*

**⭐ Q: How do AKS pods get Azure credentials securely?**
**Workload Identity** (the current best practice) — federates a Kubernetes ServiceAccount with an Azure AD identity, so pods get short-lived tokens to access Azure resources (Key Vault, Storage) with *no stored secrets*. The older approach was pod-managed identities (AAD Pod Identity), now deprecated. *Why: security question; "no stored secrets, short-lived tokens" is the key phrase.*

**⭐ Q: Azure CNI vs kubenet in AKS — the tradeoff?**
Azure CNI gives every pod a real IP from the VNet subnet — pods are directly routable, better performance, integrate with VNet features, but consume lots of VNet IP space (nodes pre-allocate IP blocks), so large clusters can exhaust the subnet. Kubenet gives pods IPs from a separate overlay range and NATs through the node — conserves VNet IPs but adds a hop and has routing limitations. Choose Azure CNI for VNet integration when you have IP space; kubenet to conserve IPs. *Why: AKS-specific networking — high value for your platform. (Cross-ref the networking deep-dive.)*

**Q: How do AKS upgrades work and how do you do them safely?**
AKS supports upgrading the control plane and node pools (Azure manages the control-plane upgrade). For nodes, AKS does a rolling upgrade — cordons and drains a node (respecting PodDisruptionBudgets), upgrades it, brings it back, moves to the next. Best practice: upgrade control plane first, then node pools; have PDBs and multiple replicas so draining doesn't cause outages; test in non-prod first. *Why: real operational task; ties PDBs to a concrete scenario.*

---

# PART 2 — NETWORKING

**⭐ Q: What is a VNet, subnet, and NSG?**
VNet = your isolated private network in Azure. Subnet = a segment of the VNet's address space (separate tiers). NSG (Network Security Group) = stateful firewall rules applied to subnets or NICs, controlling inbound/outbound traffic by source/dest/port. **Stateful** means allowing inbound automatically allows the return traffic. *Why: foundational; NSG misconfig is a common "can't reach the service" cause.*

**⭐ Q: Azure Load Balancer vs Application Gateway vs Front Door vs Traffic Manager?**
- **Azure Load Balancer** — L4 (TCP/UDP), regional, raw traffic distribution.
- **Application Gateway** — L7 (HTTP), regional, path/host routing + WAF + TLS termination.
- **Front Door** — L7, *global*, edge routing across regions + CDN + global failover + WAF.
- **Traffic Manager** — DNS-based global routing (returns different IPs based on routing method like geographic/priority).
Pick by layer (L4 vs L7) and scope (regional vs global). *Why: a design question; knowing global vs regional and L4 vs L7 is the differentiator.*

**Q: How does traffic from the internet reach a pod in AKS?**
Internet → Azure Load Balancer (provisioned by a Service of type LoadBalancer, or Application Gateway via AGIC) → Ingress controller pods → Service (ClusterIP) → pod. Typically you have one LB for the ingress controller, and the ingress does L7 routing to internal services. *Why: ties Azure networking to K8s networking. (Cross-ref.)*

**Q: What's a Private Endpoint and Service Endpoint?**
Both secure access to Azure PaaS services (like Azure SQL, Storage) from your VNet. **Private Endpoint** gives the service a private IP *inside* your VNet (traffic never touches the internet — most secure). **Service Endpoint** extends your VNet identity to the service over the Azure backbone but the service keeps a public IP. Private Endpoint is the modern, more secure choice. *Why: security/networking design; relevant to "how does AKS reach Azure SQL privately."*

**Q: What's ExpressRoute vs VPN Gateway?**
Both connect on-prem to Azure. VPN Gateway = encrypted tunnel over the public internet (cheaper, variable performance). ExpressRoute = a private, dedicated connection via a provider (bypasses the internet — consistent performance, higher cost, used by enterprises). *Why: hybrid connectivity; enterprise/GCC relevant.*

---

# PART 3 — IDENTITY & SECURITY

**⭐ Q: What is Azure AD (Entra ID) and how does it relate to RBAC?**
Azure AD (now Entra ID) is Azure's identity provider — manages users, groups, service principals, and authentication (OIDC, SAML, MFA). **Azure RBAC** controls *what* an identity can do on Azure resources via role assignments (Owner, Contributor, Reader, or custom roles) scoped to a subscription/resource group/resource. Identity (who) + RBAC (what they can do). *Why: identity is foundational to everything in Azure.*

**⭐ Q: Service Principal vs Managed Identity — when each?**
Both are non-human identities for apps/services to authenticate to Azure. **Service Principal** = you create and manage credentials (a secret or cert) — you're responsible for rotating them. **Managed Identity** = Azure manages the credentials for you, no secrets to store or rotate — preferred wherever supported. System-assigned (tied to one resource's lifecycle) vs user-assigned (standalone, shareable). *Why: security best practice; "Managed Identity = no secrets to rotate" is the key point. Maps to AKS Workload Identity.*

**⭐ Q: What is Azure Key Vault and how do AKS workloads use it?**
A managed service for storing secrets, keys, and certificates securely, with access controlled by RBAC/policies and full audit logging. AKS pods access it via the **Secrets Store CSI driver** (mounts Key Vault secrets as files in the pod) combined with Workload Identity (no stored credentials). *Why: secrets management is a guaranteed DevOps/SRE topic; the CSI driver + Workload Identity combo is the modern pattern.*

**Q: How do you implement least privilege in Azure?**
Use built-in roles (Reader, Contributor) or custom roles with minimal permissions, scope assignments as narrowly as possible (resource > resource group > subscription), use Managed Identities instead of stored credentials, use PIM (Privileged Identity Management) for just-in-time elevated access, and audit with Azure AD logs. *Why: security maturity signal.*

---

# PART 4 — STORAGE & DATA

**Q: What are the Azure Storage types?**
Within a Storage Account: **Blob** (object storage — files, backups, static content, logs), **Files** (managed SMB/NFS file shares — can be ReadWriteMany for AKS), **Queue** (simple message queue), **Table** (NoSQL key-value). Plus **Managed Disks** for VM/AKS block storage. *Why: matching storage type to use case; Blob and Files come up most.*

**⭐ Q: For AKS persistent volumes, when Azure Disk vs Azure Files?**
**Azure Disk** = block storage, **ReadWriteOnce** (one node at a time) — for single-pod stateful workloads like a database. **Azure Files** = SMB/NFS, **ReadWriteMany** (multiple nodes) — when multiple pods across nodes need shared access. The RWO limitation of Disk is why a Deployment sharing one disk across nodes fails — use Files for shared, Disk for single-pod. *Why: ties Azure storage to K8s PV access modes. (Cross-ref storage deep-dive.)*

**Q: Azure SQL vs Cosmos DB vs managed Postgres/MySQL?**
Azure SQL = managed SQL Server (relational, ACID). Cosmos DB = globally-distributed NoSQL, multi-model, low-latency, tunable consistency. Azure Database for PostgreSQL/MySQL = managed open-source relational. Choose by data model (relational vs NoSQL), scale/distribution needs, and existing stack. *Why: data-tier design questions.*

**Q: What's Event Hubs vs Service Bus?**
Service Bus = enterprise messaging (queues, topics, ordering, transactions) — for reliable command/message delivery between services. Event Hubs = high-throughput event streaming (millions of events/sec, like Kafka) — for telemetry, logs, event pipelines. Service Bus for messaging, Event Hubs for streaming. *Why: async architecture questions; both were in that SRE JD you shared.*

---

# PART 5 — MONITORING & OBSERVABILITY (your strength)

**⭐ Q: Azure Monitor vs Application Insights vs Log Analytics — how they relate?**
**Log Analytics** = the underlying data store + query engine (KQL — Kusto Query Language). **Application Insights** = the APM layer on top — request tracing, dependencies, exceptions, performance for your apps; its data lands in a Log Analytics workspace. **Azure Monitor** = the umbrella platform tying together metrics, logs, alerts, and dashboards across all Azure resources. *Why: you use these daily — be able to explain the hierarchy crisply; it's a guaranteed question for your profile.*

**⭐ Q: What is KQL and why does it matter?**
Kusto Query Language — the query language for Log Analytics and App Insights. You use it to query logs, traces, and metrics (e.g., `requests | where resultCode >= 500 | summarize count() by bin(timestamp, 5m)`). It's the Azure equivalent of PromQL/LogQL for log analysis. *Why: hands-on Azure observability requires KQL; mention you use it for incident investigation.*

**Q: How do you set up alerting in Azure Monitor?**
Alert rules based on metrics or log (KQL) queries, with conditions and thresholds, routed through **Action Groups** (which define who/what gets notified — email, SMS, webhook, PagerDuty, runbook). You can do metric alerts (fast, on platform metrics) or log alerts (KQL-based, more flexible). *Why: ties to your alerting experience; Action Groups are the routing mechanism.*

**Q: How does Azure monitoring integrate with AKS?**
Container Insights collects AKS metrics and logs into Log Analytics. You can also run Prometheus (managed Azure Monitor for Prometheus, or self-hosted) and Grafana (managed Azure Managed Grafana) — which is closer to your actual stack. *Why: connects your Prometheus/Grafana work to the Azure-native option.*

---

# PART 6 — IaC & DEVOPS TOOLING

**Q: ARM templates vs Bicep vs Terraform?**
ARM templates = Azure-native JSON IaC (verbose). Bicep = a cleaner DSL that compiles to ARM (Azure-native, more readable). Terraform = cloud-agnostic, often preferred for multi-cloud and its ecosystem. For Azure-only shops, Bicep is gaining traction; for multi-cloud or existing Terraform, Terraform. *Why: IaC choice question; you migrated to Terraform, so you can speak to why.*

**⭐ Q: What's the difference between Azure DevOps and GitHub Actions?**
Both do CI/CD. Azure DevOps = a full suite (Repos, Pipelines, Boards, Artifacts) — pipelines defined in YAML, strong Azure integration, common in enterprises. GitHub Actions = CI/CD integrated into GitHub, workflow YAML, huge marketplace of actions. You use both. Many orgs are shifting toward GitHub Actions. *Why: you support both — be ready to compare and say which you'd pick and why.*

**Q: What is ACR and how does it fit the pipeline?**
Azure Container Registry — managed private Docker registry. Pipeline: build image → push to ACR → AKS pulls from ACR (authenticated via Managed Identity, no stored creds). ACR also does image scanning (with Defender) and geo-replication. *Why: ties your container + pipeline + AKS knowledge together.*

---

# PART 7 — RELIABILITY, HA & DR (the SRE angle)

**⭐ Q: Availability Zone vs Region — and designing for failure?**
A **Region** is a geographic area; an **Availability Zone** is a physically separate datacenter within a region (independent power/cooling/network). Spreading across AZs survives a single-datacenter failure with low latency; spreading across regions survives a whole-region outage (your DR strategy) but with higher latency/complexity. For AKS: multi-zone node pools + topologySpreadConstraints to spread pods across zones. *Why: core SRE design question. (Cross-ref the AZ/region deep-dive.)*

**⭐ Q: How do you design an AKS workload to survive an AZ failure?**
Multi-zone node pool spread across AZs, `topologySpreadConstraints` to distribute replicas across zones, enough replicas with headroom to absorb losing a zone, a zone-redundant managed database (Azure SQL with zone redundancy), PodDisruptionBudgets, and health probes + autoscaling. Goal: losing a zone degrades capacity but not availability. *Why: SRE design; ties AKS + Azure HA together.*

**Q: Explain RTO and RPO and how they shape Azure DR design.**
RTO (Recovery Time Objective) = max acceptable downtime; RPO (Recovery Point Objective) = max acceptable data loss (in time). Tight RPO → synchronous geo-replication or frequent backups; tight RTO → warm/hot standby in a second region rather than cold restore. In Azure: geo-redundant storage, Azure SQL active geo-replication, multi-region AKS with Front Door/Traffic Manager for failover. Looser targets = cheaper; tighter = costlier. *Why: DR design; that SRE JD specifically wanted RTO/RPO and failover drills.*

**Q: What Azure features support high availability out of the box?**
Availability Zones, zone-redundant services (SQL, Storage, Load Balancer), VM Scale Sets with health-based instance replacement, Azure SLAs (e.g., multi-AZ deployments get higher SLA), Front Door/Traffic Manager for global failover, and managed services with built-in replication. *Why: shows you know what Azure gives you vs what you must build.*

---

# PART 1 — CORE OBJECTS & THEIR YAML

## Pod (the foundation)

**⭐ Q: What is a Pod and why not just run containers directly?**
A Pod is the smallest deployable unit — one or more containers that share a network namespace (same IP, same localhost), storage volumes, and lifecycle. You don't run bare containers because Kubernetes schedules, heals, and networks at the Pod level. Multiple containers in one Pod is for tightly-coupled helpers (sidecars). *Why: foundational.*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  labels:
    app: checkout          # labels = how everything selects this pod
spec:
  containers:
  - name: checkout
    image: myacr.azurecr.io/checkout:v1.2.3
    ports:
    - containerPort: 8080
    resources:
      requests:            # scheduler places based on this
        cpu: 100m
        memory: 128Mi
      limits:              # exceed memory = OOMKilled; exceed CPU = throttled
        cpu: 500m
        memory: 256Mi
```

You rarely create bare Pods — you use a controller (Deployment) that manages Pods for you.

## Deployment ⭐ (the workhorse)

**⭐ Q: What does a Deployment do and what's underneath it?**
A Deployment manages stateless apps. It creates a ReplicaSet, which creates Pods. It handles rolling updates (new ReplicaSet alongside old, gradually shifting), rollbacks, and scaling. You change the Deployment's pod template → it creates a new ReplicaSet and transitions. *Why: the most-used object; cross-ref the kubectl-apply lifecycle from our internals deep-dive.*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout
spec:
  replicas: 3
  selector:
    matchLabels:
      app: checkout          # MUST match template labels below
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # max extra pods above desired during update
      maxUnavailable: 0      # max pods below desired (0 = zero-downtime)
  template:
    metadata:
      labels:
        app: checkout        # pods get this label; selector finds them
    spec:
      containers:
      - name: checkout
        image: myacr.azurecr.io/checkout:v1.2.3
        ports:
        - containerPort: 8080
        readinessProbe:       # gates traffic
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:        # gates restart
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 10
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits: {cpu: 500m, memory: 256Mi}
```

**⭐ Key YAML points to explain in an interview:**
- **selector must match template labels** — if they don't, the Deployment can't find its pods (a common error).
- **maxSurge/maxUnavailable** control rollout speed vs availability. `maxUnavailable: 0` = never drop below desired count = zero-downtime, but needs surge capacity.
- **Probes** — readiness gates traffic, liveness gates restart (cross-ref the probe deep-dive).

## Service ⭐

**⭐ Q: What is a Service and the YAML for each type?**
A Service is a stable virtual IP (ClusterIP) load-balancing to a set of Pods selected by labels. Pods are ephemeral (IPs change); the Service gives a stable endpoint. (Cross-ref: how it routes — CoreDNS → ClusterIP → kube-proxy iptables DNAT → Pod IP — from our networking deep-dive.)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: checkout
spec:
  type: ClusterIP            # default; internal only
  selector:
    app: checkout            # routes to pods with this label
  ports:
  - port: 80                 # the Service's port
    targetPort: 8080         # the container's port
```

**The types (cross-ref earlier):** ClusterIP (internal), NodePort (port on every node), LoadBalancer (cloud LB), ExternalName (CNAME). The `selector` is the link — it's how the Service finds its backend Pods, via the EndpointSlice the controller maintains.

**⭐ Interview point:** `port` vs `targetPort` — `port` is what clients hit on the Service; `targetPort` is the container's actual port. People confuse them.

## ConfigMap & Secret

**⭐ Q: ConfigMap vs Secret, and how do Pods consume them?**
ConfigMap = non-sensitive config (env-specific values, config files). Secret = sensitive data (passwords, tokens, keys) — base64-encoded (NOT encrypted by default; that's a common misconception). Both can be consumed as environment variables or mounted as files.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  DB_HOST: "postgres.prod.svc.cluster.local"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0    # base64 — NOT encrypted, just encoded
```

Consuming in a Pod:
```yaml
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD
```

**⭐ Interview gotcha:** "Secrets are base64-*encoded*, not encrypted — anyone with API access can decode them. For real security you enable encryption-at-rest in etcd, use RBAC to restrict access, and ideally an external secret store (Key Vault via CSI driver, External Secrets Operator, Vault)." That answer shows real security awareness.

## StatefulSet ⭐

**⭐ Q: StatefulSet vs Deployment — when and why?**
StatefulSet gives Pods **stable, ordered identity**: named `name-0`, `name-1`, created/scaled in order, each with a **stable network name** (via a headless Service) and its **own persistent volume** that follows it across reschedules. Use for databases, Kafka, Zookeeper, anything needing stable identity or per-pod storage. Deployments treat pods as interchangeable; StatefulSets don't.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless    # headless Service for stable DNS
  replicas: 3
  selector:
    matchLabels: {app: postgres}
  template:
    metadata:
      labels: {app: postgres}
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:               # each pod gets its OWN PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

**⭐ Interview point:** `volumeClaimTemplates` — each Pod gets its *own* PVC (postgres-data-0, postgres-data-1), unlike a Deployment where pods would share or have no stable storage. The headless Service (`clusterIP: None`) gives each pod a stable DNS name like `postgres-0.postgres-headless`.

## DaemonSet

**Q: What's a DaemonSet and when do you use it?**
Runs exactly one Pod per node (or per matching node). Used for node-level agents: log collectors (Promtail, Fluent Bit), monitoring (node-exporter), CNI plugins, kube-proxy itself. When a new node joins, the DaemonSet automatically schedules a pod on it. *Why: tests knowledge of the "one per node" pattern.*

## Job & CronJob

**Q: Job vs CronJob?**
A Job runs a Pod to completion (batch task — a migration, a backup) and tracks success/retries. A CronJob runs Jobs on a schedule (like cron). *Why: batch/scheduled work; your disk-cleanup could run as a CronJob.*

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: disk-cleanup
spec:
  schedule: "0 2 * * *"        # 2am daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: cleanup:v1
          restartPolicy: OnFailure
```

---

# PART 2 — CONFIGURATION & SCHEDULING YAML

## Resource requests/limits ⭐ (covered above, but the concept)

**⭐ Q: Requests vs limits and QoS classes?**
Requests = guaranteed minimum the scheduler reserves (used for placement). Limits = hard ceiling (exceed memory → OOMKilled exit 137; exceed CPU → throttled, not killed). The request/limit ratio sets QoS: Guaranteed (requests==limits), Burstable (requests<limits), BestEffort (none) — which determines eviction order under pressure (cross-ref the QoS deep-dive). *Why: directly ties to your OOMKilled incident.*

## Probes ⭐

**⭐ Q: The three probes and a real misconfiguration scenario?**
Readiness (gates traffic — fail = removed from endpoints), Liveness (gates restart — fail = killed), Startup (grace period for slow-starting apps — disables the others until ready). Your CrashLoopBackOff story is the perfect illustration: liveness initialDelay too short for a slow startup → kubelet kills it before it's ready → fix with higher initialDelaySeconds + a startupProbe. (Cross-ref earlier.)

```yaml
        startupProbe:           # for slow starters
          httpGet: {path: /healthz, port: 8080}
          failureThreshold: 30  # allows 30 × periodSeconds for startup
          periodSeconds: 10
```

## Affinity, taints, topology spread

**Q: nodeSelector vs affinity vs taints (the YAML)?**
nodeSelector = simplest, exact label match. nodeAffinity = richer (required/preferred, operators). podAffinity/anti-affinity = schedule relative to other pods. Taints (on nodes) + tolerations (on pods) = repel/allow. topologySpreadConstraints = distribute replicas across zones/nodes. (Cross-ref the scheduling deep-dive.)

```yaml
      topologySpreadConstraints:    # spread replicas across zones for HA
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels: {app: checkout}
```

**⭐ Interview point:** topologySpreadConstraints is the modern, lighter way to ensure replicas spread across AZs — key for surviving a zone failure.

## PodDisruptionBudget

**Q: What's a PDB and what does it protect against?**
Sets minimum available pods during *voluntary* disruptions (node drains, upgrades, autoscaler scale-down). Does NOT protect against involuntary disruptions (node crash). (Cross-ref the failure-modes deep-dive.)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: checkout-pdb
spec:
  minAvailable: 2              # never drain below 2 pods
  selector:
    matchLabels: {app: checkout}
```

## HPA ⭐

**⭐ Q: HorizontalPodAutoscaler — YAML and how it works?**
Scales replica count based on a metric vs target. Formula: `desired = ceil(current × currentMetric / targetMetric)`. Needs resource requests set (to compute CPU %). Polls metrics-server ~every 15s.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # scale to keep avg CPU ~70%
```

---

# PART 3 — NETWORKING YAML

## Ingress ⭐

**⭐ Q: Ingress vs Service, and the YAML?**
A Service (LoadBalancer) is L4, one cloud LB per service — expensive. An Ingress is L7 HTTP routing (host/path) to multiple services behind a *single* LB, with TLS termination. An Ingress controller (nginx) runs as pods and implements the Ingress rules. (Cross-ref the Ingress-vs-LB networking deep-dive.)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts: [app.example.com]
    secretName: app-tls         # TLS cert stored as a Secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /checkout
        pathType: Prefix
        backend:
          service:
            name: checkout
            port:
              number: 80
      - path: /orders
        pathType: Prefix
        backend:
          service:
            name: orders
            port:
              number: 80
```

**⭐ Interview point:** One Ingress + one LoadBalancer routes to many services by path/host — that's why you use Ingress instead of a LoadBalancer Service per app.

## NetworkPolicy ⭐

**⭐ Q: NetworkPolicy YAML and the critical gotcha?**
Defines allowed ingress/egress by label selector. **The gotcha: the CNI must support it** (Calico, Cilium do; Flannel alone doesn't) — apply a policy on an unsupporting CNI and it silently does nothing. Also: once a policy selects a pod, that pod becomes default-deny for that direction. (Cross-ref networking deep-dive.)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: checkout-allow-from-gateway
spec:
  podSelector:
    matchLabels: {app: checkout}     # applies to checkout pods
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: api-gateway}   # only gateway can reach checkout
    ports:
    - protocol: TCP
      port: 8080
```

---

# PART 4 — STORAGE YAML

**⭐ Q: PV / PVC / StorageClass — the relationship and YAML?**
StorageClass defines *how* to provision (which provisioner, disk type). PVC = a request for storage. PV = the actual storage. Dynamic provisioning: PVC → StorageClass → PV created automatically. (Cross-ref storage deep-dive.)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]      # one node at a time (cloud block disk)
  storageClassName: managed-premium   # Azure managed disk
  resources:
    requests:
      storage: 20Gi
```

**⭐ Interview point:** Access modes — ReadWriteOnce (one node, cloud block disks like Azure Disk), ReadWriteMany (many nodes, needs Azure Files/NFS). RWO is why a Deployment with a cloud disk can't easily run pods across nodes — cross-ref the storage deep-dive.

---

# PART 5 — RBAC YAML

**⭐ Q: RBAC — Role, ClusterRole, RoleBinding, and the YAML?**
RBAC controls who can do what. Role (namespace-scoped) / ClusterRole (cluster-wide) define *permissions*. RoleBinding / ClusterRoleBinding *attach* those permissions to a user, group, or ServiceAccount.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]      # read-only on pods
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**⭐ Interview point:** Roles define permissions, Bindings connect them to identities. Principle of least privilege — give the minimum verbs/resources needed. ServiceAccounts are how *pods* authenticate to the API (cross-ref the admission/authz part of the internals deep-dive).

---

# PART 6 — SCENARIO-BASED QUESTIONS (interview gold)

**⭐ Scenario 1: "A pod is in CrashLoopBackOff. Walk me through debugging."**
`kubectl describe pod` (events + last state + exit code) → `kubectl logs --previous` (the crashed container's logs) → identify the cause: app error on startup, failed liveness probe, missing config/secret, OOMKilled (137), bad image. Your real story: liveness initialDelay too short for slow startup → fix with startupProbe. (Cross-ref.) *Narrate the investigation, don't just list commands.*

**⭐ Scenario 2: "A pod is Pending and won't schedule. Why?"**
`kubectl describe pod` → read the FailedScheduling event. Causes: insufficient resources (no node has enough for requests), taints not tolerated, nodeSelector/affinity matches nothing, unbound PVC, pod anti-affinity unsatisfiable. The gotcha: "insufficient resources" is based on *requests* not actual usage — nodes can be "full" on requests while CPU usage is low. (Cross-ref.)

**⭐ Scenario 3: "A Service has endpoints but traffic intermittently fails. Debug it."**
Possible: a pod is Ready but app isn't truly serving (readiness too lenient); kube-proxy reconciliation lag after endpoint changes; a terminating pod still in endpoints getting traffic mid-shutdown (fix: preStop hook + graceful shutdown); conntrack table full. Check `kubectl get endpoints`, the readiness probe, and the pod termination handling. (Cross-ref.)

**⭐ Scenario 4: "A Deployment rollout is stuck — `kubectl rollout status` hangs. Why?"**
`kubectl get rs` (old vs new replica counts), `kubectl describe` the new ReplicaSet's pods. Causes: new pods failing readiness (rollout won't progress past maxUnavailable), image pull errors, insufficient resources for surge pods, CrashLoopBackOff in new version, PDB blocking old pods from terminating. The rollout gates on new pods becoming Ready. (Cross-ref.)

**Scenario 5: "Pod is Running but receives no traffic. Walk through it."**
Check READY column — Running but 0/1 Ready means readiness failing → not added to Service endpoints → no traffic. `kubectl get endpoints <svc>` shows `<none>`. `kubectl describe pod` shows the readiness probe failure. This is the "readiness misconfig = silent traffic loss" pattern from your Athena incident 05. (Cross-ref.)

**Scenario 6: "A node went NotReady. What happens to its pods and when?"**
kubelet stops heartbeating → node controller marks NotReady after ~40s, applies a NotReady:NoExecute taint → pods without toleration evicted after tolerationSeconds (default 300s) → recreated elsewhere by their controller. The 5-min delay avoids mass rescheduling on transient blips. (Cross-ref.)

**Scenario 7: "A pod is stuck Terminating forever. Why and how to handle?"**
Causes: a finalizer waiting on cleanup that never completes; kubelet on the node is down; volume won't unmount; process ignoring SIGTERM. `kubectl describe`. If a stuck finalizer, patch it out carefully. If node is dead, force-delete (`--grace-period=0 --force`) — but understand it just removes it from the API; dangerous for StatefulSets (split-brain risk). (Cross-ref.)

**Scenario 8: "How do you do a zero-downtime deployment?"**
RollingUpdate with `maxUnavailable: 0` and `maxSurge: 1` (new pods come up before old ones go down); proper readiness probes (don't route until truly ready); preStop hook + graceful shutdown (drain in-flight requests); PodDisruptionBudget; multiple replicas. For higher safety, blue-green or canary (cross-ref your progressive-delivery prep). *Why: ties together probes, rollout strategy, and PDBs.*

**Scenario 9: "Memory usage keeps growing and pods get OOMKilled every few hours. Debug it."**
Your real incident. Memory climbing on a steady slope regardless of traffic = leak not load. Confirm OOMKilled via exit 137 in last state. Correlate with logs/traces to find the leaking endpoint. Short-term: increase limit / add replicas; real fix: the code leak. Prevention: alert at 80% of limit *before* the OOMKill. (Cross-ref your incident story.)

**Scenario 10: "How would you debug a pod that can't reach an external service / another pod?"**
Exec in (or use a netshoot debug pod). Test DNS (`nslookup`), connectivity (`nc -zv`), check NetworkPolicy blocking egress, check if it's pod vs node level, check the target's health. Isolate layer by layer: DNS → connectivity → policy → target. (Cross-ref the networking debugging.)

---

# PART 1 — TERRAFORM FUNDAMENTALS

## Basics

**⭐ Q: What is Terraform and what problem does it solve?**
Terraform is an Infrastructure-as-Code tool — you declare your desired infrastructure in config files (HCL), and Terraform makes the real world match it. It solves manual, error-prone, inconsistent provisioning: infra becomes version-controlled, reviewable, repeatable, and auditable, with no config drift or snowflake servers. *Why: the "why IaC" opener.*

**⭐ Q: Declarative vs imperative — which is Terraform and why does it matter?**
Terraform is **declarative** — you describe the *desired end state* ("I want 3 VMs"), and Terraform figures out *how* to get there (create, update, or delete to reach it). Imperative (like a bash script) describes the *steps*. Declarative means you can run it repeatedly and it converges to the same state (idempotent) without you tracking what already exists. *Why: tests core understanding; declarative + idempotent is the whole point.*

**Q: What's the difference between Terraform and Ansible / CloudFormation?**
Terraform: cloud-agnostic provisioning (creating infra) across many providers, declarative, state-based. CloudFormation: AWS-only provisioning. Ansible: primarily configuration management (configuring existing servers — installing software, managing config), procedural. They overlap but Terraform is for *provisioning infra*, Ansible for *configuring it*. Often used together. *Why: you migrated FROM CloudFormation, so expect this comparison — and you can speak to it honestly.*

**Q: What's the core Terraform workflow?**
`terraform init` (download providers, set up backend) → `terraform plan` (preview changes — the diff between config and real state) → `terraform apply` (execute) → `terraform destroy` (tear down). *Why: basic fluency.*

**⭐ Q: What's the difference between `plan`, `apply`, and `refresh`?**
`plan` = dry run; shows what *would* change by comparing config vs state vs real infrastructure — no changes made. `apply` = executes the plan, makes real changes, updates state. `refresh` = updates state to match real-world resources without changing infra (now folded into plan/apply by default). *Why: `plan` before `apply` is the safety habit; tests if you understand the dry-run.*

## State

**⭐ Q: What is Terraform state and why is it critical?**
State (`terraform.tfstate`) is Terraform's record of what it has provisioned — the mapping between your config and real resources (resource IDs, attributes, dependencies). Terraform compares config → state → reality to decide what to change. Without state, Terraform wouldn't know what already exists and would try to recreate everything. *Why: state is THE concept that separates people who understand Terraform from people who just ran it.*

**⭐ Q: Why should state be stored remotely, not locally / in Git?**
Local state breaks team collaboration (everyone has a different copy) and risks loss. Git is worse — state often contains *secrets in plaintext* (passwords, keys), so committing it is a security leak. A remote backend (Azure Storage, S3, Terraform Cloud) gives a single shared source of truth, encryption at rest, and — critically — state locking. *Why: a top Terraform question; the "state has secrets" point shows real awareness.*

**⭐ Q: What happens if two engineers run `terraform apply` at the same time?**
Without locking, they can corrupt the state file or make conflicting changes (race condition). The solution is **state locking** — a remote backend acquires an exclusive lock during operations (Azure Storage uses a blob lease, S3 uses a DynamoDB lock table). The second apply waits or fails until the lock releases. *Why: the single most-asked Terraform scenario; the answer is "remote backend with locking."*

**Q: What is `terraform import`?**
It brings *existing* infrastructure (created manually or by another tool) under Terraform management by adding it to state — without recreating it. Useful when adopting Terraform for resources that already exist. You still have to write the matching config. *Why: comes up when discussing brownfield adoption / migrating to Terraform.*

---

# PART 2 — MODULES, VARIABLES, STRUCTURE

**⭐ Q: What is a Terraform module and why use one?**
A module is a reusable, parameterized package of resources — a folder of `.tf` files you can call with different inputs. Instead of copy-pasting the same VNet/AKS config across dev/staging/prod, you write a module once and call it with per-environment variables. Promotes DRY, consistency, and easier maintenance. Every Terraform config is technically a "root module" that can call child modules. *Why: modules are how you scale Terraform across an org — the key intermediate concept.*
*(Your honest add if drilled deep: "I've worked within existing modules — passing variables, environment configs — during our CloudFormation-to-Terraform migration. Authoring complex modules from scratch is something I'm building.")*

**Q: Variables, outputs, locals — what's each?**
**Variables** (`variable`) = inputs to a module (parameterize it). **Outputs** (`output`) = values a module exposes to its caller or to other modules (e.g., the VNet ID). **Locals** (`locals`) = computed/reused values within a config (like local constants, DRY within a file). *Why: tests understanding of how data flows through Terraform.*

**Q: How do you manage multiple environments (dev/staging/prod)?**
Common approaches: separate state files per environment (separate backend keys), call shared modules with per-env variable files (`dev.tfvars`, `prod.tfvars`), or use a directory structure (`environments/dev/`, `environments/prod/`). Terraform **workspaces** are another option but are often discouraged for prod separation because they share config and it's easy to apply to the wrong one. *Why: real-world structure question; mentioning the workspace caveat shows maturity.*

**Q: What are Terraform workspaces and their limitation?**
Workspaces let one config have multiple independent states (e.g., `dev`, `prod`) without duplicating files. Limitation: they share the *same* code, so it's easy to accidentally apply to the wrong environment, and they don't isolate well for very different environments. Many teams prefer separate directories/state for prod isolation. *Why: a known gotcha — knowing *not* to over-rely on workspaces is the senior signal.*

**Q: What's the difference between `count` and `for_each`?**
Both create multiple instances of a resource. `count` uses an integer index (`count = 3`) — but if you remove an item from the middle, everything after it re-indexes and gets recreated. `for_each` uses a map/set with stable keys — adding/removing one item doesn't disturb the others. Prefer `for_each` when items can change. *Why: tests deeper module-authoring knowledge; the re-indexing gotcha is advanced.*

---

# PART 3 — DRIFT, DEPENDENCIES, LIFECYCLE

**⭐ Q: What is drift and how do you detect/handle it?**
Drift is when real infrastructure differs from what's in state — almost always because someone made a *manual change* in the cloud console. `terraform plan` detects it (it refreshes real state and shows the difference from config). Handling: either `apply` to revert reality back to config, or update the config to match the intended change, then apply. Best practice: prevent drift by making *all* changes through Terraform — no manual console edits. *Why: drift is a core operational reality; the "no manual changes" discipline is the key takeaway.*

**Q: How does Terraform handle dependencies between resources?**
Mostly *implicitly* — if resource B references resource A's attribute (e.g., a subnet referencing a VNet's ID), Terraform infers A must be created first and builds a dependency graph. For cases with no direct reference, `depends_on` declares an explicit dependency. Terraform creates/destroys in dependency order, parallelizing where it can. *Why: tests understanding of the dependency graph — the engine behind apply ordering.*

**Q: What does the `lifecycle` block do?**
Controls resource behavior: `create_before_destroy` (make the new one before destroying the old — avoids downtime), `prevent_destroy` (guard against accidental deletion of critical resources like a prod database), `ignore_changes` (ignore drift on specific attributes managed outside Terraform). *Why: shows production-hardening awareness.*

**Q: What's the difference between changing a resource that updates in-place vs forces replacement?**
Some attribute changes can be applied in-place (e.g., a tag); others require destroying and recreating the resource (e.g., changing certain immutable properties). `terraform plan` shows this — `~` = update in place, `-/+` = destroy and recreate. The danger: a small config change can trigger a replacement of a critical resource (downtime/data loss). Always read the plan carefully. *Why: a real "I almost deleted prod" scenario; reading the plan symbols is crucial.*

---

# PART 4 — PROVISIONERS, FUNCTIONS, SECRETS

**Q: What are provisioners and why are they discouraged?**
Provisioners (`local-exec`, `remote-exec`) run scripts during apply (e.g., run a command on a new VM). They're a last resort because they're not declarative, don't track state, and break idempotency — if a provisioner fails, the resource is marked tainted. Prefer cloud-init, configuration management (Ansible), or baked images instead. *Why: tests whether you know the "right" declarative way vs the escape hatch.*

**⭐ Q: How do you handle secrets in Terraform?**
Never hardcode secrets in `.tf` files (they'd be in version control). Mark variables `sensitive = true` (hides them from plan/apply output). Pull secrets from a secret manager (Azure Key Vault, Vault, AWS Secrets Manager) at apply time via data sources. And crucially — **state contains secrets in plaintext**, so the remote backend must be encrypted and access-controlled. *Why: security question; the "state holds secrets" point is the depth signal.*

**Q: What's a data source vs a resource?**
A `resource` block *creates/manages* infrastructure. A `data` source *reads* existing information (e.g., look up an existing VNet, fetch a Key Vault secret, get the latest AMI) without managing it. Data sources let you reference things Terraform doesn't own. *Why: tests understanding of read vs manage.*

**Q: What's a `.terraform.lock.hcl` file?**
The dependency lock file — it pins provider versions so everyone on the team and CI uses the exact same provider versions, ensuring reproducible runs. Commit it to version control. *Why: reproducibility question; analogous to package-lock.json.*

---

# PART 5 — SCENARIO-BASED QUESTIONS

**⭐ Scenario 1: "Someone manually changed a resource in the cloud console. What happens on the next `terraform apply`?"**
On the next `plan`, Terraform refreshes real state, detects the drift, and shows it wants to revert the manual change back to match the config. If you `apply`, it overwrites the manual change. The lesson: manual changes get clobbered — all changes should go through Terraform. If the manual change was intentional, you update the config (or use `ignore_changes`) first. *Why: the canonical drift scenario.*

**⭐ Scenario 2: "Two engineers ran apply simultaneously and the state looks corrupted. What went wrong and how do you prevent it?"**
No state locking — concurrent applies raced and corrupted the state file. Prevention: remote backend with locking (Azure Storage blob lease / S3 + DynamoDB). Recovery: restore state from a backup/version (remote backends version state), or carefully reconcile. *Why: the locking scenario; tests prevention + recovery.*

**Scenario 3: "`terraform plan` wants to destroy and recreate your production database. You only changed a tag. What do you do?"**
Stop — do NOT apply blindly. Read the plan: a `-/+` means replacement. Figure out *why* a tag change triggers replacement (maybe it's actually a different attribute, or a provider quirk). For a critical resource, use `prevent_destroy` in a lifecycle block as a guardrail, and investigate before applying. Never apply a plan that destroys prod data without understanding it. *Why: tests the "read the plan carefully" discipline and lifecycle guardrails — a real near-miss scenario.*

**Scenario 4: "Your state file is out of sync with reality — a resource exists in the cloud but not in state (or vice versa). How do you fix it?"**
Resource exists in cloud but not state → `terraform import` to bring it under management. Resource in state but deleted in cloud → `terraform plan` will want to recreate it (or use `terraform state rm` to forget it if intentional). For more surgery, `terraform state` subcommands (`mv`, `rm`, `list`) manipulate state safely. *Why: tests state management depth — `import` and `state rm` are the tools.*

**Scenario 5: "How do you safely make changes to shared production infrastructure with a team?"**
Remote state with locking; everything via pull requests with `terraform plan` output reviewed before merge (run plan in CI); separate state per environment; use `prevent_destroy` on critical resources; apply through a pipeline (not laptops) so it's audited and consistent; never make manual console changes. *Why: tests team/production workflow maturity — exactly what a DevOps role cares about.*

**Scenario 6: "A `terraform apply` failed halfway through. What's the state now and what do you do?"**
Terraform applies resources in dependency order; a mid-apply failure means *some* resources were created (and are in state) and others weren't. State reflects what succeeded. You investigate the error (often a quota, permission, or dependency issue), fix it, and re-run apply — Terraform is idempotent, so it picks up where it left off, creating only what's missing. A resource that failed mid-creation may be marked *tainted* and recreated next apply. *Why: tests understanding that Terraform is resumable and state tracks partial progress.*

---

# PART 1 — DOCKER FUNDAMENTALS

## Basics

**⭐ Q: What's the difference between an image and a container?**
An image is a read-only template — a layered filesystem plus metadata (entrypoint, env, ports). A container is a running (or stopped) instance of an image, with a thin writable layer on top. Image = the class, container = the object. You can run many containers from one image. *Why: the most fundamental Docker concept; everything builds on it.*

**⭐ Q: How is a container different from a VM?**
A VM virtualizes hardware and runs a full guest OS with its own kernel — heavy (GBs, slow boot). A container virtualizes the OS — it shares the *host kernel* and isolates only the process using Linux namespaces and cgroups — lightweight (MBs, starts in milliseconds). Containers are processes with isolation, not mini-machines. *Why: tests whether you understand containers aren't "lightweight VMs" — they're isolated host processes.*

**⭐ Q: What actually makes a container isolated? (the kernel primitives)**
Two Linux kernel features. **Namespaces** isolate *what a process can see* — its own PID namespace (own process tree), network namespace (own interfaces/IPs), mount namespace (own filesystem view), user namespace, etc. **cgroups** (control groups) limit *what a process can use* — CPU, memory, I/O. Docker/containerd just orchestrate these kernel features. *Why: senior-level — shows containers are Linux features, not magic. Comes up often.*

**Q: What is the Docker daemon, client, and registry?**
The **client** (`docker` CLI) sends commands. The **daemon** (`dockerd`) does the actual work — building, running, managing containers. The **registry** (Docker Hub, ACR, ECR) stores images. The client talks to the daemon over a socket/API; the daemon pulls/pushes to the registry. *Why: explains the architecture and why "docker build" runs server-side on the daemon.*

**Q: What's the container lifecycle / common commands?**
`docker build` (image from Dockerfile) → `docker run` (create + start a container) → `docker ps` (list running) → `docker stop`/`start`/`restart` → `docker rm` (remove container) / `docker rmi` (remove image) → `docker logs`, `docker exec -it <c> bash` (shell into running container). *Why: basic fluency check.*

**Q: `docker stop` vs `docker kill`?**
`docker stop` sends SIGTERM, waits (default 10s grace), then SIGKILL — graceful. `docker kill` sends SIGKILL immediately — forceful. Always prefer `stop` so the app can clean up. *Why: maps directly to Kubernetes pod termination — same SIGTERM-then-SIGKILL pattern.*

## Image layers & internals

**⭐ Q: How do Docker image layers work?**
Each instruction in a Dockerfile (`RUN`, `COPY`, `ADD`) creates a read-only layer. Layers stack via a union filesystem (overlay2). Containers add a thin writable layer on top. Layers are **cached and shared** — if two images share a base, that base is stored once. *Why: foundational to understanding caching, image size, and build speed.*

**⭐ Q: How does build caching work, and how do you optimize a Dockerfile for it?**
Docker caches each layer; on rebuild, it reuses a cached layer if that instruction *and its inputs* are unchanged. Once a layer's cache is invalidated, every layer after it rebuilds. Optimization: put rarely-changing instructions first (install dependencies) and frequently-changing ones last (copy source code). Classic example — copy `package.json` and run `npm install` *before* copying the rest of the source, so a code change doesn't bust the dependency cache. *Why: this is the #1 Dockerfile optimization question.*

**Q: What's a dangling image and how do you clean up?**
A dangling image is an untagged layer left over from rebuilds (`<none>:<none>`). `docker image prune` removes them; `docker system prune -a` removes unused images, containers, networks, and build cache. *Why: disk-pressure cleanup — directly relevant to your node-cleanup work.*

**Q: Where does a container's data go, and what happens when it's deleted?**
By default, data written inside a container lives in its writable layer and is *destroyed* when the container is removed. To persist data, use volumes or bind mounts. *Why: leads into the volumes question and the "why are databases hard in containers" theme.*

---

# PART 2 — DOCKERFILE (in depth)

**⭐ Q: Explain the common Dockerfile instructions.**
- `FROM` — base image (the starting layer)
- `WORKDIR` — sets the working directory
- `COPY` — copies files from build context into the image
- `ADD` — like COPY but also handles URLs and auto-extracts tarballs (prefer COPY unless you need those)
- `RUN` — executes a command at *build time* (creates a layer)
- `ENV` — sets environment variables
- `EXPOSE` — documents which port the app listens on (doesn't actually publish it)
- `CMD` — default command at *runtime* (overridable)
- `ENTRYPOINT` — the fixed executable at runtime
- `ARG` — build-time variable
*Why: you need to read and write these fluently.*

**⭐ Q: CMD vs ENTRYPOINT — the difference and how they combine?**
`ENTRYPOINT` is the fixed executable that always runs. `CMD` provides default arguments that are easily overridden at `docker run`. Common pattern: `ENTRYPOINT ["python", "app.py"]` + `CMD ["--port=8080"]` → runs `python app.py --port=8080`, but you can override the args. If you only use CMD, the whole thing is overridable. *Why: a classic Dockerfile question; people confuse these constantly.*

**Q: `COPY` vs `ADD` — which and why?**
Both copy files into the image. `ADD` additionally extracts local tarballs and can fetch URLs — but that implicit behavior is surprising and a security risk. Best practice: use `COPY` for plain file copies (explicit, predictable), use `ADD` only when you specifically need tarball extraction. *Why: tests knowledge of best practices.*

**Q: `RUN` vs `CMD` vs `ENTRYPOINT` — build-time vs runtime?**
`RUN` executes during *build* and bakes the result into a layer (e.g., installing packages). `CMD`/`ENTRYPOINT` define what runs when the *container starts*. So `RUN apt-get install` happens once at build; `CMD ["./server"]` happens every time the container runs. *Why: a common point of confusion.*

**⭐ Q: What's a multi-stage build and why is it important?**
A Dockerfile with multiple `FROM` stages. You build/compile in a heavy stage with all the build tools, then `COPY --from=builder` only the final artifact into a slim runtime stage. Benefits: dramatically smaller final image, smaller attack surface (no compilers/build tools shipped), faster pulls. Example: compile a Go binary in a `golang` stage, copy just the binary into `alpine` or `scratch`.
```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o server .

FROM alpine:3.19
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
```
*Why: THE most-asked Dockerfile optimization; expect to explain it.*

**⭐ Q: How do you reduce Docker image size? (list the techniques)**
Multi-stage builds; slim/minimal base images (`alpine`, `distroless`, `slim`); combine `RUN` commands with `&&` to reduce layers; clean package caches in the *same* layer (`apt-get install ... && rm -rf /var/lib/apt/lists/*`); use `.dockerignore` to avoid copying junk (node_modules, .git) into the build context; remove build dependencies. *Why: image size affects pull speed, storage cost, and attack surface — a frequent question.*

**Q: What's `.dockerignore` and why does it matter?**
Like `.gitignore` but for the Docker build context — it excludes files from being sent to the daemon and copied into the image (`.git`, `node_modules`, secrets, large test data). Without it, `COPY . .` drags everything in, bloating the image and slowing builds. *Why: easy to forget, real impact; also a security point (don't copy secrets in).*

**⭐ Q: Why shouldn't you run a container as root, and how do you fix it?**
If a container runs as root and there's a container-escape vulnerability, root in the container can become root on the host. Fix: create and switch to a non-root user in the Dockerfile:
```dockerfile
RUN adduser --disabled-password appuser
USER appuser
```
Many security policies and Kubernetes admission controllers reject root containers. *Why: a top container-security question.*

**Q: How do you handle secrets in a Docker build?**
Never `COPY` secrets in or put them in `ENV` (they persist in image layers and history — `docker history` reveals them). Use BuildKit secret mounts (`--mount=type=secret`) which don't persist in the image, or inject secrets at *runtime* via environment variables / mounted secret files. *Why: a common security mistake interviewers probe.*

**Q: What's the difference between an image built `FROM scratch` vs `FROM alpine` vs `FROM ubuntu`?**
`scratch` = empty, no OS at all (only works for fully static binaries like Go) — smallest, most secure. `alpine` = ~5MB minimal Linux with a package manager (musl libc — occasional compatibility quirks). `ubuntu/debian` = full-featured but large (~70MB+). Choose the smallest that runs your app. `distroless` (Google) is a popular middle ground — minimal but with libc. *Why: base-image choice affects size, security, and compatibility.*

---

# PART 3 — DOCKER NETWORKING (in depth)

**⭐ Q: What are the Docker network drivers / types?**
- **bridge** (default) — containers on a private internal network on the host; they talk to each other and reach outside via NAT. Each container gets an internal IP.
- **host** — the container shares the host's network namespace directly (no isolation, no NAT) — fast, but ports conflict with the host.
- **none** — no networking at all.
- **overlay** — spans multiple Docker hosts (for Swarm/multi-host) — containers on different hosts communicate as if on one network.
- **macvlan** — gives the container its own MAC and appears as a physical device on the network.
*Why: networking is heavily tested; knowing when to use each is the depth signal.*

**⭐ Q: How do containers on the same bridge network communicate?**
On a *user-defined* bridge network, Docker provides automatic DNS — containers resolve each other by container name. So `app` can reach `db` at `http://db:5432`. On the *default* bridge, this DNS doesn't work — you'd need `--link` (legacy) or IPs. *Why: this is why Compose works (it creates a user-defined network with name-based DNS) — a key practical point.*

**⭐ Q: What's the difference between `EXPOSE`, `-p`, and `--expose`?**
`EXPOSE` in the Dockerfile is documentation only — it declares the port the app uses but doesn't publish it. `-p 8080:80` (publish) at `docker run` actually maps host port 8080 to container port 80, making it reachable from outside the host. `-p` is what genuinely exposes a container to the host network. *Why: people confuse EXPOSE with actually publishing — a common gotcha.*

**Q: How does a container reach the internet, and how does outside traffic reach a container?**
Outbound: on a bridge network, the container's traffic is NAT'd through the host's IP. Inbound: traffic only reaches a container if you publish a port (`-p host:container`), which sets up an iptables DNAT rule on the host forwarding that host port to the container. *Why: explains the full traffic path, ties to your networking knowledge.*

**Q: What is `host.docker.internal`?**
A special DNS name that resolves to the host machine from inside a container — useful when a container needs to reach a service running on the host. *Why: practical, comes up in local dev scenarios.*

**Q: How does Docker networking relate to Kubernetes networking?**
Docker's per-host bridge networking is single-host. Kubernetes needs cross-host pod-to-pod networking with every pod getting a routable IP — so it uses CNI plugins (not Docker's bridge) to implement its flat network model. Docker handles the container runtime; the CNI handles the cluster network. *Why: bridges your Docker and Kubernetes knowledge — a nice senior connection.*

---

# PART 4 — VOLUMES & STORAGE

**⭐ Q: Volumes vs bind mounts vs tmpfs?**
- **Volume** — managed by Docker (`/var/lib/docker/volumes/...`), the preferred way to persist data; portable, backed up easily, decoupled from the host path.
- **Bind mount** — maps a specific host directory into the container; tight coupling to host filesystem, good for local dev (mounting source code).
- **tmpfs** — in-memory, never written to disk; for sensitive/temporary data.
*Why: data persistence is core; volumes are the right default.*

**Q: Why does container data disappear, and how do you persist it?**
Data in the container's writable layer is destroyed when the container is removed. Mount a volume to persist data beyond the container's life (`docker run -v mydata:/var/lib/postgresql/data`). *Why: explains why stateful apps need volumes.*

**Q: Why are databases tricky to run in containers?**
Containers are designed to be ephemeral and stateless; databases need durable state, careful volume management, backup/restore, and don't love being killed and rescheduled. You *can* run them with volumes, but in production, managed databases (RDS, Azure SQL) are usually preferred — same reasoning as not running stateful workloads in Kubernetes. *Why: ties to the architecture judgment from your earlier prep.*

---

# PART 5 — DOCKER COMPOSE (in depth)

**⭐ Q: What is Docker Compose and when do you use it?**
A tool to define and run multi-container applications with a single YAML file (`docker-compose.yml`) and one command (`docker compose up`). It's for *local development and testing* of multi-service apps — defining the services, their networks, volumes, and dependencies declaratively. *Why: tests whether you know its purpose (dev/test) vs Kubernetes (production orchestration).*

**⭐ Q: Walk through a typical docker-compose.yml.**
```yaml
services:
  web:
    build: ./web              # build from local Dockerfile
    ports:
      - "8080:80"             # host:container
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - dbdata:/var/lib/postgresql/data
volumes:
  dbdata:
```
Key parts: `services` (each container), `build` vs `image`, `ports`, `environment`, `depends_on`, `volumes` (named volume for persistence). *Why: expect to read or write one.*

**Q: How do services in Compose communicate with each other?**
Compose creates a default user-defined bridge network, so services reach each other *by service name* via Docker's built-in DNS. `web` connects to `db` at hostname `db`. No IP hardcoding needed. *Why: this is the practical payoff of user-defined bridge DNS.*

**Q: What does `depends_on` actually do — and what's the catch?**
`depends_on` controls *startup order* (start `db` before `web`), but it does NOT wait for the dependency to be *ready* — only *started*. So `web` might start before Postgres is accepting connections. The fix: a healthcheck with `condition: service_healthy`, or retry logic in the app. *Why: a classic gotcha — "depends_on doesn't mean ready."*

**Q: Compose vs Kubernetes — when each?**
Compose: local dev, simple multi-container apps on a single host, quick testing — simple and fast. Kubernetes: production orchestration across many nodes, with scaling, self-healing, rolling updates, service discovery, and high availability. Compose doesn't do multi-node scheduling or self-healing. *Why: tests that you know Compose isn't a production orchestrator.*

**Q: How do you scale a service in Compose?**
`docker compose up --scale web=3` runs 3 instances of `web`. But there's no built-in load balancing or self-healing like Kubernetes — it's limited. *Why: shows the boundary of Compose's capabilities.*

---

# PART 6 — SCENARIO-BASED QUESTIONS (the interview gold)

**⭐ Scenario 1: "Your container exits immediately after starting. How do you debug?"**
Check `docker ps -a` (it's in Exited state) → `docker logs <container>` to see why it exited → check the exit code (`docker inspect`). Common causes: the main process crashed on startup (bad config, missing env var, missing dependency), the CMD/ENTRYPOINT is wrong, or the foreground process finished (containers exit when PID 1 exits — e.g., running a non-blocking command). Key insight: a container runs only as long as its main process; if that process exits or runs in the background, the container stops. *Why: extremely common; tests the "container = one foreground process" mental model.*

**⭐ Scenario 2: "A container is consuming too much memory and getting killed. Walk me through it."**
The container exceeded its memory limit and was OOMKilled (exit 137). Check `docker stats` for live memory usage, `docker inspect` for the limit and the OOMKilled flag, and the app logs. Causes: memory leak in the app, limit set too low, or a workload spike. Fix: find the leak (heap profiling), or right-size the limit. This maps exactly to Kubernetes OOMKilled. *Why: ties to your real incident story; shows the container/cgroup memory relationship.*

**Scenario 3: "Two containers can't talk to each other. How do you debug?"**
Are they on the *same* network? (`docker network inspect <net>` to see attached containers). Are you using the default bridge (no name DNS) instead of a user-defined network? Use the container/service *name*, not localhost (localhost inside a container is the container itself, not other containers). Test from inside: `docker exec -it app ping db` or `curl db:5432`. *Why: tests Docker networking understanding; the "localhost is the container itself" point catches many people.*

**Scenario 4: "Your Docker image is 1.2GB. How do you make it smaller?"**
Multi-stage build (ship only the artifact, not the build tools); switch to a slim/alpine/distroless base; combine RUN layers and clean package caches in the same layer; add a `.dockerignore`; remove unnecessary dependencies. Check what's taking space with `docker history <image>` to see per-layer size. *Why: optimization scenario; `docker history` is the diagnostic tool that impresses.*

**Scenario 5: "A build that used to take 2 minutes now takes 10. The Dockerfile didn't change much. Why?"**
Likely the cache is being busted early. If you moved a `COPY . .` before the dependency install, or a frequently-changing file is copied before `RUN npm install`, every build re-runs the expensive install. Fix: reorder so dependencies are installed before source is copied. Check with `docker build` output — "Using cache" vs rebuilding tells you where the cache breaks. *Why: tests deep understanding of layer caching.*

**Scenario 6: "A container works on your machine but fails in production. What could differ?"**
Different architecture (built on ARM Mac, running on x86 — need multi-arch builds or `--platform`); different env variables/config; missing mounted volumes or secrets; network/DNS differences; the image tag is `latest` and production pulled a different version; resource limits in production that don't exist locally (OOM/CPU throttle). *Why: the classic "works on my machine" — tests breadth of thinking about environment differences.*

**Scenario 7: "How would you debug a running container you can't shell into (no bash/shell in the image)?"**
Distroless/scratch images have no shell. Options: `docker exec` won't help. Use `docker logs`, `docker inspect`, `docker stats`. For deeper debugging, attach an ephemeral debug container sharing the target's namespaces (`docker run --pid container:<target> --network container:<target> nicolaka/netshoot`) to get tools without modifying the image. In Kubernetes this is `kubectl debug` (ephemeral containers). *Why: advanced — distroless debugging is a real modern problem.*

---

# PYTHON (for SRE/DevOps)

## Beginner

**Q: List vs tuple vs dict vs set — when each?**
List `[]` = ordered, mutable, allows duplicates (a sequence of things). Tuple `()` = ordered, *immutable* (fixed records, can be dict keys). Dict `{}` = key-value pairs, fast lookup by key. Set = unordered, unique values, fast membership testing. *Why: "I need to count unique errors" → set or dict; "I need fast lookup" → dict. Picking the right structure is what interviews probe.*

**Q: How do you read a file safely?**
`with open("file.txt") as f: for line in f: ...`. The `with` block auto-closes the file even if an error occurs. Iterating line-by-line streams the file instead of loading it all into memory. *Why: the `with` context manager is the correct, leak-free pattern; reading `f.read()` on a huge log file is a memory mistake interviewers catch.*

**Q: Difference between `==` and `is`?**
`==` compares values (are they equal?). `is` compares identity (are they the same object in memory?). Use `==` for value checks; use `is` only for `None`, `True`, `False` (`if x is None`). *Why: `if x is "string"` is a subtle bug — interviewers test this.*

**⭐ Q: How do you handle errors in Python?**
`try/except`: `try: risky() except SpecificError as e: handle(e)`. Catch *specific* exceptions, not bare `except:` (which hides real bugs and catches things like KeyboardInterrupt). Use `finally:` for cleanup that must always run, or `else:` for code that runs only if no exception. *Why: robust error handling is what separates a script that fails gracefully from one that crashes mid-run — core for ops automation.*

## Intermediate

**⭐ Q: How would you parse a log file and count error types?**
Stream the file line-by-line, use a `collections.Counter` to tally:
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
*Why: this is THE canonical ops Python task. `Counter.most_common()` is the tool. Your log-analyzer script does exactly this.*

**Q: How do you parse JSON in Python?**
`import json`. `json.loads(string)` parses a JSON string to a dict; `json.load(file)` reads from a file object; `json.dumps(obj)` serializes back to a string. *Why: APIs and structured logs are JSON; this is why you reach for Python over Bash for structured data.*

**⭐ Q: How do you make an HTTP API call in Python?**
Using `requests`:
```python
import requests
resp = requests.get(url, headers={"Authorization": f"Bearer {token}"}, timeout=10)
resp.raise_for_status()      # raises on 4xx/5xx
data = resp.json()
```
Always set a `timeout` (without it, a hung server hangs your script forever) and check the status. *Why: calling the Kubernetes API, cloud APIs, webhooks — and the `timeout` + `raise_for_status` details show production awareness.*

**Q: How do you run a shell command from Python and capture output?**
`subprocess.run`:
```python
result = subprocess.run(["kubectl", "get", "pods"], capture_output=True, text=True, check=True, timeout=30)
print(result.stdout)
```
Pass args as a *list* (not a string) to avoid shell-injection, use `check=True` to raise on failure, `text=True` for string output, and a `timeout`. *Why: ops scripts often wrap CLI tools; the list-not-string detail is a security signal.*

**Q: What's a list comprehension and when should you use it?**
A concise way to build a list: `[x*2 for x in items if x > 0]`. Readable for simple transforms/filters. But don't cram complex logic into one — if it needs multiple conditions or side effects, use a regular loop for readability. *Why: shows Pythonic style without overusing it.*

**Q: Difference between `append()` and `extend()`? Mutable default argument trap?**
`append(x)` adds one item; `extend([a,b])` adds each item of an iterable. The trap: `def f(items=[])` — the default list is created *once* and shared across calls, so it accumulates. Use `def f(items=None): items = items or []`. *Why: the mutable-default-argument bug is a classic Python gotcha interviewers love.*

## Advanced

**Q: What's the difference between a list and a generator, and why does it matter for large data?**
A list holds all items in memory at once. A generator (`yield`, or `(x for x in ...)`) produces items lazily, one at a time, using almost no memory. For processing a huge log file or streaming API results, a generator avoids loading everything into RAM. *Why: memory-efficiency on large data is a real ops concern — processing a 10GB log shouldn't OOM your script.*

**⭐ Q: How do you write a script that's safe to run repeatedly (idempotent) and handles partial failure?**
Check state before acting (does the resource already exist? skip it). Wrap each operation in try/except so one failure doesn't abort everything. Log what was done. For external calls, add retries with backoff. Make operations reversible or checkpoint progress so a re-run resumes rather than duplicates. *Why: idempotency is core to automation — a remediation script must be safe to re-run after a partial failure.*

**Q: How do you implement retries with backoff?**
```python
import time
for attempt in range(max_retries):
    try:
        return call_api()
    except TransientError:
        if attempt == max_retries - 1:
            raise
        time.sleep(2 ** attempt)   # exponential backoff
```
Add jitter (randomness) in production to avoid thundering-herd. *Why: directly maps to the resilience patterns SRE roles ask about — retries with backoff and jitter.*

**Q: What are context managers and how do you write one?**
They manage setup/teardown automatically (the `with` statement). Built-in for files, locks, connections. You can write your own with `@contextmanager`:
```python
from contextlib import contextmanager
@contextmanager
def timer():
    start = time.time()
    yield
    print(f"took {time.time()-start:.2f}s")
```
*Why: ensures cleanup (closing connections, releasing locks) even on error — the Python equivalent of Bash's `trap`.*

**Q: How do you handle secrets and config in a Python script?**
Never hardcode. Read from environment variables (`os.environ["TOKEN"]` or `os.getenv` with a default), or a secret manager / mounted secret file. Keep config separate from code. Don't log secrets. *Why: security awareness; same principle as everywhere — secrets come from the environment at runtime, not the source.*

**Q: What's the difference between `os.environ.get()` and `os.environ[]`?**
`os.environ["KEY"]` raises KeyError if the variable is missing (fail-fast — good when the secret is required). `os.environ.get("KEY", default)` returns a default if missing (graceful — good for optional config). *Why: choosing the right one shows you think about "is this required or optional?"*

**Q: When should you NOT use Python, and reach for Bash instead?**
For simple command orchestration — chaining a few CLI commands, basic file operations, glue between tools — Bash is lighter and more direct. Reach for Python when you need structured data parsing (JSON/YAML), complex logic, error handling, API calls, or maintainability. *Why: judgment about the right tool is a senior signal — it's not "Python for everything."*

---

# Connecting to YOUR scripts

Your log-analyzer Python script is the perfect interview artifact. Be ready to explain:

- **Why Python, not Bash** → "It parses structured log data and aggregates error patterns — `json` parsing and `Counter` make that clean; doing it in Bash would be fragile."
- **`Counter` + `most_common`** → "tallies error signatures and surfaces the top ones instantly, instead of scrolling thousands of lines."
- **The normalize/regex step** → "strips out IDs, GUIDs, timestamps so similar errors group together — otherwise every line looks unique."
- **Streaming the file** → "I iterate line-by-line so it doesn't load a huge log into memory."
- **What you'd improve** → "add argparse for flexibility, handle malformed JSON gracefully, maybe output to a structured format for dashboards."

---

# Drilling priority

The ⭐ ones are what ops/SRE Python interviews actually hit:
- **Error handling** (try/except, specific exceptions) — guaranteed
- **Parse a log + count with Counter** — the canonical ops task
- **HTTP API call with requests** (timeout, raise_for_status) — very common
- **Idempotent/safe-to-rerun automation** — the senior-level concept

If you can write a log-parser, make a safe API call with error handling, and explain idempotency — plus walk through your own script — you're solidly mid-level on Python for SRE/DevOps. They're not testing whether you can implement quicksort; they're testing whether you can automate ops tasks safely.

---

# BASH

## Beginner

**Q: What's the shebang line and why does it matter?**
`#!/bin/bash` (or `#!/usr/bin/env bash`) on line 1 tells the system which interpreter runs the script. Without it, the script runs in whatever shell invoked it, which may not be Bash — and Bash-specific syntax (arrays, `[[ ]]`) breaks under plain `sh`. *Why: a script that works for you but breaks on a server is often a shebang/shell mismatch.*

**Q: Single quotes vs double quotes?**
Single quotes `'...'` are literal — no variable expansion. Double quotes `"..."` allow variable and command expansion (`$var`, `$(cmd)`) but preserve spaces. Rule: use double quotes around variables (`"$var"`) to avoid word-splitting; use single quotes for literal strings. *Why: unquoted variables are the #1 source of Bash bugs.*

**⭐ Q: Why must you quote variables — `"$var"` instead of `$var`?**
If a variable contains spaces or is empty, unquoted use causes word-splitting and globbing. `rm $file` where `file="my file.txt"` tries to delete two files, "my" and "file.txt". `"$file"` treats it as one. With an empty variable, `[ $x = "y" ]` becomes `[ = "y" ]` — a syntax error. *Why: this is the single most common Bash mistake, and interviewers love catching it.*

**Q: How do you read command-line arguments?**
`$1`, `$2`, etc. for positional args; `$0` is the script name; `$#` is the count; `$@` is all args; `$*` is all args as one string. *Why: every real script takes input.*

**Q: What's the difference between `>`, `>>`, and `2>`?**
`>` overwrites stdout to a file, `>>` appends, `2>` redirects stderr, `&>` (or `2>&1`) redirects both. `2>&1` means "send stderr to wherever stdout is going." *Why: log handling in scripts; `2>&1` order matters and trips people up.*

## Intermediate

**⭐ Q: What does `set -euo pipefail` do and why use it in every production script?**
`-e` = exit immediately if any command fails. `-u` = error on undefined variables (catches typos). `-o pipefail` = a pipeline fails if *any* command in it fails, not just the last. Without these, Bash silently continues after errors — dangerous for ops scripts that might, say, `cd` to a directory that doesn't exist and then `rm -rf *` in the wrong place. *Why: the #1 "do you write safe scripts" signal. Always lead with this in any script you show.*

**Q: Difference between `[ ]` and `[[ ]]`?**
`[ ]` is the POSIX test command — works everywhere but needs careful quoting and uses `-a`/`-o` for and/or. `[[ ]]` is a Bash keyword — safer (no word-splitting issues), supports `&&`/`||`, pattern matching, and regex (`=~`). Prefer `[[ ]]` in Bash scripts. *Why: shows you know the modern, safer construct.*

**Q: How do you check if a command succeeded?**
Check `$?` (exit code of the last command — 0 = success, non-zero = failure), or use it directly: `if command; then ... fi`. `&&` runs the next only on success, `||` runs the next only on failure. *Why: error handling is what makes a script production-grade.*

**Q: How do you loop over files or lines safely?**
Over files: `for f in *.log; do ...; done`. Over lines in a file: `while IFS= read -r line; do ...; done < file` — `IFS=` preserves whitespace, `-r` prevents backslash mangling. *Why: the naive `for line in $(cat file)` breaks on spaces — interviewers test if you know the safe pattern.*

**Q: What's command substitution and which syntax should you use?**
Capturing a command's output into a variable: `result=$(command)`. Use `$(...)` not backticks `` `...` `` — `$(...)` nests cleanly and is more readable. *Why: backticks are legacy; `$()` signals modern Bash.*

**Q: How do you give a variable a default value?**
`${var:-default}` uses "default" if var is unset or empty. `${var:=default}` also assigns it. `${var:?error message}` exits with an error if unset. *Why: makes scripts robust to missing input — `NAMESPACE="${1:-default}"`.*

## Advanced

**⭐ Q: How do you handle errors and cleanup in a script?**
Use `trap` to run cleanup on exit or error: `trap 'rm -f "$tmpfile"' EXIT` runs the cleanup whenever the script exits, success or failure. `trap 'echo "failed at line $LINENO"' ERR` catches errors. Combined with `set -e`, this ensures temp files/locks get cleaned up even when the script dies unexpectedly. *Why: shows you write scripts that don't leave garbage behind on failure — a real production concern.*

**⭐ Q: How do you prevent a script from running twice simultaneously?**
Use a lock with `flock`: `exec 200>/var/lock/myscript.lock; flock -n 200 || exit 1`. This acquires an exclusive lock on a file descriptor; if another instance holds it, `-n` (non-blocking) makes this one exit immediately. *Why: critical for cron jobs — without locking, a slow run can overlap the next scheduled run and corrupt state. This is exactly the improvement I'd mention for the disk-cleanup script.*

**Q: How do you debug a Bash script?**
`bash -x script.sh` (or `set -x` inside) prints each command as it executes with expanded variables — you see exactly what ran. `set -v` prints lines as read. Combine with `set -euo pipefail` to fail fast at the problem. *Why: tests whether you can troubleshoot your own scripts, not just write them.*

**Q: What's the difference between running a script with `./script.sh`, `bash script.sh`, and `source script.sh`?**
`./script.sh` and `bash script.sh` run it in a *new subshell* — variables set inside don't affect your current shell. `source script.sh` (or `. script.sh`) runs it in the *current shell* — so it can change your environment (used for sourcing config/env files). *Why: explains "why didn't my variable persist?" confusion.*

**Q: How do you process a large file efficiently in Bash, and when should you NOT use Bash?**
For line processing use streaming tools — `grep`, `awk`, `sed` — which process line-by-line without loading the whole file into memory. `awk` is great for column extraction and aggregation. But when you need structured data (JSON), complex logic, or maintainability, switch to Python — parsing JSON or doing real data manipulation in Bash is painful and fragile. *Why: shows judgment about the right tool — exactly why your log-analyzer is Python, not Bash.*

**Q: What does `xargs` do and why is it useful?**
`xargs` builds and runs commands from stdin input — e.g., `find . -name "*.log" | xargs rm` deletes all found files. It batches arguments efficiently rather than running the command once per item. Use `xargs -0` with `find -print0` to handle filenames with spaces safely. *Why: common in cleanup/automation scripts; the `-0` safety detail shows depth.*

**Q: How do you make a script read configuration or secrets safely?**
Source a config file (`source config.env`) for non-secrets. For secrets, never hardcode — read from environment variables injected at runtime, or pull from a secret manager. In a script, avoid echoing secrets (they end up in logs/`set -x` output); use `set +x` around the sensitive section. *Why: security-awareness in scripting is a senior signal.*

---

# Walking through YOUR scripts (interview-critical)

Since you have real scripts, interviewers may say "show me one and explain it." For your **disk-cleanup script**, be ready to explain:

- **`set -euo pipefail`** → "fails fast so it can't continue after an error and do damage"
- **The threshold check first** → "it's a no-op when there's nothing to clean, avoids unnecessary work"
- **`crictl` not `docker`** → "AKS uses containerd, so the runtime CLI is crictl"
- **truncate vs delete on logs** → "deleting a log a process has open doesn't free space until the handle closes; truncate frees it immediately"
- **What you'd improve** → "add `flock` to prevent concurrent runs, add a `--dry-run` flag, and push freed-space metrics to Prometheus"

That last point — *what you'd improve* — is gold. It shows you think about production-hardening, not just "it works on my machine."

---

# Drilling priority

The ⭐ ones are what interviews actually hit:
- `set -euo pipefail` (why) — guaranteed question if you discuss scripting
- Quoting variables (`"$var"`) — the classic gotcha
- `trap` for cleanup — separates basic from production-grade
- `flock` for locking — senior-level awareness
- When NOT to use Bash (→ Python) — shows judgment

If you can fluently explain those five plus walk through your own scripts, you're solidly mid-level on Bash for any SRE/DevOps interview.

---

# LINUX

## Beginner

**Q: What's the difference between a process and a thread?**
A process is an independent program with its own memory space. A thread is a lighter unit of execution *inside* a process, sharing that process's memory. Threads are cheaper to create and share data easily; processes are isolated and safer. *Why it matters: explains why a multi-threaded app can leak/corrupt shared memory but multiple processes can't touch each other's.*

**Q: What does `chmod 755` mean?**
Permissions as three digits: owner/group/others, where read=4, write=2, execute=1. 755 = owner rwx (7), group r-x (5), others r-x (5). *Why: file permission questions are common, and 755 (executables/dirs) and 644 (regular files) are the two you'll see most.*

**Q: Difference between a hard link and a soft (symbolic) link?**
A hard link is another name pointing to the same inode (same actual data) — deleting the original doesn't remove the data. A soft link is a pointer to a path — if the original is deleted, the link breaks. *Why: explains weird "file deleted but still there" behavior.*

**Q: What's the difference between `/etc`, `/var`, `/tmp`, `/proc`?**
`/etc` = config files. `/var` = variable data (logs, caches). `/tmp` = temporary files (often cleared on reboot). `/proc` = a virtual filesystem exposing kernel/process info (not real files on disk). *Why: `/proc/<pid>/` is a debugging goldmine, and `/var` filling up is a common incident.*

## Intermediate

**⭐ Q: `top` shows high load average but low CPU usage. What's going on?**
Load average counts processes that are *runnable* AND those in *uninterruptible sleep* (D-state, usually blocked on I/O). High load + low CPU = processes waiting on I/O (disk, NFS), not CPU saturation. Check the `wa` (I/O wait) column and `iostat`. *Why: classic SRE question — tests whether you understand load avg isn't just CPU.*

**⭐ Q: `df` says the disk is full but `du` doesn't add up. Why?**
A process is holding an open file handle to a file that was deleted. The directory entry is gone (so `du`, which walks the tree, doesn't count it), but the blocks aren't freed until the process closes the handle (so `df`, which reads free blocks, still shows them used). Find it: `lsof +L1` or `lsof | grep deleted`. Fix: restart the process or truncate the open fd. *Why: extremely common real incident; tests deep understanding of how the filesystem works.*

**Q: How do you find what's using a port? What's eating memory?**
Port: `ss -tlnp | grep :8080` (shows the listening process + PID). Memory: `ps aux --sort=-rss | head` or `top` sorted by RES. *Why: every troubleshooting scenario starts with "what's running and what's it consuming."*

**Q: What's the difference between `kill`, `kill -9`, and what signals do they send?**
`kill` sends SIGTERM (15) — graceful, asks the process to clean up and exit. `kill -9` sends SIGKILL — forceful, the kernel terminates it immediately, no cleanup. Always try SIGTERM first; SIGKILL can leave things in a bad state (open files, locks). *Why: tests whether you understand graceful vs forced shutdown — directly relevant to pod termination too.*

**Q: What's a zombie process and a defunct process?**
A zombie (defunct) is a process that has finished but whose parent hasn't read its exit status yet, so it stays in the process table. They consume no resources except a table entry. Too many means the parent isn't reaping children properly. *Why: shows up in process-table questions.*

**Q: How do you schedule a recurring job? Difference between cron and systemd timers?**
Cron: `crontab -e`, classic time-based scheduler. Systemd timers: more powerful, integrated with systemd units, better logging via journald, can handle dependencies and missed runs (`Persistent=true`). *Why: automation/toil questions; modern systems prefer systemd timers.*

## Advanced

**⭐ Q: RSS vs VSZ vs PSS — what's the difference and which matters for OOM?**
VSZ = total virtual address space the process could use (includes unmapped/reserved — misleadingly large). RSS = resident physical RAM in use, but counts shared pages (like libc) fully in every process. PSS = proportional set size, splits shared pages fairly — the most accurate "real cost." For OOM concerns, RSS is the practical number the kernel acts on. *Why: tests real memory understanding; summing RSS across processes can exceed total RAM because of double-counted shared pages.*

**⭐ Q: How do you investigate why a process was OOM-killed?**
`dmesg -T | grep -i oom` or `journalctl -k | grep -i oom` shows the OOM killer's victim, its memory usage, and oom_score. The kernel kills the highest oom_score process (influenced by memory use + oom_score_adj). In containers, exceeding the cgroup memory limit triggers a cgroup-OOM (exit 137); node-wide exhaustion is a system OOM. *Why: directly maps to Kubernetes OOMKilled — your incident story.*

**Q: A process is hung. How do you find what it's blocked on without killing it?**
`strace -p <pid>` shows the syscall it's stuck in (`read` on a socket = network wait, `futex` = lock contention, `fsync` = disk). `cat /proc/<pid>/stack` shows the kernel stack; `/proc/<pid>/wchan` shows the sleeping function. `lsof -p <pid>` shows open files/sockets. *Caveat: strace pauses the process on every syscall — heavy on production, use briefly.*

**Q: What's a file descriptor and what causes "too many open files"?**
An FD is a kernel handle to an open file or socket. Each process has a limit (`ulimit -n`). "Too many open files" = the process hit that limit, usually from an FD leak (not closing connections/files) or genuinely high concurrency with a low limit. Check: `ls /proc/<pid>/fd | wc -l`. Fix: raise the limit (`LimitNOFILE` in systemd) and fix the leak. *Why: a service hitting this stops accepting new connections while still "running" — a confusing incident.*

**Q: What is a cgroup and a namespace, and how do they relate to containers?**
Namespaces isolate *what a process can see* (its own PID space, network, mounts, users) — that's container isolation. cgroups limit *what a process can use* (CPU, memory, I/O) — that's container resource limits. Together they're the kernel primitives that *make* containers; Docker/containerd just orchestrate them. *Why: shows you understand containers aren't magic — they're Linux kernel features.*

---

# NETWORKING

## Beginner

**Q: Walk through what happens when you type a URL and hit enter.**
DNS resolves the domain to an IP → TCP connection established (3-way handshake) → TLS handshake if HTTPS → HTTP request sent → server responds → browser renders. *Why: the classic opener; tests end-to-end mental model.*

**Q: TCP vs UDP — when each?**
TCP is connection-oriented, reliable, ordered (handshake, acknowledgments, retransmission) — used for HTTP, databases, anything needing reliability. UDP is connectionless, fast, no guarantees — used for DNS, streaming, VoIP where speed beats reliability. *Why: foundational; explains why DNS (UDP) behaves differently from HTTP (TCP).*

**Q: What's the difference between a public and private IP?**
Public IPs are globally routable on the internet. Private IPs (10.x, 172.16-31.x, 192.168.x) are for internal networks and not routable on the internet — they reach out via NAT. *Why: underpins VPC/subnet design.*

**⭐ Q: What does DNS do and what are the common record types?**
DNS resolves names to IPs. Records: A (name→IPv4), AAAA (name→IPv6), CNAME (alias to another name), MX (mail), TXT (verification/SPF), NS (nameservers). *Why: DNS issues cause a huge share of incidents; knowing record types is table stakes.*

## Intermediate

**⭐ Q: Explain the TCP 3-way handshake.**
SYN (client → server, "I want to connect") → SYN-ACK (server → client, "ok, and I want to connect back") → ACK (client → server, "confirmed"). Now the connection is established. *Why: connection-establishment issues (SYN floods, half-open connections) trace back to this.*

**⭐ Q: Connection refused vs connection timed out — what does each tell you?**
Refused = the packet reached the host and was actively rejected (RST returned) — nothing listening on that port, service down, or wrong port. An app/host issue. Timed out = no response at all — a firewall/security group silently dropping packets, wrong IP, or a routing problem. A network issue. *Why: the single most useful diagnostic distinction — it tells you which layer to investigate.*

**Q: What's the difference between a load balancer at L4 vs L7?**
L4 routes on IP/port, fast, no payload inspection. L7 routes on HTTP content (host, path, headers), enables TLS termination, path routing, rate limiting. *Why: cloud LB design questions.*

**Q: Walk through a TLS handshake.**
Client hello (supported ciphers, TLS version) → server hello + certificate → client verifies the cert chain against trusted CAs → key exchange (ECDHE) creates a shared session key → encrypted communication begins. TLS 1.3 reduced this to one round trip. *Why: HTTPS/cert issues are common; tests whether you understand what "TLS error" actually means.*

**Q: What's NAT and why is it needed?**
Network Address Translation maps private IPs to a public IP for internet access. A NAT gateway lets many private resources share one public IP for outbound traffic, tracking connections so responses return correctly. *Why: explains how private subnets reach the internet.*

**Q: What are the main HTTP status code categories?**
2xx success, 3xx redirect, 4xx client error (400 bad request, 401 unauthorized, 403 forbidden, 404 not found, 429 rate-limited), 5xx server error (500 internal, 502 bad gateway, 503 unavailable, 504 gateway timeout). *Why: 5xx debugging is core SRE; knowing 502 vs 503 vs 504 points you to different causes (502 = bad upstream response, 503 = no capacity, 504 = upstream too slow).*

## Advanced

**⭐ Q: What's the difference between 502, 503, and 504, and what does each suggest?**
502 Bad Gateway = the proxy/LB got an invalid response from the upstream (upstream crashed or returned garbage). 503 Service Unavailable = no healthy backends / overloaded / nothing to route to. 504 Gateway Timeout = the upstream took too long to respond (slow backend, exhausted connection pool). *Why: in an incident, the specific 5xx code immediately narrows the cause — this is a favorite SRE drill.*

**⭐ Q: What is conntrack and how does it cause production incidents?**
conntrack is the Linux kernel's connection-tracking table that remembers NAT mappings so return traffic routes correctly. It has a max size; under high connection churn it fills up and new connections get dropped — `dmesg` shows "nf_conntrack: table full, dropping packet," and you see intermittent failures that look like app bugs. Fix: raise `nf_conntrack_max`, reduce churn with connection pooling/keepalive. *Why: a sneaky, high-level incident that separates people who've actually run busy production systems.*

**Q: What's the MTU and how does it cause mysterious failures?**
MTU is the max packet size a link can carry (typically 1500 bytes). In overlay networks (VXLAN adds ~50 bytes of header), if the pod MTU isn't lowered, large packets get fragmented or dropped while small ones succeed. Classic symptom: TLS handshake works (small packets), then the connection hangs on the first large payload. Diagnose: `ping -M do -s <size>` to find where it breaks. *Why: one of the hardest-to-diagnose networking issues; impressive if you know it.*

**Q: How does HTTP keep-alive / connection pooling improve reliability and performance?**
Keep-alive reuses a TCP connection for multiple requests instead of opening a new one each time. This avoids repeated handshakes (lower latency), reduces conntrack churn, and prevents port/FD exhaustion under high load. Connection pooling on the client side does the same for backend calls. *Why: connects to conntrack exhaustion and the resilience patterns SRE roles ask about.*

**Q: What's the difference between TCP and a reverse proxy / forward proxy?**
A forward proxy sits in front of *clients* (outbound — e.g., a corporate proxy filtering employee traffic). A reverse proxy sits in front of *servers* (inbound — e.g., nginx/LB terminating TLS, routing, caching). *Why: clarifies LB/ingress architecture questions.*

**Q: How would you debug high latency to an external API from a server/pod?**
Break it down by phase with `curl -w` timing: `time_namelookup` (DNS slow?), `time_connect` (TCP/network slow?), `time_appconnect` (TLS slow?), `time_starttransfer` (server processing — TTFB), `time_total`. This isolates exactly which phase is slow — DNS, network, TLS, or the server itself. Then check if it's your host, the network path (`mtr`/`traceroute`), or the external service. *Why: a structured latency-debugging answer is exactly what SRE interviews want.*

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
