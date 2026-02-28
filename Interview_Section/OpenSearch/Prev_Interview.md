Explain your day-to-day responsibilities.
In my day-to-day work, I focus on building and maintaining CI/CD pipelines, mainly using GitHub Actions with multi-branch strategies for automated deployments. I handle Kubernetes operations like pod-level troubleshooting, scaling workloads, and resolving deployment failures. This works by ensuring smooth build-to-release flow and stable production systems. It is used to maintain reliability and reduce manual effort. A limitation is that production issues can be unpredictable and require quick debugging. In real scenarios, monitoring dashboards and alerts help detect issues early.

How did you deploy your application on EKS?
I containerized the application using Docker and deployed it on EKS using Kubernetes manifests and Helm charts. I configured deployments, services, and ingress along with environment-specific values. This works by defining desired state and letting Kubernetes manage orchestration. It is used for scalable and consistent deployments. A limitation is managing configuration across environments. In real scenarios, proper IAM roles and networking setup are critical for EKS access.

How do you use Helm charts?
I use Helm charts to package Kubernetes resources and manage deployments with reusable templates. I customize values.yaml for environment-specific configurations and use helm install and upgrade commands. This works by templating Kubernetes manifests for consistency. It is used to simplify deployment and versioning. A limitation is debugging complex templates. In real scenarios, linting and dry runs help catch errors early.

What type of application are you currently handling? Walk me through it.
I am working on microservices-based applications deployed on Kubernetes. The services are containerized and communicate over internal networking, and deployments are handled via CI/CD pipelines. This works by decoupling services for scalability and independent deployment. It is used to handle distributed workloads efficiently. A limitation is increased complexity in debugging service interactions. In real scenarios, proper logging and monitoring are essential.

How do you connect two applications internally like DEV and QA?
I connect applications using Kubernetes services and internal DNS names within the cluster. For cross-environment communication, I use APIs or secured endpoints with proper network rules. This works by enabling service discovery and communication over internal networks. It is used to isolate environments while allowing controlled access. A limitation is network misconfiguration can block communication. In real scenarios, service endpoints and DNS resolution must be verified.

How do you use Prometheus and Grafana?
I use Prometheus to scrape metrics from Kubernetes and applications, and Grafana to visualize them in dashboards. I configure exporters and set up queries for monitoring CPU, memory, and application metrics. This works by collecting and visualizing time-series data. It is used for observability and performance tracking. A limitation is managing large volumes of metrics. In real scenarios, retention policies and filtering are important.

Have you set up or configured any alerts?
Yes, I configure alerts in Prometheus using alert rules and route them via Alertmanager to channels like email or Slack. I define thresholds for metrics like CPU usage or pod failures. This works by triggering notifications when conditions are met. It is used to enable proactive issue detection. A limitation is alert noise if thresholds are not tuned. In real scenarios, alert tuning is critical to avoid fatigue.

How do you handle deployments?
I handle deployments through CI/CD pipelines with automated build, test, and deployment stages. I use strategies like rolling updates to ensure zero downtime. This works by gradually updating instances without service disruption. It is used to maintain availability. A limitation is rollback complexity in stateful systems. In real scenarios, proper versioning and rollback plans are required.

How do you use ITSM services like RFC and INC?
I use RFC for planned changes and approvals before deployments, and INC for handling production incidents. I update tickets with deployment details and link them to pipeline executions. This works by ensuring traceability and compliance. It is used for audit and operational processes. A limitation is dependency on manual updates. In real scenarios, automation of ticket updates improves efficiency.

How do you keep yourself updated regarding Helm charts?
I follow official Helm repositories, release notes, and community updates. I also test new chart versions in non-production environments. This works by staying aware of updates and changes. It is used to maintain compatibility and security. A limitation is time required to evaluate updates. In real scenarios, controlled upgrades are necessary.

Explain APIGEE proxy and how it is configured.
Apigee proxy acts as an API gateway that manages and secures API traffic. It is configured by defining proxy endpoints, target endpoints, and policies like authentication and rate limiting. This works by routing and controlling API requests. It is used for API management and security. A limitation is added latency. In real scenarios, proper policy configuration is important.

If something changes in a Helm chart, how do you track and install it?
I track changes using version control and Helm chart versioning. I compare changes and test them using helm diff or dry runs before upgrading. This works by ensuring controlled updates. It is used to maintain stability. A limitation is missing changes if not properly reviewed. In real scenarios, CI/CD integration helps automate upgrades.

With whom do you collaborate and how?
I collaborate with developers, QA, and SRE teams to ensure smooth deployments and issue resolution. I work through tickets, standups, and shared dashboards. This works by aligning development and operations. It is used to improve delivery efficiency. A limitation is communication gaps. In real scenarios, clear documentation helps.

How do you expose the application to the outside world? What services do you know?
I expose applications using Kubernetes services like ClusterIP, NodePort, and LoadBalancer, and use Ingress for routing. This works by making services accessible internally or externally. It is used to control traffic flow. A limitation is misconfiguration can expose services insecurely. In real scenarios, Ingress with TLS is preferred for production.
