What happens if an InitContainer fails with restartPolicy set to Never?
An InitContainer runs before the main containers and must complete successfully for the pod to start. If restartPolicy is set to Never and the InitContainer fails, Kubernetes will not retry the container and the pod will fail permanently. This works because InitContainers are designed to prepare the environment before application startup. It is used for tasks like database initialization or configuration setup. A limitation is that failure blocks the entire pod lifecycle. In real scenarios, monitoring and retry logic should be implemented carefully.

What happens when you delete a pod in a StatefulSet?
When a pod in a StatefulSet is deleted, Kubernetes automatically recreates it with the same name and ordinal index. This works because StatefulSets maintain stable network identity and persistent storage for each pod. It is used for stateful workloads like databases where identity and storage must remain consistent. A limitation is slower recovery compared to stateless workloads. In real scenarios, persistent volumes are reattached to the recreated pod.

Why does a DaemonSet run on control plane nodes even if they have NoSchedule taints?
DaemonSets often include tolerations that allow them to run on nodes with certain taints like NoSchedule on control plane nodes. This works by allowing system-level workloads such as monitoring agents to run on every node. It is used to ensure cluster-wide services operate consistently. A limitation is that misconfigured tolerations can overload control plane nodes. In real scenarios, DaemonSets are used for logging, monitoring, or security agents.

What happens if you push a new image during an ongoing rollout?
If a new image is deployed while a rollout is in progress, Kubernetes stops the current rollout and starts a new one for the updated version. This works because Deployments maintain a history of ReplicaSets and ensure the desired state is updated. It is used to manage continuous updates in production environments. A limitation is partial rollout states during frequent updates. In real scenarios, teams usually wait for rollout completion before pushing another update.

Can containers in the same pod use the same port?
Containers in the same pod share the same network namespace, so they cannot bind to the same port simultaneously. This works because all containers inside a pod share the same IP address. It is used to allow containers to communicate locally through localhost. A limitation is port conflicts between containers. In real scenarios, sidecar containers usually communicate through different ports.

What happens if the HPA metrics server goes down?
If the metrics server is unavailable, the Horizontal Pod Autoscaler cannot collect resource metrics. This causes scaling operations to freeze at the current replica count. This works because HPA relies on metrics APIs to make scaling decisions. It is used to dynamically scale applications based on workload. A limitation is inability to respond to traffic spikes without metrics. In real scenarios, monitoring the metrics server is critical.

What happens if a ServiceAccount is deleted while pods are running?
When a ServiceAccount is deleted, the tokens already mounted in existing pods remain valid until the pod is restarted. This works because tokens are mounted at pod creation time and are not refreshed automatically. It is used to authenticate pods to the Kubernetes API. A limitation is security risks if tokens remain active. In real scenarios, restarting pods is required to remove invalid tokens.

Can strict pod anti-affinity cause scheduling problems?
Strict anti-affinity rules can prevent pods from being scheduled if no nodes meet the required conditions. This works because the scheduler enforces rules that prevent pods from running on the same node or topology domain. It is used to distribute workloads for high availability. A limitation is scheduling deadlocks if cluster resources are limited. In real scenarios, preferred rules are often used instead of strict ones.

Can multiple pods mount a ReadWriteOnce volume?
ReadWriteOnce volumes can be mounted by multiple pods only if those pods run on the same node. This works because the storage system restricts access to a single node at a time. It is used for workloads that require exclusive storage access. A limitation is reduced flexibility for distributed workloads. In real scenarios, ReadWriteMany volumes are preferred for shared storage.

What happens if a NetworkPolicy defines only ingress rules?
If a NetworkPolicy specifies only ingress rules, outgoing traffic from pods remains unrestricted by default. This works because Kubernetes network policies treat ingress and egress rules independently. It is used to control incoming traffic without affecting outgoing communication. A limitation is potential data exfiltration risk. In real scenarios, both ingress and egress policies should be defined.

How do you debug CrashLoopBackOff with empty logs?
If logs are empty, I use kubectl describe pod to inspect events and identify reasons like image errors or resource limits. I also check container startup commands and configuration. This works by analyzing Kubernetes event messages when logs are not available. It is used to diagnose startup failures. A limitation is limited debugging information. In real scenarios, checking previous container logs can help.

Why might the cluster autoscaler fail to scale nodes?
The autoscaler may fail if node group limits are reached or resource requests are misconfigured. It works by evaluating pending pods and deciding whether new nodes are needed. It is used to dynamically adjust cluster capacity. A limitation is scaling delays during sudden demand spikes. In real scenarios, correct resource requests are essential for scaling.

Why might rolling updates cause downtime?
Downtime can occur if readiness probes are missing or if PodDisruptionBudgets are not configured properly. This works because Kubernetes may terminate old pods before new ones are ready. It is used to update applications gradually. A limitation is incorrect probe configuration. In real scenarios, readiness probes ensure traffic is only routed to healthy pods.

What issues can affect etcd performance?
etcd performance issues often occur due to high disk latency or slow fsync operations. etcd stores the cluster state and must perform frequent disk writes. This works by maintaining consistent cluster data. It is used to ensure Kubernetes control plane reliability. A limitation is dependency on fast disk storage. In real scenarios, SSD storage and monitoring etcd metrics are recommended.

How can you enforce usage of an internal container registry?
This can be enforced using admission controllers or policy engines like OPA Gatekeeper. These tools validate container images before they are deployed to the cluster. This works by rejecting deployments that use unauthorized registries. It is used to maintain security and compliance. A limitation is additional policy management overhead. In real scenarios, organizations restrict deployments to trusted registries only.

---