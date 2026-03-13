Explain Taints and Tolerations in Kubernetes.
Taints are applied to nodes to prevent pods from being scheduled on them unless the pod explicitly allows it. Tolerations are defined in pod specifications so that the pod can be scheduled on nodes with matching taints. This works by controlling workload placement and isolating certain nodes for specific workloads. It is used when some nodes should only run particular applications like GPU workloads. A limitation is misconfiguration can prevent pods from scheduling. In real scenarios, taints are often used to reserve nodes for critical workloads.

Explain Node Affinity in Kubernetes.
Node affinity is a scheduling rule that allows pods to run only on nodes that match specific labels. It works by defining rules like required or preferred node selection in the pod specification. It is used to control where applications run based on hardware, region, or environment. A limitation is strict rules may prevent scheduling if no nodes match. In real scenarios, preferred affinity is often used to avoid scheduling failures.

Explain Kubernetes architecture components.
Kubernetes architecture consists of control plane components and worker node components that manage containerized workloads. The control plane includes components like API server, scheduler, controller manager, and etcd for cluster management. Worker nodes run kubelet, kube-proxy, and container runtime to execute pods. This works by separating management logic from workload execution. It is used to maintain scalability and reliability. A limitation is control plane failure can affect cluster operations if not highly available.

What is the difference between Control Plane and Worker Nodes?
Control plane nodes manage the cluster and make decisions about scheduling and state management. Worker nodes run the actual application workloads inside pods. This works by delegating orchestration tasks to the control plane and execution tasks to workers. It is used to maintain separation of responsibilities in cluster architecture. A limitation is control plane nodes must be protected and highly available. In real scenarios, production clusters run multiple control plane nodes.

Explain stages in your CI/CD pipeline.
My pipeline starts with source code checkout, followed by dependency installation and build stages. Then it runs unit tests and security scans before packaging the application into a container image. After that, the image is pushed to a registry and deployed to Kubernetes environments. This works by automating the entire delivery lifecycle. It is used to ensure reliable and repeatable deployments. A limitation is longer execution time if many checks are included.

What is Continuous Feedback in CI/CD?
Continuous feedback means collecting information from builds, tests, monitoring, and users to improve the development process. It works by integrating monitoring tools, alerts, and test results into the pipeline and development workflow. It is used to detect issues quickly and improve product quality. A limitation is large amounts of feedback can create noise. In real scenarios, dashboards and alerts help prioritize important signals.

Explain Continuous Delivery.
Continuous delivery ensures that code changes are automatically built, tested, and prepared for deployment at any time. It works by keeping the application in a deployable state through automated pipelines. It is used to enable frequent and reliable releases. A limitation is that final deployment may still require manual approval. In real scenarios, strong automated testing is required to maintain stability.

How do you secure Terraform state files?
Terraform state files contain sensitive infrastructure details, so they must be stored securely. I store them in a remote backend like S3 with encryption enabled and access controlled using IAM policies. This works by preventing unauthorized access and enabling team collaboration. It is used to protect infrastructure metadata. A limitation is dependency on backend availability. In real scenarios, state locking is also used to prevent concurrent modifications.

Explain Terraform remote backend configuration.
A remote backend stores Terraform state outside the local machine in services like S3 or Terraform Cloud. It works by centralizing the state file and enabling state locking for collaborative workflows. It is used to manage infrastructure changes safely across teams. A limitation is network dependency. In real scenarios, backend encryption and access control are required.

What is null_resource in Terraform?
null_resource is a Terraform resource used to run scripts or commands that are not tied to infrastructure resources. It works using provisioners like local-exec or remote-exec to execute tasks. It is used for operations like configuration scripts or triggers. A limitation is that it can create dependency complexity. In real scenarios, it is used sparingly to avoid breaking infrastructure as code principles.

Explain deployment strategies like Rolling, Blue-Green, and Canary.
Rolling deployment updates pods gradually to ensure minimal downtime during upgrades. Blue-Green deployment runs two identical environments and switches traffic between them after validation. Canary deployment releases changes to a small portion of users before full rollout. These strategies work by reducing risk during updates. They are used to improve deployment reliability. A limitation is increased infrastructure complexity.

What is a DaemonSet?
A DaemonSet ensures that one pod runs on every node in a Kubernetes cluster. It works by automatically scheduling pods on new nodes when they join the cluster. It is used for workloads like monitoring agents or log collectors. A limitation is increased resource usage across nodes. In real scenarios, DaemonSets are used for tools like logging or security agents.

Explain Docker networking concepts.
Docker networking allows containers to communicate with each other and external systems. It works using network drivers like bridge, host, overlay, and macvlan. These networks provide isolation and connectivity for containers. They are used to enable communication between services. A limitation is network misconfiguration can break connectivity.

Explain multi-stage Dockerfile and why it is used.
A multi-stage Dockerfile uses multiple build stages to separate build dependencies from runtime dependencies. It works by compiling the application in one stage and copying only required artifacts into the final image. It is used to reduce image size and improve security. A limitation is more complex build configurations. In real scenarios, smaller images improve deployment speed.

Explain Docker image security best practices.
Docker image security involves using minimal base images, scanning images for vulnerabilities, and avoiding running containers as root. It works by reducing attack surface and identifying vulnerabilities early. It is used to maintain secure container environments. A limitation is compatibility issues with minimal images. In real scenarios, automated image scanning is integrated into CI/CD.

Explain rollback strategies in case of deployment failure.
Rollback strategies restore the previous stable version of an application when a deployment fails. This works by maintaining versioned images and deployment history in Kubernetes. It is used to minimize downtime and recover quickly. A limitation is database changes may not be reversible. In real scenarios, backward compatibility is important.

Explain Route 53 working.
Route 53 is a DNS service that routes user requests to AWS resources based on domain names. It works by translating domain names into IP addresses using DNS records. It is used for traffic routing, health checks, and failover. A limitation is DNS propagation delays. In real scenarios, low TTL values help faster switching.

Explain end-to-end DNS resolution flow.
DNS resolution starts when a user enters a domain name in the browser. The request goes to a recursive resolver, which queries root, TLD, and authoritative name servers to find the IP address. The browser then connects to the server using that IP. This works by mapping human-readable names to machine addresses. It is used to locate services on the internet. A limitation is caching delays.

How do you handle SSL certificate management in Kubernetes?
SSL certificates are managed using tools like cert-manager integrated with Kubernetes ingress. It works by automatically requesting and renewing certificates from certificate authorities. It is used to enable secure HTTPS communication. A limitation is certificate expiration if automation fails. In real scenarios, monitoring certificate expiry is essential.

How do you handle failed pipeline stages?
When a pipeline stage fails, I analyze logs and identify the failing step. I work with developers if the issue is related to code or configuration. This works by isolating the root cause and fixing it quickly. It is used to restore pipeline functionality. A limitation is complex pipelines take longer to debug.

How do you debug CI/CD pipeline issues with your team?
I first check pipeline logs and recent commits to identify the issue. Then I collaborate with developers or infrastructure teams depending on the problem. This works by combining debugging with team communication. It is used to resolve issues faster. A limitation is dependency on team availability. In real scenarios, documentation and monitoring help speed up troubleshooting.
