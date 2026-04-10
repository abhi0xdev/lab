1. What is Azure DevOps and how is it used in CI/CD?
   Azure DevOps is a platform that provides services like repositories, pipelines, and artifacts to manage the software lifecycle. It works by integrating code repositories with automated pipelines that build, test, and deploy applications. It is used to standardize delivery and reduce manual intervention in deployments. A limitation is that pipeline complexity increases with scale and requires careful management. In real projects, misconfigured service connections often break deployments.

2. What is GitHub Actions and how does it compare with Azure DevOps?
   GitHub Actions is a CI/CD tool integrated directly into GitHub repositories for automating workflows. It works by defining workflows in YAML files that trigger on events like commits or pull requests. It is used for tight integration with code hosting and simpler pipeline setups. A trade-off is that enterprise-level governance is weaker compared to Azure DevOps. In real scenarios, improper secret handling can expose credentials.

3. What is a CI/CD pipeline and how does it work?
   A CI/CD pipeline automates building, testing, and deploying applications. It works by triggering steps like code compilation, testing, and deployment based on code changes. It is used to ensure consistent and fast delivery of software. A limitation is that poorly written pipelines can slow down development instead of improving it. In practice, flaky tests can cause pipeline instability.

4. What is Azure Kubernetes Service (AKS)?
   AKS is a managed Kubernetes service in Azure used to deploy and manage containerized applications. It works by abstracting control plane management while allowing users to manage worker nodes and workloads. It is used to reduce operational overhead of managing Kubernetes clusters. A trade-off is limited control over certain Kubernetes configurations. In real cases, node pool misconfiguration can lead to resource exhaustion.

5. How does Kubernetes scheduling work in AKS?
   Kubernetes scheduling assigns pods to nodes based on resource availability and constraints. It works by evaluating CPU, memory, and affinity rules before placing pods. It is used to efficiently utilize cluster resources. A limitation is that scheduling decisions may not always be optimal for custom workloads. In production, incorrect resource requests can lead to pod starvation.

6. What is Docker and why is it used?
   Docker is a containerization platform that packages applications with dependencies into containers. It works by using images to create isolated runtime environments. It is used for consistency across development and production. A limitation is that large images increase deployment time. In real use, improper base image selection can cause security vulnerabilities.

7. What is a Dockerfile and how does it work?
   A Dockerfile defines instructions to build a container image. It works by executing steps like copying files, installing dependencies, and defining entry points. It is used to standardize container builds. A limitation is that inefficient layering can increase image size. In practice, not using multi-stage builds leads to bloated images.

8. What is Azure Container Registry (ACR)?
   ACR is a private registry for storing Docker images in Azure. It works by allowing secure push and pull of images integrated with Azure services. It is used to manage container images centrally. A limitation is cost increases with storage and usage. In real scenarios, improper access control can expose images.

9. How does authentication work between AKS and ACR?
   AKS authenticates with ACR using managed identities or service principals. It works by granting pull permissions to the cluster identity. It is used to securely fetch images during deployment. A limitation is misconfigured roles can block deployments. In real environments, expired credentials often cause image pull errors.

10. What is Helm and why is it used?
    Helm is a package manager for Kubernetes that simplifies application deployment. It works by using charts that define Kubernetes resources. It is used to manage complex deployments with reusable templates. A limitation is debugging Helm templates can be difficult. In practice, incorrect values files can break deployments.

11. What is a Kubernetes Deployment?
    A Deployment is a resource that manages replica sets and ensures desired pod state. It works by maintaining the specified number of pod replicas. It is used for rolling updates and scaling. A limitation is it does not handle stateful workloads well. In real use, incorrect rollout strategy can cause downtime.

12. What is a Kubernetes Service?
    A Service exposes pods as a network service. It works by routing traffic to pod IPs using selectors. It is used for stable communication between components. A limitation is it does not handle advanced routing by itself. In real scenarios, misconfigured selectors can route traffic incorrectly.

13. What is an Ingress in Kubernetes?
    Ingress manages external access to services in a cluster. It works by routing HTTP/HTTPS traffic based on rules. It is used to expose services with load balancing. A limitation is it requires an Ingress controller. In production, incorrect rules can cause routing failures.

14. What is Azure Application Gateway?
    Application Gateway is a web traffic load balancer in Azure. It works by routing HTTP requests based on URL paths and hostnames. It is used for Layer 7 routing and SSL termination. A limitation is higher cost compared to basic load balancers. In real cases, improper health probes cause backend failures.

15. What is Terraform and why is it used?
    Terraform is an Infrastructure as Code tool used to provision cloud resources. It works by defining infrastructure in declarative configuration files. It is used for repeatable and version-controlled infrastructure. A limitation is state file management complexity. In practice, state file corruption can break deployments.

16. What is Terraform state and why is it important?
    Terraform state stores information about managed infrastructure. It works by mapping resources to real-world cloud objects. It is used to track changes and dependencies. A limitation is sensitive data may be stored in state. In real projects, not using remote state can cause conflicts.

17. What is Azure Key Vault?
    Azure Key Vault stores secrets, keys, and certificates securely. It works by providing controlled access using identities. It is used to protect sensitive data. A limitation is latency when accessing secrets frequently. In practice, missing access policies can block applications.

18. What are private endpoints in Azure?
    Private endpoints allow secure access to Azure services over a private network. They work by assigning private IPs to services. They are used to avoid public exposure. A limitation is increased network complexity. In real scenarios, DNS misconfiguration can break connectivity.

19. What is SAST and DAST in CI/CD?
    SAST analyzes source code for vulnerabilities, while DAST tests running applications. They work by scanning for security issues during development and runtime. They are used to improve application security. A limitation is false positives. In real pipelines, ignoring findings can lead to breaches.

20. What is a Kubernetes pod failure and how do you troubleshoot it?
    A pod failure occurs when a container fails to start or crashes. It works by showing status like CrashLoopBackOff. It is used to identify runtime issues. A limitation is logs may not always be clear. In real scenarios, checking logs and events is the first step.

21. How do you troubleshoot a CrashLoopBackOff error in Kubernetes?
    CrashLoopBackOff indicates a container repeatedly failing to start. It works by restarting the container until a backoff limit is reached. It is used to detect unstable applications or misconfigurations. A limitation is logs may not clearly show root cause if the container exits quickly. In real scenarios, I check container logs, environment variables, and entrypoint commands.

22. What happens if a pod is stuck in Pending state?
    A pod in Pending state means it cannot be scheduled on any node. It works by waiting for resources or constraints like node selectors to be satisfied. It is used to signal scheduling issues. A limitation is it does not specify exact reasons without deeper inspection. In real cases, I check node capacity, taints, and events.

23. How do you perform an AKS cluster upgrade?
    AKS upgrade updates Kubernetes version and node pools. It works by upgrading control plane first and then nodes in a rolling manner. It is used to get new features and security patches. A limitation is potential downtime if not handled carefully. In real scenarios, I test upgrades in staging before production.

24. What is a rolling deployment in Kubernetes?
    Rolling deployment updates pods gradually without downtime. It works by replacing old pods with new ones incrementally. It is used for zero-downtime releases. A limitation is issues may spread gradually if not monitored. In real cases, I set proper readiness probes to avoid traffic to unhealthy pods.

25. How do readiness and liveness probes work?
    Readiness probes check if a pod is ready to serve traffic, while liveness probes check if it should be restarted. They work by sending HTTP or command checks. They are used to maintain application health. A limitation is misconfiguration can cause unnecessary restarts. In real use, aggressive probe timing can destabilize apps.

26. What is a scenario where image pull fails in AKS?
    Image pull failure happens when Kubernetes cannot fetch container images. It works by throwing ImagePullBackOff errors. It is used to signal authentication or image issues. A limitation is error messages may be generic. In real scenarios, incorrect ACR permissions or wrong image tags cause failures.

27. How do you secure secrets in Kubernetes?
    Secrets store sensitive data like passwords and tokens. They work by encoding values and injecting them into pods. They are used to separate configuration from code. A limitation is base64 encoding is not encryption. In real environments, I integrate with Azure Key Vault for better security.

28. What is Infrastructure as Code and why is it important?
    Infrastructure as Code defines infrastructure using code instead of manual setup. It works by provisioning resources through scripts. It is used for consistency and automation. A limitation is debugging scripts can be complex. In real cases, version control helps track changes.

29. How does Ansible work for automation?
    Ansible automates tasks using playbooks written in YAML. It works by connecting to systems over SSH and executing modules. It is used for configuration management and deployments. A limitation is it may be slower for large-scale operations. In real use, idempotency ensures repeated runs are safe.

30. What is a scenario where CI/CD pipeline fails during deployment?
    Pipeline failures occur due to misconfigurations or code issues. It works by stopping execution at failed steps. It is used to prevent faulty deployments. A limitation is debugging multi-stage pipelines can be complex. In real scenarios, incorrect environment variables often cause failures.

31. What is blue-green deployment?
    Blue-green deployment uses two environments for deployment. It works by switching traffic from old to new environment after validation. It is used to reduce downtime and risk. A limitation is higher infrastructure cost. In real scenarios, database compatibility must be ensured.

32. What is canary deployment?
    Canary deployment releases changes to a small subset of users. It works by gradually increasing traffic to the new version. It is used to minimize impact of failures. A limitation is requires monitoring and traffic control. In real cases, improper metrics can lead to unnoticed failures.

33. How does Git branching strategy impact CI/CD?
    Branching strategy defines how code changes are managed. It works by controlling merge and release workflows. It is used to maintain code quality and stability. A limitation is complex strategies slow down development. In real scenarios, GitFlow may be overkill for small teams.

34. What is a pull request and why is it important?
    A pull request is a request to merge code changes. It works by enabling code review before merging. It is used to maintain code quality. A limitation is delays if reviewers are unavailable. In real cases, lack of proper review leads to bugs in production.

35. What is kubectl and how is it used?
    kubectl is a command-line tool to interact with Kubernetes. It works by sending commands to the API server. It is used to manage resources and debug issues. A limitation is steep learning curve. In real use, incorrect commands can affect production workloads.

36. What is a node pool in AKS?
    Node pool is a group of nodes with same configuration. It works by grouping workloads based on requirements. It is used for scaling and workload isolation. A limitation is managing multiple pools increases complexity. In real scenarios, improper sizing leads to cost issues.

37. What is autoscaling in AKS?
    Autoscaling adjusts resources based on demand. It works by increasing or decreasing nodes or pods. It is used to optimize cost and performance. A limitation is scaling delay can impact performance. In real cases, incorrect thresholds cause unnecessary scaling.

38. What is Azure Managed Identity?
    Managed Identity provides identity for Azure resources. It works by eliminating the need for storing credentials. It is used for secure authentication. A limitation is limited cross-cloud usage. In real scenarios, missing role assignments cause failures.

39. What is network policy in Kubernetes?
    Network policy controls traffic between pods. It works by defining rules for ingress and egress. It is used to enhance security. A limitation is requires compatible network plugin. In real cases, misconfigured policies block valid traffic.

40. What is a real scenario where AKS deployment fails after upgrade?
    Deployment may fail due to API deprecations or version incompatibility. It works by rejecting outdated configurations. It is used to enforce newer standards. A limitation is backward compatibility issues. In real cases, deprecated APIs must be updated before upgrade.

41. How do you optimize Docker image size?
    Image size optimization reduces build and deployment time. It works by using smaller base images and multi-stage builds. It is used to improve efficiency. A limitation is debugging becomes harder. In real scenarios, removing unnecessary packages is key.

42. What is a scenario where Terraform apply fails?
    Terraform apply fails due to configuration or dependency issues. It works by halting execution on errors. It is used to prevent inconsistent infrastructure. A limitation is error messages may not be clear. In real cases, incorrect variables or missing permissions cause failures.

43. What is state locking in Terraform?
    State locking prevents concurrent modifications of infrastructure. It works by locking state during execution. It is used to avoid conflicts. A limitation is locking issues can block deployments. In real scenarios, backend like Azure Storage is used for locking.

44. What is ARM template vs Bicep?
    ARM templates are JSON-based IaC scripts, while Bicep is a simplified syntax. They work by defining Azure resources declaratively. They are used for automation. A limitation is ARM templates are verbose. In real use, Bicep improves readability.

45. How do you debug a failing Kubernetes service?
    Service failure affects communication between components. It works by routing traffic incorrectly. It is used to expose applications. A limitation is issues may be network-related. In real scenarios, checking endpoints and selectors helps.

46. What is a scenario where DNS fails in AKS?
    DNS failure prevents service discovery. It works by failing to resolve service names. It is used for internal communication. A limitation is debugging is complex. In real cases, CoreDNS issues or misconfiguration cause failures.

47. What is resource request and limit in Kubernetes?
    Requests define minimum resources, limits define maximum. They work by guiding scheduler and enforcing constraints. They are used for resource management. A limitation is incorrect values cause instability. In real scenarios, overcommitting leads to throttling.

48. How do you secure CI/CD pipelines?
    Pipeline security ensures safe code delivery. It works by using secrets management and access control. It is used to prevent breaches. A limitation is complexity increases. In real cases, storing secrets in plain text is a major risk.

49. What is a real-world scenario for high latency in AKS?
    High latency occurs due to resource contention or network issues. It works by slowing down response times. It is used to identify performance problems. A limitation is root cause may be distributed. In real scenarios, monitoring tools help identify bottlenecks.

50. How do you monitor AKS clusters?
    Monitoring tracks performance and health. It works using tools like Azure Monitor and Prometheus. It is used for proactive issue detection. A limitation is alert fatigue. In real cases, proper alert thresholds are essential.

51. What is a scenario where a Kubernetes pod is running but application is not accessible?
    This happens when the pod is healthy but networking or service configuration is incorrect. It works by having the container running but traffic not reaching it. It is used to identify issues outside container execution. A limitation is health checks may still pass even if app is unreachable. In real cases, wrong service port or ingress rule is the root cause.

52. How do you troubleshoot high CPU usage in AKS?
    High CPU usage indicates resource pressure on pods or nodes. It works by monitoring metrics and identifying top consumers. It is used to prevent performance degradation. A limitation is temporary spikes may mislead analysis. In real scenarios, I check HPA configuration and optimize application code.

53. What is Horizontal Pod Autoscaler (HPA)?
    HPA automatically scales pods based on metrics like CPU or memory. It works by adjusting replica count dynamically. It is used to handle varying workloads. A limitation is scaling delay can affect performance. In real cases, incorrect thresholds lead to over-scaling.

54. What is Vertical Pod Autoscaler (VPA)?
    VPA adjusts resource requests and limits of containers. It works by analyzing usage and recommending or applying changes. It is used to optimize resource allocation. A limitation is it may restart pods during updates. In real scenarios, it is not used with HPA for same resource.

55. What is a scenario where ingress is not routing traffic?
    Ingress issues occur when rules or controller are misconfigured. It works by failing to match host or path rules. It is used to manage external access. A limitation is debugging requires checking multiple layers. In real cases, missing annotations or wrong backend service cause failure.

56. How do you handle secrets in CI/CD pipelines?
    Secrets are managed securely using vaults or secret managers. They work by injecting values during runtime. They are used to protect sensitive information. A limitation is improper access control can leak secrets. In real scenarios, using Azure Key Vault integration is best practice.

57. What is a scenario where Helm deployment fails?
    Helm deployment fails due to template errors or invalid values. It works by failing during chart rendering or resource creation. It is used to detect configuration issues early. A limitation is error messages may be hard to interpret. In real cases, incorrect values.yaml causes failure.

58. What is Git rebase and when do you use it?
    Git rebase rewrites commit history by applying changes on top of another branch. It works by creating a linear history. It is used to keep commit history clean. A limitation is it can cause conflicts and rewrite shared history. In real scenarios, avoid rebasing public branches.

59. What is Git merge conflict and how do you resolve it?
    Merge conflict occurs when changes overlap in the same file. It works by requiring manual resolution. It is used to ensure correct code integration. A limitation is conflicts can be complex. In real cases, reviewing differences carefully avoids breaking code.

60. What is a scenario where pipeline build passes but deployment fails?
    Build success does not guarantee deployment success. It works by separating build and deploy stages. It is used to validate code independently. A limitation is environment differences cause failures. In real scenarios, missing runtime dependencies cause deployment issues.

61. What is Azure Virtual Network (VNet)?
    VNet is a private network in Azure for resources. It works by isolating resources and controlling traffic. It is used for secure communication. A limitation is complex configuration. In real cases, subnet misconfiguration can block connectivity.

62. What is subnetting in Azure?
    Subnetting divides a VNet into smaller networks. It works by allocating IP ranges. It is used for organization and security. A limitation is IP exhaustion if not planned. In real scenarios, overlapping ranges cause issues.

63. What is Network Security Group (NSG)?
    NSG controls inbound and outbound traffic. It works by defining rules based on IP and ports. It is used to secure resources. A limitation is rule conflicts can occur. In real cases, wrong priority blocks traffic.

64. What is a scenario where AKS nodes are not joining cluster?
    Nodes fail to join due to networking or identity issues. It works by failing during node registration. It is used to detect infrastructure problems. A limitation is logs may not be clear. In real cases, misconfigured subnet or permissions cause issue.

65. What is Azure Load Balancer?
    Load Balancer distributes traffic across instances. It works at Layer 4 using IP and port. It is used for high availability. A limitation is limited routing capabilities. In real scenarios, health probe misconfiguration causes downtime.

66. What is a scenario where Terraform destroy fails?
    Destroy fails due to dependencies or locked resources. It works by not being able to delete resources in order. It is used to protect infrastructure integrity. A limitation is manual cleanup may be required. In real cases, orphaned resources cause issues.

67. What is drift in Infrastructure as Code?
    Drift occurs when actual infrastructure differs from code. It works by manual changes outside IaC. It is used to detect inconsistencies. A limitation is detection requires regular checks. In real scenarios, drift causes unexpected behavior.

68. How do you handle rollback in CI/CD?
    Rollback reverts to previous stable version. It works by redeploying older artifacts. It is used to recover from failures. A limitation is data compatibility issues. In real cases, versioning strategy is critical.

69. What is a scenario where Kubernetes node becomes NotReady?
    Node becomes NotReady due to resource or network issues. It works by marking node unhealthy. It is used to prevent scheduling. A limitation is workloads may be disrupted. In real scenarios, kubelet or network failure is common cause.

70. What is logging in AKS and how is it handled?
    Logging captures application and system logs. It works by sending logs to monitoring systems. It is used for debugging and auditing. A limitation is high storage cost. In real cases, log retention policies are important.

71. What is a sidecar container?
    Sidecar is a helper container running alongside main container. It works by sharing resources within pod. It is used for logging, monitoring, or proxying. A limitation is increased resource usage. In real scenarios, sidecar misconfiguration affects main app.

72. What is init container?
    Init container runs before main container starts. It works by preparing environment. It is used for setup tasks. A limitation is it delays startup. In real cases, failure blocks pod startup.

73. What is a scenario where service is exposed but still not reachable externally?
    This occurs due to firewall or ingress issues. It works by blocking traffic at network level. It is used to identify external connectivity issues. A limitation is multiple layers need checking. In real cases, NSG rules or DNS misconfiguration cause issue.

74. What is Kubernetes ConfigMap?
    ConfigMap stores non-sensitive configuration data. It works by injecting values into pods. It is used to separate config from code. A limitation is not secure for secrets. In real scenarios, incorrect config leads to app failure.

75. What is a scenario where application crashes after deployment?
    Crash happens due to runtime issues or config errors. It works by failing during execution. It is used to detect deployment problems. A limitation is logs may not be sufficient. In real cases, environment mismatch is common.

76. What is immutable infrastructure?
    Immutable infrastructure means not modifying resources after creation. It works by replacing instead of updating. It is used for consistency. A limitation is higher resource usage. In real scenarios, rollback becomes easier.

77. What is idempotency in DevOps?
    Idempotency ensures same result on repeated execution. It works by avoiding unintended changes. It is used for reliable automation. A limitation is harder to implement. In real cases, Ansible uses idempotent tasks.

78. What is a scenario where autoscaling does not work?
    Autoscaling fails due to metric or config issues. It works by not triggering scale events. It is used to detect misconfiguration. A limitation is debugging metrics is complex. In real cases, missing metrics server causes issue.

79. What is Azure Bastion?
    Azure Bastion provides secure RDP/SSH access. It works through browser without exposing ports. It is used for secure access. A limitation is cost. In real scenarios, it avoids opening public ports.

80. What is a scenario where container works locally but fails in AKS?
    This happens due to environment differences. It works by failing in cluster environment. It is used to detect dependency issues. A limitation is debugging requires logs. In real cases, missing environment variables cause failure.

81. What is dependency management in CI/CD?
    Dependency management handles external libraries. It works by installing required packages during build. It is used to ensure consistency. A limitation is version conflicts. In real scenarios, lock files prevent issues.

82. What is artifact in CI/CD?
    Artifact is output of build process. It works by storing compiled code or images. It is used for deployment. A limitation is storage cost. In real cases, versioning artifacts is important.

83. What is a scenario where AKS pod is evicted?
    Pod eviction happens due to resource pressure. It works by removing pods to free resources. It is used to maintain node stability. A limitation is service disruption. In real cases, insufficient memory causes eviction.

84. What is Azure Policy?
    Azure Policy enforces governance rules. It works by auditing and restricting configurations. It is used for compliance. A limitation is complexity in setup. In real cases, policy violations block deployments.

85. What is a scenario where Helm rollback fails?
    Rollback fails due to missing revision or dependency issues. It works by not restoring previous state. It is used to detect release problems. A limitation is partial rollback risk. In real scenarios, corrupted release history causes issue.

86. What is infrastructure scaling strategy?
    Scaling strategy defines how resources grow. It works by horizontal or vertical scaling. It is used for performance. A limitation is cost trade-off. In real cases, horizontal scaling is preferred for resilience.

87. What is service mesh?
    Service mesh manages communication between services. It works by using proxies like Istio. It is used for observability and security. A limitation is added complexity. In real scenarios, it improves traffic control.

88. What is a scenario where network latency increases suddenly?
    Latency increases due to congestion or misrouting. It works by slowing communication. It is used to detect network issues. A limitation is root cause identification is hard. In real cases, cross-region traffic causes latency.

89. What is chaos engineering?
    Chaos engineering tests system resilience. It works by injecting failures. It is used to improve reliability. A limitation is risk if not controlled. In real scenarios, it helps identify weak points.

90. What is blue-green vs canary deployment difference?
    Blue-green switches entire traffic at once, canary shifts gradually. They work by different traffic strategies. They are used for safe deployments. A limitation is complexity in canary. In real cases, canary is preferred for critical apps.

91. What is API versioning issue in AKS upgrades?
    API versioning issues occur when deprecated APIs are removed. It works by rejecting old manifests. It is used to enforce updates. A limitation is migration effort. In real scenarios, pre-upgrade checks are required.

92. What is a scenario where Key Vault access fails?
    Access fails due to permission or network issues. It works by denying secret retrieval. It is used to detect security issues. A limitation is debugging identity issues is complex. In real cases, missing role assignment causes failure.

93. What is zero-downtime deployment?
    Zero-downtime ensures no service interruption. It works using rolling or blue-green strategies. It is used for high availability. A limitation is complex setup. In real scenarios, proper health checks are critical.

94. What is pipeline caching?
    Caching stores dependencies to speed up builds. It works by reusing previous downloads. It is used to improve performance. A limitation is cache invalidation complexity. In real cases, stale cache causes build issues.

95. What is a scenario where GitHub Actions workflow fails randomly?
    Random failures occur due to environment or network issues. It works by failing intermittently. It is used to detect instability. A limitation is hard to reproduce. In real cases, flaky tests are common cause.

96. What is container security scanning?
    Security scanning checks images for vulnerabilities. It works by analyzing dependencies. It is used to prevent attacks. A limitation is false positives. In real scenarios, integrating with pipeline is best practice.

97. What is rate limiting in Application Gateway?
    Rate limiting controls request rate. It works by restricting traffic. It is used to prevent overload. A limitation is legitimate traffic may be blocked. In real cases, tuning thresholds is important.

98. What is a scenario where Terraform plan shows unexpected changes?
    Unexpected changes occur due to drift or config errors. It works by detecting differences. It is used to ensure consistency. A limitation is false positives possible. In real scenarios, manual changes cause drift.

99. What is observability in DevOps?
    Observability provides insight into system behavior. It works using logs, metrics, and traces. It is used for monitoring and debugging. A limitation is data overload. In real cases, proper dashboards are needed.

100. What is best practice for production-ready DevOps system?
     A production-ready system ensures reliability, scalability, and security. It works by combining automation, monitoring, and testing. It is used to deliver stable applications. A limitation is high complexity. In real scenarios, continuous improvement is essential.

---
