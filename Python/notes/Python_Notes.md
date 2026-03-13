# Python for DevOps and Platform Engineering — Mid-Level Interview Notes

---

# 1. Core Concepts (Mid-Level Focus)

## Automation with Python

**Definition:** Using Python scripts to automate operational tasks such as deployments, monitoring, configuration, and infrastructure management.

**How it works**

* Python interacts with system tools, APIs, and infrastructure services.
* Scripts replace repetitive manual tasks.

**Example**

* Deploying applications
* Creating cloud resources
* Parsing logs
* Health checks

**Why DevOps engineers use it**

* Rapid scripting
* Strong ecosystem
* Easy integration with APIs and CLI tools.

**Constraints**

* Scripts must handle failures and retries.
* Logging and error handling must be robust.
* Idempotency is critical for infrastructure automation.

---

## Infrastructure as Code (IaC) Integration

Python is often used to automate infrastructure tools.

Common interactions:

Python → Terraform CLI
Python → AWS SDK (Boto3)
Python → Kubernetes API
Python → Configuration management tools

Example workflow:

Python Script
→ Trigger Terraform
→ Validate Infrastructure
→ Deploy Application

---

## API Interaction

DevOps platforms expose APIs.

Examples:

* Kubernetes API
* GitHub API
* AWS API
* Monitoring APIs

Python libraries like `requests` allow automation.

Example tasks:

* Trigger CI pipelines
* Retrieve deployment status
* Rotate credentials
* Manage secrets

---

## System Automation

Python interacts with OS utilities.

Typical operations:

* File management
* Process monitoring
* Log parsing
* Cron job automation

Modules used:

* os
* subprocess
* shutil
* pathlib

---

## Observability Automation

Python is used for:

* Metrics collection
* Log parsing
* Alert automation
* Health monitoring

Example:

* Query Prometheus
* Send alerts to Slack
* Auto-scale resources.

---

## Idempotency in Automation

**Definition:** Running a script multiple times produces the same result.

Important in:

* Deployment automation
* Infrastructure management

Example:
Script checks if resource exists before creating it.

---

## Configuration Management

Python scripts manage configuration files:

* YAML
* JSON
* TOML
* ENV files

Example:
Update Kubernetes manifests automatically.

---

## Parallel Execution

Python can run multiple tasks simultaneously.

Used for:

* Deploying across servers
* Running checks across clusters

Methods:

* threading
* multiprocessing
* async programming

---

# 2. Key Terminology & Definitions

**SDK**
Software Development Kit used to interact with services (example: AWS Boto3).

**CLI Automation**
Executing command line tools through Python scripts.

**Idempotency**
Running the same operation multiple times without changing the result.

**Infrastructure as Code (IaC)**
Managing infrastructure using version-controlled code.

**Webhook**
HTTP callback triggered by events.

**CI/CD Pipeline**
Automated process for build, test, and deployment.

**Immutable Infrastructure**
Infrastructure that is replaced instead of modified.

**Artifact**
Compiled output of a build (container image, binary).

**Observability**
Understanding system state through metrics, logs, and traces.

**Configuration Drift**
When system configuration deviates from expected state.

---

# 3. Tools, Frameworks, and Technologies

## Python Libraries

### Boto3

AWS SDK for Python.

Use cases:

* Manage EC2
* Deploy Lambda
* Create S3 buckets

Pros:

* Full AWS integration

Cons:

* Verbose API calls

---

### Requests

HTTP client library.

Used for:

* API automation
* Webhooks
* Service monitoring.

---

### PyYAML

Used to manipulate YAML files (Kubernetes manifests).

---

### Click / Argparse

CLI development tools.

Used for building DevOps utilities.

---

### Fabric

SSH automation tool.

Use case:

* Run commands across servers.

---

### Paramiko

SSH library used for remote execution.

---

### Docker SDK for Python

Automates Docker operations.

Example:

* Build containers
* Push images
* Manage containers.

---

### Kubernetes Python Client

Automates Kubernetes cluster operations.

Example:

* Deploy pods
* Monitor services
* Scale deployments.

---

# 4. Comprehensive Interview Questions & Answers

---

## Q1 What are common use cases of Python in DevOps?

**Answer**

Python is widely used for:

Automation
Infrastructure management
Monitoring
Deployment pipelines

Examples include:

* Automating infrastructure provisioning using Boto3
* Writing health-check scripts for services
* Parsing logs and generating alerts
* Automating CI/CD tasks
* Managing Kubernetes resources.

**Junior answer**
Focuses only on scripting.

**Mid-level answer**
Explains integration with APIs, IaC tools, and CI/CD pipelines.

**Senior answer**
Discusses reliability, idempotency, scaling, and observability.

---

## Q2 How do you interact with shell commands in Python?

**Answer**

Use the `subprocess` module.

Example:

```python
import subprocess

result = subprocess.run(["kubectl", "get", "pods"], capture_output=True, text=True)

print(result.stdout)
```

Best practice:

* Avoid `shell=True`
* Handle return codes
* Capture errors.

---

## Q3 How would you automate EC2 instance creation?

**Answer**

Using Boto3.

Steps:

1. Authenticate using IAM
2. Create EC2 instance
3. Attach security groups
4. Monitor instance status.

Example:

```python
import boto3

ec2 = boto3.resource('ec2')

instance = ec2.create_instances(
    ImageId='ami-123',
    InstanceType='t2.micro',
    MinCount=1,
    MaxCount=1
)
```

---

## Q4 How do you implement retries in Python automation?

Use retry mechanisms.

Example:

```python
import time

for i in range(3):
    try:
        deploy()
        break
    except Exception:
        time.sleep(5)
```

Better approach:
Use `tenacity` library.

---

## Q5 How would you parse logs using Python?

Use regex or string parsing.

Example:

```python
import re

pattern = r"ERROR"

with open("app.log") as f:
    for line in f:
        if re.search(pattern, line):
            print(line)
```

---

## Q6 How would you build a CLI tool for DevOps tasks?

Use `argparse`.

Example:

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--env")
args = parser.parse_args()

print(f"Deploying to {args.env}")
```

---

# 5. Code Snippets / Best Practices

## Environment Variables

```python
import os

token = os.getenv("API_TOKEN")
```

Avoid hardcoding secrets.

---

## Logging

```python
import logging

logging.basicConfig(level=logging.INFO)

logging.info("Deployment started")
```

---

## API Request

```python
import requests

response = requests.get("https://api.github.com/repos")
print(response.json())
```

---

# 6. Practical & Real-World Scenarios

## Scenario: Automated Kubernetes Deployment

Pipeline:

Code Push
↓
CI Pipeline
↓
Docker Build
↓
Push to Registry
↓
Python script triggers deployment.

Script tasks:

* Update image tag
* Apply manifests
* Verify rollout.

---

## Scenario: Log Monitoring

Python script:

* Reads log files
* Detects error patterns
* Sends Slack alerts.

---

## Scenario: Auto-Scaling Monitor

Python script:

* Reads CPU metrics from Prometheus
* Triggers scaling via Kubernetes API.

---

# 7. Performance, Optimization & Edge Cases

Considerations:

Large log files
→ Use streaming instead of loading into memory.

API rate limits
→ Implement retries and backoff.

Network failures
→ Use timeouts.

Parallel operations
→ Use multiprocessing or async.

---

# 8. Debugging & Troubleshooting

Common issues:

Script failing in production.

Troubleshooting steps:

Check logs
Validate environment variables
Verify API authentication
Check network connectivity
Test CLI commands manually.

---

# 9. Security & Reliability Considerations

Never store secrets in code.

Use:

* Vault
* AWS Secrets Manager
* Kubernetes secrets

Implement:

Retry logic
Timeouts
Audit logs.

---

# 10. Follow-Up & Probing Questions

Why did you use Python instead of Bash?

How would you scale this script to thousands of servers?

How would you handle API rate limiting?

What would you do if deployment partially fails?

---

# 11. Common Mistakes & Red Flags

Hardcoding credentials.

No error handling.

Using `shell=True` blindly.

No logging.

No retry logic.

No timeout handling.

---

# 12. Behavioral / Soft Skills Signals

Good signals:

Explains trade-offs.

Mentions monitoring and observability.

Discusses failure handling.

Ownership of automation pipelines.

---

# 13. Evaluation Guide

Junior:

Basic scripting knowledge.

Mid-Level:

API integrations
Automation design
Error handling
Idempotent workflows

Senior:

System architecture
Scalable automation platforms
Reliability engineering.

---

# 14. Flashcards (Quick Revision)

What library interacts with AWS?
Boto3.

Which Python module runs shell commands?
subprocess.

Which library makes HTTP requests?
requests.

How do you access environment variables?
os.getenv()

How do you build CLI tools?
argparse or click.

---