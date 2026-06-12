---
```
k8s deployment is ready
Pods looks healthy
YAML files look perfect
```
---
yet PersistentVolumeClaim refuses to bind and pod is stuck in ContainerCreating stage and k8s showing "Pending"

No error and warn

---
```
$ kubectl get pvc (list of pvc)

$ kubectl get pods (list of pods)

$ kubectl describe pvc <pvc_name> (describing the pvc for pods)

$ kubectl get storageclass (list of storageclass and storage provider)
```

---
if pvc is stuck in pending due to
StorageClass doesnt exist
StorageClass spelling mistake
No dynamic provisioner
PV Size mistmatch
AccessMode mismatch
