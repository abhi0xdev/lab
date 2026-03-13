How do you design a multi-environment CI/CD pipeline in Jenkins?
A multi-environment pipeline is designed with stages that promote builds from dev to staging and then production. I usually parameterize environment variables and use conditional stages with approval gates before production deployment. This works by using the same artifact across environments to maintain consistency. It is used to ensure controlled releases and traceability. A limitation is increased pipeline complexity. In real scenarios, environment-specific configuration and secrets must be handled securely.

Scripted vs Declarative pipeline, when do you choose which?
Declarative pipelines use a structured syntax that is easier to read and maintain, while scripted pipelines provide full Groovy flexibility for complex logic. Declarative is usually chosen for standard CI/CD workflows, while scripted is used when custom logic or dynamic pipeline generation is needed. This works by balancing maintainability and flexibility. It is used to standardize pipelines across teams. A limitation is that scripted pipelines can become difficult to debug. In real scenarios, most teams prefer declarative with small scripted blocks when necessary.

How do you secure Jenkins credentials and users?
I secure Jenkins by storing secrets in the Jenkins credentials store and restricting access using role-based access control. I also integrate Jenkins with external authentication systems like LDAP or SSO. This works by ensuring credentials are injected at runtime rather than stored in code. It is used to protect sensitive information like API keys and passwords. A limitation is that improper permission configuration can expose credentials. In real scenarios, rotating credentials and auditing access is important.

How do you trigger Jenkins automatically using Git webhooks?
I configure a webhook in the Git repository that sends HTTP POST requests to Jenkins whenever a commit or pull request occurs. Jenkins is configured with a webhook endpoint that triggers the pipeline job. This works by enabling event-driven automation instead of manual builds. It is used to run CI pipelines immediately after code changes. A limitation is that webhook misconfiguration can stop triggers. In real scenarios, checking webhook delivery logs helps debug issues.

How do you clean the workspace and handle pipeline failures?
I use Jenkins workspace cleanup plugins or pipeline steps like deleteDir to remove leftover files before builds. For failures, I implement post conditions that send alerts or trigger rollback processes. This works by ensuring clean builds and proper failure handling. It is used to avoid dependency conflicts between builds. A limitation is accidental deletion of useful artifacts. In real scenarios, artifact storage systems are used for persistence.

How will you auto-scale an application when traffic spikes?
I would use an Auto Scaling Group with policies based on metrics like CPU utilization or request count. These policies automatically add or remove instances as demand changes. This works by monitoring metrics and adjusting capacity dynamically. It is used to maintain performance during traffic spikes. A limitation is scaling delay during sudden spikes. In real scenarios, combining predictive scaling with load testing improves reliability.

Difference between ALB, NLB, and CLB and when to use them?
Application Load Balancer works at layer 7 and supports HTTP and HTTPS routing based on paths and hostnames. Network Load Balancer works at layer 4 and handles high-performance TCP or UDP traffic with low latency. Classic Load Balancer is the older generation and supports basic load balancing features. They are used depending on application protocol and performance needs. A limitation is configuration complexity. In real scenarios, ALB is preferred for microservices architectures.

How will you secure an S3 bucket for both public and private access?
I would use bucket policies and IAM roles to control access while enabling public access only for required objects. I would also enable encryption and block public access by default. This works by enforcing security policies on storage resources. It is used to protect sensitive data while allowing controlled access. A limitation is misconfigured policies exposing data. In real scenarios, enabling access logging and monitoring is important.

Your EC2 instance becomes unreachable. How will you troubleshoot?
I would first check security groups and network ACLs to confirm SSH access is allowed. Then I would verify instance status checks and inspect system logs from the AWS console. This works by isolating network and system-level issues. It is used to quickly identify connectivity problems. A limitation is limited access if the OS is completely unresponsive. In real scenarios, using AWS Systems Manager helps recover access.

How do you implement Blue-Green deployment using AWS services?
I create two identical environments, one active and one idle, and deploy new changes to the idle environment. After testing, traffic is switched using a load balancer or DNS update. This works by minimizing downtime and allowing quick rollback. It is used for safe production deployments. A limitation is higher infrastructure cost. In real scenarios, automated health checks validate the new environment before switching.

Explain Docker multi-stage builds with an example.
Multi-stage builds use multiple build stages within a Dockerfile to reduce final image size. The first stage compiles the application and the final stage copies only required artifacts. This works by separating build dependencies from runtime environment. It is used to create lightweight production images. A limitation is complexity in debugging build steps. In real scenarios, smaller images improve security and performance.

How do you troubleshoot high CPU usage in a container?
I check container resource usage using commands like docker stats and inspect running processes inside the container. I also analyze application logs and configuration. This works by identifying resource-intensive processes. It is used to diagnose performance issues. A limitation is limited visibility without monitoring tools. In real scenarios, setting CPU limits prevents resource exhaustion.

Best practices for production-ready Dockerfiles?
I use minimal base images, avoid running containers as root, and reduce layers by combining commands. I also use multi-stage builds and proper caching. This works by improving security and performance. It is used to maintain efficient container images. A limitation is compatibility issues with minimal images. In real scenarios, scanning images for vulnerabilities is essential.

How do you handle environment variables securely in containers?
I store sensitive variables in secret management systems rather than hardcoding them. They are injected at runtime through environment variables or secret volumes. This works by keeping secrets separate from application code. It is used to secure credentials. A limitation is secret management overhead. In real scenarios, rotating secrets regularly is important.

What happens internally when you run docker run?
When docker run executes, Docker checks if the image exists locally and pulls it if necessary. It then creates a container, sets up namespaces and cgroups, and starts the process defined in the image. This works by isolating processes using containerization features of the Linux kernel. It is used to run applications consistently across environments. A limitation is dependency on the host kernel. In real scenarios, proper resource limits prevent container abuse.

Pod in CrashLoopBackOff error, how do you debug?
I check pod logs using kubectl logs and describe the pod to review events and configuration issues. I verify environment variables, resource limits, and application errors. This works by identifying repeated container failures. It is used to diagnose application startup problems. A limitation is incomplete logs if the container crashes immediately. In real scenarios, checking previous logs helps identify root cause.

Deployment vs StatefulSet vs DaemonSet real use cases?
Deployment is used for stateless applications with scalable replicas like web services. StatefulSet is used for stateful applications requiring stable identity and storage like databases. DaemonSet ensures one pod runs on each node for tasks like monitoring agents. These controllers manage pods according to workload requirements. A limitation is complexity in managing stateful workloads. In real scenarios, correct controller choice improves reliability.

How do you expose a service externally in Kubernetes?
I expose services using types like NodePort, LoadBalancer, or Ingress depending on requirements. NodePort exposes services through node IPs, while LoadBalancer integrates with cloud load balancers. Ingress provides HTTP routing and TLS termination. This works by routing external traffic into the cluster. It is used to make applications accessible. A limitation is misconfiguration leading to security risks.

How do ConfigMaps and Secrets work?
ConfigMaps store non-sensitive configuration data while Secrets store sensitive data like passwords. They can be injected into pods as environment variables or mounted as files. This works by separating configuration from application code. It is used to simplify configuration management. A limitation is limited size and security considerations. In real scenarios, secrets should be encrypted and access controlled.

RollingUpdate vs Recreate strategy, when to use which?
RollingUpdate gradually replaces old pods with new ones to maintain availability. Recreate stops all old pods before starting new ones. RollingUpdate is used for zero-downtime deployments while Recreate is used when applications cannot run multiple versions simultaneously. This works by controlling how updates are applied. A limitation is slower updates with rolling strategy. In real scenarios, readiness probes ensure safe rollout.

How do you manage Terraform state in teams?
I store Terraform state in a remote backend like S3 with state locking enabled using DynamoDB. This works by preventing concurrent modifications and ensuring consistency. It is used for collaboration in teams. A limitation is dependency on backend availability. In real scenarios, proper access control to state storage is important.

What happens internally during Terraform apply?
Terraform first reads the current state and compares it with the desired configuration. It generates an execution plan and then performs resource creation, update, or deletion accordingly. This works by maintaining infrastructure as code consistency. It is used to automate infrastructure changes. A limitation is risk of unintended changes if configuration is incorrect.

Explain Terraform modules with a scenario.
Terraform modules are reusable blocks of infrastructure configuration. For example, I can create a module for deploying EC2 instances and reuse it across environments. This works by parameterizing variables and outputs. It is used to maintain consistency and reduce duplication. A limitation is module complexity in large systems.

How do you prevent accidental deletion of resources in Terraform?
I use lifecycle rules like prevent_destroy in resource configuration. This works by blocking destructive operations unless explicitly overridden. It is used to protect critical infrastructure. A limitation is that it may block legitimate changes. In real scenarios, approvals and reviews are used before applying changes.

How do you fix drift when changes are done manually in AWS?
I run terraform plan to detect differences between the current state and configuration. I either update Terraform configuration or import resources into state. This works by reconciling infrastructure with code. It is used to maintain infrastructure consistency. A limitation is complexity in large environments. In real scenarios, manual changes should be restricted to avoid drift.

---

What is the difference between Git pull, Git fetch, and Git merge?
Git fetch downloads changes from the remote repository but does not modify the working branch. Git merge combines changes from another branch into the current branch. Git pull is essentially a combination of fetch followed by merge to update the local branch. These commands work by synchronizing local repositories with remote updates. They are used to keep development environments up to date. A limitation is that pull may cause unexpected merge conflicts if changes are not reviewed first.

How do you configure Maven in Jenkins?
I configure Maven in Jenkins by installing it on the Jenkins server and adding its path under Global Tool Configuration. In the pipeline, I reference the Maven installation and use commands like mvn clean install for builds. This works by integrating Jenkins with build automation tools. It is used to compile and package Java applications. A limitation is dependency resolution failures if repositories are not accessible.

How do you configure SonarQube in Jenkins pipelines?
I configure SonarQube by adding the SonarQube server in Jenkins system settings and installing the SonarQube plugin. In the pipeline, I add a stage that runs the Sonar scanner using mvn sonar:sonar or sonar-scanner commands. This works by sending code analysis results to SonarQube. It is used to detect bugs and security vulnerabilities. A limitation is that analysis increases pipeline execution time.

How do you write Jenkins pipeline stages for Build, Test, and Deploy?
I define stages in a Jenkinsfile where the build stage compiles the code, the test stage runs unit tests, and the deploy stage deploys the application. Each stage contains commands or scripts required for that step. This works by breaking the pipeline into manageable phases. It is used to automate the software delivery lifecycle. A limitation is debugging complex pipelines can take time.

How do you manage Jenkins parameters and credentials?
Jenkins parameters allow users to provide input values like environment or version during pipeline execution. Credentials are stored securely in Jenkins credentials manager and injected into pipelines when required. This works by separating configuration and secrets from code. It is used to protect sensitive information like API keys. A limitation is incorrect permissions may expose credentials.

How do you handle pipeline failures due to SonarQube quality gates?
When the quality gate fails due to low code coverage or vulnerabilities, the pipeline stops automatically. I review the SonarQube report and work with developers to fix issues before rerunning the pipeline. This works by enforcing code quality standards. It is used to prevent poor-quality code from reaching production. A limitation is that strict gates may slow development.

How is a WAR file generated and where are artifacts stored?
A WAR file is generated during the Maven build process using commands like mvn package. Jenkins stores the generated artifact in the workspace or archives it using the artifact storage feature. This works by packaging application code and dependencies into a deployable file. It is used for deploying Java web applications. A limitation is large artifacts increase storage usage.

What is the Jenkins upgrade process?
The Jenkins upgrade process involves backing up configuration and plugins before updating the Jenkins version. Then I update Jenkins through package managers or the web interface and verify plugin compatibility. This works by ensuring system stability during upgrades. It is used to receive security patches and new features. A limitation is plugin incompatibility with newer versions.

What are custom quality gates in SonarQube?
Custom quality gates define conditions like minimum code coverage, maximum bug count, or vulnerability thresholds. These rules determine whether a build passes or fails. This works by enforcing code quality standards automatically. It is used to maintain software reliability. A limitation is overly strict gates can block development.

What are common deployment stage challenges in CI/CD pipelines?
Deployment challenges include environment configuration mismatches, missing dependencies, and network connectivity issues. I usually debug logs and validate configurations to identify problems. This works by verifying each step of the deployment process. It is used to maintain reliable releases. A limitation is complex environments make troubleshooting harder.

Write a multi-stage Dockerfile.
A multi-stage Dockerfile uses separate build and runtime stages to reduce final image size. The first stage compiles the application and the second stage runs it with only required files. This works by isolating build dependencies from runtime dependencies. It is used to create lightweight production images. A limitation is debugging across stages can be complex.

How do you build Docker images and push them to registries?
I build images using docker build and tag them with repository names before pushing to a registry using docker push. Authentication with the registry is required before pushing images. This works by storing container images in centralized repositories. It is used to share images across environments. A limitation is network dependency for image distribution.

How do you troubleshoot stopped containers?
I check container status using docker ps -a and inspect logs using docker logs. I also inspect container configuration and resource usage. This works by identifying runtime or configuration errors. It is used to restore container functionality. A limitation is limited debugging information if logs are not configured.

How do you change Docker container port mappings?
Port mappings are defined during container creation using the -p option in docker run. If the container is already running, it must be recreated with the new port mapping. This works by exposing container ports to host ports. It is used to allow external access to container services. A limitation is downtime during container recreation.

What is the difference between Kubernetes LoadBalancer and Ingress?
LoadBalancer exposes services directly using cloud provider load balancers. Ingress provides HTTP routing rules to manage multiple services under a single entry point. These resources work by controlling external traffic into the cluster. They are used to expose applications externally. A limitation is Ingress requires an ingress controller.

How do you configure Terraform state locking?
Terraform state locking prevents multiple users from modifying the state file simultaneously. I configure it using DynamoDB when storing state in an S3 backend. This works by locking the state during Terraform operations. It is used to prevent conflicts in team environments. A limitation is dependency on backend availability.

How do you convert shell scripts to Ansible playbooks?
I analyze the logic in shell scripts and translate commands into Ansible modules and tasks. Each step is defined as a task in the playbook with proper idempotent modules. This works by automating configuration management. It is used to ensure consistent infrastructure deployment. A limitation is some complex scripts require custom modules.

How do you monitor Kubernetes clusters?
I deploy monitoring tools like Prometheus to collect metrics and use Grafana to visualize dashboards. I monitor metrics like CPU, memory, pod health, and request latency. This works by collecting real-time cluster data. It is used to detect issues early. A limitation is high storage usage for metrics.

Write a shell command to find files larger than 200 MB.
The command uses the find utility to search for files exceeding a specified size. It works by scanning directories and filtering results based on size conditions. It is used for storage management and cleanup tasks. A limitation is scanning large directories may take time. In real scenarios, administrators often schedule such scans.

How do you check Jenkins system logs?
Jenkins system logs can be checked through the Jenkins dashboard under the system log section or directly from the server log files. These logs contain information about builds, plugins, and system events. This works by providing visibility into Jenkins operations. It is used for troubleshooting system issues. A limitation is logs may become large over time.

---

What is the difference between Security Groups and Network ACLs in AWS?
Security Groups act as virtual firewalls attached to EC2 instances and operate at the instance level. They are stateful, meaning if inbound traffic is allowed, the response traffic is automatically allowed. Network ACLs operate at the subnet level and are stateless, so both inbound and outbound rules must be defined explicitly. They work together to control network traffic in a VPC. They are used to enforce layered security for infrastructure. A limitation is misconfigured rules can block legitimate traffic.

How does Auto Scaling work with an Application Load Balancer? Explain a real scenario.
Auto Scaling works with an Application Load Balancer by dynamically adjusting the number of EC2 instances based on metrics like CPU usage or request count. The load balancer distributes incoming traffic across instances registered in the Auto Scaling group. This works by scaling infrastructure up or down depending on demand. It is used to maintain performance during traffic spikes while controlling costs during low usage. A limitation is scaling delay when sudden traffic spikes occur. In real scenarios, combining predictive scaling and health checks ensures smoother traffic handling.

What are IAM Roles vs IAM Policies? When should you use each?
IAM Policies define permissions in JSON format that specify what actions are allowed or denied. IAM Roles are identities that applications or services assume to gain those permissions temporarily. This works by attaching policies to roles which are then assigned to services like EC2 or Lambda. It is used to avoid storing long-term credentials in applications. A limitation is overly permissive policies can create security risks. In real scenarios, the principle of least privilege is always recommended.

Explain the difference between S3 Standard, S3 IA, and Glacier.
S3 Standard is designed for frequently accessed data with high availability and low latency. S3 Infrequent Access is used for data accessed less often but still requiring quick retrieval. Glacier is designed for long-term archival storage with very low cost but slower retrieval times. These storage classes work by balancing cost and access frequency. They are used to optimize storage expenses. A limitation is retrieval cost and delay for Glacier.

How would you secure secrets in a CI/CD pipeline running on AWS?
Secrets should be stored in secure services like AWS Secrets Manager or AWS Systems Manager Parameter Store. The CI/CD pipeline retrieves secrets dynamically using IAM roles instead of storing them in code. This works by providing encrypted storage and controlled access to credentials. It is used to protect sensitive data like database passwords and API keys. A limitation is additional configuration and access management. In real scenarios, secret rotation should also be implemented.

What is the difference between ECS and EKS?
ECS is AWS’s native container orchestration service that integrates closely with other AWS services. EKS is a managed Kubernetes service that runs standard Kubernetes clusters on AWS. Both work by orchestrating containerized workloads across compute resources. They are used to deploy scalable microservices applications. A limitation is higher operational complexity with Kubernetes. In real scenarios, ECS is simpler while EKS is preferred for Kubernetes-based ecosystems.

How do you deploy a Docker container to AWS using CI/CD?
The application is first containerized using Docker and pushed to a container registry like Amazon ECR. A CI/CD pipeline then pulls the image and deploys it to services like ECS, EKS, or EC2. This works by automating build, test, and deployment stages. It is used to deliver updates consistently across environments. A limitation is dependency on registry availability. In real scenarios, image scanning and tagging strategies are important.

What are the steps to troubleshoot a failing EC2 instance in production?
I first check instance status checks and system logs from the AWS console. Then I verify network configurations such as security groups and network ACLs. This works by isolating infrastructure-level issues from application issues. It is used to restore connectivity and service availability. A limitation is limited debugging if the instance becomes completely unresponsive. In real scenarios, using AWS Systems Manager helps access instances without SSH.

Explain how Terraform manages AWS infrastructure state.
Terraform maintains a state file that maps infrastructure resources to configuration definitions. When Terraform runs, it compares the desired configuration with the current state to determine necessary changes. This works by tracking resource attributes and dependencies. It is used to automate infrastructure provisioning reliably. A limitation is risk of state corruption if not managed properly. In real scenarios, remote state backends with locking are recommended.

How would you design a highly available architecture on AWS for a web application?
I would deploy application servers across multiple Availability Zones behind an Application Load Balancer. Auto Scaling groups would ensure sufficient capacity while RDS Multi-AZ or read replicas handle database reliability. This works by distributing workloads and eliminating single points of failure. It is used to maintain uptime during failures or traffic spikes. A limitation is increased cost due to redundant infrastructure. In real scenarios, monitoring and backup strategies are also required.

---

How do you change file permissions in Linux?
File permissions in Linux are changed using the chmod command, which modifies read, write, and execute permissions for users, groups, or others. It works by applying numeric or symbolic modes to control access to files and directories. This is used to secure files and control who can modify or execute them. A limitation is incorrect permissions can expose sensitive data or break applications. In real scenarios, administrators carefully apply least privilege permissions to avoid security risks.

How do you search for a file by name in Linux?
A file can be searched using the find command followed by the directory path and file name pattern. It works by recursively scanning directories and matching file names or attributes. This is used to locate configuration files, logs, or scripts in large systems. A limitation is that searching large file systems can take time. In real scenarios, administrators often combine find with filters like size or time for faster results.

What is an inode and how does it relate to files?
An inode is a data structure in Linux that stores metadata about a file such as permissions, owner, size, and disk location. It works by linking file names to actual storage blocks on disk. This is used by the file system to manage and access files efficiently. A limitation is that inode limits can restrict file creation even when disk space is available. In real scenarios, systems running many small files can exhaust inode capacity.

What is the difference between CMD and ENTRYPOINT in Docker?
CMD defines the default command that runs when a container starts, while ENTRYPOINT defines the main executable that always runs. They work together where CMD can provide arguments to the ENTRYPOINT command. This is used to control container startup behavior. A limitation is confusion when both are defined incorrectly. In real scenarios, ENTRYPOINT is used for fixed commands and CMD for default parameters.

What is the difference between COPY and ADD in a Dockerfile?
COPY transfers files from the local system into the container image. ADD performs the same function but also supports remote URLs and automatic extraction of compressed archives. This works by adding application files during image build. COPY is generally preferred because it is simpler and more predictable. A limitation is that ADD can introduce unintended behavior. In real scenarios, COPY is recommended for most use cases.

What is ImagePullBackOff and how do you fix it?
ImagePullBackOff occurs when Kubernetes cannot pull a container image from the registry. It works by retrying the image pull operation repeatedly until successful. This is used to indicate image availability or authentication issues. A limitation is that deployments cannot start until the image is accessible. In real scenarios, checking image names, registry credentials, and network connectivity usually resolves the issue.

What command is used to build a Docker image from a Dockerfile?
The docker build command is used to create an image from a Dockerfile. It works by executing instructions in the Dockerfile sequentially to build layers. This is used to package applications and dependencies into container images. A limitation is that large builds take time and consume storage. In real scenarios, build caching and multi-stage builds help optimize the process.

How do you clean unused Docker images and containers safely?
Unused resources can be removed using docker system prune or docker image prune commands. It works by deleting stopped containers, unused networks, and dangling images. This is used to reclaim disk space and maintain system performance. A limitation is accidental deletion of needed images if not reviewed. In real scenarios, administrators check resource lists before pruning.

A container keeps restarting. How do you debug it?
I check container logs using docker logs and inspect container configuration using docker inspect. It works by identifying application errors or configuration issues causing container crashes. This is used to diagnose runtime problems quickly. A limitation is limited logs if the container crashes immediately. In real scenarios, running the container interactively helps identify issues.

What is the purpose of the docker tag command?
The docker tag command assigns a new name or version tag to an existing image. It works by referencing the same image ID under a different repository or version label. This is used to organize images and prepare them for pushing to registries. A limitation is confusion if version tags are not managed properly. In real scenarios, version tags are aligned with release versions.

How do you ensure zero-downtime deployments in Kubernetes?
Zero-downtime deployments are achieved using rolling update strategies with readiness probes configured. Kubernetes gradually replaces old pods with new ones while ensuring new pods are ready before receiving traffic. This works by maintaining service availability during updates. It is used to deploy application changes without user interruption. A limitation is misconfigured probes can still cause downtime.

Explain rolling updates and rollbacks.
Rolling updates replace old application pods with new ones gradually while maintaining service availability. Rollbacks revert deployments to the previous stable version if issues occur. This works by maintaining deployment history and controlling pod replacement. It is used to safely release updates. A limitation is slower deployment speed. In real scenarios, readiness probes ensure safe rollout.

Pod stuck in CrashLoopBackOff. How do you troubleshoot it?
I check pod logs and describe the pod to review events and errors. This works by identifying issues such as misconfiguration, missing dependencies, or resource limits. It is used to diagnose repeated container failures. A limitation is limited debugging information if logs are empty. In real scenarios, checking previous container logs and resource settings helps resolve the issue.

What is the difference between Pod, Deployment, ReplicaSet, StatefulSet, and DaemonSet?
A Pod is the smallest deployable unit that runs containers. A Deployment manages stateless application updates and scaling using ReplicaSets. ReplicaSets maintain the required number of pod replicas. StatefulSets manage stateful applications requiring stable identity and storage. DaemonSets run one pod per node for tasks like monitoring agents. These controllers manage different workload types. A limitation is increased complexity in stateful workloads.

What is the difference between ClusterIP, NodePort, and LoadBalancer?
ClusterIP exposes services internally within the cluster. NodePort exposes services through a specific port on each node. LoadBalancer integrates with cloud load balancers to expose services externally. These service types work by routing traffic to pods. They are used to manage application accessibility. A limitation is NodePort security risks if misconfigured.

What are Taints and Tolerations?
Taints are applied to nodes to prevent pods from scheduling on them unless allowed. Tolerations are defined in pods to permit scheduling on tainted nodes. This works by controlling workload placement. It is used to reserve nodes for specific workloads. A limitation is misconfiguration may block pod scheduling.

How do you pass secrets securely in Jenkins pipelines?
Secrets are stored in Jenkins credentials manager and injected into pipelines when needed. It works by providing encrypted storage and secure access during pipeline execution. This is used to protect passwords, tokens, and keys. A limitation is improper permission control exposing credentials. In real scenarios, role-based access control protects secret usage.

What types of pipeline failures have you encountered during build or deployment stages?
Common failures include dependency installation issues, test failures, configuration errors, and deployment permission problems. These failures occur when pipeline stages cannot complete successfully. They are used to indicate issues early in the development process. A limitation is debugging complex pipelines can take time. In real scenarios, logs and monitoring tools help identify root causes.

How do you troubleshoot CI/CD pipeline failures?
I start by reviewing pipeline logs and identifying the stage where the failure occurred. Then I analyze configuration files, dependencies, and environment variables. This works by isolating the exact failure point. It is used to quickly restore pipeline functionality. A limitation is incomplete logs may slow debugging.

Jenkins job not triggering after a Git commit. How do you fix it?
I check webhook configuration in the Git repository and ensure Jenkins is reachable. Then I verify job trigger settings and repository permissions. This works by confirming event-driven triggers are correctly configured. It is used to automate CI builds. A limitation is network issues may block webhook delivery.

What is Terraform state and why is it important?
Terraform state is a file that stores the current infrastructure configuration and resource mapping. It works by tracking resources created by Terraform. This is used to compare desired and actual infrastructure states during operations. A limitation is corruption or conflicts if not managed properly. In real scenarios, remote state backends are used for safety.

What is a remote backend and state locking?
A remote backend stores Terraform state in shared storage like S3 instead of locally. State locking prevents multiple users from modifying infrastructure simultaneously. This works by locking the state during Terraform operations. It is used to maintain consistency in team environments. A limitation is dependency on backend services.

What is AWS Lambda?
AWS Lambda is a serverless compute service that runs code in response to events. It works by automatically scaling and managing infrastructure without requiring server management. It is used for event-driven applications such as APIs or data processing. A limitation is execution time limits for functions. In real scenarios, Lambda integrates with services like API Gateway and S3.

What is the difference between a public subnet and private subnet?
A public subnet has direct access to the internet through an Internet Gateway. A private subnet does not allow direct internet access and is typically used for backend resources like databases. This works by controlling routing rules in the VPC. It is used to isolate sensitive infrastructure. A limitation is additional configuration required for outbound connectivity.

What is a NAT Gateway and Internet Gateway?
An Internet Gateway allows resources in a public subnet to communicate directly with the internet. A NAT Gateway allows instances in private subnets to access the internet for updates without being publicly reachable. These gateways work by routing traffic between VPC networks and the internet. They are used to secure network architecture. A limitation is NAT Gateway adds extra cost.

---

