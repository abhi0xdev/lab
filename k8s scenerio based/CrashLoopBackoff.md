In CrashLoopBackoff -->
```
App unable to alive
its showing green
deployment succeed
no warning
but container dies every sec
```
---
```
$ kubectl get pods

$ kubctl decribe pod <pod_name>

$ kubectl logs <pod_name>
```
---

CrashLoopBackoff almost always cost by -->
```
bad env var
missing dependencies
typos
wrong image tags
bad startup commands
unhandled exceptions

k8s is just messenger but real problem always in inside the container
```
