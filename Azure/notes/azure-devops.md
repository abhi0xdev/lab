## 🔷 Azure DevOps — Mid-Level Interview Notes

---

# 1. Core Concepts (Mid-Level Focus)

### 🔹 What is Azure DevOps?

A cloud-based platform by Microsoft that provides tools for **CI/CD, version control, project management, and testing**.

---

### 🔹 Core Services in Azure DevOps

#### 1. Azure Repos

* Git-based version control
* Supports branching strategies, PRs, and policies
* **Why it works**: Enables collaboration and history tracking

---

#### 2. Azure Pipelines

* CI/CD automation service
* Supports YAML & classic pipelines
* Works with multiple languages/platforms

**How it works:**

* Trigger → Build → Test → Deploy

---

#### 3. Azure Boards

* Work tracking (Epics, Features, User Stories, Tasks)
* Agile/Kanban support

---

#### 4. Azure Artifacts

* Package management (NuGet, npm, Maven, Python)

---

#### 5. Azure Test Plans

* Manual + automated testing support

---

### 🔹 CI vs CD

| Concept                     | Meaning                                            |
| --------------------------- | -------------------------------------------------- |
| CI (Continuous Integration) | Frequent code integration + automated builds/tests |
| CD (Continuous Delivery)    | Code is always ready for release                   |
| CD (Continuous Deployment)  | Automatic production deployment                    |

---

### 🔹 Pipeline Types

* **YAML Pipelines (Preferred)**

  * Version-controlled
  * Declarative
* **Classic Pipelines**

  * UI-based
  * Easier for beginners

---

### 🔹 Agents

* Machines that run pipeline jobs
* Types:

  * Microsoft-hosted
  * Self-hosted

---

### 🔹 Key Concepts

* **Stages → Jobs → Steps hierarchy**
* **Artifacts**: Output of builds
* **Triggers**: Push, PR, schedule
* **Service Connections**: Secure external access (Azure, AWS)

---

# 2. Key Terminology & Definitions

* **Pipeline**: Automated workflow for build/deploy
* **Agent Pool**: Collection of agents
* **Artifact**: Build output stored for deployment
* **Release Pipeline**: Deployment workflow
* **YAML**: Declarative config language
* **PR (Pull Request)**: Code review mechanism
* **Branch Policy**: Rules for merging code
* **Service Principal**: Identity for automation
* **Task**: Predefined pipeline step

---

# 3. Tools, Frameworks, and Technologies

| Tool              | Use Case                | Pros                | Cons                        |
| ----------------- | ----------------------- | ------------------- | --------------------------- |
| Azure Pipelines   | CI/CD                   | Multi-cloud support | YAML learning curve         |
| Git (Azure Repos) | Version control         | Integrated          | Limited vs GitHub ecosystem |
| Terraform         | IaC                     | Multi-cloud         | Complex debugging           |
| ARM Templates     | Azure infra             | Native              | Verbose                     |
| Docker            | Containerization        | Portable            | Requires orchestration      |
| Kubernetes (AKS)  | Container orchestration | Scalable            | Complex setup               |

---

# 4. Comprehensive Interview Questions & Answers

---

## 🔹 Q1: Explain CI/CD pipeline in Azure DevOps

**Answer:**

* Code pushed → triggers pipeline
* Build stage compiles code
* Test stage validates
* Artifact published
* Deployment to environments

**Mid-level depth:**

* Mentions YAML pipelines, environments, approvals

**Senior-level:**

* Talks about rollback, canary deployments, infra as code

---

## 🔹 Q2: YAML vs Classic pipelines?

**Answer:**

* YAML:

  * Version-controlled
  * Reusable templates
* Classic:

  * GUI-based
  * Easier initial setup

**Trade-off:**

* YAML preferred for scalability

---

## 🔹 Q3: What are stages, jobs, steps?

* **Stage**: Logical grouping (Build/Deploy)
* **Job**: Runs on agent
* **Step**: Individual task

---

## 🔹 Q4: How do you secure pipelines?

**Answer:**

* Use:

  * Service connections
  * Key Vault integration
  * Secret variables
  * RBAC

---

## 🔹 Q5: How do you handle failures?

**Answer:**

* Retry logic
* Logs analysis
* Alerts
* Rollback strategy

---

## 🔹 Q6: Branching strategy?

* Git Flow / Trunk-based
* Use PR validation
* Enforce policies

---

## 🔹 Q7: Scenario — Pipeline failing randomly

**Answer:**

* Check:

  * Agent issues
  * Dependency version
  * Timeout
  * Network issues

---

## 🔹 Q8: How to deploy to multiple environments?

**Answer:**

* Use stages (Dev → QA → Prod)
* Add approvals
* Use environment variables

---

## 🔹 Q9: How to reuse pipeline code?

**Answer:**

* YAML templates
* Variable groups

---

## 🔹 Q10: What is artifact usage?

* Store build output
* Used in deployment stages

---

# 5. Code Snippets / YAML Example

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Build
  jobs:
  - job: BuildJob
    steps:
    - script: echo "Building..."
    
- stage: Deploy
  dependsOn: Build
  jobs:
  - job: DeployJob
    steps:
    - script: echo "Deploying..."
```

### ✅ Best Practices:

* Use templates
* Avoid hardcoding values
* Use variables

---

# 6. Practical & Real-World Scenarios

### 🔹 Scenario: ETL Pipeline Deployment

* Code in repo
* Pipeline builds data pipeline scripts
* Deploys to Azure (Data Factory / Databricks)
* Uses environment configs

---

### 🔹 Scenario: Microservices Deployment

* Docker build
* Push to ACR
* Deploy to AKS

---

# 7. Performance, Optimization & Edge Cases

* Parallel jobs for faster execution
* Cache dependencies
* Avoid unnecessary builds
* Edge case:

  * Large artifacts slowing pipeline
  * Agent timeout

---

# 8. Debugging & Troubleshooting

### Steps:

1. Check logs
2. Validate YAML
3. Re-run failed stage
4. Check agent health
5. Verify secrets/config

---

# 9. Security / Reliability Considerations

* Use Key Vault
* Avoid plain text secrets
* RBAC roles
* Audit logs
* Environment approvals

---

# 10. Follow-Up & Probing Questions

* Why YAML over classic?
* How would you scale pipelines?
* What if deployment fails in production?
* How do you manage secrets securely?

---

# 11. Common Mistakes & Red Flags

❌ Hardcoding credentials
❌ No rollback strategy
❌ Ignoring logs
❌ Not using branch policies
❌ Manual deployments in CI/CD setup

---

# 12. Behavioral / Soft Skills Signals

* Ownership of failures
* Clear debugging explanation
* Collaboration with QA/Dev teams
* Proactive monitoring

---

# 13. Evaluation Guide

| Level     | Expectation                        |
| --------- | ---------------------------------- |
| Junior    | Basic CI/CD understanding          |
| Mid-Level | Hands-on pipelines, debugging      |
| Senior    | Architecture, scaling, reliability |

---

# 14. Flashcards / Quick Revision

* CI = Build + Test
* CD = Deploy
* YAML = Pipeline as code
* Agent = Executes pipeline
* Artifact = Build output
* Service Connection = Secure access

---

# 15. References / Further Reading

* Microsoft Docs: Azure DevOps
* Azure Pipelines Documentation
* Git branching strategies
* Terraform & ARM guides

---

If you want, I can also:

* Create **real interview mock Q&A based on your experience**
* Or **map Azure DevOps to your current ETL/Data Engineer role** (very useful for your switch)
