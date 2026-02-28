Introduce yourself.
I am a DevOps Engineer with around 3.5 years of experience working on CI/CD pipelines, Kubernetes, and production support. I build and maintain CI/CD pipelines using GitHub Actions with a multi-branch strategy, handling end-to-end automation from build to deployment. In my daily work, I troubleshoot pipeline failures and resolve Kubernetes issues like pod crashes, scaling problems, and deployment failures. This helps ensure stable releases and reliable systems. One limitation is handling unexpected production issues under time pressure, so I rely on monitoring tools like Prometheus and Grafana for faster debugging.

What is the difference between ReplicaSet and ReplicationController?
A ReplicationController ensures a specified number of pod replicas are running at any time, while a ReplicaSet is its newer version with more advanced selector support. ReplicaSet works using set-based selectors, allowing more flexible label matching compared to ReplicationController which only supports equality-based selectors. Both are used to maintain desired pod count and provide self-healing in Kubernetes. A limitation is that they do not handle rolling updates directly, which is why Deployments are preferred. In real scenarios, ReplicaSet is mostly managed by Deployments and rarely used directly.

What is the difference between Deployment and DeploymentConfig?
A Deployment is a native Kubernetes resource used to manage stateless applications with features like rolling updates and rollback. DeploymentConfig is specific to OpenShift and provides additional features like triggers based on image changes or config changes. Both work by managing ReplicaSets to maintain desired pod state. They are used to automate application lifecycle management. A limitation is that DeploymentConfig is not portable outside OpenShift. In real scenarios, Deployment is preferred for standard Kubernetes environments.

What is the difference between ENTRYPOINT and CMD in Docker?
ENTRYPOINT defines the main command that always runs when a container starts, while CMD provides default arguments to that command. They work together where CMD can be overridden but ENTRYPOINT usually remains fixed. They are used to control container execution behavior. A limitation is confusion when both are used incorrectly. In real scenarios, ENTRYPOINT is used for fixed commands and CMD for flexible arguments.

How did you set up dashboards for Kubernetes microservices and which metrics do you show?
I set up Prometheus to scrape metrics from Kubernetes and applications, and used Grafana to create dashboards. I monitor metrics like CPU, memory, pod restarts, request latency, and error rates. This works by collecting and visualizing real-time system data. It is used to improve observability and detect issues early. A limitation is high metric volume. In real scenarios, focusing on critical metrics reduces noise.

Tell me the biggest incident you faced and how you handled it.
We faced an issue where multiple pods were crashing due to incorrect environment configuration, causing service downtime. I analyzed logs, identified the misconfiguration, and rolled back to a stable version. This works by quickly restoring service while fixing root cause. It is used to minimize downtime. A limitation is dependency on rollback readiness. In real scenarios, proper monitoring helps detect such issues early.

How do you troubleshoot Kubernetes pods?
I start by checking pod status using kubectl get and describe commands, then review logs using kubectl logs. I verify events, resource limits, and configuration like environment variables. This works by isolating issues step by step. It is used to diagnose failures quickly. A limitation is multiple possible root causes. In real scenarios, logs and events are the primary debugging tools.

How do you validate security and which tools do you use in CI/CD?
I use tools like SonarQube for code quality and vulnerability scanning, and integrate them into CI/CD pipelines. I also use container scanning tools to check images before deployment. This works by enforcing security checks at different stages. It is used to prevent vulnerabilities from reaching production. A limitation is false positives. In real scenarios, tuning rules is important.

Explain the CI/CD process you are using with GitHub Actions.
I use GitHub Actions to trigger pipelines on code commits with multi-branch strategy. The pipeline includes build, test, security scan, and deployment stages. This works by automating the entire delivery process. It is used to ensure consistent and fast releases. A limitation is debugging complex workflows. In real scenarios, reusable workflows improve maintainability.

How do you get a specific commit ID like 9tty464?
I use git log or git show commands to search for the commit ID in the repository. I can also use git checkout with the commit ID to access that version. This works by referencing Git history. It is used for debugging or rollback. A limitation is partial IDs may not be unique. In real scenarios, full commit hashes are safer.

If we want to clone remote repositories how do we do it?
I use git clone followed by the repository URL to copy it locally. It works by downloading all files and history from the remote repository. It is used for development and collaboration. A limitation is large repositories take time. In real scenarios, shallow clone can reduce size.

How do you configure SonarQube in Jenkins?
I configure SonarQube server in Jenkins global settings and add credentials. I use SonarQube scanner in pipeline stages to analyze code. This works by sending code data to SonarQube for analysis. It is used to enforce quality gates. A limitation is added pipeline time. In real scenarios, failing builds on quality gate ensures compliance.

How do you configure security scan tools in GitHub Actions?
I add steps in workflow YAML to run security tools like SonarQube or container scanners. I use secrets for authentication and define thresholds for failure. This works by integrating security into CI/CD. It is used for DevSecOps practices. A limitation is increased pipeline duration. In real scenarios, scanning should be optimized.

What is AppDynamics?
AppDynamics is an application performance monitoring tool that tracks application health and performance. It works by collecting metrics and traces from applications and infrastructure. It is used to identify bottlenecks and improve performance. A limitation is setup complexity. In real scenarios, it helps in root cause analysis during incidents.

How did you perform unit testing in your CI/CD pipeline?
I implemented unit testing as part of the CI stage in GitHub Actions, where tests are triggered automatically on every commit or pull request. The process starts with installing dependencies, then running unit test commands based on the tech stack, and finally validating results before moving to the next stage. This works by catching issues early before deployment. It is used to ensure code quality and prevent broken builds. A limitation is that unit tests do not cover integration or real environment issues. In real scenarios, failing tests immediately stop the pipeline to avoid bad deployments.

Can you explain the step-by-step process with commands?
First, I install dependencies using the appropriate package manager like npm install or pip install -r requirements.txt. Then I run unit tests using commands like npm test for Node.js, pytest for Python, or mvn test for Java projects. This works by executing predefined test cases written by developers. It is used to validate individual components of the application. A limitation is dependency on proper test coverage. In real scenarios, test reports are generated and stored for visibility.

Example GitHub Actions workflow for unit testing.
In GitHub Actions, I define a workflow YAML file that runs on push or pull request and executes test commands. It works by automating test execution in a clean environment. It is used to ensure consistent testing across all changes. A limitation is longer pipeline time. In real scenarios, caching dependencies improves speed.

```yaml id="u7k3lm"
name: Unit Tests

on:
  push:
    branches: [ "main", "dev" ]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm install

      - name: Run unit tests
        run: npm test
```

How do you handle test failures in pipeline?
If unit tests fail, the pipeline stops immediately and prevents further stages like deployment. I analyze logs to identify the failing test cases and work with developers to fix them. This works by enforcing quality gates in CI/CD. It is used to maintain code reliability. A limitation is false failures due to flaky tests. In real scenarios, retries or fixing flaky tests is necessary.
