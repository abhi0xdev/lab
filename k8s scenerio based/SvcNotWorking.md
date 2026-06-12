```
k8s pod are healthy
deployment is green
service is created 
u dont see any error and no warning

but u r application is dead and every req hangs and fails
```
---

k8s service is perfectly fine but refused to send traffic to pods bcoz of tiny labels mismatch

(service that refuses to send traffic)
---
```
$ kubectl get svc (list of svc name)

$ kubectl get pods (list of all pods)

$ kubectl describe svc <svc_name> (describe the svc)

$ kubectl get pods --show-labels (shows the svc labels of pods)
```
---

if ur k8s svc is not working then -->
```
always check the service exists
check the pod is running
check labels match exactly
targetport should match containerport
should be same namespace
debug network policies & firewall
```
