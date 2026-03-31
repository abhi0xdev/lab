What is DevSecOps and how is it different from DevOps?
DevSecOps is the practice of integrating security into every stage of the software delivery lifecycle instead of treating it as a separate phase. It works by embedding security checks like code scanning, dependency checks, and runtime protections directly into CI/CD pipelines. It is used to detect vulnerabilities early and reduce the cost of fixing them later in production. A key trade-off is that adding security checks can slow down pipeline execution time. In real-world setups, teams often balance speed and security by running lightweight scans on every commit and deeper scans on scheduled builds.

How would you implement DevSecOps in GitHub Actions?
In GitHub Actions, DevSecOps is implemented by adding security jobs into the workflow YAML alongside build and test stages. It works by triggering tools like CodeQL for SAST, dependency scanning tools, and container scanning tools before deployment steps. It is used to automatically block insecure code from being merged or deployed. A limitation is that misconfigured workflows can either miss vulnerabilities or create too many false positives. In practice, teams use branch protection rules and required checks to enforce that security jobs must pass before merging.

How do you implement DevSecOps in Jenkins pipelines?
In Jenkins, DevSecOps is implemented by adding security stages in declarative or scripted pipelines using plugins and external tools. It works by integrating tools like SonarQube, OWASP Dependency Check, and container scanners into pipeline stages. It is used to enforce security gates before artifacts are published or deployed. A trade-off is that Jenkins requires more manual setup and maintenance compared to managed CI tools. In real-world cases, plugin compatibility and version mismatches can break pipelines, so version control of plugins is important.

What is SAST and how do you integrate it in CI/CD?
SAST is Static Application Security Testing that analyzes source code without executing it to find vulnerabilities. It works by scanning code for patterns like insecure functions, hardcoded secrets, or unsafe logic. It is used to catch issues early in the development phase before runtime. A limitation is that it can produce false positives and may not detect runtime issues. In practice, teams tune rules and integrate tools like SonarQube or CodeQL into CI pipelines to run on every pull request.

What is DAST and where does it fit in the pipeline?
DAST is Dynamic Application Security Testing that tests a running application for vulnerabilities from an external perspective. It works by sending requests to the application and analyzing responses for issues like injection or misconfigurations. It is used after deployment in staging environments to validate real-world behavior. A trade-off is that it requires a deployed environment and takes longer to execute. In real-world scenarios, DAST is often scheduled or run nightly instead of on every commit to avoid slowing down pipelines.

How do you secure secrets in GitHub Actions?
Secrets in GitHub Actions are stored in encrypted form using GitHub Secrets and accessed as environment variables during workflow execution. It works by injecting secrets only at runtime and masking them in logs. It is used to securely manage credentials like API keys, tokens, and passwords. A limitation is that secrets can still leak if printed accidentally in logs. In practice, teams use least privilege access and avoid echoing sensitive variables to logs.

How do you secure secrets in Jenkins?
Jenkins uses credentials management to securely store secrets like usernames, passwords, and tokens. It works by binding credentials to environment variables or files during pipeline execution. It is used to prevent hardcoding secrets in scripts or configuration files. A trade-off is that improper access control can expose credentials to unauthorized users. In real-world setups, role-based access control and credential rotation policies are enforced to minimize risk.

How do you integrate container security in DevSecOps?
Container security is integrated by scanning images for vulnerabilities during the build stage of the pipeline. It works by using tools like Trivy or Clair to analyze image layers and installed packages. It is used to ensure that deployed containers do not contain known vulnerabilities. A limitation is that scans can slow down builds and may require updating base images frequently. In practice, teams use minimal base images like Alpine and automate regular image rebuilds.

What is dependency scanning and why is it important?
Dependency scanning checks third-party libraries for known vulnerabilities using vulnerability databases. It works by comparing project dependencies against CVE databases. It is used because many attacks exploit outdated or vulnerable libraries. A trade-off is that frequent updates can break compatibility with existing code. In real-world projects, teams use tools like Dependabot or Snyk and test updates in staging before production rollout.

How do you enforce security gates in CI/CD?
Security gates are enforced by failing the pipeline if vulnerabilities exceed a defined threshold. It works by integrating tools like SonarQube quality gates or custom scripts to evaluate scan results. It is used to prevent insecure code from being deployed. A limitation is that strict gates can block development progress if not tuned properly. In practice, teams define severity-based thresholds and allow exceptions with proper approvals.

How do you handle false positives in security scans?
False positives are handled by tuning rules, suppressing known safe issues, and validating findings manually. It works by customizing tool configurations and maintaining ignore lists. It is used to reduce noise and improve developer trust in security tools. A trade-off is that suppressing issues may hide real vulnerabilities if not reviewed carefully. In real-world scenarios, teams periodically review suppressed findings to ensure they are still valid.

How do you secure Kubernetes in a DevSecOps pipeline?
Kubernetes security is enforced by scanning manifests, enforcing policies, and securing runtime configurations. It works by using tools like kube-bench, OPA Gatekeeper, or Kyverno to validate configurations. It is used to prevent misconfigurations like privileged containers or open network policies. A limitation is that strict policies can block deployments if not properly designed. In practice, teams test policies in staging and gradually enforce them in production.

What is shift-left security?
Shift-left security means moving security checks earlier in the development lifecycle. It works by integrating security tools in coding, commit, and build stages. It is used to reduce the cost and impact of fixing vulnerabilities later. A trade-off is increased complexity in early development workflows. In real-world environments, developers are trained to fix issues early to avoid pipeline failures later.

How do you scan Infrastructure as Code (IaC) for security?
IaC scanning checks Terraform, CloudFormation, or Kubernetes YAML files for misconfigurations. It works by analyzing code against security best practices and policies. It is used to prevent insecure infrastructure setups like open ports or weak permissions. A limitation is that not all runtime issues can be detected statically. In practice, tools like Checkov or tfsec are integrated into pipelines before deployment.

What happens if a security scan fails in production pipeline?
If a security scan fails, the pipeline is usually stopped to prevent deployment of vulnerable code. It works by marking the job as failed based on scan results. It is used to enforce security compliance automatically. A trade-off is that urgent fixes may be delayed due to strict enforcement. In real-world cases, teams implement override mechanisms with approvals for critical situations.

How do you monitor security after deployment?
Post-deployment security is monitored using runtime tools like intrusion detection, logging, and alerting systems. It works by collecting logs and analyzing anomalies or suspicious activities. It is used to detect attacks that were not caught during pre-deployment scans. A limitation is that monitoring generates large volumes of data requiring proper filtering. In practice, teams use tools like Prometheus, Grafana, or SIEM systems and set alert thresholds carefully.
