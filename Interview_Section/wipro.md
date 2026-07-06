# Wipro DevOps Engineer Interview — 100 Q&A (4 Years Experience)

**Role:** DevOps Engineer (client round)
**Level:** Mid-level (4 YOE)
**Toolchain covered:** Linux, Git, CI/CD (Jenkins/GitLab/GitHub Actions), Docker, Kubernetes, Terraform, Ansible, AWS, Prometheus/Grafana, Networking & Security
**How to use:** Read answers as talking points, not scripts. Anchor every answer to something you actually did ("In my pipeline we used..."). 💡 = framing/interview tip. Commands are included because 4-YOE client rounds test hands-on depth, not just definitions.

---

## Section 1 — DevOps Fundamentals & Culture (Q1–Q8)

**Q1. What is DevOps in your own words?**
DevOps is a culture and set of practices that unify development and operations to deliver software faster and more reliably. It's built on automation (CI/CD, IaC), collaboration (shared ownership, breaking silos), continuous feedback (monitoring, alerting), and continuous improvement. It's not a tool or a job title — it's a way of working where the team that builds it also runs it. 💡 Avoid reciting "CALMS" like a textbook; give a practical version.

**Q2. What is CALMS?**
A framework for assessing DevOps maturity: **C**ulture (collaboration, shared responsibility), **A**utomation (CI/CD, IaC, testing), **L**ean (small batches, reduce waste), **M**easurement (metrics, monitoring, feedback), **S**haring (knowledge, tooling, blameless learning).

**Q3. What are the DORA metrics?**
Four key metrics that measure software delivery performance: **Deployment Frequency** (how often you ship), **Lead Time for Changes** (commit to production time), **Change Failure Rate** (% of deploys causing incidents), and **MTTR** (time to restore service). The first two measure speed, the last two measure stability — elite teams score high on both, proving speed and stability aren't a trade-off.

**Q4. What's the difference between Continuous Integration, Continuous Delivery, and Continuous Deployment?**
CI: developers merge to a shared branch frequently, each merge auto-builds and runs tests. Continuous Delivery: every passing build is *ready* to deploy to prod, but the final release is a manual/gated decision. Continuous Deployment: every passing build is *automatically* released to prod with no manual gate.

**Q5. Why is DevOps important for a business?**
Faster time to market, higher release frequency with lower failure rates, quicker recovery from incidents, better collaboration, and lower operational cost through automation. It turns releases from risky, rare events into routine, low-risk ones.

**Q6. What is "shift-left"?**
Moving activities like testing, security, and quality checks earlier in the development lifecycle (left on the timeline) instead of at the end. Catching a bug or vulnerability at commit time is far cheaper than in production. Examples: unit tests and SAST scans in CI, linting on commit, security in the pipeline (DevSecOps).

**Q7. What is Infrastructure as Code and why does it matter?**
IaC means managing infrastructure through machine-readable definition files (Terraform, CloudFormation, Ansible) instead of manual clicking. Benefits: version control, repeatability, consistency across environments, peer review, easy rollback, and no configuration drift. It makes infra reproducible and auditable.

**Q8. What is configuration drift and how do you prevent it?**
Drift is when the actual state of infrastructure diverges from the defined/desired state due to manual changes. Prevent it with IaC as the single source of truth, immutable infrastructure (replace instead of modify), regular `terraform plan`/drift detection, and configuration management (Ansible) that continuously enforces desired state. GitOps also prevents drift by reconciling live state to Git.

---

## Section 2 — Linux & Shell Scripting (Q9–Q18)

**Q9. How do you check what's consuming CPU/memory on a Linux server?**
`top` or `htop` for a live view, `ps aux --sort=-%cpu | head` for top CPU processes, `ps aux --sort=-%mem | head` for memory, `free -h` for overall memory, `vmstat 1` for system-wide stats. For historical/deeper analysis, `sar`. To find what a specific process is doing, `strace -p <pid>`.

**Q10. A disk is full — how do you find what's using space?**
`df -h` to see which mount is full, then `du -sh /path/*` drilling down into directories, or `du -ah / | sort -rh | head -20` for the biggest files. Common culprits: log files (`/var/log`), old Docker images/volumes (`docker system prune`), and deleted-but-open files (`lsof | grep deleted` — a process holding a deleted file keeps the space until restarted).

**Q11. What's the difference between hard and soft links?**
A soft (symbolic) link is a pointer to a path — breaks if the target is deleted, can cross filesystems, can link directories. A hard link is another directory entry pointing to the same inode (same actual data) — survives deletion of the original, can't cross filesystems, can't link directories.

**Q12. Explain Linux file permissions and what 755 means.**
Permissions are read(4)/write(2)/execute(1) for owner/group/others. 755 = owner rwx (7), group r-x (5), others r-x (5) — common for executables and directories. 644 = owner rw, group/others read-only — common for files. Set with `chmod`, ownership with `chown user:group file`.

**Q13. How do you find and kill a process running on a specific port?**
`lsof -i :8080` or `netstat -tulpn | grep 8080` or `ss -tulpn | grep 8080` to find the PID, then `kill <pid>` (graceful, SIGTERM) or `kill -9 <pid>` (force, SIGKILL). Prefer SIGTERM first so the process can clean up.

**Q14. What is the difference between SIGTERM and SIGKILL?**
SIGTERM (15) politely asks a process to terminate — it can catch it, clean up, and exit gracefully. SIGKILL (9) forcibly kills it immediately at the kernel level — it can't be caught or ignored, so no cleanup happens. Always try SIGTERM first.

**Q15. Write a shell script to check if a service is running and restart it if not.**
```bash
#!/bin/bash
SERVICE="nginx"
if ! systemctl is-active --quiet "$SERVICE"; then
  echo "$(date): $SERVICE is down, restarting" >> /var/log/watchdog.log
  systemctl restart "$SERVICE"
fi
```

**Q16. How do you search for a string inside files recursively?**
`grep -r "search_term" /path` or with line numbers and filename: `grep -rn "search_term" /path`. To search only certain file types: `grep -rn --include="*.log" "ERROR" /var/log`. For faster large-scale search, `ripgrep (rg)`.

**Q17. What does `set -euo pipefail` do in a bash script?**
`set -e` exits on any command failure, `set -u` errors on undefined variables, `set -o pipefail` makes a pipeline fail if any command in it fails (not just the last). Together they make scripts fail fast and safe instead of silently continuing after an error — important in CI/CD scripts.

**Q18. How do you schedule a recurring task in Linux?**
`cron` — edit with `crontab -e`. Format: `minute hour day month weekday command`. Example, run at 2 AM daily: `0 2 * * * /path/backup.sh`. For system-wide or more complex scheduling with dependencies and logging, `systemd timers` are a modern alternative.

---

## Section 3 — Git & Version Control (Q19–Q25)

**Q19. What's the difference between `git merge` and `git rebase`?**
`merge` combines branches and creates a merge commit, preserving the true history (branchy graph). `rebase` replays your commits on top of another branch, creating a linear history but rewriting commit hashes. Rule: rebase local/private branches for a clean history; never rebase shared/public branches (it rewrites history others depend on).

**Q20. What is `git cherry-pick`?**
It applies a specific commit from one branch onto another without merging the whole branch. Useful for hotfixes — e.g., picking a single bug-fix commit from `develop` into `release` without bringing everything else.

**Q21. Difference between `git fetch` and `git pull`?**
`fetch` downloads remote changes but doesn't merge them into your working branch — safe, lets you review first. `pull` = `fetch` + `merge` (or rebase) — it downloads and immediately integrates. Use fetch when you want to inspect before merging.

**Q22. How do you undo the last commit?**
`git reset --soft HEAD~1` keeps changes staged (undo commit, keep work). `git reset --mixed HEAD~1` keeps changes unstaged. `git reset --hard HEAD~1` discards everything (dangerous). If the commit was already pushed and shared, use `git revert <commit>` instead — it creates a new commit that undoes it without rewriting history.

**Q23. What's the difference between `git reset` and `git revert`?**
`reset` moves the branch pointer back, rewriting history — fine for local, unshared commits. `revert` creates a new commit that reverses a previous one — safe for shared branches because it doesn't rewrite history.

**Q24. Explain a branching strategy you've used.**
Common ones: **GitFlow** (main, develop, feature/*, release/*, hotfix/* — structured, good for scheduled releases but heavy). **GitHub Flow** (main + short-lived feature branches, deploy from main — simple, good for CD). **Trunk-based development** (everyone commits to main frequently behind feature flags — best for high-frequency CI/CD). 💡 Say which you used and *why* it fit the team's release cadence.

**Q25. How do you resolve a merge conflict?**
Git marks conflicting sections with `<<<<<<<`, `=======`, `>>>>>>>`. You open the file, manually choose/combine the correct code, remove the markers, then `git add` the resolved file and `git commit` (or `git rebase --continue`). Tools like VS Code or `git mergetool` make it visual. Prevention: small frequent merges and communication reduce conflicts.

---

## Section 4 — CI/CD (Jenkins, GitLab, GitHub Actions) (Q26–Q37)

**Q26. Walk me through a CI/CD pipeline you built.**
Trigger on push/PR → build → unit tests → static code analysis (SonarQube) → security scan (SAST/dependency scan) → build & push Docker image to registry → deploy to dev/staging → integration/smoke tests → gated promotion to prod → post-deploy verification (health checks, smoke tests, watch metrics). 💡 Name the actual tools and describe one gate/approval step to show maturity.

**Q27. What is a Jenkinsfile and declarative vs scripted pipeline?**
A Jenkinsfile defines the pipeline as code in the repo. Declarative is structured with a fixed `pipeline { stages { } }` syntax — readable, easier, preferred for most cases. Scripted is Groovy-based, more flexible/programmable but more complex. Example declarative:
```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { sh 'mvn clean package' } }
    stage('Test')  { steps { sh 'mvn test' } }
    stage('Docker'){ steps { sh 'docker build -t app:$BUILD_NUMBER .' } }
  }
  post { failure { mail to: 'team@x.com', subject: 'Build failed' } }
}
```

**Q28. How do you handle secrets in a pipeline?**
Never hard-code them. Use the platform's secret store (Jenkins Credentials, GitLab CI/CD variables (masked/protected), GitHub Actions Secrets) or an external manager (Vault, AWS Secrets Manager). Inject at runtime as masked env vars, scope by least privilege, rotate regularly, and never echo/log them.

**Q29. What is a Jenkins agent/node and why use them?**
The Jenkins controller (master) orchestrates; agents (nodes) actually run the build jobs. You use multiple agents to distribute load, run builds in parallel, and provide different environments (different OS, tools, labels). Ephemeral agents (Docker/Kubernetes) spin up fresh per build for clean, isolated builds.

**Q30. How do you trigger a pipeline?**
Webhooks on push/PR (most common), scheduled (cron/`triggers`), manually, upstream/downstream job triggers, or via API. Webhooks give the fastest feedback — the SCM notifies CI immediately on a commit.

**Q31. What's the difference between GitLab CI, Jenkins, and GitHub Actions?**
Jenkins: self-hosted, huge plugin ecosystem, very flexible, but you maintain it. GitLab CI: built into GitLab, `.gitlab-ci.yml`, uses runners, tightly integrated with the repo. GitHub Actions: built into GitHub, YAML workflows, marketplace of reusable actions, easy for GitHub-hosted projects. 💡 The concepts transfer — pipeline-as-code, stages, runners/agents, artifacts.

**Q32. What is a build artifact and how do you manage it?**
An artifact is the output of a build (JAR, Docker image, binary) that later stages consume or deploy. Store in an artifact repository (Nexus, Artifactory) or container registry (ECR, Docker Hub, GCR). Version them (build number, git SHA, semver), and promote the *same* artifact through environments rather than rebuilding — rebuilding risks producing a different artifact than what you tested.

**Q33. How do you make a pipeline faster?**
Cache dependencies, run independent stages in parallel, fail fast (cheap checks first), use incremental builds, reuse Docker layer caching, split test suites, use faster/ephemeral runners, and skip unchanged components (monorepo change detection). Also quarantine flaky tests instead of re-running the whole pipeline.

**Q34. What deployment strategies do you know?**
Rolling (gradual instance replacement), Blue-Green (two environments, instant switch/rollback), Canary (small % traffic first, watch metrics, then ramp), Recreate (down-then-up, causes downtime), and Feature Flags (deploy dark, toggle on gradually). Canary and blue-green give the safest production releases.

**Q35. How do you roll back a failed deployment automatically?**
Have the pipeline verify health/smoke tests and metrics post-deploy; if they fail, trigger rollback: blue-green flips traffic back, canary halts and shifts back to stable, GitOps reverts the commit, and rolling redeploys the last known-good image. Prerequisite: keep previous artifacts and tag "last known good."

**Q36. What quality gates do you put in a pipeline?**
Unit test pass + coverage threshold, static analysis (SonarQube quality gate), security scans (SAST, dependency/CVE scan, container image scan), linting, integration/smoke tests, and sometimes an SLO/error-budget or manual approval gate before prod. A failing gate blocks promotion.

**Q37. What is DevSecOps?**
Integrating security into every stage of the pipeline rather than bolting it on at the end — "shift security left." Practices: SAST (code scanning), DAST (running-app scanning), dependency/SCA scanning, container image scanning, secrets detection, and IaC scanning (Checkov, tfsec). Security becomes a shared responsibility and an automated gate.

---

## Section 5 — Docker & Containers (Q38–Q49)

**Q38. What is the difference between a container and a VM?**
A VM virtualizes hardware and runs a full guest OS with its own kernel — heavy, slow to boot, strong isolation. A container virtualizes the OS, sharing the host kernel and isolating via namespaces and cgroups — lightweight, fast, portable, but weaker isolation. Containers package the app + dependencies, not a whole OS.

**Q39. What is the difference between an image and a container?**
An image is a read-only template (built from a Dockerfile) — the blueprint. A container is a running (or stopped) instance of an image with a writable layer on top. One image → many containers.

**Q40. Explain a Dockerfile you'd write and best practices.**
```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```
Best practices: use small base images (alpine/distroless), multi-stage builds to shrink the final image, order layers from least- to most-frequently-changing for caching, combine RUN commands, use `.dockerignore`, run as non-root, pin versions, and never bake secrets into the image.

**Q41. What is a multi-stage build and why use it?**
It uses multiple `FROM` stages in one Dockerfile — build the app in a heavy stage (with compilers/tools), then copy only the final artifact into a slim runtime stage. Result: a much smaller, more secure final image without build tools. Shown in Q40.

**Q42. What's the difference between CMD and ENTRYPOINT?**
`ENTRYPOINT` sets the fixed executable that always runs. `CMD` provides default arguments (or a default command) that can be overridden at `docker run`. Common pattern: `ENTRYPOINT ["python"]` + `CMD ["app.py"]` — runs `python app.py` by default but lets you override the script.

**Q43. What's the difference between COPY and ADD?**
Both copy files into the image. `COPY` just copies local files/dirs — preferred for clarity. `ADD` also auto-extracts local tar archives and can fetch from URLs — use only when you need those features. Best practice: default to COPY.

**Q44. How do you reduce Docker image size?**
Use a minimal base (alpine/distroless), multi-stage builds, remove build dependencies and caches in the same layer (`apt-get clean`, `rm -rf /var/lib/apt/lists/*`), use `.dockerignore`, combine RUN steps, and avoid installing unnecessary packages. Smaller images = faster pulls, less attack surface.

**Q45. How do containers persist data?**
Containers are ephemeral — the writable layer dies with the container. For persistence use **volumes** (managed by Docker, best for persistent data) or **bind mounts** (map a host path, good for dev). Named volumes survive container removal and can be backed up. `tmpfs` mounts keep data only in memory.

**Q46. How do Docker containers communicate?**
Via Docker networks. Default `bridge` isolates containers on the host; a user-defined bridge network lets containers resolve each other by name (built-in DNS). `host` network shares the host's stack. `overlay` networks span multiple hosts (Swarm/Kubernetes). Best practice: user-defined bridge for name-based service discovery.

**Q47. What does `docker system prune` do?**
Removes unused data — stopped containers, dangling images, unused networks, and (with `-a`) all unused images, plus `--volumes` for unused volumes. Frees disk space. Useful on build servers where images pile up. 💡 Common answer to "the CI server disk is full."

**Q48. How do you troubleshoot a container that keeps crashing?**
`docker ps -a` to see exit status, `docker logs <container>` for output, `docker inspect <container>` for config/exit code, check resource limits (OOMKilled = out of memory), verify the ENTRYPOINT/CMD and health check, and exec into a shell (`docker exec -it <c> sh`) if it stays up long enough or run it interactively to debug. Exit code 137 usually means OOM/SIGKILL.

**Q49. What is Docker Compose and when do you use it?**
A tool to define and run multi-container applications with a single YAML file (`docker-compose.yml`) — you declare services, networks, and volumes, then `docker compose up`. Great for local development and testing multi-service apps. Not typically for production orchestration at scale — that's Kubernetes.

---

## Section 6 — Kubernetes (Q50–Q64)

**Q50. What is Kubernetes and why use it?**
An open-source container orchestration platform that automates deployment, scaling, self-healing, load balancing, and rollout/rollback of containerized apps across a cluster. You use it because managing many containers manually across hosts doesn't scale — K8s handles scheduling, health, scaling, and networking declaratively.

**Q51. Explain Kubernetes architecture.**
**Control plane:** `kube-apiserver` (front door, all requests go through it), `etcd` (key-value store holding cluster state), `kube-scheduler` (assigns pods to nodes), `controller-manager` (runs controllers that reconcile desired vs actual state). **Worker nodes:** `kubelet` (agent that runs pods and reports status), `kube-proxy` (networking/routing), and the container runtime (containerd). Everything is declarative and reconciliation-based.

**Q52. What is a Pod?**
The smallest deployable unit — one or more containers that share the same network namespace (IP), storage volumes, and lifecycle. Usually one main container per pod; sidecars (logging, proxy) share the pod. Pods are ephemeral — you don't manage them directly, you use controllers (Deployments) that recreate them.

**Q53. Difference between Deployment, StatefulSet, and DaemonSet?**
**Deployment:** manages stateless replicas, supports rolling updates/rollback — the default for stateless apps. **StatefulSet:** for stateful apps needing stable network identity and persistent storage per pod, ordered startup (databases, Kafka). **DaemonSet:** runs one pod per node — for node-level agents (log collectors, monitoring like node_exporter, CNI).

**Q54. What is a ReplicaSet?**
It ensures a specified number of identical pod replicas are running. Deployments manage ReplicaSets for you (a new ReplicaSet per version, enabling rollout/rollback). You rarely create ReplicaSets directly.

**Q55. What are Kubernetes Services and their types?**
A Service gives a stable network endpoint to a set of pods (which have changing IPs), with load balancing via label selectors. Types: **ClusterIP** (internal-only, default), **NodePort** (exposes on each node's port), **LoadBalancer** (provisions a cloud LB for external access), **ExternalName** (DNS alias). Ingress sits on top for HTTP routing.

**Q56. What is an Ingress?**
An API object that manages external HTTP/HTTPS access to services, providing routing by host/path, TLS termination, and load balancing — through an Ingress Controller (nginx, Traefik, ALB). It lets many services share one external entry point instead of a LoadBalancer per service.

**Q57. How does Kubernetes do self-healing?**
Controllers continuously reconcile actual state to desired state. If a pod crashes, the ReplicaSet recreates it; if a node dies, pods are rescheduled elsewhere; liveness probes restart unhealthy containers; readiness probes remove unhealthy pods from Service load balancing until they recover.

**Q58. Difference between liveness, readiness, and startup probes?**
**Liveness:** is the app alive? If it fails, K8s restarts the container (recovers deadlocks). **Readiness:** is the app ready to serve traffic? If it fails, the pod is removed from Service endpoints but not restarted (handles temporary unavailability). **Startup:** for slow-starting apps — delays liveness/readiness checks until the app has started, preventing premature restarts.

**Q59. How do you manage configuration and secrets in Kubernetes?**
**ConfigMaps** for non-sensitive config (env vars, config files), **Secrets** for sensitive data (base64-encoded, ideally encrypted at rest with KMS). Both mount as env vars or files. For real security use external secret managers (External Secrets Operator + Vault/AWS Secrets Manager) and enable etcd encryption + RBAC. Base64 is encoding, not encryption — call that out.

**Q60. How does scaling work in Kubernetes?**
Manual: `kubectl scale deployment app --replicas=5`. Automatic: **HPA** (Horizontal Pod Autoscaler) scales pod count based on CPU/memory/custom metrics; **VPA** adjusts pod resource requests; **Cluster Autoscaler** adds/removes nodes when pods can't be scheduled. HPA + Cluster Autoscaler together scale both pods and infrastructure.

**Q61. What are requests and limits?**
**Requests** = guaranteed resources the scheduler uses to place a pod (reserved). **Limits** = the ceiling a container can use. If a container exceeds its memory limit it's OOMKilled; exceeding CPU limit throttles it. Setting them right prevents noisy-neighbor problems and enables proper scheduling and autoscaling.

**Q62. How do you perform a rolling update and rollback?**
Update the image: `kubectl set image deployment/app app=img:v2` — the Deployment gradually replaces pods (controlled by `maxSurge`/`maxUnavailable`), keeping the app available. Check status: `kubectl rollout status deployment/app`. Rollback: `kubectl rollout undo deployment/app` reverts to the previous ReplicaSet. History: `kubectl rollout history`.

**Q63. What is Helm?**
A package manager for Kubernetes. A Helm **chart** templates K8s manifests with configurable `values.yaml`, so you deploy the same app across environments with different values, version releases, and roll back easily (`helm rollback`). It avoids copy-pasting near-identical YAML for every environment.

**Q64. How do you troubleshoot a pod stuck in CrashLoopBackOff / Pending?**
CrashLoopBackOff: `kubectl logs <pod> --previous` (logs from the crashed container), `kubectl describe pod <pod>` for events/exit codes — usually a bad config, missing dependency, failed probe, or OOM. Pending: `kubectl describe pod` shows scheduling failures — insufficient resources, no matching node (taints/affinity), unbound PVC, or image pull errors. Work from events, then logs, then config.

---

## Section 7 — Infrastructure as Code: Terraform & Ansible (Q65–Q76)

**Q65. What is Terraform and how does it work?**
A declarative IaC tool that provisions infrastructure across providers (AWS, Azure, GCP). You write `.tf` files describing desired state; Terraform builds a dependency graph, `plan` shows what will change, `apply` makes it so, and it tracks reality in a **state file**. It's provider-agnostic and cloud-neutral.

**Q66. What is the Terraform state file and why is it important?**
`terraform.tfstate` maps your config to real-world resources — it's how Terraform knows what it already created. It's critical and sensitive (can contain secrets). Store it **remotely** (S3 + DynamoDB lock, Terraform Cloud) so the team shares one state and it's locked against concurrent applies. Never edit it by hand; never commit it to Git.

**Q67. What is state locking and why does it matter?**
Locking prevents two people from running `apply` simultaneously and corrupting the state. With an S3 backend you use a DynamoDB table for locks; Terraform Cloud handles it automatically. Without locking, concurrent applies can create conflicting or duplicate resources.

**Q68. Difference between `terraform plan` and `terraform apply`?**
`plan` is a dry run — it shows what Terraform *would* create/change/destroy without touching anything (great for PR review). `apply` executes those changes. Best practice: review the plan (ideally in CI as a PR comment) before applying, and require approval for prod applies.

**Q69. What are Terraform modules?**
Reusable, parameterized groups of resources — like functions for infrastructure. You write a module once (e.g., a "VPC" or "EKS cluster" module) and reuse it across environments/projects with different inputs. Promotes DRY, consistency, and easier maintenance.

**Q70. How do you manage multiple environments (dev/stage/prod) in Terraform?**
Options: separate directories per environment with shared modules and per-env `tfvars` (clear, explicit — common at scale), or Terraform workspaces (same config, separate state — lighter but easy to misuse). Separate state per environment either way, so a prod apply can never touch dev.

**Q71. What's the difference between `count` and `for_each`?**
Both create multiple resource instances. `count` uses a number and indexes by position (0,1,2) — reordering the list can force recreation. `for_each` uses a map/set and keys by a stable identifier — safer for lists that change, because adding/removing one item doesn't shuffle the others.

**Q72. What is Ansible and how is it different from Terraform?**
Ansible is a configuration management tool — it configures/installs software on existing servers (procedural, agentless over SSH, uses playbooks). Terraform is a provisioning tool — it creates infrastructure (declarative, state-based). Common combo: Terraform provisions the servers, Ansible configures them. Terraform = "build the house," Ansible = "furnish it."

**Q73. What is idempotency in Ansible?**
Running the same playbook multiple times produces the same result without unintended changes — if the desired state already exists, Ansible reports "ok/unchanged" rather than re-doing it. Ansible modules are designed to be idempotent (e.g., the `package` module won't reinstall an already-installed package). This makes runs safe and repeatable.

**Q74. What are Ansible playbooks, roles, and inventory?**
**Inventory:** the list of managed hosts (static file or dynamic). **Playbook:** a YAML file of plays/tasks describing what to do on which hosts. **Roles:** a reusable, structured way to organize tasks, handlers, variables, and templates (like modules) so playbooks stay clean and shareable.

**Q75. How does Ansible handle secrets?**
**Ansible Vault** encrypts sensitive files/variables (`ansible-vault encrypt`), decrypted at runtime with a password or key. You can also integrate external secret managers. Never store plaintext secrets in playbooks or inventory committed to Git.

**Q76. What is immutable infrastructure?**
Instead of modifying running servers (mutable), you build a new image/instance for every change and replace the old one — servers are never patched in place. Benefits: no configuration drift, predictable/reproducible, easy rollback (redeploy old image), consistent across environments. Tools: Packer (build images) + Terraform (deploy) + auto-scaling groups.

---

## Section 8 — Cloud / AWS (Q77–Q86)

**Q77. What is a VPC and its key components?**
A Virtual Private Cloud is your isolated network in AWS. Components: subnets (public/private, per AZ), route tables, an Internet Gateway (public internet access), NAT Gateway (outbound internet for private subnets), security groups (stateful, instance-level firewall), and NACLs (stateless, subnet-level). Public subnet = route to IGW; private subnet = route to NAT for egress only.

**Q78. Difference between a security group and a NACL?**
Security groups are stateful (return traffic auto-allowed), operate at the instance/ENI level, and support allow rules only. NACLs are stateless (must allow return traffic explicitly), operate at the subnet level, and support both allow and deny rules. SGs are the primary control; NACLs add a subnet-level layer.

**Q79. What's the difference between EC2, ECS, EKS, Lambda, and Fargate?**
EC2: raw virtual machines you manage. ECS: AWS-native container orchestration. EKS: managed Kubernetes. Fargate: serverless compute for containers (no node management) — works under ECS/EKS. Lambda: serverless functions, event-driven, pay-per-invocation, no servers at all. Choice depends on control vs operational overhead.

**Q80. What is IAM and how do roles differ from users?**
IAM controls who can do what in AWS. **Users** are long-lived identities with credentials (for humans/CI). **Roles** are assumable identities with temporary credentials and no permanent keys — used by services (EC2, Lambda) and for cross-account access. Best practice: use roles and temporary credentials over long-lived access keys; apply least privilege.

**Q81. What's the difference between S3, EBS, and EFS?**
S3: object storage, unlimited, accessed via API, great for backups/static assets/data lakes. EBS: block storage attached to a single EC2 instance (like a virtual disk). EFS: shared file storage (NFS) mountable by many instances at once. Pick by access pattern: object vs single-attach block vs shared file.

**Q82. How do you make an application highly available in AWS?**
Deploy across multiple Availability Zones behind a load balancer, use Auto Scaling Groups (replace failed instances, scale on demand), multi-AZ RDS for the database, health checks, and for higher tiers multi-region with Route 53 failover. No single point of failure at the AZ level.

**Q83. What is auto-scaling and how does it work?**
An Auto Scaling Group maintains a desired number of instances and scales in/out based on policies — target tracking (keep CPU at 50%), step scaling, or scheduled. It replaces unhealthy instances automatically and works with the load balancer. Ensures capacity matches demand and improves resilience.

**Q84. How do you manage cost in the cloud?**
Right-size instances, use auto-scaling to avoid over-provisioning, use Spot instances for fault-tolerant workloads, Reserved Instances/Savings Plans for steady baseline load, delete unused resources (orphaned volumes, idle LBs), set budgets/alerts, use lifecycle policies on S3, and tag resources for cost attribution. Cost visibility (Cost Explorer) drives decisions.

**Q85. What's the difference between horizontal and vertical scaling?**
Vertical (scale up): bigger instance (more CPU/RAM) — simple, has a ceiling, often needs restart. Horizontal (scale out): more instances behind a load balancer — near-limitless, better fault tolerance, needs stateless design. Cloud/DevOps favors horizontal scaling with auto-scaling.

**Q86. What is a load balancer and the AWS types?**
It distributes incoming traffic across multiple targets for availability and scale. AWS: **ALB** (Layer 7, HTTP/HTTPS, path/host routing — for web apps), **NLB** (Layer 4, TCP/UDP, ultra-high performance, static IP), and **Gateway LB** (for virtual appliances). Choose ALB for HTTP routing, NLB for raw TCP/low latency.

---

## Section 9 — Monitoring & Observability (Prometheus/Grafana) (Q87–Q94)

**Q87. What are the four golden signals?**
Latency (time to serve requests), Traffic (demand/requests per second), Errors (rate of failed requests), and Saturation (how full the system is — CPU, memory, queue depth). They give a user-focused view of service health and map directly to what to monitor and alert on.

**Q88. How does Prometheus work?**
Pull-based: Prometheus scrapes metrics from HTTP `/metrics` endpoints on targets on a schedule, stores them as time-series in its local TSDB, and evaluates recording/alerting rules. Exporters expose metrics for third-party systems; Alertmanager handles alert routing. Targets are found via static config or service discovery (Kubernetes, Consul).

**Q89. What are the Prometheus metric types?**
Counter (only increases — request counts; use `rate()`), Gauge (up and down — memory, temperature), Histogram (bucketed observations — latencies; enables `histogram_quantile()`, aggregatable), and Summary (client-side quantiles, not aggregatable across instances).

**Q90. Write a PromQL query for the request error rate.**
```promql
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```
This is the ratio of 5xx responses to total requests over a 5-minute window.

**Q91. What is Alertmanager?**
It receives alerts from Prometheus and handles grouping (bundling related alerts), inhibition (suppress lower-priority when a higher one fires), silencing (mute during maintenance), deduplication, and routing to receivers (PagerDuty, Slack, email) based on labels.

**Q92. What does Grafana do and how does it pair with Prometheus?**
Grafana visualizes metrics from data sources like Prometheus in dashboards, and can also alert. Prometheus stores/computes; Grafana displays. Use template variables for reusable dashboards, annotate deploys to correlate releases with metric changes, and store dashboards as code (JSON in Git) for versioning.

**Q93. What's the difference between monitoring and observability?**
Monitoring watches for known/predefined failure conditions (dashboards, alerts you set up in advance). Observability is the ability to ask arbitrary new questions about system state from its telemetry to debug "unknown unknowns." The three pillars — metrics, logs, traces — enable observability.

**Q94. How would you set up log aggregation?**
Ship logs from all sources to a central store: ELK/EFK stack (Elasticsearch + Logstash/Fluentd + Kibana) or Loki + Promtail + Grafana (lighter, label-based, pairs with Prometheus). Structured (JSON) logs, centralized search, retention policies, and correlation with metrics/traces. In Kubernetes, a DaemonSet log collector (Fluentd/Fluent Bit/Promtail) reads container logs from each node.

---

## Section 10 — Networking & Security (Q95–Q100 core) + Scenario/Behavioral

**Q95. Explain the OSI layers relevant to DevOps.**
You mainly care about Layer 3 (Network — IP routing), Layer 4 (Transport — TCP/UDP, ports, load balancers like NLB), and Layer 7 (Application — HTTP/HTTPS, ALB, Ingress). Knowing which layer a component operates at explains routing, load balancing, and where TLS terminates.

**Q96. What happens when you type a URL and hit enter? (DevOps lens)**
DNS resolves the domain to an IP → TCP handshake (or QUIC) to the server → TLS handshake for HTTPS → HTTP request sent → hits a load balancer / ingress → routed to a backend service/pod → response returned and rendered. As a DevOps engineer you'd relate each step to DNS (Route 53), LB (ALB), TLS certs, and the service mesh/ingress.

**Q97. How do you secure a CI/CD pipeline and infrastructure?**
Least-privilege IAM/RBAC, secrets in a vault (never in code), signed/scanned container images, SAST/DAST/dependency and IaC scanning as gates, network segmentation and security groups, immutable infra, audit logging, MFA, rotate credentials, and pin/verify dependencies. Security as code and as a pipeline gate.

**Q98. Scenario: A production deployment caused high error rates. What do you do?**
First mitigate — roll back to the last known-good version (stop the bleeding) rather than debug live. Then confirm recovery via metrics (error rate back to baseline), communicate status to stakeholders, and only then investigate root cause using logs, traces, and the deploy timeline. Finally, a blameless postmortem with action items (e.g., add a canary stage or a smoke-test gate that would've caught it). 💡 Order matters: mitigate → communicate → root-cause → prevent.

**Q99. How do you manage competing priorities from multiple stakeholders? (from the JD)**
Make trade-offs explicit and data-driven. Prioritize by user/business impact, communicate constraints and timelines transparently, and when two urgent things conflict, surface it to the owner with the trade-off clearly stated rather than silently dropping one. Use shared metrics (SLOs, error budgets, pipeline health) as the neutral language so it's not opinion vs opinion. 💡 Have a real STAR example ready — a time you balanced a feature deadline against a stability concern.

**Q100. Tell me about a time you improved a process / had impact. (STAR closer)**
Structure it: **Situation** (e.g., slow/flaky pipeline, frequent failed deploys, manual toil), **Task** (your goal), **Action** (what you built — parallelized stages, added canary + auto-rollback, containerized builds, IaC'd the environment, added Prometheus alerting), **Result** — quantified ("cut deploy time from 40 to 12 min, reduced failed deploys by X%, eliminated Y hours of manual work weekly"). Always land on a measurable outcome. 💡 Prepare two of these — one technical, one collaboration/conflict.

---

## Rapid-Fire Cheat Sheet (last-minute revision)

| Topic | One-liner |
|---|---|
| CI vs CD vs CD | Integrate → ready-to-deploy (gated) → auto-deploy |
| DORA metrics | Deploy freq, Lead time, Change fail rate, MTTR |
| IaC | Infra in version-controlled code (Terraform) |
| Config mgmt | Configure existing servers (Ansible) |
| Terraform state | Maps config to real resources; store remote + locked |
| plan vs apply | Dry run vs execute |
| count vs for_each | Index by position vs stable key |
| Ansible idempotency | Re-runs produce same state, no side effects |
| Container vs VM | Shares host kernel vs full guest OS |
| Image vs container | Blueprint vs running instance |
| Multi-stage build | Build heavy, ship slim |
| CMD vs ENTRYPOINT | Default args vs fixed executable |
| Pod | Smallest K8s unit, shares net/storage |
| Deployment vs StatefulSet vs DaemonSet | Stateless / stateful+identity / one-per-node |
| Service types | ClusterIP, NodePort, LoadBalancer, ExternalName |
| Liveness vs Readiness | Restart if dead vs remove from LB if not ready |
| HPA / Cluster Autoscaler | Scale pods / scale nodes |
| requests vs limits | Reserved vs ceiling (OOMKilled if over mem) |
| Helm | K8s package manager (charts + values) |
| SG vs NACL | Stateful instance-level vs stateless subnet-level |
| IAM role vs user | Temp assumable creds vs long-lived identity |
| Blue-green vs canary | Instant switch vs gradual % ramp |
| 4 golden signals | Latency, Traffic, Errors, Saturation |
| Counter vs Gauge | Only up vs up-and-down |
| Merge vs Rebase | Preserve history vs linear (rewrites) |
| Reset vs Revert | Rewrite history (local) vs safe new commit (shared) |
| Blameless postmortem | Fix the system, not blame the person |
| Mitigate → Communicate → Root-cause | Incident response order |

---

*Client-round prep tips:*
1. Be ready to **whiteboard one real pipeline** end to end: commit → build → test → scan → image → registry → deploy (with strategy) → verify. Name the actual tools you used.
2. Have **two STAR stories** ready — one deeply technical, one about collaboration/conflict/stakeholders (directly maps to the JD).
3. Expect **"why did you choose X over Y"** follow-ups (Jenkins vs GitHub Actions, rolling vs canary, count vs for_each) — the *reasoning* is what separates 4 YOE from 1 YOE.
4. Know **your own resume cold** — every tool/project on it is fair game for deep-dive questions.
