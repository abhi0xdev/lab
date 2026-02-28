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
