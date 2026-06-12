```
Feature realese
Traffic increasing
Teams on slack celebrating

but suddenly pods stuck in pending
Services down
Cluster refusing to schedule anything

no errors and no crashes just pending state
```
---
```
its not YAML file mistake but Node itself

they push CPU from 500m to 8 cores
Memory from 512mi to 10 Gi

they think k8s has autoscaling so automatically k8s will figured out but k8s didnt scale fast enough suddenly all pods in k8s become Unschedulable
```
---

when pod is pending state due to -->
```
Node has insufficient CPU
insufficient memory
resource exceed cluster capacity
Autoscaler didnt add nodes
Taints/Tolerations mismatch
Nodeselector rules cant be satisfied
Wrong affinity/antiaffinity rules

but real prod issue is Over requested resources
```
