## 1. Core Concepts (Mid-Level Focus)

### Infrastructure as Code (IaC)

* Managing infrastructure using declarative code instead of manual processes
* Ensures **consistency, repeatability, and version control**
* Terraform uses **declarative syntax** → you define *desired state*, not steps

---

### Terraform Workflow

* **Write → Plan → Apply → Destroy**

  * `terraform init`: Initialize providers & modules
  * `terraform plan`: Preview changes
  * `terraform apply`: Execute changes
  * `terraform destroy`: Tear down infrastructure

**Why it matters:** Prevents unintended changes and enables safe deployments

---

### Providers

* Plugins that allow Terraform to interact with APIs
* Examples: AWS, Azure, GCP, Kubernetes

```hcl
provider "aws" {
  region = "ap-south-1"
}
```

---

### Resources

* The smallest unit of infrastructure (e.g., EC2, S3)

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123"
  instance_type = "t2.micro"
}
```

---

### State File (`terraform.tfstate`)

* Tracks current infrastructure state
* Maps **real-world resources ↔ Terraform config**

**Key point:** Without state, Terraform can’t determine changes

---

### Desired vs Current State

* Terraform compares:

  * Config (desired)
  * State file (current)
* Generates execution plan (diff)

---

### Variables & Outputs

* **Variables:** Parameterize configs
* **Outputs:** Export values (e.g., IPs)

---

### Modules

* Reusable Terraform configurations
* Helps in scaling and standardization

---

### Dependency Graph

* Terraform builds a graph to determine execution order
* Uses implicit + explicit (`depends_on`) dependencies

---

## 2. Key Terminology & Definitions

* **Plan:** Execution preview of infrastructure changes
* **Apply:** Execution of changes
* **Destroy:** Deletes infrastructure
* **State:** Snapshot of infrastructure
* **Drift:** When real infra ≠ Terraform state
* **Provisioner:** Executes scripts on resources (last resort)
* **Backend:** Where state is stored (local/remote)
* **Workspace:** Separate environments (dev/staging/prod)

---

## 3. Tools, Frameworks, and Technologies

### Terraform CLI

* Main tool to manage infrastructure

### Terraform Cloud / Enterprise

* Remote execution, state management, collaboration

### Remote Backends

* AWS S3 + DynamoDB (state locking)
* Azure Blob Storage
* GCS

**Why remote state?**

* Team collaboration
* Avoid state conflicts

---

### Terragrunt

* Wrapper for Terraform
* Reduces duplication and manages environments

---

### CI/CD Integration

* GitHub Actions, Jenkins, GitLab CI
* Automate plan/apply

---

## 4. Comprehensive Interview Questions & Answers

### Q1: How does Terraform work internally?

**Answer:**

* Parses HCL config
* Loads provider plugins
* Builds dependency graph
* Compares desired vs current state
* Generates execution plan
* Applies changes in correct order

**Mid-level insight:**
Understands graph-based execution and state comparison

**Senior insight:**
Explains provider SDK, refresh phase, and parallelism

---

### Q2: What is Terraform state and why is it important?

**Answer:**

* Stores metadata about resources
* Tracks IDs, dependencies, attributes

**Why important:**

* Enables diff calculation
* Avoids recreating resources

**Risk:**

* Sensitive data exposure → must secure

---

### Q3: Local vs Remote State?

| Aspect        | Local      | Remote           |
| ------------- | ---------- | ---------------- |
| Storage       | Local file | Cloud (S3, etc.) |
| Collaboration | Poor       | Good             |
| Locking       | No         | Yes              |

**Mid-level answer:** Use remote state in teams
**Senior answer:** Also discuss locking & encryption

---

### Q4: What is state locking?

**Answer:**

* Prevents concurrent modifications
* Example: DynamoDB lock for S3 backend

---

### Q5: What is drift and how do you handle it?

**Answer:**

* Drift = Infra changed outside Terraform

**Handling:**

* `terraform plan`
* `terraform refresh`
* Import resources if needed

---

### Q6: When would you use modules?

**Answer:**

* Reusability
* Standardization

Example: VPC module reused across environments

---

### Q7: Implicit vs Explicit dependencies?

**Implicit:**

* Terraform auto-detects via references

**Explicit:**

```hcl
depends_on = [aws_instance.db]
```

---

### Q8: What are provisioners and why avoid them?

**Answer:**

* Run scripts on resources
* Not idempotent → breaks declarative model

---

### Q9: How do you manage multiple environments?

**Answer:**

* Workspaces
* Separate state files
* Folder structure + modules

---

### Q10: How do you handle secrets?

**Answer:**

* Use:

  * Vault
  * AWS Secrets Manager
  * Environment variables

**Never store secrets in state**

---

## 5. Code Snippets / Pseudocode

### Variable Example

```hcl
variable "instance_type" {
  default = "t2.micro"
}
```

---

### Output Example

```hcl
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

---

### Module Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
}
```

---

## 6. Practical & Real-World Scenarios

### Scenario: Deploying scalable web app

* VPC → Subnets → EC2 → Load Balancer
* Use modules for each component
* Store state in S3

---

### Scenario: CI/CD Pipeline

* Code commit triggers:

  * `terraform plan`
  * Manual approval
  * `terraform apply`

---

## 7. Performance, Optimization & Edge Cases

* Parallel resource creation (default ~10 threads)
* Large state files → slow operations
* Use:

  * `-target` cautiously
  * Split configs into modules

**Edge Cases:**

* Partial failures
* State corruption
* API rate limits

---

## 8. Debugging & Troubleshooting

### Common Issues:

* State mismatch
* Provider errors
* Dependency cycles

### Steps:

1. Run `terraform plan`
2. Check logs (`TF_LOG=DEBUG`)
3. Validate config
4. Inspect state

---

## 9. Security / Reliability Considerations

* Encrypt state (S3 + KMS)
* Enable versioning
* Restrict access (IAM policies)
* Avoid hardcoding secrets

---

## 10. Follow-Up & Probing Questions

* Why Terraform over CloudFormation?
* What happens if state file is lost?
* How do you handle rollback?
* How would you design Terraform for multi-region?

---

## 11. Common Mistakes & Red Flags

* Using local state in teams
* Hardcoding values
* Overusing provisioners
* Not understanding state

---

## 12. Behavioral / Soft Skills Signals

* Explains trade-offs
* Mentions collaboration issues
* Talks about real incidents

---

## 13. Evaluation Guide

| Level  | Signal                     |
| ------ | -------------------------- |
| Junior | Knows commands             |
| Mid    | Understands state, modules |
| Senior | Designs scalable infra     |

---

## 14. Flashcards / Quick Revision

* **What is Terraform?** IaC tool
* **State file?** Infra snapshot
* **Drift?** Config ≠ reality
* **Module?** Reusable code
* **Backend?** State storage

---




