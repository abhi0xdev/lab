Explain how you would design a highly available EC2 architecture across multiple AZs for a web application.
I would deploy EC2 instances across at least two Availability Zones behind an Application Load Balancer. I would use Auto Scaling Groups to maintain desired capacity and distribute traffic evenly. This works by isolating failures to a single AZ while keeping the application available. It is used to achieve fault tolerance and high availability. A limitation is increased cost due to multi-AZ resources. In real scenarios, health checks and proper subnet design are critical to avoid cascading failures.

How would you migrate a running EC2 instance to another region with minimal downtime?
I would create an AMI of the instance, copy it to the target region, and launch a new instance there. I would sync data using tools like rsync or database replication before switching traffic. This works by preparing the target environment before cutover. It is used to minimize downtime during migration. A limitation is that some downtime is still unavoidable during final sync. In real scenarios, DNS cutover with low TTL helps reduce impact.

If your EC2 instance is not reachable via SSH, what troubleshooting steps would you perform?
I would check security groups, NACLs, and ensure port 22 is open to my IP. I would verify the instance status checks and confirm the key pair and username are correct. This works by isolating network, OS, and credential issues. It is used to quickly identify access problems. A limitation is that OS-level issues may require deeper access. In real scenarios, using EC2 Instance Connect or SSM Session Manager helps bypass SSH issues.

Your web application experiences unpredictable traffic spikes. How would you configure auto scaling policies?
I would configure target tracking policies based on CPU or request count per target and set min and max limits. I would also use predictive scaling if patterns exist. This works by automatically adjusting capacity based on demand. It is used to maintain performance while optimizing cost. A limitation is delayed scaling during sudden spikes. In real scenarios, combining scaling with load testing ensures better tuning.

Explain a scenario where scheduled scaling is more appropriate than dynamic scaling.
Scheduled scaling is useful when traffic patterns are predictable, such as daily peak hours. I would pre-scale instances before expected load increases. This works by proactively allocating resources. It is used to avoid latency during scale-up time. A limitation is wasted resources if traffic does not occur. In real scenarios, combining scheduled and dynamic scaling gives better balance.

Your S3 bucket storing critical backups becomes unavailable. How would you recover?
I would enable cross-region replication and access backups from another region. I would also check versioning to restore deleted objects. This works by ensuring redundancy across regions. It is used for disaster recovery. A limitation is replication delay. In real scenarios, lifecycle and backup policies must be tested regularly.

You want to audit all IAM users and roles for compliance. How would you do this?
I would use IAM Access Analyzer and AWS Config to review permissions and detect overly permissive policies. I would generate credential reports to audit usage. This works by providing visibility into access patterns. It is used for security compliance. A limitation is that analysis may not catch business logic risks. In real scenarios, periodic reviews and least privilege enforcement are essential.

Your organization needs to grant cross-account access to a partner. How would you implement it?
I would create an IAM role in my account and allow the partner account to assume it using a trust policy. I would attach least privilege permissions to the role. This works by enabling secure temporary access. It is used to avoid sharing credentials. A limitation is misconfigured trust relationships. In real scenarios, external IDs should be used to prevent confused deputy issues.

How would you secure S3 data in transit and at rest?
I would enforce HTTPS for data in transit and enable server-side encryption using SSE-S3 or SSE-KMS. I would also use bucket policies to restrict access. This works by protecting data from interception and unauthorized access. It is used for compliance requirements. A limitation is key management complexity with KMS. In real scenarios, auditing access logs is important.

How would you connect an on-premises network securely to your VPC?
I would use a site-to-site VPN or AWS Direct Connect for secure connectivity. I would configure routing and security policies accordingly. This works by establishing encrypted tunnels between networks. It is used for hybrid cloud setups. A limitation is latency in VPN connections. In real scenarios, redundancy with multiple tunnels is recommended.

How would you deploy Lambda functions across multiple environments securely?
I would use environment-specific configurations and IAM roles with least privilege. I would deploy using CI/CD pipelines with versioning and aliases. This works by isolating environments and controlling access. It is used to ensure safe deployments. A limitation is managing configuration drift. In real scenarios, secrets should be stored in services like Secrets Manager.

How do you handle multi-cloud Docker deployments with compliance restrictions?
I would standardize container images and use private registries with access controls. I would enforce policies using tools like admission controllers or scanning tools. This works by maintaining consistency across clouds. It is used to meet compliance requirements. A limitation is increased operational complexity. In real scenarios, image signing and scanning are critical.

You need live-patching of Docker host kernel without downtime. How do you achieve it?
I would use tools like Ksplice or KernelCare for live patching. I would also ensure workloads are distributed across nodes. This works by applying patches without rebooting. It is used to maintain uptime. A limitation is dependency on specific tools. In real scenarios, rolling updates of nodes are still preferred for full safety.

How do you troubleshoot container DNS resolution failures?
I would check container DNS settings and verify the cluster DNS service like CoreDNS. I would inspect network policies and logs. This works by identifying resolution path issues. It is used to fix connectivity problems. A limitation is multiple layers of networking. In real scenarios, misconfigured resolv.conf is a common issue.

How do you enforce policy-as-code for Docker security?
I would use tools like OPA or Docker Bench to enforce policies. I would integrate them into CI/CD pipelines. This works by automating compliance checks. It is used to prevent insecure configurations. A limitation is initial setup complexity. In real scenarios, policies must be updated regularly.

Your Ingress controller crashes repeatedly under heavy load. How do you stabilize it?
I would scale the Ingress controller and optimize resource limits. I would also analyze logs and enable rate limiting. This works by handling increased traffic load. It is used to maintain availability. A limitation is increased resource usage. In real scenarios, using multiple replicas improves resilience.

An entire Kubernetes region goes down. How do you failover workloads?
I would use multi-region clusters with traffic routing via DNS or load balancers. I would replicate data across regions. This works by shifting traffic to healthy regions. It is used for disaster recovery. A limitation is data consistency challenges. In real scenarios, active-active setups are preferred.

All pods in one namespace fail readiness checks. What would you do?
I would check application logs, readiness probe configuration, and dependencies like databases. I would verify recent deployments. This works by identifying root cause quickly. It is used to restore service health. A limitation is multiple possible causes. In real scenarios, config changes often trigger such issues.

Your cluster cannot scale fast enough during traffic surge. What would you do?
I would combine HPA with Cluster Autoscaler and pre-scale nodes if needed. I would also optimize resource requests. This works by scaling both pods and nodes. It is used to handle high traffic. A limitation is scaling delay. In real scenarios, buffer capacity helps absorb spikes.

A node hosting critical workloads crashes. How do you recover?
Kubernetes automatically reschedules pods to healthy nodes. I would ensure proper replica settings and node health checks. This works by maintaining desired state. It is used for self-healing. A limitation is temporary downtime. In real scenarios, PodDisruptionBudgets improve stability.

A Pod is stuck in ImagePullBackOff. How do you troubleshoot?
I would check image name, registry access, and credentials. I would verify network connectivity. This works by identifying pull failures. It is used to fix deployment issues. A limitation is dependency on external registries. In real scenarios, expired credentials are common.

Your Terraform apply succeeded but resources misbehave. What would you do?
I would compare actual state with Terraform state and inspect logs. I would validate configurations and dependencies. This works by identifying drift or misconfiguration. It is used to fix inconsistencies. A limitation is limited visibility in state files. In real scenarios, manual changes often cause drift.

How do you run Terraform safely in CI/CD pipelines?
I would use remote state with locking and run plan before apply. I would enforce approvals for production changes. This works by preventing conflicts. It is used for safe deployments. A limitation is slower pipeline execution. In real scenarios, state locking is critical.

Your Terraform state file is too large. What would you do?
I would split infrastructure into smaller modules and separate state files. I would use remote backends. This works by improving manageability. It is used for large projects. A limitation is increased complexity. In real scenarios, proper module design helps.

How do you optimize Terraform performance?
I would use parallelism, modular design, and avoid unnecessary dependencies. I would cache providers. This works by reducing execution time. It is used for faster deployments. A limitation is debugging complexity. In real scenarios, state size impacts performance.

How do you implement Kubernetes monitoring with Prometheus?
I would deploy Prometheus using Helm and configure it to scrape metrics from nodes and pods. I would use exporters for additional metrics. This works by collecting time-series data. It is used for monitoring cluster health. A limitation is storage overhead. In real scenarios, retention policies must be defined.

How do you integrate Prometheus with Alertmanager and Slack?
I would configure Alertmanager with Slack webhook and define alert rules in Prometheus. I would route alerts based on severity. This works by sending real-time notifications. It is used for incident response. A limitation is alert noise. In real scenarios, proper alert tuning is essential.

---

Your application running on EC2 suddenly starts failing health checks behind an ALB. How would you troubleshoot it step by step?
I would first check the Application Load Balancer target group health status to see which instances are failing checks. Then I would verify the health check path, port, and response codes configured in the target group. This works by confirming whether the application or infrastructure is causing the failure. I would also check EC2 instance logs, security groups, and CPU or memory usage to detect runtime issues. A limitation is that ALB health checks only test a specific endpoint. In real scenarios, application logs and monitoring metrics help identify root causes.

A production RDS database is running out of storage. What immediate actions would you take?
I would immediately increase the allocated storage for the RDS instance because AWS allows storage scaling without downtime for most engines. Then I would check database logs and storage metrics to identify the reason for rapid growth. This works by preventing database crashes due to full disk space. I would also remove unnecessary data, optimize queries, and configure automated backups. A limitation is storage scaling cannot be reduced later. In real scenarios, setting storage alarms prevents such incidents.

Your company wants to reduce AWS cost for idle resources. What strategies would you implement?
I would identify idle EC2 instances, unused volumes, and unattached load balancers using cost analysis tools. Then I would implement auto scaling, scheduled shutdowns, and rightsizing for resources. This works by aligning infrastructure usage with actual demand. It is used to optimize cloud spending without affecting performance. A limitation is aggressive cost reduction may impact availability. In real scenarios, tagging resources helps track ownership and cost allocation.

If an S3 bucket accidentally gets deleted, how would you recover the data?
If versioning and replication were enabled, data can be recovered from replicated buckets or previous versions. I would also check backups stored in other regions or lifecycle storage systems. This works by relying on redundancy and backup strategies. It is used to protect against accidental deletion. A limitation is recovery may not be possible without backups. In real scenarios, enabling versioning and cross-region replication is recommended.

How would you implement centralized logging for multiple AWS accounts?
I would configure services to send logs to a centralized logging account using CloudWatch Logs or a logging pipeline. Then logs can be aggregated and analyzed using services like OpenSearch or external monitoring tools. This works by collecting logs from multiple environments into a single system. It is used for security monitoring and troubleshooting. A limitation is increased storage and management overhead. In real scenarios, cross-account roles are used for secure log access.

Your Lambda function is timing out frequently. How do you troubleshoot and optimize it?
I would first check CloudWatch logs to identify slow operations or errors. Then I would analyze memory allocation and increase it if needed since higher memory improves CPU performance. This works by optimizing resource allocation for serverless workloads. It is used to improve performance and prevent execution failures. A limitation is longer functions increase execution cost. In real scenarios, caching and asynchronous processing help reduce latency.

How do you securely store secrets for applications running on EC2 or Lambda?
Secrets should be stored in services like AWS Secrets Manager or Systems Manager Parameter Store with encryption enabled. Applications retrieve them at runtime using IAM roles instead of hardcoding credentials. This works by separating secrets from application code. It is used to improve security and simplify secret rotation. A limitation is additional access control management. In real scenarios, automatic secret rotation is configured.

One of your containers keeps restarting in production. How do you identify the root cause?
I check container logs using docker logs or Kubernetes logs to identify application errors. Then I inspect container configuration, resource limits, and startup commands. This works by identifying runtime failures that cause container crashes. It is used to debug application or environment issues quickly. A limitation is logs may be incomplete if the container crashes instantly. In real scenarios, running the container interactively helps identify problems.

Your Docker image size has grown very large. How do you optimize it?
I use multi-stage builds to separate build dependencies from runtime components. I also switch to minimal base images and remove unnecessary packages. This works by reducing the number of layers and dependencies in the image. It is used to improve performance and security. A limitation is compatibility issues with minimal images. In real scenarios, scanning images ensures optimized builds.

How do you scan Docker images for vulnerabilities before deployment?
I integrate image scanning tools like Trivy or registry scanners in the CI/CD pipeline. These tools analyze container layers for known vulnerabilities. This works by detecting security risks before deployment. It is used to enforce secure container practices. A limitation is false positives in vulnerability reports. In real scenarios, pipelines block deployments if critical vulnerabilities are detected.

How would you manage secrets in Docker containers securely?
Secrets should not be stored in the Docker image or environment variables directly. Instead, they should be injected at runtime using secret management tools or orchestrator features. This works by keeping credentials outside the container image. It is used to prevent credential exposure. A limitation is increased configuration complexity. In real scenarios, orchestrators like Kubernetes manage container secrets.

Some pods are running but not receiving traffic from the service. How do you troubleshoot?
I verify that the service selector labels match the pod labels correctly. Then I check service endpoints and confirm that pods are ready and passing readiness probes. This works by ensuring the service can route traffic to the correct pods. It is used to diagnose connectivity issues. A limitation is network policies may also block traffic.

A deployment rollout caused a production outage. How would you roll back quickly?
I would use the deployment rollback command to revert to the previous stable version. Kubernetes maintains deployment history which allows quick restoration. This works by redeploying the last working replica set. It is used to recover services quickly after failed deployments. A limitation is database changes may not roll back automatically.

Your cluster nodes are running out of memory. What steps would you take to stabilize the cluster?
I would first identify pods consuming excessive memory using resource metrics. Then I would adjust resource limits or scale nodes using cluster autoscaling. This works by redistributing workloads and preventing node failures. It is used to maintain cluster stability. A limitation is scaling nodes may take time during peak traffic.

How would you implement zero-downtime deployments in Kubernetes?
Zero-downtime deployments are implemented using rolling update strategies with readiness probes configured. Kubernetes gradually replaces old pods with new ones while maintaining available replicas. This works by ensuring new pods are healthy before traffic is routed. It is used to maintain service availability. A limitation is incorrect readiness probes can cause downtime.

You need to restrict communication between namespaces. How would you implement this?
I would use Kubernetes network policies to control traffic between pods and namespaces. These policies define allowed ingress and egress communication rules. This works by enforcing network isolation within the cluster. It is used to improve security between services. A limitation is dependency on the cluster network plugin.

How would you debug a pod stuck in CrashLoopBackOff state?
I check container logs and pod events to identify startup failures. Then I verify environment variables, configuration files, and resource limits. This works by identifying repeated container crashes. It is used to diagnose application issues quickly. A limitation is incomplete logs if containers fail immediately.

Your Terraform state file gets accidentally deleted. How would you recover it?
If a remote backend is used, I would restore the state from the backend storage or backup. Then I would re-import resources if necessary to rebuild state. This works by reconstructing the infrastructure mapping. It is used to maintain Terraform resource tracking. A limitation is manual effort during recovery.

How do you manage secrets in Terraform while working in teams?
Secrets are stored outside Terraform code using secret managers or encrypted variables. Terraform retrieves them securely during execution. This works by keeping sensitive data out of version control. It is used to protect credentials in collaborative environments. A limitation is dependency on external secret systems.

Multiple teams are modifying the same Terraform code. How do you avoid conflicts?
I store Terraform state in a remote backend with state locking enabled. This prevents simultaneous modifications by multiple users. It works by locking the state file during operations. It is used to maintain infrastructure consistency. A limitation is dependency on backend availability.

How would you implement reusable Terraform modules for different environments?
I create reusable modules with parameterized variables and outputs. Each environment uses different variable values for configuration. This works by separating reusable infrastructure logic from environment-specific settings. It is used to maintain consistent infrastructure deployments. A limitation is complexity in module version management.

How do you handle drift detection between Terraform code and existing infrastructure?
I run Terraform plan regularly to compare the desired configuration with the current infrastructure state. This works by identifying manual changes outside Terraform. It is used to maintain infrastructure consistency. A limitation is manual remediation may be required.

Your application latency suddenly increases. How would you use metrics to identify the root cause?
I analyze monitoring metrics such as CPU usage, memory consumption, request latency, and error rates. This works by correlating performance metrics with application behavior. It is used to identify bottlenecks in infrastructure or application layers. A limitation is incomplete monitoring data.

How do you design a monitoring strategy for microservices on Kubernetes?
I deploy metrics collection tools like Prometheus and visualization dashboards like Grafana. Each service exposes metrics that are aggregated for monitoring. This works by providing observability across distributed services. It is used to detect failures and performance issues. A limitation is managing large volumes of metrics.

If Prometheus storage is filling up quickly, how would you optimize it?
I would reduce metric retention time and limit the number of high-cardinality metrics. This works by controlling the amount of data stored by Prometheus. It is used to manage storage consumption. A limitation is reduced historical visibility.

How do you create meaningful alerts that reduce false positives?
I design alerts based on sustained threshold conditions rather than single metric spikes. This works by filtering temporary fluctuations. It is used to improve incident response quality. A limitation is overly strict thresholds may miss early warnings.

---