Intro -->

I’m a DevOps Engineer with 3.5+ years of experience working on CI/CD, Kubernetes, and cloud platforms like AWS and Azure.

In my current role, I design and improve CI/CD pipelines using GitHub Actions with environment-based workflows, approvals, and rollback strategies. This has helped reduce deployment failures by around 30–40%.

I work closely with Docker and Kubernetes, managing microservices deployments, handling scaling, and resolving issues like crash loops and failed rollouts to ensure high availability.

I’ve also implemented GitOps using ArgoCD to manage deployments across dev, staging, and production.

On the monitoring side, I use Prometheus, Grafana, and Application Insights to quickly detect and respond to issues.

Overall, I focus on building reliable systems, automating processes, and handling production challenges effectively.

---

DAY-TO-DAY RESPONSIBILITIES -->

A typical day for me usually starts with checking our monitoring dashboards like Grafana and any alerts from Alertmanager or Application Insights. I just want to make sure everything is stable and nothing broke overnight.

Once that looks good, I go through emails, tickets, or any tasks assigned for the day. If there’s a production issue, that becomes my top priority. For example, I’ll jump into Kubernetes, check pod status, look at logs and events, and try to quickly find what’s going wrong and fix it.

After that, I usually check our CI/CD pipelines in GitHub Actions. If any build or deployment has failed, I dig into the logs, fix the issue, and re-run the pipeline to get things back on track.

The rest of my day is a mix of improving things and handling ongoing work. Sometimes I’m optimizing pipelines, sometimes updating Docker images or working on Kubernetes deployments. If there are scaling or performance issues, I spend time tuning resources or debugging them. I also try to automate repetitive tasks using Bash or Python whenever I see an opportunity.

Throughout the day, I stay in touch with developers—whether it’s for release planning, debugging environment-specific issues, or making sure deployments across dev, staging, and production are smooth.

So overall, it’s a mix of monitoring, troubleshooting, automation, and collaboration, with a strong focus on keeping systems stable and releases smooth.

---

# 🔹 1. GitHub Actions CI/CD Pipeline (Real Example)

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ "dev", "staging", "main" ]
  pull_request:
    branches: [ "main" ]

env:
  IMAGE_NAME: my-app

jobs:
  build-test-scan:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Install Dependencies
        run: npm install

      - name: Run Unit Tests
        run: npm test

      # 🔐 SAST - Code Scan (CodeQL)
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

      # 🐳 Build Docker Image
      - name: Build Docker Image
        run: docker build -t $IMAGE_NAME:${{ github.sha }} .

      # 🔍 Image Scan (Trivy)
      - name: Scan Docker Image
        uses: aquasecurity/trivy-action@v0.20.0
        with:
          image-ref: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          severity: HIGH,CRITICAL

      # 🔐 Push to ECR
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-role
          aws-region: ap-south-1

      - name: Login to ECR
        run: |
          aws ecr get-login-password \
          | docker login --username AWS --password-stdin <account>.dkr.ecr.ap-south-1.amazonaws.com

      - name: Push Image
        run: |
          docker tag $IMAGE_NAME:${{ github.sha }} <ECR_URL>/$IMAGE_NAME:${{ github.sha }}
          docker push <ECR_URL>/$IMAGE_NAME:${{ github.sha }}

  deploy:
    needs: build-test-scan
    runs-on: ubuntu-latest

    steps:
      - name: Update Helm Values (GitOps Repo)
        run: |
          git clone https://github.com/org/gitops-repo.git
          cd gitops-repo
          sed -i 's/tag:.*/tag: "${{ github.sha }}"/' values.yaml
          git commit -am "Update image tag"
          git push
```

---

# 🔹 2. Helm Chart (Deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  selector:
    matchLabels:
      app: my-app

  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:
            - containerPort: 3000

          envFrom:
            - configMapRef:
                name: my-app-config

          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10

          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

# 🔹 3. DevSecOps Integration (Explain Like This)

Instead of just tools, say this:

* SAST → GitHub CodeQL (code-level vulnerabilities)
* Image scanning → Trivy (HIGH/CRITICAL blocking)
* Secrets scanning → GitHub Secret Scanning + pre-commit hooks
* Dependency scan → npm audit / Snyk
* IAM security → Role-based access (no hardcoded credentials)
* Policy → Fail pipeline if critical vulnerability found

👉 Strong line:

> “Security is integrated as a gate in the pipeline, not as a separate step.”

---

# 🔹 4. Real Metrics (VERY IMPORTANT for panel)

Use numbers like this (don’t be generic):

* Deployment frequency → **2–5 deployments per day**
* MTTR (Mean Time to Recovery) → **Reduced from ~1 hour to ~15–20 minutes**
* Deployment success rate → **~95%+**
* Downtime → **Reduced using rolling/canary deployments**
* Pipeline time → **Reduced by ~30% using caching & parallel jobs**
* Incident detection → **Improved by ~40% using alerts & dashboards**

👉 Strong line:

> “We focused on reducing MTTR and improving deployment reliability rather than just increasing speed.”

---

YAML YOU CAN SPEAK (SHORT VERSION)

👉 GitHub Actions snippet:

```yaml
- name: Build Docker Image
  run: docker build -t myapp:${{ github.sha }} .

- name: Push to ECR
  run: |
    aws ecr get-login-password | docker login ...
    docker push myapp:${{ github.sha }}

```

---

👉 Kubernetes Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 80
```
---


---

Production Issues -->

Story 1 -->

One interesting production issue I handled was when our Kubernetes pods kept getting OOMKilled, and honestly, it was quite confusing at the beginning.

From our dashboards, everything looked completely normal. Memory usage was stable around 50–60%, there were no alerts, no spikes, and even application logs were clean. So initially, nothing indicated a memory problem.

But the pods were still restarting randomly.

So we started digging deeper. I checked kubectl describe pod, and that’s where we saw exit code 137, which confirmed it was OOMKilled. That’s when we realized something was off—because the metrics were not matching the actual behavior.

Then we took a step back and thought about how we were looking at the data. Most of our dashboards were showing averaged metrics over time. That’s when it clicked—what if the issue is not sustained usage, but short-lived spikes?

We investigated further and understood that these spikes were happening for very short durations—just milliseconds or seconds—so they were not getting captured due to Prometheus scrape intervals and the way dashboards smooth data.

And since Kubernetes enforces strict memory limits, even a tiny spike beyond the limit can immediately kill the container, without giving the application any chance to log an error.

To validate this, we started correlating pod restarts with container-level memory metrics like container_memory_working_set_bytes, and that gave us more clarity.

To fix the issue, we slightly increased the memory limits and added some buffer so the application had room to handle spikes. Along with that, we improved our monitoring by focusing on peak values instead of averages and adjusting queries to better capture short-term behavior.

We also worked with the development team to optimize parts of the application that were causing sudden memory usage during heavy processing.

After these changes, the issue was resolved.

This incident really changed how I look at production systems. I learned that most real-world failures don’t happen in steady state—they happen in short bursts or edge cases that you might not see unless you’re looking at the right metrics in the right way.

---

---

# 🔥 1. IF INTERVIEWER ASKS: “WHAT WAS YOUR ROLE?”

👉
“In this issue, I was mainly responsible for Kubernetes-level debugging and monitoring analysis.

I identified the OOMKilled pattern from pod events, correlated it with metrics, and worked with the team to analyze why dashboards were not showing the real issue.

I also helped improve monitoring queries and suggested increasing memory limits and adding buffer capacity to handle spikes.”

---

# 🔥 2. IF THEY ASK: “HOW DID YOU CONFIRM SPIKES?”

👉
“We couldn’t directly see spikes initially due to scrape intervals, but we inferred them by correlating pod restarts with workload patterns and analyzing peak memory metrics instead of averages.

After that, we adjusted monitoring queries using max-based functions and shorter windows to better capture such behavior.”

---

# 🔥 3. STRONG TECHNICAL LINE (VERY IMPRESSIVE)

Use this exact line 👇

👉
“Kubernetes doesn’t care about average memory usage—it enforces hard limits, so even a millisecond-level spike beyond the limit can trigger OOMKilled.”

---

# 🔥 4. COMMON FOLLOW-UP QUESTIONS (BE READY)

---

### 👉 “Why no logs were generated?”

👉
“Because OOMKill sends SIGKILL, the application doesn’t get time to log anything.”

---

### 👉 “Why dashboards didn’t show spikes?”

👉
“Due to scrape interval and use of average metrics instead of peak metrics.”

---

### 👉 “How to prevent this in future?”

👉
“Add memory headroom, monitor peak usage, reduce scrape interval, and optimize application behavior.”

---

### 👉 “Why not just increase memory always?”

👉
“That increases cost and doesn’t solve root cause, so we balance between optimization and capacity.”

---

# 🔥 5. FINAL UPGRADE (THIS MAKES YOU STAND OUT)

End your answer with this:

👉
“This incident changed how I approach monitoring—I now focus not just on averages, but also on peak behavior and edge cases, especially in production systems.”

---

# 🚀 REALITY CHECK

Most candidates say:
❌ “OOMKilled means memory issue”

You are now saying:
✅ “Short-lived spikes hidden by averaging + scrape interval + container-level metrics”


---
 
Story 2 -->

One major issue I handled was during a production deployment where new pods suddenly started going into CrashLoopBackOff.

I first noticed it through Grafana, where pod restarts had spiked right after the release. I quickly checked the pods using kubectl describe and logs, and found that the application was failing at startup due to a missing environment variable.

Digging deeper, I realized the issue came from a Helm values file that wasn’t updated properly during the pipeline run.

To minimize impact, I immediately rolled back to the previous stable version so the service could recover. Then I fixed the configuration in our GitOps repository and redeployed using ArgoCD.

After everything was stable, I added validation checks in the CI pipeline to ensure required configurations are present before deployment.

This helped us avoid similar issues in future and made the deployment process more reliable.

---


---

# 🔥 CI/CD & DEVOPS (Story Style)

---

### 👉 How exactly GitHub Actions integrates with ECR?

👉
“In our setup, when I initially implemented the pipeline, we wanted to avoid storing AWS credentials in GitHub for security reasons. So instead of using access keys, I configured OIDC-based authentication.

When the pipeline runs, GitHub Actions assumes an IAM role using `aws-actions/configure-aws-credentials`. Once the role is assumed, we use AWS CLI to log in to ECR and push the Docker image.

I verified this by checking that no static credentials were stored, and access was controlled through IAM policies. This made our pipeline more secure and easier to manage.”

---

### 👉 What happens if pipeline fails at image scan stage?

👉
“Yes, we actually faced this when we integrated Trivy. Initially, builds were failing frequently because of vulnerabilities in base images.

What we did was configure Trivy to fail the pipeline only for HIGH and CRITICAL vulnerabilities. When it fails, I check the report, identify whether it’s from the base image or application dependency, and then update the base image or patch dependencies.

This helped us enforce security without blocking deployments unnecessarily.”

---

### 👉 How do you rollback deployment?

👉
“We had a situation where a deployment caused API failures. Since we were using Kubernetes with rolling updates, I immediately checked the rollout status.

To minimize impact, I executed `kubectl rollout undo` to revert to the previous stable version. In our GitOps setup, we also had the option to revert the commit in the GitOps repo, which ArgoCD would automatically sync.

This allowed us to restore service quickly while we investigated the root cause.”

---

# 🔥 KUBERNETES (Story Style)

---

### 👉 Difference between liveness and readiness probe?

👉
“I understood the importance of this when one of our services was getting traffic even though it wasn’t fully initialized.

We added readiness probes to ensure traffic is only sent when the application is ready. Liveness probes helped us detect if the app got stuck and needed a restart.

After implementing both correctly, we saw fewer failed requests during deployments.”

---

### 👉 How do you debug CrashLoopBackOff?

👉
“One time, after a deployment, pods started crashing repeatedly. I first checked `kubectl get pods` and saw CrashLoopBackOff status.

Then I used `kubectl describe pod` to check events and `kubectl logs` to see application errors. It turned out a required environment variable was missing.

After fixing the config and redeploying, the issue was resolved. Later, I added validation in pipeline to avoid such issues.”

---

### 👉 How does HPA work?

👉
“We configured HPA when we noticed increased load on our services.

HPA monitors metrics like CPU usage from the metrics server. Based on thresholds, it automatically scales the number of pods up or down.

After implementing HPA, our application handled traffic spikes better without manual intervention.”

---

# 🔥 SECURITY (Story Style)

---

### 👉 How do you avoid secrets leakage?

👉
“Initially, we had some configs where secrets were manually passed, which wasn’t secure.

So we moved to using GitHub Secrets for CI and IAM roles for AWS access. For Kubernetes, we used Kubernetes Secrets and ensured they were not exposed in logs or code.

We also restricted access using least privilege policies. This significantly improved our security posture.”

---

### 👉 Difference IAM role vs access key?

👉
“In earlier setups, access keys were used, but they had risks like exposure and manual rotation.

So we moved to IAM roles, especially with OIDC in GitHub Actions. IAM roles provide temporary credentials and are more secure since they are not stored anywhere.

This change helped us eliminate long-lived credentials from our pipeline.”

---

# 🔥 GITOPS (Story Style)

---

### 👉 Why ArgoCD over Jenkins CD?

👉
“We initially evaluated push-based deployment using Jenkins, but managing credentials and direct cluster access was complex and less secure.

With ArgoCD, we adopted a pull-based GitOps model where the cluster pulls changes from Git. This reduced security risks and made deployments more transparent.

Also, rollback became easier by just reverting Git commits.”

---

### 👉 What is pull vs push model?

👉
“In push model, like Jenkins, the CI tool directly deploys to the cluster, which requires giving cluster access to CI.

In pull model, like ArgoCD, the cluster continuously watches the Git repository and pulls changes. This is more secure and aligns with GitOps principles.

In our setup, we preferred pull model because it reduced external access to the cluster and improved auditability.”

---

# 🔥 FINAL TIP (VERY IMPORTANT)

When you answer like this:

❌ “We use HPA, ArgoCD, ECR…”

You sound average.

When you say:

✅ “We faced X issue → I checked logs → fixed config → improved pipeline…”

You sound like:
👉 **REAL PRODUCTION ENGINEER**

---
