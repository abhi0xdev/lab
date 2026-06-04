# Si2 Technologies — Spoken Answer Scripts
### Self-Intro · Project Workflow · Day-to-Day · Production Story

> These are written the way you'd *say* them. Read each aloud 2–3 times, then deliver it in your own words — don't memorize word-for-word, memorize the *structure*. Aim for the timing noted on each.

---

## 1. SELF-INTRODUCTION  (~45–60 seconds)

"Hi, I'm Abhinandan Gayaki. I'm a DevOps Engineer with around 4 years of experience, currently at Cognizant, where I work mostly on building and running production CI/CD pipelines and Kubernetes platforms on Azure and AWS.

Day to day, my work centers on three things — automating delivery, keeping production reliable, and improving observability. On the delivery side, I've redesigned CI/CD pipelines using GitHub Actions with environment-based workflows, approval gates, and rollback mechanisms, which cut our deployment failures significantly. On the platform side, I manage AKS workloads across multiple environments — scaling, troubleshooting, and keeping uptime high. And I've built out monitoring using Prometheus, Grafana, and Application Insights so we catch issues before users do.

I'm also Azure certified — AZ-400 DevOps Expert, AZ-204, and AZ-900. I enjoy the troubleshooting side most — getting into a production issue, finding the real root cause, and then automating it so it doesn't happen again. I'm looking for a role where I can go deeper on Kubernetes and CI/CD at scale, which is why this opportunity at Si2 interested me."

**Delivery tips:** Start with name + years + current role. Don't list every tool — group them by *what they achieve*. End by connecting to *this* role. Keep it under a minute; they'll dig into details after.

---

## 2. PROJECT WORKFLOW  (~60–90 seconds)

Frame this as "how a code change goes from a developer's laptop to production safely." Walk it as a journey:

"Let me walk you through our end-to-end workflow.

A developer pushes code to a feature branch and opens a pull request. We have **branch protection rules** on main, so nothing merges without passing a **PR validation pipeline** — that runs the build, unit tests, linting, and a code review approval. This is the first quality gate, and it stopped a lot of faulty merges.

Once merged, the **CI pipeline in GitHub Actions** kicks in: it builds the app, runs the full test suite, runs security and image scans, then builds a **Docker image** and pushes it to **ACR** — tagged with the commit SHA, so every image is traceable and immutable.

For delivery, we use **environment-based workflows**. The same image promotes through dev → staging → production. Dev deploys automatically. Staging and prod are gated behind **GitHub Environments with required approvers** — so production needs a human sign-off, which gives us control and an audit trail.

Deployment to **AKS** happens through progressive strategies — rolling updates, and blue-green for the riskier releases — so there's no downtime. In one project I took this further with **GitOps using ArgoCD**: the manifests live in Git, ArgoCD continuously syncs the cluster to match, and rollback is just reverting a commit.

Finally, **observability closes the loop** — Prometheus and Grafana plus Application Insights validate that the release is healthy. If metrics or alerts go wrong, we roll back, either with `kubectl rollout undo` or by reverting the Git commit in the ArgoCD setup."

**Why this works:** It's a clean narrative with quality gates at each stage, and it naturally surfaces every keyword on your resume (GitHub Actions, ACR, AKS, approval gates, blue-green, ArgoCD, Prometheus). Pause after each stage so they can interrupt with questions.

---

## 3. DAY-TO-DAY ACTIVITIES  (~45–60 seconds)

"My day usually starts by checking our **Grafana and Azure Monitor dashboards** and triaging any overnight alerts — making sure all environments are healthy before anything else.

After standup, the work splits across a few areas. A big part is **maintaining and improving CI/CD pipelines** — fixing flaky builds, adding stages, speeding things up. I review **pull requests** and validation pipelines for the team. I handle **deployments** through to production, including the approval steps for prod releases.

A good chunk is **troubleshooting** — investigating pod issues, failed deployments, resource problems, and digging into logs and metrics to find root cause. I also do **infrastructure work** through Terraform when we need new resources, and I write **Bash and Python scripts** to automate anything repetitive — that's an ongoing theme, removing manual toil.

And I'm regularly **collaborating with developers** — helping them with deployment issues, container config, or pipeline problems. When I'm on-call, incident response takes priority over everything else."

**Tip:** Group the day into themes — monitoring, pipelines, deployments, troubleshooting, automation, collaboration. It signals you understand the *breadth* of the role, not just one tool.

---

## 4. PRODUCTION STORY — "Pod stuck in Pending, but CPU and metrics look fine"  (~90 seconds)

This is your flagship technical story. Tell it in **STAR** format. The power of it is that you *ruled out the obvious cause* and found the real one.

### The script

**Situation:**
"We had a production incident where a deployment to our AKS cluster wasn't completing — new pods were stuck in **Pending**. What made it confusing was that all the dashboards looked perfectly healthy. Node **CPU and memory utilization were low**, well under 50%, so at first glance there was no obvious resource pressure. The team's instinct was 'the cluster has plenty of capacity, why won't it schedule?'"

**Task:**
"My job was to figure out why the scheduler couldn't place the pod and get the release moving again without just throwing more nodes at it blindly."

**Action:**
"I started with the basics — `kubectl describe pod` on the Pending pod and went straight to the **Events** section. The event said **'0/N nodes are available: insufficient cpu/memory.'** That seemed to contradict the dashboards, and that contradiction was the key.

The insight is that the **Kubernetes scheduler places pods based on resource *requests*, not actual usage.** Our monitoring showed *actual* consumption, which was low — but the existing pods had **over-provisioned requests**. So even though the nodes were only 40–50% utilized in reality, almost all of their **allocatable** capacity was already *reserved* by those requests. From the scheduler's point of view the nodes were full, so the new pod's request couldn't fit and it sat in Pending.

I confirmed this with `kubectl describe node` — comparing **Allocatable** versus **Allocated requests**, which showed requests were nearly maxed out while real usage wasn't. I also quickly ruled out the other usual suspects — no unsatisfied nodeSelector or taints, and no unbound PVC."

**Result:**
"For the immediate fix, I scaled the node pool to unblock the release. But the real fix was **right-sizing the resource requests** — we'd set them far higher than what the apps actually used. After correcting the requests based on real metrics, the same nodes could schedule far more pods, scheduling stopped failing, and we actually **reduced our node count and cost**. I also added an alert on the **allocated-requests-versus-allocatable** ratio, not just raw CPU, so we'd catch this *before* a deployment got stuck rather than during one."

### Why this is a great story
- It shows the single most important Kubernetes scheduling concept: **requests reserve capacity; the scheduler doesn't care about real-time usage.**
- You demonstrate a **systematic diagnostic process** (describe pod → events → describe node → rule out alternatives).
- It ends with **prevention and cost savings**, not just a patch — that's senior thinking.

### Be ready for these follow-ups
- **"What else could cause Pending besides this?"** → Unsatisfied **nodeSelector / node affinity**, **taints** without tolerations, an **unbound PVC** (no PV / storage class / wrong zone), **pod anti-affinity or topology spread** that can't be met, **cluster autoscaler** at max nodes or out of quota, and on AKS specifically the **max-pods-per-node limit** (with Azure CNI a node can run out of pod IP slots even with free CPU).
- **"Requests vs limits again?"** → Request = guaranteed reservation the scheduler uses for placement. Limit = hard ceiling; over-limit memory gets OOMKilled, over-limit CPU gets throttled.
- **"How did you decide the right request values?"** → Looked at actual P95/P99 usage from Prometheus over a representative period, set requests near real usage with some headroom, kept limits higher for burst.
- **"Why not just always add nodes?"** → It hides the real problem and wastes money; the requests were wrong, so more nodes would just get reserved-but-idle the same way.

---

## QUICK DELIVERY REMINDERS
- Speak slowly. Pause after each section so they can interrupt — interviews are conversations, not monologues.
- If you don't know something, say "I haven't used that directly, but my understanding is…" — never bluff a fake answer.
- Tie answers back to real experience whenever you can: "We actually hit this at Cognizant when…"
- For the Pending story, the one line that lands it: **"the scheduler schedules on requests, not on actual usage."** Land that and the interviewer knows you get it.
