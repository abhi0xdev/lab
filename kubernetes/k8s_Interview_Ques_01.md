How do you list Kubernetes pods based on their age?
Pods can be listed based on age using the command `kubectl get pods --sort-by=.metadata.creationTimestamp`. This works by sorting pod objects according to their creation timestamp in ascending order. It is used to quickly identify older or recently created pods during troubleshooting or rollout analysis. A limitation is that sorting does not directly show restart history or lifecycle events. In real scenarios, engineers often combine this command with `kubectl describe pod` to inspect pod history.

Is there any downtime in Blue/Green deployment? Explain why or why not.
Blue/Green deployment usually avoids downtime because two identical environments run simultaneously. Traffic is switched from the old environment to the new one using a load balancer or routing change. This works by validating the new environment before redirecting users. It is used to achieve safe deployments and fast rollback capability. A limitation is increased infrastructure cost because two environments run at the same time.

How does Blue/Green deployment work?
Blue/Green deployment maintains two separate environments where one serves live traffic while the other runs the new version of the application. Once the new version is verified, traffic is switched to the new environment using DNS or load balancer configuration. This works by separating deployment from user traffic exposure. It is used to reduce deployment risk and enable quick rollback. A limitation is complexity in maintaining identical environments.

How do you design a highly available Kubernetes cluster?
A highly available cluster is designed by running multiple control plane nodes and distributing worker nodes across availability zones. It works by replicating critical components like etcd and API servers to avoid single points of failure. This is used to maintain cluster operations during node or infrastructure failures. A limitation is increased infrastructure and operational cost. In real scenarios, load balancers and automated health checks are also implemented.

What recent production issue have you troubleshooted?
A recent issue involved pods repeatedly crashing due to incorrect environment variables after a deployment. I checked logs and pod events, identified the configuration mismatch, and rolled back to the previous stable version. This worked by restoring the last working deployment quickly. It is used to minimize downtime during incidents. A limitation is dependency on accurate monitoring and rollback readiness.

What is a headless service in Kubernetes and how does it work?
A headless service is created by setting `clusterIP: None` in the service definition. It works by returning individual pod IP addresses instead of a single service IP. This is used for applications requiring direct pod communication like databases or stateful applications. A limitation is lack of load balancing at the service level. In real scenarios, StatefulSets commonly use headless services.

How do you create and manage Kubernetes clusters using Terraform?
Terraform provisions infrastructure resources such as VPCs, nodes, and Kubernetes clusters using infrastructure as code. It works by defining resources in configuration files and applying them to create or update infrastructure. This is used to automate cluster creation and maintain consistent environments. A limitation is dependency on accurate state management. In real scenarios, remote state storage with locking prevents conflicts.

What is the role of Master node and Worker node?
The master node manages the Kubernetes cluster and runs components like API server, scheduler, and controller manager. Worker nodes run the application workloads inside pods. This works by separating orchestration from execution tasks. It is used to maintain cluster management and workload scheduling. A limitation is control plane failure can affect cluster operations if redundancy is not implemented.

What common Kubernetes errors have you faced and how did you resolve them?
Common errors include CrashLoopBackOff and ImagePullBackOff during deployments. CrashLoopBackOff usually occurs due to application startup errors or configuration problems, while ImagePullBackOff occurs when the image cannot be fetched from the registry. This works by identifying failure events and logs from Kubernetes. It is used to diagnose container startup issues quickly. A limitation is incomplete logs if containers crash immediately.

How do you access a running pod and how do you define Kubernetes objects?
A running pod can be accessed using the command `kubectl exec -it pod-name -- /bin/sh`. Kubernetes objects are defined using YAML or JSON configuration files that describe desired resource states. This works by allowing direct interaction with containers and declarative infrastructure management. It is used for debugging and configuration control. A limitation is manual changes inside pods are temporary.

How does Horizontal Pod Autoscaler work internally?
Horizontal Pod Autoscaler monitors metrics such as CPU or memory usage from the metrics server. It works by calculating desired replica counts based on current metrics and configured thresholds. This is used to automatically scale application pods during traffic changes. A limitation is scaling delay if metrics are not updated quickly. In real scenarios, proper resource requests are required for accurate scaling.

How do Services and Kubernetes Ingress differ?
A Service exposes pods internally or externally and provides stable networking within the cluster. Ingress manages HTTP routing rules and external access to multiple services. This works by separating service discovery from routing logic. It is used to simplify external traffic management. A limitation is that Ingress requires an ingress controller.

How will you debug networking issues inside a Kubernetes cluster?
I check service configurations, endpoints, and DNS resolution using commands like `kubectl get svc` and `kubectl get endpoints`. I also verify network policies and test connectivity between pods. This works by identifying whether the issue lies in service routing or networking policies. It is used to resolve connectivity issues. A limitation is network debugging can be complex in large clusters.

How do you manage secrets securely in Kubernetes?
Secrets are stored as Kubernetes objects and accessed by pods through environment variables or mounted volumes. They work by separating sensitive information from application configuration. This is used to protect credentials like database passwords and API keys. A limitation is secrets are base64 encoded but not encrypted by default. In real scenarios, encryption at rest and external secret managers are recommended.

Pods in CrashLoopBackOff problem – what will you check?
I check container logs and pod events using `kubectl logs` and `kubectl describe pod`. I verify configuration values, environment variables, and resource limits. This works by identifying repeated container startup failures. It is used to diagnose application issues quickly. A limitation is limited logs when containers crash immediately.

Application not accessible via Service – what will you check?
I verify that the service selector correctly matches pod labels and check service endpoints. I also confirm that the correct port mappings and target ports are configured. This works by ensuring traffic is routed to the correct pods. It is used to resolve connectivity issues. A limitation is incorrect label configuration can silently break routing.

If a pod is stuck in Pending state what will you check?
I check node availability, resource requests, and scheduling constraints like node selectors or taints. I also review cluster events to identify scheduling failures. This works by identifying why the scheduler cannot place the pod on any node. It is used to resolve resource or scheduling issues. A limitation is cluster capacity limits.

Pod shows ImagePullBackOff error – what will you check?
I verify the image name, tag, and registry authentication credentials. I also check network connectivity and registry availability. This works by identifying why Kubernetes cannot pull the container image. It is used to resolve deployment issues quickly. A limitation is dependency on external registry services.

Pod logs are not showing – what will you do?
I check previous container logs using `kubectl logs --previous` and inspect pod events. This works by retrieving logs from the last terminated container instance. It is used to diagnose startup failures when logs disappear quickly. A limitation is very short-lived containers may still provide limited information.

If a production application suddenly goes down what are your first troubleshooting steps?
I first check monitoring dashboards and alerts to identify affected components. Then I verify pod health, logs, and recent deployments. This works by isolating infrastructure or application failures quickly. It is used to minimize downtime and restore services. A limitation is dependency on monitoring tools.

How do you scale an application in Kubernetes?
Applications can be scaled manually using `kubectl scale` or automatically using Horizontal Pod Autoscaler. This works by adjusting the number of pod replicas handling the workload. It is used to manage traffic demand dynamically. A limitation is scaling delay during sudden spikes.

Pods are running but not receiving traffic – what will you check?
I check service selectors, endpoints, and readiness probes. I also verify ingress rules and network policies. This works by ensuring traffic is routed correctly to pods. It is used to resolve routing problems. A limitation is misconfigured probes may block traffic.

What happens if a Kubernetes node goes down?
When a node fails, Kubernetes marks it as NotReady and reschedules affected pods on healthy nodes. This works by maintaining the desired state defined in the cluster. It is used to ensure application availability. A limitation is temporary service disruption.

How do you perform rolling updates in Kubernetes?
Rolling updates are performed by updating the deployment image or configuration. Kubernetes gradually replaces old pods with new ones while maintaining availability. This works by controlling update speed and pod readiness checks. It is used for safe application updates. A limitation is longer deployment time.

How do you rollback a deployment in Kubernetes?
Rollback is performed using the command `kubectl rollout undo deployment`. It works by restoring the previous version stored in deployment history. This is used to recover quickly from faulty updates. A limitation is limited history depending on configuration.

How do you monitor Kubernetes cluster health?
Cluster health is monitored using tools like Prometheus and Grafana that collect metrics from nodes and pods. This works by visualizing resource usage and application performance. It is used to detect issues before they affect users. A limitation is storage requirements for metrics.

What will you do if pods are getting OOMKilled?
OOMKilled occurs when a container exceeds its memory limits. I review resource limits and memory usage metrics to adjust configurations. This works by ensuring containers have sufficient memory. It is used to prevent repeated crashes. A limitation is increasing limits may impact node capacity.

How do you check which pod is consuming high CPU or memory?
I use the command `kubectl top pods` to view resource usage metrics. This works by retrieving data from the Kubernetes metrics server. It is used to identify resource-heavy workloads. A limitation is dependency on metrics server availability.

What happens if the etcd database fails in a Kubernetes cluster?
etcd stores the cluster state including configurations and metadata. If it fails, the control plane cannot manage resources properly. This works because the API server relies on etcd for cluster information. It is used to maintain consistent cluster state. A limitation is cluster operations stop until etcd is restored.

What is the difference between Deployment and StatefulSet?
Deployment manages stateless applications and supports rolling updates and scaling. StatefulSet manages stateful workloads requiring stable network identities and persistent storage. These controllers work by maintaining the desired number of pods. They are used for different workload types. A limitation is StatefulSet deployments are slower due to ordered operations.

---