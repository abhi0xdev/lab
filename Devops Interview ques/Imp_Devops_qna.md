1. What is the DevOps mindset and why is it important?
   DevOps mindset is about owning the full lifecycle of software from development to deployment and operations. It works by breaking silos between teams and focusing on automation and collaboration. It is used to deliver software faster with fewer failures. A trade-off is that it requires cultural change which can be slow and resisted. In real projects, lack of ownership often causes delays, so clear responsibility is critical.

2. How would you deploy an application on cloud securely?
   Deploying securely means creating infrastructure using VPC, subnets, and IAM roles to control access. It works by isolating resources in private networks and granting least privilege permissions. It is used to prevent unauthorized access and data breaches. A limitation is increased complexity in managing policies and networking. In practice, misconfigured security groups often expose services unintentionally.

3. What is a VPC and how does it work?
   A VPC is a virtual private network in the cloud that isolates resources. It works by defining IP ranges, subnets, and routing rules. It is used to control network traffic and improve security. A trade-off is added complexity in configuration and troubleshooting. In real scenarios, incorrect routing tables can break connectivity between services.

4. What are subnets and why are they used?
   Subnets are smaller network segments within a VPC used to organize resources. They work by dividing IP ranges into public and private sections. They are used to isolate services like databases from the internet. A limitation is managing multiple subnets increases operational overhead. In practice, placing sensitive services in public subnets is a common mistake.

5. What is IAM and how does it ensure security?
   IAM manages access to cloud resources through users, roles, and policies. It works by assigning permissions based on least privilege principles. It is used to secure access without hardcoding credentials. A trade-off is complex policy management can lead to misconfigurations. In real-world cases, overly permissive roles can cause major security risks.

6. What is CI/CD and why is it important?
   CI/CD is a process that automates building, testing, and deploying code changes. It works by triggering pipelines on code commits. It is used to reduce manual errors and speed up delivery. A limitation is initial setup and maintenance effort. In practice, flaky tests can break pipelines frequently.

7. How does a CI pipeline work in practice?
   A CI pipeline builds code, runs tests, and validates changes automatically. It works by integrating with version control systems like Git. It is used to catch issues early before deployment. A trade-off is increased compute cost for frequent builds. In real projects, failing tests can block releases if not handled properly.

8. What is the difference between CI and CD?
   CI focuses on integrating and testing code changes, while CD focuses on deploying them. CI works on every commit, while CD automates release to environments. It is used to ensure continuous delivery of stable software. A limitation is CD pipelines need strict controls to avoid accidental deployments. In practice, approval gates are often added for production.

9. How do you design a CI/CD pipeline?
   A pipeline is designed with stages like build, test, and deploy. It works by defining workflows in tools like GitHub Actions. It is used to automate the entire release process. A trade-off is complex pipelines become hard to debug. In real-world scenarios, separating environments like dev, staging, and prod is essential.

10. What is Docker and why is it used?
    Docker is a containerization platform that packages applications with dependencies. It works by using images and containers to run apps consistently. It is used to avoid environment-related issues. A limitation is managing container sprawl at scale. In practice, improper image sizes can slow deployments.

11. What is a Dockerfile and how does it work?
    A Dockerfile defines instructions to build a container image. It works by executing commands like install dependencies and copy files. It is used to create reproducible environments. A trade-off is inefficient layers increase image size. In real scenarios, using minimal base images improves performance.

12. What is Kubernetes and why is it needed?
    Kubernetes is an orchestration tool for managing containers at scale. It works by scheduling and managing pods across nodes. It is used for high availability and scalability. A limitation is steep learning curve. In real systems, misconfigured resources can lead to pod failures.

13. What is a Pod in Kubernetes?
    A Pod is the smallest deployable unit in Kubernetes containing one or more containers. It works by sharing network and storage resources. It is used to run tightly coupled containers together. A limitation is pods are ephemeral and can be destroyed anytime. In practice, logs must be externalized to avoid loss.

14. What is a Deployment in Kubernetes?
    A Deployment manages replica sets and ensures desired state of pods. It works by rolling updates and scaling pods. It is used for maintaining application availability. A trade-off is rollout failures can impact services. In real scenarios, readiness probes help prevent downtime.

15. What is the difference between Pod and Deployment?
    A Pod runs containers directly, while a Deployment manages multiple pods. Deployment works by maintaining replicas and handling updates. It is used for scalability and fault tolerance. A limitation is extra abstraction adds complexity. In real-world usage, deployments are preferred for production workloads.

16. Why are containers better than virtual machines?
    Containers share the host OS and are lightweight compared to VMs. They work by isolating processes instead of entire systems. They are used for faster startup and efficient resource usage. A trade-off is less isolation than VMs. In practice, security hardening is required to prevent container escapes.

17. What is CrashLoopBackOff in Kubernetes?
    CrashLoopBackOff occurs when a container repeatedly fails and restarts. It works as Kubernetes tries to restart failing pods with delay. It is used as a signal for debugging failures. A limitation is it hides root cause without logs. In real scenarios, checking logs and events is critical.

18. How do you debug a failing pod?
    Debugging involves checking logs, events, and pod description. It works by identifying errors like config issues or crashes. It is used to quickly resolve production issues. A trade-off is time-consuming if logs are unclear. In practice, enabling proper logging is essential.

19. What is Linux file permission and why is it important?
    Linux permissions control access to files using read, write, and execute flags. It works by assigning permissions to user, group, and others. It is used for security and access control. A limitation is incorrect permissions can break applications. In real systems, least privilege should be applied.

20. What is chmod and how is it used?
    chmod changes file permissions in Linux. It works by assigning numeric or symbolic values. It is used to control file access. A trade-off is incorrect usage can expose sensitive data. In practice, scripts should avoid overly permissive settings.

21. What is chown and when do you use it?
    chown changes file ownership in Linux. It works by assigning a user and group to a file or directory. It is used to ensure proper access control for applications. A limitation is incorrect ownership can cause permission issues. In real systems, services often fail if ownership is not set correctly.

22. How do you check running processes in Linux?
    Processes are checked using commands like ps and top. It works by listing active processes with resource usage. It is used for monitoring system health and debugging issues. A trade-off is too many processes make analysis harder. In practice, identifying high CPU or memory processes is key.

23. What is a shell script and why is it used?
    A shell script is a file containing commands executed in sequence. It works by automating repetitive tasks. It is used to reduce manual effort and errors. A limitation is debugging complex scripts can be difficult. In real-world usage, logging inside scripts helps troubleshooting.

24. How would you write a script to restart a service?
    A script checks service status and restarts if not running. It works using commands like systemctl. It is used for automation and reliability. A trade-off is false positives can restart healthy services. In practice, adding checks and logs is important.

25. What is logging and why is it important?
    Logging records events generated by applications or systems. It works by writing logs to files or centralized systems. It is used for debugging and monitoring. A limitation is large logs can be hard to analyze. In real scenarios, structured logging improves readability.

26. What are metrics in monitoring?
    Metrics are numerical data representing system performance. They work by collecting values like CPU, memory, and latency. They are used to detect trends and anomalies. A trade-off is too many metrics create noise. In practice, focusing on key metrics is essential.

27. What is alerting in monitoring?
    Alerting notifies when system thresholds are breached. It works by defining rules on metrics or logs. It is used to respond quickly to issues. A limitation is false alerts can cause alert fatigue. In real systems, tuning thresholds is critical.

28. How do you read logs effectively?
    Reading logs involves filtering, searching, and identifying error patterns. It works by using tools like grep or centralized logging systems. It is used to diagnose issues quickly. A trade-off is unstructured logs slow analysis. In practice, timestamps and log levels are crucial.

29. What is Infrastructure as Code?
    IaC is the practice of managing infrastructure using code. It works by defining resources in files like Terraform scripts. It is used for consistency and automation. A limitation is learning curve and debugging complexity. In real scenarios, version control is essential.

30. Why is Terraform used?
    Terraform is used to provision infrastructure declaratively. It works by comparing desired state with actual state. It is used for multi-cloud deployments. A trade-off is state file management can be tricky. In practice, remote state storage is recommended.

31. What is Terraform state file?
    The state file stores current infrastructure details. It works by tracking resource mappings. It is used for updates and changes. A limitation is corruption or loss causes issues. In real systems, state locking prevents conflicts.

32. How do you manage multiple environments in IaC?
    Environments are managed using separate state files or workspaces. It works by isolating configurations for dev, staging, and prod. It is used to prevent conflicts. A trade-off is duplication of configurations. In practice, reusable modules help reduce duplication.

33. What is GitHub Actions?
    GitHub Actions is a CI/CD tool integrated with GitHub. It works by running workflows on events like commits. It is used to automate pipelines. A limitation is limited control compared to self-hosted tools. In real scenarios, secrets management is important.

34. How do you secure secrets in pipelines?
    Secrets are stored in encrypted vaults or CI tools. It works by injecting them at runtime. It is used to avoid exposing credentials. A trade-off is complexity in management. In practice, rotating secrets regularly is critical.

35. What is a Kubernetes Service?
    A Service exposes pods to internal or external traffic. It works by providing a stable IP and DNS. It is used for communication between components. A limitation is incorrect configuration can block access. In real systems, proper port mapping is essential.

36. What is scaling in Kubernetes?
    Scaling increases or decreases the number of pods. It works manually or automatically using HPA. It is used to handle varying load. A trade-off is over-scaling increases cost. In practice, metrics-based scaling is preferred.

37. What is Horizontal Pod Autoscaler?
    HPA automatically scales pods based on metrics. It works by monitoring CPU or custom metrics. It is used to maintain performance under load. A limitation is delayed scaling during spikes. In real scenarios, tuning thresholds is important.

38. What is a common cause of pod failure?
    Common causes include misconfiguration, resource limits, or missing dependencies. It works as containers fail to start or crash. It is used to identify system weaknesses. A limitation is logs may not always show root cause. In practice, checking events helps.

39. What is networking in Kubernetes?
    Networking allows communication between pods and services. It works using cluster IPs and DNS. It is used for service discovery. A trade-off is complexity increases with scale. In real systems, network policies improve security.

40. What is a real-world debugging scenario you faced?
    A pod was failing due to memory limits causing OOMKilled errors. It worked by analyzing logs and metrics. It was resolved by adjusting resource limits. A limitation is over-provisioning wastes resources. In practice, monitoring memory usage is key.

41. What is least privilege principle?
    Least privilege means giving minimal required permissions. It works by restricting access to only necessary actions. It is used to improve security. A trade-off is frequent permission updates. In real systems, over-permissioning is a common risk.

42. What is EC2 instance?
    EC2 is a virtual server in AWS. It works by running applications on cloud infrastructure. It is used for scalable compute. A limitation is manual management overhead. In practice, auto-scaling helps manage load.

43. What is auto-scaling?
    Auto-scaling adjusts resources based on demand. It works by monitoring metrics like CPU usage. It is used for cost optimization and performance. A trade-off is delayed response to sudden spikes. In real systems, scaling policies must be tuned.

44. What is a load balancer?
    A load balancer distributes traffic across servers. It works by routing requests based on rules. It is used for high availability. A limitation is misconfiguration can cause uneven load. In practice, health checks are critical.

45. What is the difference between public and private subnet?
    Public subnets have internet access, private do not. They work using routing tables and gateways. They are used to isolate sensitive resources. A limitation is NAT setup adds complexity. In real systems, databases should be in private subnets.

46. What is NAT Gateway and why is it used?
    A NAT Gateway allows instances in a private subnet to access the internet securely. It works by routing outbound traffic through a managed gateway while blocking inbound connections. It is used to keep internal services secure while still allowing updates or API calls. A trade-off is additional cost compared to direct internet access. In real systems, misconfigured routes can prevent outbound connectivity.

47. What is a security group in cloud?
    A security group is a virtual firewall that controls inbound and outbound traffic. It works by defining rules based on ports, protocols, and IP ranges. It is used to restrict access to cloud resources. A limitation is managing many rules becomes complex. In practice, overly open rules like 0.0.0.0/0 can create security risks.

48. What is the difference between security group and network ACL?
    Security groups are stateful while network ACLs are stateless. They work at instance level and subnet level respectively. They are used together for layered security. A trade-off is added complexity in debugging network issues. In real scenarios, mismatched rules can block traffic unexpectedly.

49. What is container orchestration?
    Container orchestration manages deployment, scaling, and networking of containers. It works using tools like Kubernetes. It is used to automate large-scale container management. A limitation is high complexity. In real systems, misconfigured orchestration leads to downtime.

50. What is image versioning in Docker?
    Image versioning tags Docker images with versions. It works by assigning labels like v1 or latest. It is used to track changes and rollbacks. A trade-off is poor tagging leads to confusion. In practice, immutable version tags are recommended.

51. What is a rolling update in Kubernetes?
    A rolling update gradually replaces old pods with new ones. It works by updating pods in batches. It is used to avoid downtime during deployments. A limitation is failures during rollout can affect users. In real scenarios, rollback strategies must be ready.

52. What is a readiness probe?
    A readiness probe checks if a container is ready to serve traffic. It works by sending periodic checks. It is used to prevent traffic to unhealthy pods. A trade-off is incorrect probe configuration can delay deployments. In practice, proper endpoints should be defined.

53. What is a liveness probe?
    A liveness probe checks if a container is alive. It works by restarting containers when checks fail. It is used to recover from stuck states. A limitation is aggressive checks can cause unnecessary restarts. In real systems, tuning intervals is important.

54. What is a common CI/CD failure scenario?
    A common failure is pipeline breaking due to failed tests or build errors. It works by stopping deployment automatically. It is used to maintain code quality. A limitation is flaky tests cause false failures. In practice, stabilizing tests is essential.

55. How do you handle failed deployments?
    Failed deployments are handled by rollback mechanisms. It works by reverting to previous stable versions. It is used to restore system stability. A trade-off is rollback may not fix underlying issues. In real systems, root cause analysis is required.

56. What is blue-green deployment?
    Blue-green deployment uses two environments for deployment. It works by switching traffic between old and new versions. It is used for zero downtime releases. A limitation is higher infrastructure cost. In practice, database changes must be handled carefully.

57. What is canary deployment?
    Canary deployment releases changes to a small subset of users. It works by gradually increasing traffic. It is used to detect issues early. A trade-off is slower rollout. In real systems, monitoring metrics is crucial during rollout.

58. What is log aggregation?
    Log aggregation collects logs from multiple sources into one place. It works using tools like ELK stack. It is used for centralized analysis. A limitation is storage and cost overhead. In practice, log retention policies are important.

59. What is observability?
    Observability is the ability to understand system behavior using logs, metrics, and traces. It works by collecting and analyzing data. It is used to debug complex systems. A trade-off is increased tooling complexity. In real systems, correlation between logs and metrics is key.

60. What is tracing in monitoring?
    Tracing tracks requests across multiple services. It works by assigning unique IDs to requests. It is used to debug distributed systems. A limitation is overhead in instrumentation. In practice, sampling helps reduce cost.

61. What is CPU throttling in Kubernetes?
    CPU throttling occurs when containers exceed CPU limits. It works by restricting CPU usage. It is used to enforce resource allocation. A trade-off is performance degradation. In real systems, proper resource limits should be defined.

62. What is OOMKilled error?
    OOMKilled happens when a container exceeds memory limits. It works by terminating the container. It is used to prevent system crashes. A limitation is sudden termination affects application stability. In practice, monitoring memory usage is critical.

63. What is a multi-stage Docker build?
    A multi-stage build uses multiple stages to optimize images. It works by copying only required artifacts to final image. It is used to reduce image size. A trade-off is increased complexity in Dockerfile. In real systems, it improves security and performance.

64. What is caching in CI/CD?
    Caching stores dependencies to speed up builds. It works by reusing previous build artifacts. It is used to reduce pipeline time. A limitation is cache invalidation issues. In practice, cache keys must be managed carefully.

65. What is Git branching strategy?
    Branching strategy defines how code changes are managed. It works using branches like feature, develop, and main. It is used to organize development workflow. A trade-off is complex strategies slow development. In real systems, simple strategies are preferred.

66. What is merge conflict and how do you handle it?
    Merge conflict occurs when changes overlap. It works by requiring manual resolution. It is used to maintain code consistency. A limitation is time-consuming resolution. In practice, frequent merges reduce conflicts.

67. What is infrastructure drift?
    Infrastructure drift occurs when actual state differs from desired state. It works due to manual changes outside IaC. It is used to detect inconsistencies. A trade-off is difficult tracking. In real systems, avoid manual changes.

68. What is idempotency in DevOps?
    Idempotency means operations produce same result repeatedly. It works by ensuring consistent state. It is used in automation tools. A limitation is harder to design. In practice, Terraform follows this principle.

69. What is a sidecar container?
    A sidecar container runs alongside main container. It works by providing supporting functionality. It is used for logging or monitoring. A trade-off is increased resource usage. In real systems, sidecars help in observability.

70. What is a config map in Kubernetes?
    ConfigMap stores configuration data separately from code. It works by injecting configs into pods. It is used for flexibility. A limitation is not suitable for secrets. In practice, environment separation is easier.

71. What is a secret in Kubernetes?
    Secrets store sensitive data like passwords. It works by encoding and injecting into pods. It is used for security. A trade-off is base64 encoding is not encryption. In real systems, external secret managers are preferred.

72. What is a common networking issue in Kubernetes?
    A common issue is service not reachable due to DNS or port mismatch. It works due to misconfiguration. It is used to debug connectivity. A limitation is hard to trace in large clusters. In practice, checking service endpoints helps.

73. What is disk pressure in Kubernetes?
    Disk pressure occurs when node storage is low. It works by evicting pods. It is used to protect node stability. A limitation is sudden pod termination. In real systems, monitoring disk usage is essential.

74. What is node failure in Kubernetes?
    Node failure happens when a node becomes unavailable. It works by rescheduling pods to other nodes. It is used for high availability. A limitation is temporary downtime. In practice, multi-node clusters improve resilience.

75. What is a real-world CI/CD optimization?
    Pipeline time was reduced by adding caching and parallel jobs. It worked by optimizing build steps. It was used to speed up deployments. A limitation is increased complexity. In real systems, balancing speed and reliability is key.

76. What is a pipeline trigger in CI/CD?
    A pipeline trigger starts a workflow based on events like commits or pull requests. It works by listening to repository changes and executing defined steps. It is used to automate build and deployment processes. A trade-off is too many triggers can overload systems. In real scenarios, filtering branches helps avoid unnecessary runs.

77. What is artifact management in CI/CD?
    Artifact management stores build outputs like binaries or images. It works by uploading artifacts to repositories after builds. It is used for versioning and reuse in deployments. A limitation is storage cost and cleanup management. In practice, retention policies are required to control usage.

78. What is dependency management in pipelines?
    Dependency management ensures required libraries are available during builds. It works by downloading or caching dependencies. It is used to ensure consistent builds. A trade-off is dependency conflicts can break builds. In real systems, version pinning avoids unexpected failures.

79. What is parallel execution in CI/CD?
    Parallel execution runs multiple jobs simultaneously. It works by splitting tasks into independent steps. It is used to reduce pipeline execution time. A limitation is resource contention. In real scenarios, careful job design prevents bottlenecks.

80. What is a rollback strategy in CI/CD?
    Rollback strategy restores previous stable versions after failure. It works using versioned artifacts or deployments. It is used to maintain system stability. A trade-off is rollback may not fix root issues. In practice, testing before release reduces rollback need.

81. What is immutable infrastructure?
    Immutable infrastructure means servers are not modified after deployment. It works by replacing instances instead of updating them. It is used to ensure consistency and reliability. A limitation is higher resource usage. In real systems, it simplifies debugging.

82. What is configuration drift and how do you prevent it?
    Configuration drift happens when systems deviate from defined configurations. It works due to manual changes. It is used to detect inconsistencies. A trade-off is difficult tracking. In practice, enforcing IaC prevents drift.

83. What is service downtime and how do you minimize it?
    Downtime is when a service becomes unavailable. It works due to failures or deployments. It is used as a measure of reliability. A limitation is unavoidable in some cases. In real systems, redundancy and rolling updates minimize downtime.

84. What is health check in DevOps?
    Health checks verify system or service status. They work by sending periodic requests. They are used to detect failures early. A trade-off is additional overhead. In real scenarios, proper endpoints are necessary.

85. What is system bottleneck?
    A bottleneck is a component limiting system performance. It works when resources are constrained. It is used to identify optimization areas. A limitation is hard to detect in distributed systems. In practice, monitoring helps locate bottlenecks.

86. What is horizontal scaling?
    Horizontal scaling adds more instances to handle load. It works by distributing traffic across nodes. It is used for scalability and availability. A trade-off is increased infrastructure cost. In real systems, load balancers are required.

87. What is vertical scaling?
    Vertical scaling increases resources of a single instance. It works by upgrading CPU or memory. It is used for simple scaling needs. A limitation is hardware limits. In real scenarios, it causes downtime during upgrades.

88. What is infrastructure provisioning?
    Provisioning is creating resources like servers or networks. It works using tools or manual setup. It is used to prepare environments. A trade-off is manual provisioning is error-prone. In practice, IaC automates provisioning.

89. What is environment parity?
    Environment parity means dev, staging, and production environments are similar. It works by using consistent configurations. It is used to avoid deployment issues. A limitation is cost of maintaining multiple environments. In real systems, containers help achieve parity.

90. What is a build failure and how do you debug it?
    Build failure occurs when code does not compile or tests fail. It works by stopping pipeline execution. It is used to prevent bad deployments. A limitation is unclear errors can slow debugging. In practice, logs and step isolation help identify issues.

91. What is log rotation?
    Log rotation manages log file size by archiving or deleting old logs. It works by setting size or time limits. It is used to prevent disk exhaustion. A limitation is losing historical data. In real systems, centralized logging avoids this issue.

92. What is rate limiting?
    Rate limiting restricts number of requests to a service. It works by enforcing thresholds. It is used to prevent abuse and overload. A trade-off is blocking legitimate users. In real scenarios, proper limits must be configured.

93. What is API gateway?
    API gateway manages and routes API requests. It works by handling authentication and routing. It is used for centralized control. A limitation is added latency. In real systems, caching improves performance.

94. What is a reverse proxy?
    A reverse proxy forwards client requests to backend servers. It works by acting as an intermediary. It is used for load balancing and security. A trade-off is additional configuration complexity. In practice, Nginx is commonly used.

95. What is SSL/TLS?
    SSL/TLS encrypts data in transit. It works using certificates and encryption protocols. It is used for secure communication. A limitation is certificate management overhead. In real systems, expired certificates cause outages.

96. What is secret rotation?
    Secret rotation updates credentials periodically. It works by replacing old secrets with new ones. It is used to improve security. A trade-off is operational complexity. In real scenarios, automation is necessary.

97. What is backup and recovery?
    Backup stores data copies for restoration. It works by saving snapshots or files. It is used to recover from failures. A limitation is storage cost. In real systems, regular testing of recovery is important.

98. What is disaster recovery?
    Disaster recovery restores systems after major failures. It works using backups and failover systems. It is used to ensure business continuity. A trade-off is high cost. In real scenarios, defining RTO and RPO is critical.

99. What is RTO and RPO?
    RTO is recovery time objective and RPO is recovery point objective. They work as targets for recovery. They are used to define disaster recovery plans. A limitation is strict targets increase cost. In real systems, balancing cost and recovery is key.

100. What is system resilience?
     Resilience is the ability to recover from failures. It works by using redundancy and failover. It is used to maintain uptime. A trade-off is increased complexity. In real systems, testing failure scenarios is essential.

101. What is chaos engineering?
     Chaos engineering tests system reliability by introducing failures. It works by simulating real-world issues. It is used to improve resilience. A limitation is risk of unintended impact. In practice, controlled experiments are required.

102. What is service mesh?
     Service mesh manages communication between services. It works using sidecar proxies. It is used for observability and security. A limitation is added complexity. In real systems, tools like Istio are used.

103. What is container registry?
     Container registry stores Docker images. It works by hosting versioned images. It is used for deployment. A limitation is storage management. In practice, access control is important.

104. What is image scanning?
     Image scanning checks container images for vulnerabilities. It works using security tools. It is used to improve security. A limitation is false positives. In real systems, regular scanning is required.

105. What is network latency?
     Latency is delay in network communication. It works due to distance or processing time. It is used to measure performance. A limitation is hard to eliminate fully. In real systems, caching reduces latency.

106. What is throughput?
     Throughput is amount of data processed over time. It works by measuring system capacity. It is used for performance evaluation. A limitation is affected by bottlenecks. In real systems, scaling improves throughput.

107. What is system availability?
     Availability measures uptime of a system. It works as a percentage over time. It is used for reliability metrics. A limitation is achieving high availability is costly. In practice, redundancy improves availability.

108. What is SLA?
     SLA defines service level agreements for uptime and performance. It works as a contract between provider and user. It is used to set expectations. A limitation is penalties for breaches. In real systems, monitoring ensures compliance.

109. What is SLO?
     SLO is service level objective defining internal targets. It works as measurable goals. It is used to track performance. A limitation is unrealistic targets cause stress. In practice, achievable targets are important.

110. What is SLI?
     SLI is service level indicator measuring performance metrics. It works by tracking specific metrics. It is used to evaluate SLOs. A limitation is choosing wrong indicators. In real systems, latency and error rate are common SLIs.

111. What is zero downtime deployment?
     Zero downtime deployment ensures no service interruption. It works using rolling or blue-green strategies. It is used for continuous availability. A limitation is complex setup. In real systems, testing is critical.

112. What is feature flag?
     Feature flag enables or disables features dynamically. It works by toggling code paths. It is used for safe deployments. A limitation is code complexity. In real scenarios, flags must be cleaned up.

113. What is dependency injection?
     Dependency injection provides dependencies externally. It works by decoupling components. It is used for flexibility. A limitation is added abstraction. In real systems, it improves testability.

114. What is code versioning?
     Code versioning tracks changes in code. It works using Git. It is used for collaboration. A limitation is managing conflicts. In practice, proper commit messages help.

115. What is a hotfix?
     A hotfix is a quick fix for production issues. It works by patching code directly. It is used for urgent problems. A limitation is bypassing standard process. In real systems, proper testing is still required.

116. What is system outage?
     Outage is complete service failure. It works due to major issues. It is used as reliability metric. A limitation is business impact. In real systems, failover systems reduce outages.

117. What is load testing?
     Load testing evaluates system under stress. It works by simulating traffic. It is used to identify limits. A limitation is environment differences. In practice, realistic scenarios are important.

118. What is stress testing?
     Stress testing pushes system beyond limits. It works by overloading resources. It is used to find breaking points. A limitation is potential system crash. In real systems, controlled testing is required.

119. What is capacity planning?
     Capacity planning ensures sufficient resources for future demand. It works by analyzing trends. It is used for scaling decisions. A limitation is prediction errors. In practice, monitoring data helps.

120. What is a real-world incident you handled?
     An incident involved pod crashes due to memory leak. It worked by analyzing logs and metrics. It was resolved by fixing code and adjusting limits. A limitation is identifying root cause takes time. In real systems, monitoring helps early detection.

121. What is infrastructure cost optimization?
     Infrastructure cost optimization is the process of reducing cloud expenses while maintaining performance. It works by rightsizing resources, using auto-scaling, and removing unused services. It is used to control cloud spending in production environments. A trade-off is aggressive optimization can impact performance. In real systems, monitoring usage patterns helps make better decisions.

122. What is spot instance and when would you use it?
     A spot instance is a low-cost cloud instance that can be terminated anytime. It works by using unused cloud capacity at discounted prices. It is used for non-critical or batch workloads. A limitation is unpredictability of termination. In real scenarios, it should not be used for critical applications.

123. What is a stateful vs stateless application?
     A stateless application does not store session data, while a stateful one does. Stateless works by handling each request independently. It is used for scalability and simplicity. A limitation is state management becomes external. In real systems, databases or caches store state.

124. What is caching and why is it used?
     Caching stores frequently accessed data for faster retrieval. It works by keeping data in memory or fast storage. It is used to reduce latency and load. A trade-off is stale data issues. In real systems, cache invalidation must be handled carefully.

125. What is CDN?
     A CDN distributes content across global servers. It works by serving content from nearest location. It is used to reduce latency and improve performance. A limitation is additional cost. In real systems, caching static assets improves speed.

126. What is DNS and how does it work?
     DNS translates domain names to IP addresses. It works by querying distributed servers. It is used for accessing services easily. A limitation is propagation delay. In real systems, DNS misconfiguration can cause outages.

127. What is blue-green vs canary deployment difference?
     Blue-green switches all traffic at once, while canary shifts gradually. They work using separate environments or traffic splitting. They are used to reduce deployment risk. A limitation is increased complexity. In real systems, canary is safer for critical apps.

128. What is a build artifact?
     A build artifact is the output of a build process. It works by packaging compiled code or binaries. It is used for deployment. A limitation is storage management. In real systems, versioning artifacts is important.

129. What is pipeline as code?
     Pipeline as code defines CI/CD workflows in code files. It works using YAML or scripts. It is used for version control and reproducibility. A limitation is debugging pipeline code can be hard. In real systems, code reviews improve quality.

130. What is configuration management?
     Configuration management ensures systems are consistently configured. It works using tools or scripts. It is used to avoid manual errors. A limitation is tool complexity. In real systems, automation reduces drift.

131. What is service discovery?
     Service discovery allows services to find each other dynamically. It works using DNS or registries. It is used in microservices architecture. A limitation is added complexity. In real systems, Kubernetes handles this automatically.

132. What is a distributed system?
     A distributed system consists of multiple components working together. It works by communicating over a network. It is used for scalability and fault tolerance. A limitation is debugging complexity. In real systems, network failures are common.

133. What is eventual consistency?
     Eventual consistency means data becomes consistent over time. It works by allowing temporary inconsistencies. It is used in distributed systems. A trade-off is stale reads. In real systems, design must handle delays.

134. What is a race condition?
     A race condition occurs when multiple processes access shared data simultaneously. It works due to lack of synchronization. It is used to identify concurrency issues. A limitation is hard to reproduce. In real systems, locks or queues prevent it.

135. What is a deadlock?
     A deadlock happens when processes wait indefinitely for resources. It works due to circular dependencies. It is used to identify system blocking issues. A limitation is difficult detection. In real systems, proper resource ordering prevents it.

136. What is throttling?
     Throttling limits system usage to prevent overload. It works by controlling request rates. It is used for stability. A limitation is reduced performance for users. In real systems, proper limits are required.

137. What is autoscaling failure scenario?
     Autoscaling may fail due to incorrect metrics or thresholds. It works by not scaling in time. It is used to identify scaling issues. A limitation is delayed response. In real systems, testing scaling policies is important.

138. What is Kubernetes node scaling?
     Node scaling adjusts number of nodes in cluster. It works using cluster autoscaler. It is used for resource management. A limitation is scaling delay. In real systems, pre-warming nodes helps.

139. What is pod eviction?
     Pod eviction removes pods due to resource pressure. It works when node resources are low. It is used to maintain stability. A limitation is service disruption. In real systems, resource requests must be defined.

140. What is container restart policy?
     Restart policy defines when containers restart. It works using rules like Always or OnFailure. It is used for fault tolerance. A limitation is endless restart loops. In real systems, proper debugging is needed.

141. What is CI/CD security risk?
     Security risks include exposing secrets or vulnerable code. It works due to misconfiguration. It is used to identify pipeline vulnerabilities. A limitation is complex security setup. In real systems, secret scanning is important.

142. What is DevSecOps?
     DevSecOps integrates security into DevOps processes. It works by adding security checks in pipelines. It is used to detect vulnerabilities early. A limitation is increased pipeline time. In real systems, automation helps reduce impact.

143. What is vulnerability scanning?
     Vulnerability scanning detects security issues in code or images. It works using automated tools. It is used to improve security posture. A limitation is false positives. In real systems, regular scans are necessary.

144. What is firewall?
     Firewall controls network traffic based on rules. It works by filtering packets. It is used for security. A limitation is misconfiguration risks. In real systems, layered security is recommended.

145. What is identity federation?
     Identity federation allows users to access multiple systems with one identity. It works using protocols like SSO. It is used for convenience and security. A limitation is dependency on identity provider. In real systems, outages affect access.

146. What is audit logging?
     Audit logging tracks user and system actions. It works by recording events. It is used for compliance and security. A limitation is storage overhead. In real systems, logs must be protected.

147. What is compliance in DevOps?
     Compliance ensures systems meet regulations. It works by enforcing policies. It is used for legal requirements. A limitation is operational overhead. In real systems, automation helps maintain compliance.

148. What is pipeline timeout?
     Pipeline timeout stops long-running jobs. It works by setting execution limits. It is used to prevent resource waste. A limitation is incomplete tasks. In real systems, timeout values must be tuned.

149. What is debugging in production?
     Debugging in production involves analyzing live issues. It works using logs and metrics. It is used to resolve incidents. A limitation is risk of impact. In real systems, read-only debugging is preferred.

150. What is high availability architecture?
     High availability ensures system uptime through redundancy. It works using multiple instances and failover. It is used for reliability. A limitation is cost. In real systems, load balancing is key.

151. What is failover?
     Failover switches traffic to backup systems during failure. It works automatically or manually. It is used for continuity. A limitation is failover delay. In real systems, testing failover is important.

152. What is redundancy?
     Redundancy duplicates components for reliability. It works by having backups. It is used to prevent failures. A limitation is cost. In real systems, critical systems require redundancy.

153. What is service degradation?
     Service degradation reduces functionality instead of failing completely. It works by limiting features. It is used to maintain partial service. A limitation is poor user experience. In real systems, graceful degradation is useful.

154. What is circuit breaker pattern?
     Circuit breaker prevents system overload by stopping requests. It works by opening circuit on failure. It is used in microservices. A limitation is temporary blocking. In real systems, fallback mechanisms are needed.

155. What is retry mechanism?
     Retry mechanism attempts failed requests again. It works with delays or backoff. It is used for transient failures. A limitation is increased load. In real systems, retries must be limited.

156. What is exponential backoff?
     Exponential backoff increases delay between retries. It works by doubling wait time. It is used to reduce load. A limitation is slower recovery. In real systems, it prevents cascading failures.

157. What is data consistency?
     Data consistency ensures correct data across systems. It works using transactions or replication. It is used for accuracy. A limitation is performance overhead. In real systems, consistency models vary.

158. What is replication?
     Replication copies data across systems. It works for redundancy and availability. It is used for fault tolerance. A limitation is synchronization delay. In real systems, lag must be monitored.

159. What is sharding?
     Sharding splits data across multiple databases. It works by distributing load. It is used for scalability. A limitation is complex queries. In real systems, shard key selection is critical.

160. What is real-time monitoring?
     Real-time monitoring tracks systems instantly. It works by streaming metrics. It is used for quick detection. A limitation is resource usage. In real systems, alert thresholds must be set.

161. What is log level?
     Log level defines severity of logs like INFO or ERROR. It works by categorizing logs. It is used for filtering. A limitation is missing details if levels are wrong. In real systems, proper levels improve debugging.

162. What is cold start?
     Cold start is delay when starting new instances. It works due to initialization time. It is used in scaling scenarios. A limitation is latency impact. In real systems, pre-warming helps.

163. What is container lifecycle?
     Container lifecycle includes creation, running, and termination. It works through orchestration tools. It is used to manage containers. A limitation is ephemeral nature. In real systems, persistent storage is needed.

164. What is persistent volume?
     Persistent volume provides storage for containers. It works independently of pod lifecycle. It is used for stateful apps. A limitation is storage management. In real systems, backup is important.

165. What is namespace in Kubernetes?
     Namespace isolates resources within cluster. It works by grouping objects. It is used for multi-tenant environments. A limitation is complexity. In real systems, naming conventions help.

166. What is Helm?
     Helm is a package manager for Kubernetes. It works using charts to deploy apps. It is used for simplified deployments. A limitation is debugging templates. In real systems, versioning charts is important.

167. What is ingress in Kubernetes?
     Ingress manages external access to services. It works using rules and controllers. It is used for routing traffic. A limitation is configuration complexity. In real systems, TLS setup is required.

168. What is a real-world Kubernetes issue you solved?
     A service was unreachable due to wrong port mapping. It worked by debugging service and pod configs. It was fixed by correcting ports. A limitation is misconfigurations are common. In real systems, validation helps.

169. What is CI/CD best practice?
     Best practice is automation, testing, and version control. It works by defining pipelines clearly. It is used for reliability. A limitation is initial effort. In real systems, incremental improvement is key.

170. What is DevOps culture?
     DevOps culture focuses on collaboration and ownership. It works by breaking silos. It is used to improve delivery speed. A limitation is resistance to change. In real systems, leadership support is required.

171. What is root cause analysis?
     Root cause analysis identifies underlying issue. It works by investigating failures. It is used to prevent recurrence. A limitation is time-consuming. In real systems, proper documentation helps.

172. What is incident management?
     Incident management handles system failures. It works using processes and tools. It is used to restore service quickly. A limitation is coordination complexity. In real systems, runbooks help.

173. What is runbook?
     Runbook is a guide for handling incidents. It works by providing step-by-step instructions. It is used for consistency. A limitation is outdated documentation. In real systems, regular updates are needed.

174. What is escalation policy?
     Escalation policy defines who handles issues. It works by assigning responsibilities. It is used for faster resolution. A limitation is dependency on availability. In real systems, clear roles are important.

175. What is on-call support?
     On-call support handles issues outside working hours. It works by rotating responsibilities. It is used for continuous availability. A limitation is burnout risk. In real systems, proper scheduling helps.

176. What is monitoring dashboard?
     Dashboard visualizes system metrics. It works using tools like Grafana. It is used for quick insights. A limitation is cluttered dashboards. In real systems, focused metrics are better.

177. What is alert fatigue?
     Alert fatigue occurs due to excessive alerts. It works by overwhelming engineers. It is used to identify monitoring issues. A limitation is ignored alerts. In real systems, tuning alerts is critical.

178. What is service dependency?
     Service dependency is reliance between components. It works in distributed systems. It is used for architecture design. A limitation is cascading failures. In real systems, decoupling helps.

179. What is cascading failure?
     Cascading failure occurs when one failure affects others. It works due to dependencies. It is used to analyze system risks. A limitation is widespread impact. In real systems, isolation prevents this.

180. What is a real-world debugging approach?
     Debugging involves checking logs, metrics, and configs. It works by narrowing down root cause. It is used for incident resolution. A limitation is time pressure. In real systems, structured approach helps.

181. What is GitOps?
     GitOps uses Git as source of truth for deployments. It works by syncing desired state with cluster. It is used for automation. A limitation is dependency on Git workflows. In real systems, auditability improves.

182. What is drift detection in GitOps?
     Drift detection identifies differences between desired and actual state. It works by comparing configs. It is used to maintain consistency. A limitation is false positives. In real systems, automation corrects drift.

183. What is pipeline failure recovery?
     Failure recovery restarts or retries failed steps. It works using retry logic. It is used to improve reliability. A limitation is masking real issues. In real systems, proper logging is needed.

184. What is system observability challenge?
     Observability challenge is understanding complex systems. It works due to distributed nature. It is used to improve debugging. A limitation is data overload. In real systems, correlation is key.

185. What is Kubernetes upgrade challenge?
     Upgrading Kubernetes can cause compatibility issues. It works by changing cluster versions. It is used for new features. A limitation is downtime risk. In real systems, testing is required.

186. What is resource quota in Kubernetes?
     Resource quota limits resource usage in namespace. It works by enforcing limits. It is used to prevent overuse. A limitation is blocking deployments. In real systems, proper planning is needed.

187. What is pod affinity?
     Pod affinity controls pod placement based on rules. It works using labels. It is used for performance. A limitation is scheduling complexity. In real systems, careful configuration is required.

188. What is pod anti-affinity?
     Pod anti-affinity prevents pods from running together. It works by spreading workloads. It is used for high availability. A limitation is scheduling delays. In real systems, improves fault tolerance.

189. What is taints and tolerations?
     Taints restrict nodes from accepting pods. Tolerations allow pods to run on tainted nodes. It works for workload isolation. It is used for dedicated resources. A limitation is configuration complexity. In real systems, useful for special workloads.

190. What is cluster autoscaler failure?
     Failure occurs when nodes do not scale as expected. It works due to misconfiguration. It is used to debug scaling issues. A limitation is delayed response. In real systems, monitoring is required.

191. What is deployment failure in Kubernetes?
     Deployment failure happens when pods do not reach desired state. It works due to config or image issues. It is used to detect problems. A limitation is partial rollout. In real systems, rollback is needed.

192. What is log-based alerting?
     Log-based alerting triggers alerts from log patterns. It works using queries. It is used for detecting errors. A limitation is noisy alerts. In real systems, filtering is important.

193. What is metric-based alerting?
     Metric-based alerting triggers based on thresholds. It works using monitoring tools. It is used for performance tracking. A limitation is missed anomalies. In real systems, combining logs and metrics is better.

194. What is SLA breach handling?
     Handling SLA breach involves quick resolution and reporting. It works by incident management. It is used to maintain trust. A limitation is business impact. In real systems, prevention is key.

195. What is secure deployment?
     Secure deployment ensures code is safe before release. It works using scans and checks. It is used to prevent vulnerabilities. A limitation is slower pipelines. In real systems, automation balances speed.

196. What is secret leakage?
     Secret leakage exposes credentials accidentally. It works due to poor handling. It is used to identify risks. A limitation is security breach. In real systems, scanning tools help detect leaks.

197. What is audit trail?
     Audit trail records system activities. It works by logging actions. It is used for compliance. A limitation is storage cost. In real systems, retention policies are required.

198. What is system hardening?
     System hardening reduces vulnerabilities. It works by disabling unnecessary services. It is used for security. A limitation is operational effort. In real systems, regular updates are needed.

199. What is patch management?
     Patch management updates systems with fixes. It works by applying updates. It is used for security and stability. A limitation is downtime risk. In real systems, testing patches is important.

200. What is your overall approach to DevOps?
     My approach focuses on automation, monitoring, and reliability. It works by designing pipelines, managing infrastructure, and debugging issues. It is used to deliver stable software quickly. A limitation is balancing speed and quality. In real systems, continuous learning and improvement are essential.

201. What is the difference between StatefulSet and Deployment in Kubernetes?
     A StatefulSet manages stateful applications with stable identities, while a Deployment manages stateless apps. It works by assigning unique pod names and persistent storage. It is used for databases and stateful services. A trade-off is slower scaling and updates. In real systems, StatefulSets are used when data persistence is critical.

202. What is a DaemonSet in Kubernetes?
     A DaemonSet ensures a pod runs on every node in the cluster. It works by automatically scheduling pods on new nodes. It is used for logging and monitoring agents. A limitation is resource consumption on all nodes. In real systems, it is used for tools like log collectors.

203. What is an init container?
     An init container runs before the main container starts. It works by completing setup tasks like configuration. It is used to prepare environments. A trade-off is increased startup time. In real systems, it is useful for dependency checks.

204. What is etcd in Kubernetes?
     etcd is a distributed key-value store used by Kubernetes. It works by storing cluster state and configuration. It is used for coordination and consistency. A limitation is performance impact if overloaded. In real systems, backup of etcd is critical.

205. What is resource request vs limit in Kubernetes?
     Request defines minimum resources, limit defines maximum allowed usage. It works by scheduling pods based on requests. It is used for efficient resource allocation. A trade-off is incorrect values cause throttling or failures. In real systems, tuning is essential.

206. What is Kubernetes RBAC?
     RBAC controls access to Kubernetes resources. It works by defining roles and bindings. It is used for security and access management. A limitation is complex configuration. In real systems, least privilege should be followed.

207. What is Kubernetes Network Policy?
     Network policy controls pod-to-pod communication. It works by defining allowed traffic rules. It is used for security isolation. A limitation is misconfiguration can block traffic. In real systems, policies should be tested carefully.

208. What is kube-proxy?
     kube-proxy manages network routing for services. It works by handling iptables or IPVS rules. It is used for service communication. A limitation is performance overhead. In real systems, IPVS mode is preferred for scale.

209. What is CoreDNS in Kubernetes?
     CoreDNS provides DNS services inside the cluster. It works by resolving service names to IPs. It is used for service discovery. A limitation is DNS failures impact communication. In real systems, monitoring DNS is important.

210. What is ingress controller?
     Ingress controller manages external access to services. It works by routing HTTP/HTTPS traffic. It is used for centralized traffic management. A limitation is configuration complexity. In real systems, TLS setup is required.

211. What is difference between L4 and L7 load balancer?
     L4 works at transport layer, L7 works at application layer. L4 routes based on IP and port, L7 routes based on content. It is used for traffic distribution. A trade-off is L7 adds latency. In real systems, L7 is used for advanced routing.

212. How does DNS resolution work step by step?
     DNS resolution starts with local cache, then queries recursive resolvers. It works by contacting root, TLD, and authoritative servers. It is used to map domain to IP. A limitation is propagation delay. In real systems, caching improves performance.

213. What is TCP vs UDP?
     TCP is connection-oriented, UDP is connectionless. TCP ensures reliability, UDP is faster but less reliable. It is used based on application needs. A trade-off is TCP overhead vs UDP packet loss. In real systems, HTTP uses TCP.

214. What is HTTP vs HTTPS?
     HTTP is unencrypted, HTTPS uses TLS encryption. It works by securing communication. It is used for data protection. A limitation is SSL overhead. In real systems, HTTPS is mandatory.

215. What is connection pooling?
     Connection pooling reuses database connections. It works by maintaining a pool of connections. It is used to improve performance. A limitation is pool exhaustion. In real systems, proper pool size is required.

216. What is block storage vs object storage?
     Block storage provides raw disk, object storage stores data as objects. Block is used for databases, object for files. It works based on use case. A trade-off is performance vs scalability. In real systems, S3 is object storage.

217. What is file storage?
     File storage organizes data in hierarchical structure. It works like traditional file systems. It is used for shared access. A limitation is scalability issues. In real systems, NFS is commonly used.

218. How do you design CI/CD for microservices?
     CI/CD for microservices uses separate pipelines for each service. It works by independent builds and deployments. It is used for scalability and flexibility. A trade-off is pipeline complexity. In real systems, version compatibility must be managed.

219. What is artifact promotion strategy?
     Artifact promotion moves builds across environments. It works by reusing same artifact. It is used for consistency. A limitation is storage overhead. In real systems, tagging is important.

220. How do you debug pipeline stuck issue?
     Debugging involves checking logs and job dependencies. It works by identifying blocking steps. It is used to fix pipeline delays. A limitation is unclear logs. In real systems, step isolation helps.

221. What is OWASP?
     OWASP is a list of common security risks. It works by identifying vulnerabilities. It is used to secure applications. A limitation is evolving threats. In real systems, regular updates are needed.

222. What is container security best practice?
     Best practice includes minimal images and scanning. It works by reducing attack surface. It is used for secure deployments. A limitation is additional effort. In real systems, avoid running as root.

223. What is image hardening?
     Image hardening removes unnecessary components. It works by minimizing attack surface. It is used for security. A limitation is build complexity. In real systems, use official images.

224. What is system design for high traffic app?
     Design involves load balancers, scaling, and caching. It works by distributing load. It is used for performance. A limitation is cost. In real systems, CDN improves performance.

225. How do you handle 1M users traffic?
     Handling traffic involves scaling and caching. It works using load balancers and distributed systems. It is used for high availability. A trade-off is infrastructure cost. In real systems, autoscaling is key.

226. What is database replication lag?
     Replication lag is delay in syncing data. It works in distributed databases. It is used to identify consistency issues. A limitation is stale reads. In real systems, monitoring lag is important.

227. What is read replica?
     Read replica is a copy of database for read operations. It works by replicating data. It is used to reduce load. A limitation is replication delay. In real systems, writes still go to primary.

228. What is connection timeout issue?
     Timeout occurs when connection takes too long. It works due to latency or failure. It is used to detect issues. A limitation is false timeouts. In real systems, tuning timeout values is needed.

229. What is network packet loss?
     Packet loss occurs when data packets are dropped. It works due to network issues. It is used to diagnose problems. A limitation is performance degradation. In real systems, monitoring network is important.

230. What is Linux inode issue?
     Inode issue occurs when file limit is reached. It works even if disk space is available. It is used to diagnose storage problems. A limitation is hard to detect. In real systems, monitoring inode usage is important.

231. What is disk I/O bottleneck?
     Disk I/O bottleneck limits read/write performance. It works due to slow storage. It is used to identify performance issues. A limitation is hardware dependency. In real systems, SSD improves performance.

232. What is netstat command?
     netstat shows network connections. It works by listing ports and connections. It is used for debugging. A limitation is outdated in some systems. In real systems, ss command is preferred.

233. What is curl command?
     curl is used to test HTTP requests. It works by sending requests to endpoints. It is used for debugging APIs. A limitation is manual usage. In real systems, useful for quick checks.

234. What is telnet command?
     telnet tests connectivity to ports. It works by opening TCP connection. It is used for debugging network issues. A limitation is insecure protocol. In real systems, nc is preferred.

235. What is process signal in Linux?
     Signals control process behavior like kill or stop. It works using commands like kill. It is used for process management. A limitation is misuse can crash system. In real systems, SIGTERM is preferred over SIGKILL.

236. What is zombie process?
     Zombie process is a completed process not cleaned up. It works due to parent process not collecting status. It is used to identify issues. A limitation is resource waste. In real systems, fixing parent process helps.

237. What is orphan process?
     Orphan process continues after parent exits. It works by being adopted by init. It is used for process management. A limitation is unintended behavior. In real systems, proper process handling is needed.

238. What is system load average?
     Load average shows system workload. It works by measuring active processes. It is used for performance monitoring. A limitation is misinterpretation. In real systems, compare with CPU cores.

239. What is thread vs process?
     Process is independent execution, thread is lightweight unit. It works by sharing resources. It is used for concurrency. A limitation is complexity. In real systems, threads improve performance.

240. What is memory leak?
     Memory leak occurs when memory is not released. It works due to programming issues. It is used to identify performance problems. A limitation is gradual degradation. In real systems, monitoring helps detect leaks.

241. What is garbage collection?
     Garbage collection frees unused memory. It works automatically in some languages. It is used to manage memory. A limitation is performance overhead. In real systems, tuning GC improves performance.

242. What is thread pool?
     Thread pool manages multiple threads efficiently. It works by reusing threads. It is used for performance. A limitation is pool exhaustion. In real systems, proper sizing is required.

243. What is API rate limit handling?
     Handling rate limit involves retries and backoff. It works by limiting requests. It is used to prevent failures. A limitation is delayed response. In real systems, caching helps reduce calls.

244. What is microservices architecture?
     Microservices split application into small services. It works independently. It is used for scalability. A limitation is complexity. In real systems, service communication must be managed.

245. What is monolith vs microservices?
     Monolith is single application, microservices are distributed. It works differently in deployment. It is used based on scale. A limitation is migration complexity. In real systems, microservices suit large systems.

246. What is service orchestration vs choreography?
     Orchestration uses central control, choreography is event-driven. It works based on design. It is used in microservices. A limitation is complexity. In real systems, choice depends on use case.

247. What is API versioning?
     API versioning manages changes in APIs. It works by maintaining versions. It is used for backward compatibility. A limitation is maintenance overhead. In real systems, versioning strategy is important.

248. What is backward compatibility?
     Backward compatibility ensures old clients work with new systems. It works by maintaining interfaces. It is used for stability. A limitation is code complexity. In real systems, breaking changes should be avoided.

249. What is forward compatibility?
     Forward compatibility allows new clients with old systems. It works by flexible design. It is used for upgrades. A limitation is limited feature use. In real systems, rarely fully achieved.

250. What is overall advanced DevOps approach?
     Advanced DevOps focuses on scalability, security, and automation. It works by combining CI/CD, monitoring, and IaC. It is used for reliable systems. A limitation is complexity. In real systems, continuous improvement is required.

---

1. Design a CI/CD pipeline for a microservices-based application deployed on Kubernetes.
   A CI/CD pipeline for microservices involves separate pipelines per service with shared standards for build, test, and deploy stages. It works by triggering builds on commits, creating Docker images, pushing to registry, and deploying via manifests or Helm. It is used to allow independent deployments and faster releases. A trade-off is increased complexity in managing multiple pipelines. In real systems, version compatibility between services must be handled carefully.

2. How would you debug a Kubernetes pod stuck in CrashLoopBackOff?
   CrashLoopBackOff indicates repeated container failures and restarts. It works by Kubernetes retrying with backoff delay. It is used as a signal for runtime issues like config or memory errors. A limitation is logs may not persist if container crashes quickly. In real scenarios, checking logs, describe output, and resource limits helps identify root cause.

3. A deployment succeeded but the application is not accessible, how do you debug?
   This issue involves checking service, ingress, and networking layers. It works by verifying pod status, service endpoints, and port mappings. It is used to ensure connectivity between components. A limitation is multiple layers make debugging complex. In real systems, checking DNS resolution and ingress rules is critical.

4. How would you design a highly available system on AWS?
   A highly available system uses multiple AZs, load balancers, and auto-scaling groups. It works by distributing traffic and replicating services. It is used to minimize downtime. A trade-off is increased cost and complexity. In real systems, health checks and failover mechanisms are essential.

5. What happens when a node goes down in Kubernetes?
   When a node fails, pods are rescheduled to other nodes. It works through controller reconciliation. It is used for fault tolerance. A limitation is temporary downtime during rescheduling. In real systems, multi-node clusters reduce impact.

6. How do you handle secret management in CI/CD pipelines?
   Secrets are stored securely in vaults or pipeline secret stores. It works by injecting secrets at runtime. It is used to prevent exposure of credentials. A trade-off is complexity in management. In real systems, secrets should never be hardcoded.

7. How do you reduce CI/CD pipeline execution time?
   Pipeline time is reduced using caching, parallel jobs, and optimized steps. It works by eliminating redundant tasks. It is used to speed up delivery. A limitation is increased pipeline complexity. In real systems, balancing speed and reliability is important.

8. What is your approach to debugging production issues?
   Debugging involves analyzing logs, metrics, and recent changes. It works by isolating the problem step by step. It is used to restore service quickly. A limitation is time pressure and incomplete data. In real systems, having observability tools improves response.

9. How do you handle database downtime in production?
   Database downtime is handled using replication, failover, and backups. It works by switching to standby systems. It is used for high availability. A trade-off is cost and complexity. In real systems, monitoring and alerting are critical.

10. What is the difference between scaling pods and scaling nodes?
    Scaling pods increases application instances, scaling nodes increases infrastructure capacity. It works together to handle load. It is used for performance and availability. A limitation is delayed scaling response. In real systems, both must be tuned properly.

11. How would you troubleshoot high CPU usage in production?
    High CPU usage is analyzed using monitoring tools and process inspection. It works by identifying resource-heavy processes. It is used to optimize performance. A limitation is transient spikes can mislead analysis. In real systems, profiling helps identify issues.

12. What is a real incident you handled and how did you resolve it?
    An incident involved pod failures due to memory limits. It worked by analyzing logs and metrics. It was resolved by adjusting limits and fixing code. A limitation is root cause analysis takes time. In real systems, proactive monitoring helps prevent issues.

13. How do you ensure zero downtime deployment?
    Zero downtime is achieved using rolling updates or blue-green deployments. It works by gradually shifting traffic. It is used to maintain availability. A limitation is complex setup. In real systems, readiness probes are essential.

14. What is infrastructure drift and how do you handle it?
    Drift occurs when actual infrastructure differs from defined state. It works due to manual changes. It is used to identify inconsistencies. A limitation is hard detection. In real systems, enforcing IaC prevents drift.

15. How do you secure a Kubernetes cluster?
    Cluster security involves RBAC, network policies, and secure images. It works by restricting access and communication. It is used to prevent attacks. A limitation is configuration complexity. In real systems, regular audits are required.

16. How would you design monitoring for a distributed system?
    Monitoring includes logs, metrics, and tracing. It works by collecting data from all services. It is used for observability. A limitation is data overload. In real systems, correlation between signals is important.

17. What is your approach to handling alert fatigue?
    Alert fatigue is handled by tuning thresholds and reducing noise. It works by focusing on actionable alerts. It is used to improve response efficiency. A limitation is missing critical alerts. In real systems, prioritization is key.

18. How do you manage multiple environments in DevOps?
    Environments are managed using IaC and separate configurations. It works by isolating dev, staging, and prod. It is used to prevent conflicts. A limitation is duplication effort. In real systems, reusable modules help.

19. What happens if your CI/CD pipeline fails frequently?
    Frequent failures indicate unstable tests or configuration issues. It works by blocking deployments. It is used to maintain quality. A limitation is delays in delivery. In real systems, stabilizing pipelines is essential.

20. How do you handle version compatibility between microservices?
    Compatibility is managed using versioning and backward compatibility. It works by maintaining stable interfaces. It is used to avoid breaking changes. A limitation is increased maintenance. In real systems, API contracts are important.

21. How do you debug network latency issues?
    Latency issues are debugged using monitoring and tracing tools. It works by identifying slow components. It is used to improve performance. A limitation is complex root cause. In real systems, network metrics help.

22. What is your approach to cost optimization in cloud?
    Cost optimization involves rightsizing, auto-scaling, and removing unused resources. It works by analyzing usage. It is used to reduce expenses. A limitation is potential performance impact. In real systems, continuous monitoring is needed.

23. How do you handle failed deployments in production?
    Failed deployments are handled by rollback and debugging. It works by reverting to stable version. It is used to restore service. A limitation is temporary disruption. In real systems, proper testing reduces failures.

24. How do you ensure security in containerized applications?
    Security is ensured by scanning images and using minimal base images. It works by reducing vulnerabilities. It is used to protect systems. A limitation is additional effort. In real systems, regular updates are required.

25. How do you design a scalable logging system?
    A scalable logging system uses centralized storage and indexing. It works by aggregating logs. It is used for debugging and monitoring. A limitation is storage cost. In real systems, retention policies are important.

26. What is your approach to handling cascading failures?
    Cascading failures are handled using isolation and circuit breakers. It works by preventing failure propagation. It is used to maintain stability. A limitation is partial service degradation. In real systems, retries must be controlled.

27. How do you test disaster recovery plan?
    Disaster recovery is tested by simulating failures. It works by validating backup and failover. It is used to ensure readiness. A limitation is risk during testing. In real systems, regular drills are required.

28. How do you manage secrets in Kubernetes?
    Secrets are managed using Kubernetes secrets or external vaults. It works by injecting at runtime. It is used for security. A limitation is base64 encoding is not encryption. In real systems, external tools are preferred.

29. What is your approach to debugging memory leak?
    Memory leak is debugged using monitoring and profiling tools. It works by tracking memory usage. It is used to identify leaks. A limitation is hard to reproduce. In real systems, long-running monitoring helps.

30. How do you approach learning new DevOps tools quickly?
    Learning involves understanding concepts rather than tools. It works by applying existing knowledge. It is used to adapt quickly. A limitation is initial learning curve. In real systems, hands-on practice is essential.

---



