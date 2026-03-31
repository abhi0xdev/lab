## 🔐 DevSecOps with GitHub Actions — Mid-Level Interview Notes

---

# 1. Core Concepts (Mid-Level Focus)

### 🔹 DevSecOps

* **Definition:** Integrating security practices into every stage of the CI/CD pipeline.
* **Why:** Detect vulnerabilities early → reduce cost, risk, and production incidents.
* **How it works:** Shift-left security → automate scanning in pipeline.

---

### 🔹 CI/CD Security Integration

* **CI (Continuous Integration):**

  * Code commit → build → test → scan
* **CD (Continuous Deployment):**

  * Secure artifact → deploy with validation

👉 Security gates are added at:

* Code level (SAST)
* Dependencies (SCA)
* Runtime (DAST)
* Infrastructure (IaC scan)

---

### 🔹 GitHub Actions (GHA)

* Event-driven CI/CD platform
* Uses:

  * `workflow.yml`
  * `jobs`
  * `steps`
  * `runners`

👉 Security is implemented via:

* Marketplace actions (Snyk, Trivy, CodeQL)
* Custom scripts

---

### 🔹 Shift-Left Security

* Move security from production → development stage
* Detect vulnerabilities early (PR level)

---

### 🔹 Security Layers in Pipeline

1. **SAST** – Static code analysis
2. **SCA** – Dependency vulnerabilities
3. **Secrets scanning**
4. **Container scanning**
5. **IaC scanning**
6. **DAST (optional, runtime)**

---

# 2. Key Terminology & Definitions

* **SAST:** Scans source code without execution
* **DAST:** Tests running application for vulnerabilities
* **SCA:** Checks third-party dependencies
* **CVE:** Known vulnerability identifier
* **SBOM:** Software Bill of Materials (dependency list)
* **Secrets Management:** Secure handling of credentials
* **Least Privilege:** Minimal access required
* **CodeQL:** GitHub-native static analysis engine
* **Trivy:** Container & filesystem vulnerability scanner
* **OIDC:** Secure authentication without storing secrets

---

# 3. Tools, Frameworks, Technologies

### 🔹 GitHub Native

* **CodeQL**

  * Best for SAST
  * Deep integration with GitHub
* **Dependabot**

  * Auto dependency updates

### 🔹 Security Tools

* **Snyk**

  * SAST + SCA
  * Developer-friendly
* **Trivy**

  * Container + IaC scanning
* **Checkov**

  * Terraform/IaC scanning

### 🔹 Why Use Each

| Tool    | Use Case             |
| ------- | -------------------- |
| CodeQL  | Code vulnerabilities |
| Trivy   | Docker image scan    |
| Snyk    | Dependencies         |
| Checkov | Terraform scan       |

---

# 4. Comprehensive Interview Questions & Answers

---

### Q1: How do you implement DevSecOps in GitHub Actions?

**Answer:**

I design a pipeline with multiple security stages:

1. Trigger on PR/push
2. Run build & unit tests
3. Add security scans:

   * CodeQL → SAST
   * Snyk → dependencies
   * Trivy → container
4. Fail pipeline if critical vulnerabilities found
5. Store reports as artifacts
6. Deploy only if all checks pass

👉 Example flow:

```
Code → PR → Scan → Build → Scan Image → Deploy
```

**Mid-level depth:**

* Explains integration + tools
* Talks about pipeline blocking

**Senior adds:**

* Risk-based thresholds
* Exception handling
* Security SLAs

---

### Q2: How do you scan Docker images in GitHub Actions?

**Answer:**

* Build image
* Use Trivy to scan
* Fail on HIGH/CRITICAL vulnerabilities

---

### Q3: How do you manage secrets securely?

**Answer:**

* Use GitHub Secrets or OIDC
* Avoid hardcoding
* Use environment protection rules

---

### Q4: What happens if vulnerabilities are found?

**Answer:**

* Pipeline fails (for critical severity)
* Report generated
* Developer fixes before merge

---

### Q5: Difference between SAST and DAST?

| SAST        | DAST        |
| ----------- | ----------- |
| Static code | Running app |
| Early stage | Late stage  |
| Fast        | Slower      |

---

### Q6: How do you secure Terraform in pipeline?

**Answer:**

* Use Checkov / tfsec
* Validate before apply

---

### Q7: Scenario — Pipeline is slow due to scans. What do you do?

**Answer:**

* Parallel jobs
* Scan only changed files
* Cache dependencies

---

### Q8: How do you prevent secret leaks?

**Answer:**

* Pre-commit hooks
* GitHub secret scanning
* Vault integration

---

# 5. Code Snippets / GitHub Actions Example

```yaml
name: DevSecOps Pipeline

on:
  push:
    branches: [ "main" ]

jobs:

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run CodeQL
        uses: github/codeql-action/init@v2
      - uses: github/codeql-action/analyze@v2

  container_scan:
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker Image
        run: docker build -t myapp .

      - name: Scan Image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp
          severity: CRITICAL,HIGH
```

---

# 6. Practical & Real-World Scenarios

### 🔹 Scenario 1: Production Deployment Blocked

* Trivy detects critical CVE
* Pipeline stops
* Fix base image → rebuild

---

### 🔹 Scenario 2: Dependency Risk

* Snyk flags vulnerable library
* Replace version or patch

---

### 🔹 Scenario 3: Multi-Environment Deployment

* Dev → auto deploy
* Prod → manual approval + security gate

---

# 7. Performance, Optimization & Edge Cases

* Parallel scanning jobs
* Cache dependencies
* Skip scans for docs-only changes
* Handle false positives with allowlist

---

# 8. Debugging & Troubleshooting

### Common Issues:

* Scan failing → outdated DB
* False positives
* Permission issues in runner

### Approach:

1. Check logs
2. Re-run locally
3. Validate tool configs
4. Compare versions

---

# 9. Security / Reliability Considerations

* Use **OIDC instead of static secrets**
* Enable **branch protection rules**
* Enforce **signed commits**
* Store artifacts securely
* Audit logs enabled

---

# 10. Follow-Up & Probing Questions

* Why did you choose Trivy over Snyk?
* How do you handle false positives?
* What if business needs override security?
* How do you measure security maturity?

---

# 11. Common Mistakes & Red Flags

❌ Only adding scanning without enforcement
❌ Not failing pipeline on critical issues
❌ Hardcoding secrets
❌ Ignoring dependency vulnerabilities

---

# 12. Behavioral / Soft Skills Signals

* Talks about **ownership of security**
* Explains **collaboration with developers**
* Mentions **incident prevention mindset**

---

# 13. Evaluation Guide

| Level  | Signal                                |
| ------ | ------------------------------------- |
| Junior | Knows tools only                      |
| Mid    | Implements pipelines + handles issues |
| Senior | Designs strategy + risk management    |

---

# 14. Flashcards / Quick Revision

* DevSecOps = Security in CI/CD
* SAST = Code scan
* SCA = Dependency scan
* Trivy = Container scan
* CodeQL = GitHub SAST
* OIDC = No secrets auth

---

# 15. References / Further Reading

* GitHub Actions Docs
* OWASP Top 10
* Trivy Documentation
* Snyk Docs
* Terraform Security (Checkov)

---

## 🚀 Final Interview Tip

If interviewer asks:

👉 **“How will you implement DevSecOps in GitHub Actions?”**

Give this structure:

> "I design a multi-stage pipeline integrating SAST, SCA, container, and IaC scanning. I enforce security gates by failing builds on critical vulnerabilities, use OIDC for secure authentication, and optimize scans using parallel jobs and caching. I also handle false positives with allowlists and ensure production deployments pass all security checks."

---

