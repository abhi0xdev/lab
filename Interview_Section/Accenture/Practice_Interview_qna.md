What is a CI/CD pipeline and how does it work in a real DevOps setup?
A CI/CD pipeline is an automated workflow that builds, tests, and deploys code changes from version control to production environments. It works by triggering jobs on events like commits, running stages such as build, test, security scans, and deployment in sequence or parallel. It is used to reduce manual effort, improve consistency, and enable faster delivery. A key trade-off is increased pipeline complexity which requires maintenance and debugging effort. In real systems, failures in intermediate stages like flaky tests can block deployments and require proper retry or isolation strategies.

How do you design a Jenkins pipeline for multi-environment deployments?
A Jenkins pipeline for multiple environments uses stages and conditional logic to deploy code progressively from dev to staging to production. It works by parameterizing environment variables and using approval gates before sensitive deployments. It is used to ensure controlled releases and environment-specific configurations. A limitation is that complex pipelines become harder to read and maintain over time. In practice, handling secrets and environment drift between environments requires strict configuration management.

What is the difference between declarative and scripted Jenkins pipelines?
Declarative pipelines use a structured syntax with predefined blocks, while scripted pipelines provide full Groovy flexibility for custom logic. Declarative pipelines are easier to maintain and enforce standards, while scripted pipelines allow complex workflows. They are used depending on the level of control and complexity required. A trade-off is that scripted pipelines can become hard to debug and less readable. In real use, mixing both styles carefully helps handle edge cases without losing maintainability.

How do you implement approval gates in Jenkins pipelines?
Approval gates are implemented using input steps or integrations with external systems like Jira for manual approvals. They work by pausing pipeline execution until a user or system approves the next stage. They are used to enforce compliance and prevent unauthorized production deployments. A limitation is that manual approvals can slow down delivery and create bottlenecks. In practice, timeouts and fallback mechanisms must be added to avoid stuck pipelines.

What are Git branching strategies and how do they affect CI/CD?
Git branching strategies define how code changes are managed and merged, such as Git Flow or trunk-based development. They work by organizing feature development, releases, and hotfixes into structured branches. They are used to align development workflows with CI/CD processes and deployment frequency. A trade-off is that complex strategies like Git Flow can slow down releases. In real scenarios, merge conflicts and long-lived branches can break pipelines and require disciplined usage.

How does trunk-based development improve CI/CD performance?
Trunk-based development involves frequent commits to a single main branch with short-lived feature branches. It works by encouraging continuous integration and reducing merge conflicts. It is used to speed up delivery and simplify pipeline triggers. A limitation is that it requires strong test automation to prevent unstable code from reaching main. In practice, feature flags are often used to control incomplete features safely.

How do you integrate SonarQube into a CI/CD pipeline?
SonarQube is integrated as a pipeline stage where code is scanned for quality issues and vulnerabilities. It works by analyzing code during the build phase and generating reports with quality gates. It is used to enforce coding standards and detect bugs early. A trade-off is increased pipeline execution time. In real-world usage, failing quality gates must be tuned properly to avoid blocking development unnecessarily.

What is the role of OWASP ZAP in DevSecOps pipelines?
OWASP ZAP is a dynamic application security testing tool used to scan running applications for vulnerabilities. It works by simulating attacks like SQL injection or XSS during pipeline execution. It is used to identify runtime security issues before deployment. A limitation is that it may produce false positives requiring manual validation. In practice, scan duration and environment setup must be optimized to avoid slowing pipelines.

How does Ansible help in DevOps automation?
Ansible is a configuration management and automation tool that uses playbooks to define desired system states. It works by executing tasks over SSH without requiring agents on target systems. It is used for provisioning, deployment, and configuration consistency. A trade-off is that large playbooks can become complex and harder to debug. In real scenarios, idempotency must be maintained to avoid unintended repeated changes.

What is idempotency in Ansible and why is it important?
Idempotency means running the same playbook multiple times produces the same result without side effects. It works by checking the current state before applying changes. It is used to ensure predictable and safe automation. A limitation is that writing idempotent tasks requires careful module selection and design. In practice, improper idempotency can lead to configuration drift or repeated failures.

How do you manage secrets in CI/CD pipelines?
Secrets are managed using secure storage systems like Jenkins credentials, Vault, or Kubernetes secrets. They work by injecting sensitive data at runtime without exposing it in code. They are used to protect credentials like API keys and passwords. A trade-off is added complexity in managing access and rotation. In real-world systems, misconfigured permissions can lead to secret leaks.

What is Helm and how is it used in Kubernetes deployments?
Helm is a package manager for Kubernetes that uses charts to define application deployments. It works by templating Kubernetes manifests and managing releases with versioning. It is used to simplify and standardize deployments across environments. A limitation is that debugging Helm templates can be complex. In practice, improper value overrides can cause deployment inconsistencies.

How does GitOps work with Kubernetes and Helm?
GitOps uses Git as the single source of truth for Kubernetes configurations and deployments. It works by automatically syncing cluster state with repository changes using tools like ArgoCD. It is used to improve traceability and rollback capabilities. A trade-off is dependency on Git workflows for all changes. In real scenarios, unauthorized direct cluster changes can cause drift and must be restricted.

What is RBAC in Kubernetes and why is it critical?
RBAC controls access to Kubernetes resources based on roles and permissions. It works by defining roles and binding them to users or service accounts. It is used to enforce security and limit access to sensitive operations. A limitation is that misconfiguration can either over-restrict or over-permit access. In practice, auditing and least privilege principles must be applied carefully.

How do you debug a failed Jenkins pipeline?
Debugging involves analyzing console logs, checking stage outputs, and verifying environment configurations. It works by isolating the failing stage and identifying errors in scripts or integrations. It is used to quickly restore pipeline functionality. A trade-off is that debugging complex pipelines can be time-consuming. In real scenarios, adding proper logging and retries helps reduce debugging effort.

How do you implement rollback strategies in CI/CD pipelines?
Rollback strategies involve reverting to previous stable versions using artifacts or deployment history. They work by storing versioned builds and redeploying them when failures occur. They are used to minimize downtime and recover quickly from issues. A limitation is that database changes may not be easily reversible. In practice, backward compatibility must be ensured before deploying changes.

How do you integrate Jira with CI/CD pipelines?
Jira is integrated by linking commits, builds, and deployments to tickets using APIs or plugins. It works by updating ticket statuses automatically based on pipeline events. It is used to ensure traceability and visibility in the SDLC. A trade-off is dependency on consistent ticket usage by developers. In real-world cases, incorrect ticket tagging can break traceability.

How do you handle flaky tests in CI/CD pipelines?
Flaky tests are handled by identifying unstable tests and isolating or retrying them. It works by analyzing test patterns and applying retry mechanisms or quarantining tests. It is used to prevent false pipeline failures. A limitation is that retries can hide real issues. In practice, flaky tests should be fixed or removed rather than ignored.

How do you optimize CI/CD pipeline performance?
Pipeline performance is optimized by parallelizing stages, caching dependencies, and reducing unnecessary steps. It works by minimizing execution time while maintaining reliability. It is used to speed up feedback cycles for developers. A trade-off is increased pipeline complexity. In real scenarios, over-optimization can reduce visibility into failures.

How do you ensure security in DevSecOps pipelines?
Security is ensured by integrating static, dynamic, and dependency scanning tools into pipelines. It works by enforcing security checks at different stages before deployment. It is used to detect vulnerabilities early and maintain compliance. A limitation is increased pipeline execution time and potential false positives. In practice, balancing security checks with developer productivity is critical.

---

A deployment to production failed after a successful build and test stage. How would you investigate and fix it?
I would start by checking Jenkins logs and deployment stage outputs to identify where the failure occurred. I would verify environment-specific variables, credentials, and connectivity to the target system like Kubernetes or OpenShift. This approach works because most deployment failures are due to misconfigurations or runtime issues rather than code defects. It is used to quickly isolate whether the issue is pipeline-related or infrastructure-related. A limitation is that logs may not always clearly indicate root cause, so deeper debugging may be needed. In real scenarios, having structured logging and alerts helps reduce investigation time.

Write a Jenkins declarative pipeline that builds a Maven project, runs tests, and deploys only after manual approval.
In this pipeline, I define stages for build, test, and deploy, and use an input step before deployment to enforce approval. It works by pausing execution until a user approves the deployment. It is used to ensure controlled production releases. A limitation is that manual approval slows down automation. In practice, timeouts should be added to avoid stuck pipelines.

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean install'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Approval') {
            steps {
                input message: 'Approve deployment?'
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo Deploying to production'
            }
        }
    }
}
```

Your pipeline is slow and takes 30 minutes to complete. What would you do?
I would analyze pipeline stages to identify bottlenecks such as long-running tests or redundant steps. I would introduce parallel execution for independent stages and enable caching for dependencies like Maven or npm. This works by reducing overall execution time through concurrency and reuse. It is used to improve developer feedback speed. A limitation is that parallel stages can make debugging harder. In real scenarios, over-parallelization can overload build agents or infrastructure.

Write an Ansible playbook to install and start Nginx on a server.
This playbook defines tasks to install Nginx and ensure the service is running. It works by using idempotent modules that check system state before applying changes. It is used for consistent and repeatable configuration management. A limitation is that errors in playbooks can affect multiple servers at once. In practice, testing in staging environments is critical.

```yaml
- name: Install and start Nginx
  hosts: webservers
  become: yes
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

A security scan fails in the pipeline due to vulnerabilities. What is your approach?
I would first review the scan report to understand the severity and type of vulnerabilities. I would prioritize fixing critical and high issues and decide whether to block the pipeline based on policy. This works by enforcing security gates in CI/CD. It is used to prevent vulnerable code from reaching production. A limitation is that false positives can delay deployments. In real scenarios, teams often whitelist acceptable risks temporarily with proper documentation.

Write a shell script to check if a service is running and restart it if not.
This script checks the service status and restarts it if it is not active. It works by using system commands like systemctl to query and control services. It is used for basic operational automation. A limitation is that it may not handle all failure scenarios like partial crashes. In real usage, logging and alerting should be added.

```bash
#!/bin/bash
SERVICE="nginx"

if systemctl is-active --quiet $SERVICE; then
  echo "$SERVICE is running"
else
  echo "$SERVICE is not running. Restarting..."
  systemctl restart $SERVICE
fi
```

Your Kubernetes deployment is failing with CrashLoopBackOff. How do you debug it?
I would check pod logs and describe the pod to identify errors like misconfiguration or missing dependencies. I would verify environment variables, secrets, and resource limits. This works by identifying runtime issues inside containers. It is used to quickly diagnose deployment failures. A limitation is that logs may not always capture root cause. In real scenarios, readiness and liveness probes misconfiguration often causes repeated restarts.

Write a Kubernetes deployment YAML for a simple Nginx app with 2 replicas.
This YAML defines a deployment with two replicas of an Nginx container. It works by ensuring the desired number of pods are running and managed by Kubernetes. It is used for scalable and self-healing deployments. A limitation is that misconfigured resources can lead to instability. In real usage, resource limits should always be defined.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

A developer pushed code but the pipeline did not trigger. What could be the issue?
I would check webhook configurations between Git and Jenkins to ensure triggers are properly set. I would also verify branch filters and pipeline triggers in Jenkins configuration. This works by ensuring events are correctly propagated to CI/CD systems. It is used to maintain automation reliability. A limitation is that silent failures can occur without alerts. In real scenarios, webhook delivery logs are essential for debugging.

Write a Python script to parse a log file and count error occurrences.
This script reads a log file and counts lines containing the word ERROR. It works by iterating through file lines and applying string matching. It is used for basic log analysis and monitoring. A limitation is that it does not handle structured logs. In real scenarios, tools like ELK are preferred for scalability.

```python
error_count = 0

with open("app.log", "r") as file:
    for line in file:
        if "ERROR" in line:
            error_count += 1

print(f"Total errors: {error_count}")
```

Your OpenShift deployment works in dev but fails in prod. What would you check?
I would compare configurations between environments including secrets, routes, and resource quotas. I would verify RBAC permissions and network policies in production. This works by identifying environment-specific differences. It is used to ensure consistent deployments across environments. A limitation is that differences may not be documented. In real scenarios, configuration drift is a common cause of such failures.

Write a Helm values.yaml snippet for configuring image and replica count.
This snippet defines configurable parameters for image and replicas in a Helm chart. It works by allowing dynamic overrides during deployment. It is used to maintain flexibility across environments. A limitation is that incorrect values can break deployments. In real usage, validation of values is important.

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent
```

A rollback failed and made things worse. What went wrong?
The rollback may have restored application code but not database or external dependencies. It works incorrectly when stateful changes are not reversible. It is used to recover from failures but requires proper design. A limitation is that not all systems support safe rollback. In real scenarios, backward compatibility and database migration strategies must be planned.

Write a Git command sequence to create a feature branch, commit, and push it.
This sequence creates a branch, adds changes, commits, and pushes to remote. It works by isolating feature development from the main branch. It is used to support collaborative workflows. A limitation is that improper branching can cause conflicts. In real scenarios, consistent naming and commit linking to Jira tickets is important.

```bash
git checkout -b feature/new-feature
git add .
git commit -m "Add new feature"
git push origin feature/new-feature
```

A pipeline is passing but production has bugs. What is the issue?
The issue is likely insufficient test coverage or missing real-world scenarios in tests. It works as a gap between CI validation and production behavior. It is used to highlight weaknesses in testing strategy. A limitation is that not all edge cases can be tested. In real scenarios, adding integration and performance tests improves reliability.
