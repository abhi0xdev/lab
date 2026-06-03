# Si2 Technologies — DevOps Engineer Interview Prep
### Focus: Kubernetes & CI/CD | Interview: 04 June 2026

> How to use this: Read the answer, then say it out loud in your own words. The interviewer cares more that you *understand* than that you recite. Wherever you see **[Your resume]**, expect a follow-up tying the concept to something you claimed — be ready to tell that story.

---

## PART 1 — KUBERNETES

### A. Architecture & Core Concepts

**1. Explain the Kubernetes architecture.**
Kubernetes has a **control plane** and **worker nodes**.
- Control plane: `kube-apiserver` (front door / all communication goes through it), `etcd` (key-value store holding cluster state), `kube-scheduler` (decides which node a pod runs on), `kube-controller-manager` (runs controllers that reconcile desired vs actual state), and `cloud-controller-manager` (talks to the cloud provider).
- Worker nodes: `kubelet` (agent that ensures containers are running in pods), `kube-proxy` (handles networking/routing rules), and the container runtime (e.g., containerd).
Key idea to emphasize: Kubernetes is **declarative** — you describe desired state, controllers continuously reconcile actual state toward it.

**2. What is a Pod? Why not just run containers directly?**
A Pod is the smallest deployable unit — one or more containers that share the same network namespace (same IP), storage volumes, and lifecycle. You usually run one main container per pod, plus optional sidecars (e.g., a logging or proxy container). Kubernetes schedules and scales pods, not raw containers, so the pod is the unit of scheduling.

**3. Pod vs Deployment vs ReplicaSet — how do they relate?**
- **Pod**: a running instance.
- **ReplicaSet**: ensures N identical pods are always running.
- **Deployment**: manages ReplicaSets and gives you rolling updates, rollbacks, and version history. You almost always create a Deployment, which creates a ReplicaSet, which creates Pods.

**4. What is a Service, and what are the types?**
A Service gives a stable network endpoint to a set of pods (pods are ephemeral and get new IPs). Types:
- **ClusterIP** (default): reachable only inside the cluster.
- **NodePort**: exposes on a static port on every node.
- **LoadBalancer**: provisions a cloud load balancer (on AKS, an Azure LB).
- **ExternalName**: maps to an external DNS name.
Services select pods via **labels/selectors**.

**5. ClusterIP vs Ingress — when do you use Ingress?**
A Service exposes one app. **Ingress** is an L7 (HTTP/HTTPS) router that lets you expose *many* services behind one IP using host/path rules, plus TLS termination. You need an **Ingress Controller** (NGINX, Traefik, AGIC on Azure) running for Ingress resources to actually do anything. **[Your resume — you list "configured ingress"]**

**6. ConfigMap vs Secret.**
Both inject config into pods. ConfigMap = non-sensitive config (env vars, config files). Secret = sensitive data (passwords, tokens), base64-encoded and can be encrypted at rest in etcd. Best practice: never bake secrets into images; mount them or pull from a vault (Azure Key Vault via CSI driver).

**7. What are Namespaces used for?**
Logical partitioning of a cluster — separating environments (dev/staging/prod) or teams, with resource quotas and RBAC scoped per namespace. **[Your resume — you ran workloads across multiple environments]**

**8. Labels vs Annotations.**
Labels are key-value identifiers used for *selection* (Services, selectors, grouping). Annotations hold *non-identifying metadata* (build info, tooling config) and aren't used for selection.

---

### B. Scheduling, Scaling & Reliability

**9. How does scheduling work? What influences pod placement?**
The scheduler filters nodes (which *can* run the pod) then scores them (which is *best*). Influences: resource **requests/limits**, **nodeSelector**, **node affinity/anti-affinity**, **taints and tolerations**, and **topology spread constraints**.

**10. Requests vs Limits.**
- **Request**: guaranteed minimum the scheduler reserves; used for placement.
- **Limit**: hard ceiling. CPU over-limit → throttled. Memory over-limit → pod **OOMKilled**.
Setting these correctly is how you avoid noisy-neighbor problems and throttling. **[Your resume — you mention resource throttling]**

**11. Explain HPA (Horizontal Pod Autoscaler).**
HPA scales the *number of pods* based on metrics (CPU/memory or custom metrics). It needs the **metrics-server**. Contrast with **VPA** (adjusts requests/limits of existing pods) and **Cluster Autoscaler** (adds/removes *nodes* — on AKS this is the AKS cluster autoscaler). **[Your resume — "scaling applications to handle increased load"]**

**12. Liveness, Readiness, and Startup probes — the difference.**
- **Liveness**: is the container alive? Fails → kubelet restarts it.
- **Readiness**: is it ready to serve traffic? Fails → pod removed from Service endpoints (no restart).
- **Startup**: for slow-starting apps; delays liveness/readiness until the app boots.
This is a *very* common question — know that liveness restarts, readiness gates traffic.

**13. How do you achieve high availability / >99.9% uptime?**
Multiple replicas across nodes (anti-affinity / topology spread), readiness probes so traffic only hits healthy pods, **PodDisruptionBudgets** to limit voluntary disruptions, rolling updates, multiple node pools / availability zones, and resource requests so pods aren't evicted under pressure. **[Your resume — you claim >99.9% uptime; be ready to explain HOW]**

**14. What is a StatefulSet and when do you use it?**
For stateful apps needing stable identity and storage (databases, Kafka). Gives stable network IDs (pod-0, pod-1), ordered deployment/scaling, and per-pod persistent volumes. Deployments are for stateless apps.

**15. DaemonSet?**
Runs one pod per node — used for node-level agents like log collectors (Fluentd), monitoring agents (node-exporter), or CNI plugins.

---

### C. Troubleshooting (expect heavy focus — your resume claims this)

**16. A pod is in CrashLoopBackOff. How do you debug it?**
Walk through it as a process:
1. `kubectl get pods` — confirm status and restart count.
2. `kubectl describe pod <name>` — check Events (OOMKilled? image pull error? failed probe?).
3. `kubectl logs <pod>` and `kubectl logs <pod> --previous` — see why the last run died.
4. Check probes (a too-aggressive liveness probe causes false restarts), resource limits (OOM), missing config/secrets, or a bad command/entrypoint.
5. If needed, `kubectl exec -it` into a running container or run a debug container.
Common root causes: app crashes on startup, missing env/config, OOMKilled, failing dependency, misconfigured liveness probe. **[Your resume — "resolved pod crash loops"; have a real example ready]**

**17. Pod stuck in `Pending`. Why?**
Scheduler can't place it: insufficient CPU/memory on nodes, no node matching nodeSelector/affinity, unsatisfied taints, or no PV available for a PVC. `kubectl describe pod` Events will say.

**18. `ImagePullBackOff` / `ErrImagePull`?**
Wrong image name/tag, private registry without imagePullSecret, or registry auth failure. On AKS, often the AKS cluster isn't attached to ACR (`az aks update --attach-acr`). **[Your resume — you use ACR]**

**19. What does `kubectl describe` give you that `kubectl get` doesn't?**
`get` is a quick status list; `describe` shows detailed spec, conditions, and the **Events** stream — which is usually where the real cause of a failure is.

**20. How do you debug a Service that's not routing traffic?**
Check the Service selector matches pod labels, check `kubectl get endpoints <svc>` (empty = no pods matched / readiness failing), verify the targetPort matches the container port, and confirm the pods are Ready. Then test with `kubectl port-forward` or a debug pod inside the cluster.

---

### D. AKS / Cloud-specific (your stated platform)

**21. What does AKS manage for you vs self-managed Kubernetes?**
AKS manages the control plane (you don't pay for or maintain it). You manage node pools, workloads, scaling config, and networking choices. Integrations: ACR for images, Azure Monitor/Container Insights for observability, Azure AD for RBAC, Azure Key Vault via CSI for secrets, and the Azure Load Balancer for LoadBalancer services. **[Your resume — AKS, ACR, Azure Monitor, Application Insights]**

**22. How do you upgrade an AKS cluster safely?**
Upgrade control plane first, then node pools one at a time using surge upgrades (cordon + drain nodes, schedule onto new nodes), respecting PodDisruptionBudgets, ideally in a non-prod cluster first. Test workloads against the new K8s version beforehand.

---

## PART 2 — CI/CD

### A. Fundamentals

**23. CI vs CD vs CD — define them.**
- **Continuous Integration**: developers merge to a shared branch frequently; every commit triggers automated build + tests.
- **Continuous Delivery**: every change that passes is *automatically prepared* for release; deploy to prod is a manual approval click.
- **Continuous Deployment**: every passing change goes to prod *automatically*, no manual gate.
Know the Delivery-vs-Deployment distinction — it's a favorite "gotcha."

**24. Walk me through a typical CI/CD pipeline.**
Source (commit/PR) → Build → Unit tests → Static analysis / lint / security scan (SAST, image scan) → Build & push container image → Deploy to dev → Integration/smoke tests → approval gate → staging → prod. Artifacts are versioned and immutable between stages. **[Your resume — "build, test, deploy with approval gates and rollback"]**

**25. Why approval gates and rollback mechanisms?**
Approval gates put a human checkpoint before sensitive environments (prod), giving control and an audit trail. Rollback lets you revert quickly when a release misbehaves — in K8s that's `kubectl rollout undo` or redeploying a previous image tag / previous Git commit (in GitOps). **[Your resume — you claim both; explain your exact mechanism]**

**26. What is an artifact, and why immutable artifacts matter?**
An artifact is the built, deployable output (a container image, a binary). "Build once, deploy everywhere" — you build the image *once* and promote that *same* image through dev → staging → prod. This guarantees what you tested is what you shipped. Never rebuild per environment.

---

### B. GitHub Actions (your primary tool)

**27. Explain the structure of a GitHub Actions workflow.**
A workflow (YAML in `.github/workflows/`) has **events** (triggers like push, pull_request, workflow_dispatch), **jobs** (run on runners, can run in parallel or with `needs:` dependencies), and **steps** (individual commands or reusable **actions**). Jobs run on **runners** (GitHub-hosted or self-hosted).

**28. How do you manage secrets in GitHub Actions?**
GitHub **Secrets** (repo/org/environment-scoped), referenced as `${{ secrets.NAME }}`. They're masked in logs. For cloud auth, prefer **OIDC federation** (short-lived tokens) over long-lived stored credentials — e.g., GitHub OIDC → Azure, so you don't store a service principal secret.

**29. How do you do environment-based workflows with approvals in GitHub Actions?**
Use **Environments** (dev/staging/prod) with **protection rules** — required reviewers (the approval gate), wait timers, and environment-scoped secrets. A job targeting the `prod` environment pauses until an approver clicks approve. **[Your resume — "environment-based workflows, approval gates"]**

**30. Matrix builds — what are they for?**
Run the same job across multiple combinations (OS versions, language versions, regions) in parallel from one definition. Good for testing across variants quickly.

**31. How do you make pipelines faster?**
Caching dependencies (`actions/cache`), parallel jobs, only building what changed (path filters), smaller/multi-stage Docker builds with layer caching, and reusable workflows to avoid duplication. **[Your resume — "release time from hours to minutes"]**

**32. Reusable workflows vs composite actions?**
Reusable workflows = call an entire workflow from another (`workflow_call`). Composite actions = bundle multiple steps into one reusable action. Both reduce duplication across repos.

---

### C. Deployment Strategies (you list blue-green & rolling)

**33. Compare deployment strategies.**
- **Rolling update** (K8s default): replace pods gradually, a few at a time. Zero downtime, but two versions run briefly. Controlled by `maxSurge`/`maxUnavailable`.
- **Blue-Green**: run two full environments (blue=current, green=new). Switch traffic all at once. Instant rollback (switch back), but doubles resources during cutover.
- **Canary**: route a small % of traffic to the new version, watch metrics, gradually increase. Lowest risk, needs good observability / traffic-splitting (Ingress, service mesh, Argo Rollouts).
**[Your resume — "rolling updates / blue-green"; be ready to say which you used and why]**

**34. How do you roll back a Kubernetes deployment?**
`kubectl rollout undo deployment/<name>` reverts to the previous ReplicaSet. In GitOps, you revert the Git commit and the controller re-syncs. Blue-green: flip the Service selector back to blue.

**35. How do you achieve zero-downtime deployments?**
Rolling/blue-green/canary + readiness probes (don't send traffic to not-ready pods) + `maxUnavailable: 0` if you can't lose capacity + graceful shutdown (handle SIGTERM, `terminationGracePeriodSeconds`) + PodDisruptionBudgets.

---

### D. GitOps / ArgoCD (your project)

**36. What is GitOps?**
Git is the single source of truth for *both* app and infra state. A controller continuously reconciles the cluster to match Git. You deploy by committing/merging, not by running imperative commands. Benefits: full audit trail, easy rollback (revert commit), and drift detection. **[Your resume — GitOps project with ArgoCD]**

**37. How does ArgoCD work?**
ArgoCD runs in the cluster, watches a Git repo of manifests (plain YAML, Helm, or Kustomize), compares desired (Git) vs live (cluster) state, and syncs — automatically or on approval. It shows sync status and health, and can auto-heal drift. **[Be ready to describe your dev/staging/prod setup]**

**38. Push-based CI/CD vs pull-based GitOps — the difference.**
Push (classic GitHub Actions deploy step) = the pipeline has cluster credentials and pushes changes in. Pull (ArgoCD) = the in-cluster agent pulls from Git; the cluster's credentials never leave it — generally more secure for the cluster.

---

## PART 3 — INFRA, IaC & OBSERVABILITY (likely cross-questions)

**39. What is Terraform and what problem does it solve?**
Declarative Infrastructure as Code — define cloud resources in config, `plan` to preview changes, `apply` to provision. Tracks **state** (what it manages). Benefits: reproducible, version-controlled, reviewable infra. **[Your resume — Terraform]**

**40. Terraform state — why does it matter, and remote state?**
State maps your config to real resources. It must be shared and locked for teams — store it remotely (Azure Storage / S3) with locking so two people don't apply at once and corrupt it. Never commit state to Git (it can contain secrets).

**41. Docker multi-stage builds — why use them?**
Build in one stage (with compilers/dev deps), copy only the final artifact into a slim runtime stage. Result: much smaller, more secure images. **[Your resume — "optimized image sizes by 25%"; this is likely how]**

**42. Prometheus + Grafana — how do they fit together?**
Prometheus scrapes and stores time-series metrics and runs alerting rules (Alertmanager sends alerts). Grafana is the visualization/dashboard layer that queries Prometheus (and other sources). Prometheus uses a **pull** model and **PromQL** for queries. **[Your resume — observability stack]**

**43. What's the difference between metrics, logs, and traces?**
The three pillars of observability. Metrics = numeric aggregates over time (CPU, request rate). Logs = discrete event records. Traces = the path of a single request across services (latency per hop). Prometheus = metrics; Application Insights/Dynatrace add traces and APM. **[Your resume — App Insights, Dynatrace]**

---

## PART 4 — RESUME DEEP-DIVE (rehearse these — they WILL ask)

For every quantified claim, prepare a 30-second story: **Situation → what you did → result → how you measured it.**

- **"Reduced deployment failures by 30–40%"** — *How did you measure failures before/after?* (e.g., failed pipeline runs / rollbacks per month). What change drove it — approval gates? better tests? environment isolation?
- **"Cut manual deployment effort by 60% / hours to minutes"** — what was manual before, what did you automate?
- **">99.9% uptime on AKS"** — how measured? (uptime monitoring, SLO). What kept it up — replicas, probes, autoscaling?
- **"Reduced incident resolution time by 40%"** — give one concrete incident (a crash loop or throttling) and how observability helped you find it faster.
- **"Optimized image sizes by 25%"** — multi-stage builds? smaller base image (alpine/distroless)? removing build deps?
- **"Reduced alert response time by 50%"** — better dashboards? actionable alerts vs noise? routing to the right team?

If a number is an estimate, that's fine — say "approximately, based on [the metric]." Don't invent precise detail you can't defend.

---

## PART 5 — LIKELY BEHAVIORAL / SCENARIO QUESTIONS

1. **Tell me about a production incident you resolved.** → Use the crash-loop or throttling example. STAR format (Situation, Task, Action, Result).
2. **A deployment to prod failed at 2 AM. Walk me through your response.** → Stabilize first (rollback), then diagnose (logs/describe/metrics), communicate, then post-mortem to prevent recurrence.
3. **How do you decide between rolling, blue-green, and canary?** → Risk tolerance, resource budget, observability maturity, rollback speed needed.
4. **How do you keep secrets out of pipelines and images?** → GitHub Environment secrets / OIDC, Key Vault CSI driver, never in Git or image layers.
5. **Disagreement with a developer about a release.** → Show collaboration + data-driven reasoning.

---

## PART 6 — SMART QUESTIONS TO ASK THEM

(Asking good questions signals seniority.)
- What does the current CI/CD setup look like, and where do they want to improve it?
- Managed Kubernetes (AKS/EKS) or self-hosted? Single or multi-cluster?
- What does their on-call / incident process look like?
- GitOps in place, or still push-based deploys?
- What would success in this role look like in the first 90 days?

---

## FINAL-NIGHT CHECKLIST

- [ ] Be able to draw the K8s architecture on a whiteboard from memory.
- [ ] Memorize the CrashLoopBackOff debug sequence — it's the highest-probability question for you.
- [ ] Know the 3 deployment strategies cold + how to roll back each.
- [ ] Know CI vs Continuous Delivery vs Continuous Deployment.
- [ ] Rehearse 2–3 resume stories in STAR format with real numbers.
- [ ] Refresh: liveness vs readiness, requests vs limits, ConfigMap vs Secret, Service types.
- [ ] Have `kubectl` commands ready to mention: `get`, `describe`, `logs --previous`, `rollout undo`, `get endpoints`, `exec`.

You've got real, relevant experience — the goal tomorrow is just to explain it clearly and back up your resume. Good luck.
