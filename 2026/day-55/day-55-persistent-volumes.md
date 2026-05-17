# Persistent Volumes (PV) and Persistent Volume Claims (PVC)
A hands-on guide to understanding ephemeral vs persistent storage in Kubernetes, provisioning PersistentVolumes manually and dynamically, and verifying data survival across Pod restarts.
 
## Table of Contents
- [The Problem - Data Lost on Pod Deletion](#1-see-the-problem--data-lost-on-pod-deletion-emptyDir)
- [Create a PersistentVolume (Static Provisioning)](#2-create-a-persistentvolume-static-provisioning)
- [Create a PersistentVolumeClaim](#3-create-a-persistentvolumeclaim)
- [Use the PVC in a Pod](#4-use-the-pvc-in-a-pod--data-that-survives)
- [StorageClasses and Dynamic Provisioning](#5-storageclasses-and-dynamic-provisioning)
- [Dynamic Provisioning in Practice](#6-dynamic-provisioning-in-practice)
- [Clean Up](#7-clean-up)

## 1. See the Problem - Data Lost on Pod Deletion (emptyDir)
Before introducing persistent storage, this exercise demonstrates what `emptyDir` volumes actually are and why they cannot survive a Pod deletion.
### Manifest - `ephemeral-pod.yaml`
```yaml
apiVersion: v1                      #tells K8s which API version to use, v1 is stable core API group for basic resources like Pods, Services, ConfigMaps
kind: Pod                           #defines type of resource, here it’s a Pod, smallest deployable unit in K8s
metadata:
  name: ephemeral-pod               #unique Pod name in namespace
spec:                               #desired state of Pod, defines containers, volumes, restart policy, etc.
  containers:                       #list of containers inside Pod.
  - name: busybox                   #logical name for container
    image: busybox                  #lightweight BusyBox image from Docker Hub, useful for testing
    command: ["sh", "-c", "while true; do date >> /data/message.txt; sleep 10; done"]     #overrides default container command - runs a shell (sh) with -c flag to execute a command string - 'while true; do ...; done' → Infinite loop - 'date >> /data/message.txt' → Appends current timestamp to /data/message.txt - 'sleep 10' → Waits 10 seconds before repeating
    volumeMounts:                   #defines where volume is mounted inside container
    - mountPath: /data              #container sees volume at /data
      name: ephemeral-storage       #refers to volume defined below
  volumes:                          #defines storage volumes available to Pod
  - name: ephemeral-storage
    emptyDir: {}                    #creates empty directory on node’s filesystem - exists only while Pod runs - when Pod is deleted, directory and its contents are wiped - this is why data written to /data/message.txt disappears after Pod deletion
```
### Apply the Pod
```bash
kubectl apply -f ephemeral-pod.yaml
```

Check that it’s running:
```bash
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(797).png)

Verify the data exists
```bash
#Exec into the Pod and read the file:
kubectl exec ephemeral-pod -- cat /data/message.txt
```
- We should see a timestamp line written by the container.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(798).png)

### Delete the Pod
```bash
kubectl delete pod ephemeral-pod
```

### Recreate the Pod
```bash
#Reapply the manifest:
kubectl apply -f ephemeral-pod.yaml
```
Wait until it’s running:
```bash
kubectl get pods
```
Check the file again
```bash
kubectl exec ephemeral-pod -- cat /data/message.txt
```
- The file now contains only **new** timestamps. The previous data is gone.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(800).png)

### Why?
`emptyDir` is created fresh every time a Pod starts and is destroyed when the Pod is deleted. It is scoped to the Pod's lifetime, not the data's lifetime.
 
| Storage type | Survives container restart? | Survives Pod deletion? |
|---|---|---|
| `emptyDir` | ✓ Yes | ✗ No |
| PersistentVolume (PVC) | ✓ Yes | ✓ Yes |

## 2. Create a PersistentVolume (Static Provisioning)
A **PersistentVolume (PV)** is a piece of storage provisioned by an administrator - independent of any Pod. It exists as a cluster-level resource and persists beyond the lifecycle of any individual Pod.
 
### Manifest - `pv-static.yaml`
```yaml
apiVersion: v1                                #uses core K8s API group v1, PersistentVolumes are part of stable API
kind: PersistentVolume                        #declares resource type, here it’s a PersistentVolume (PV), which represents storage available to cluster
metadata:
  name: pv-static                             #unique name for this PV object in cluster
spec:                                         #defines desired configuration of PV
  capacity:                                   #declares how much storage this PV provides
    storage: 1Gi
  accessModes:                                #defines how Pods can use volume
    - ReadWriteOnce                           #volume can be mounted as read/write by a single node - Other options (not used here): ReadOnlyMany (ROX), ReadWriteMany (RWX)
  persistentVolumeReclaimPolicy: Retain       #defines what happens to PV when its PVC is deleted - Retain → Keeps data and PV object - Status changes to Released, requiring manual cleanup - alternatives: Delete (auto‑delete PV and data), Recycle (deprecated)
  hostPath:                                   #uses a directory on node’s filesystem as storage
    path: /tmp/k8s-pv-data                    #data is stored in '/tmp/k8s-pv-data' on whichever node Pod runs
```

### Apply the PV
```bash
kubectl apply -f pv-static.yaml
```
Verify the PV
```bash
#List all PVs:
kubectl get pv
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(802).png)

`STATUS: Available` means the PV exists but has not yet been claimed by a PVC.
 
### Access modes reference
| Mode | Short | Description |
|---|---|---|
| `ReadWriteOnce` | RWO | Read-write by a **single node** |
| `ReadOnlyMany` | ROX | Read-only by **many nodes** |
| `ReadWriteMany` | RWX | Read-write by **many nodes** |
 
> `hostPath` mounts a directory from the node's filesystem into the Pod. It is suitable for local development and learning, but should never be used in production - data is tied to a specific node and will be lost if that node is replaced.

## 3. Create a PersistentVolumeClaim
A **PersistentVolumeClaim (PVC)** is a request for storage made by a developer. Kubernetes matches the PVC to an available PV based on capacity and access mode.
 
### Manifest - `pvc-static.yaml`
```yaml
apiVersion: v1                      #uses core K8s API group v1, PVCs are part of stable API
kind: PersistentVolumeClaim         #declares resource type, here it’s a PVC, which is a request for storage by a Pod
metadata:
  name: pvc-static                  #unique name for this PVC object in namespace
spec:                               #defines desired configuration of PVC
  accessModes:                      #defines how Pod can use volume
    - ReadWriteOnce                 #volume can be mounted as read/write by a single node
  resources:                        #specifies storage request
    requests:
      storage: 500Mi                #PVC is asking for 500 MiB of storage
  storageClassName: ""    #controls dynamic provisioning - setting it to "" (empty string) disables dynamic provisioning - this forces K8s to bind PVC only to a manually created PV (like our pv-static) - if we omit this field, K8s uses cluster’s default StorageClass (e.g., standard) and auto‑creates a PV
```
### Apply the PVC
```bash
kubectl apply -f pvc-static.yaml
```
Verify PVC status
```bash
kubectl get pvc
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(805).png)

Verify PV status
```bash
kubectl get pv
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(808).png)

Both the PV and PVC show `Bound`, confirming Kubernetes matched them by capacity and access mode. The PVC requested 500Mi and was bound to a 1Gi PV - Kubernetes satisfies the request with the smallest available PV that meets or exceeds the requirement.
 
### The PV–PVC relationship
```
Developer creates PVC (request: 500Mi, RWO)
         │
         ▼
Kubernetes finds a matching PV (capacity: 1Gi, RWO)
         │
         ▼
Both show STATUS: Bound
         │
         ▼
PVC is now ready to be mounted in a Pod
```

## 4. Use the PVC in a Pod — Data That Survives
### Create a Pod that mounts the PVC at /data - `pvc-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pod                     #unique Pod name in namespace
spec:                               #defines desired configuration of Pod
  containers:                       #list of containers inside Pod
  - name: busybox                   #logical name for container
    image: busybox                  #lightweight BusyBox image, great for testing
    command: ["sh", "-c", "while true; do echo $(date) >> /data/message.txt; sleep 10; done"]         #overrides default container command - runs a shell (sh) with -c to execute - 'while true; do ...; done' → Infinite loop - 'echo $(date) >> /data/message.txt' → Appends current timestamp to /data/message.txt - sleep 10 → Waits 10 seconds before repeating
    volumeMounts:                   #defines where volume is mounted inside container
    - mountPath: /data              #container sees volume at /data
      name: persistent-storage      #refers to volume defined below
  volumes:                          #defines storage volumes available to Pod
  - name: persistent-storage
    persistentVolumeClaim:          #attaches a PVC to Pod
      claimName: pvc-static         #refers to PVC we created earlier
```

### Apply Pod
```bash
#Deploy the Pod to our cluster.
kubectl apply -f pvc-pod.yaml

#Confirm with:
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(810).png)

### Write and verify data
```bash
#Check that the Pod writes to the PVC.
kubectl exec pvc-pod -- cat /data/message.txt
```
Observe timestamp entries

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(812).png)

### Delete and recreate Pod
```bash
#Remove the Pod and redeploy it.
kubectl delete pod pvc-pod

#Reapply manifest:
kubectl apply -f pvc-pod.yaml
kubectl get pods
```
Wait until Pod is Running

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(814).png)

### Check file again
```bash
#Verify persistence across Pod lifecycle.
kubectl exec pvc-pod -- cat /data/message.txt
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(815).png)


The file contains entries from both Pod runs. The PVC retained all data through the Pod deletion - unlike `emptyDir`, which was wiped clean.

## 5. StorageClasses and Dynamic Provisioning
Static provisioning requires an administrator to manually create PVs before developers can claim them.

**Dynamic provisioning** removes this step - developers simply create a PVC and Kubernetes automatically provisions a matching PV using a **StorageClass**.
### List StorageClasses
```bash
kubectl get storageclass
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(817).png)

### Describe the StorageClass
```bash
#Pick the default one (often called standard) and run:
kubectl describe storageclass standard
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(819).png)

### Key StorageClass fields 
| Field | Description |
|---|---|
| **Provisioner** | The backend that creates the actual storage (e.g., `kubernetes.io/aws-ebs`, `rancher.io/local-path`) |
| **ReclaimPolicy** | What happens to the PV when the PVC is deleted - `Delete` removes the PV, `Retain` keeps it |
| **VolumeBindingMode** | `Immediate` creates the PV when the PVC is created; `WaitForFirstConsumer` waits until a Pod uses the PVC |
 
### Static vs dynamic provisioning comparison
| | Static Provisioning | Dynamic Provisioning |
|---|---|---|
| Who creates PVs | Administrator, manually | Kubernetes, automatically |
| PV manifest required | ✓ Yes | ✗ No |
| StorageClass required | ✗ No | ✓ Yes |
| Flexibility | Low | High |
| Typical environment | On-prem, learning | Cloud, production |

## 6. Dynamic Provisioning
### Create a PVC that references the default StorageClass - `pvc-dynamic.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic                 #unique PVC name in namespace
spec:                               #defines desired configuration of PVC
  accessModes:                      #defines how Pod can use volume
    - ReadWriteOnce                 #volume can be mounted as read/write by a single node
  resources:                        #specifies storage request
    requests:
      storage: 1Gi                  #PVC is asking for 1 GiB of storage, K8s will try to match this request with a PV that has at least this capacity
  storageClassName: standard        #tells K8s which StorageClass to use - standard → Refers to default StorageClass in our cluster
```

### Apply the PVC
```bash
#Deploy the PVC to the cluster.
kubectl apply -f pvc-dynamic.yaml
#PVC will be created and bound automatically

#Check that a new PV was created automatically.
kubectl get pvc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(826).png)

- Our PVC is stuck in Pending because of the StorageClass’s `volumeBindingMode: WaitForFirstConsumer`. That means Kubernetes will not provision a PV until a Pod that actually uses the PVC is scheduled. Right now we’ve only created the PVC, so the controller is waiting for a “first consumer.”

### Create a Pod that mounts the dynamic PVC at /data - `pvc-dynamic-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-dynamic-pod           #unique Pod name in namespace
spec:                             #defines desired configuration of Pod
  containers:                     #list of containers inside Pod
  - name: busybox
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) >> /data/message.txt; sleep 10; done"]       #runs a shell (sh) with -c to execute - 'while true; do ...; done' → Infinite loop - 'echo $(date) >> /data/message.txt' → Appends current timestamp to /data/message.txt - 'sleep 10' → Waits 10 seconds before repeating
    volumeMounts:                 #defines where volume is mounted inside container
    - mountPath: /data            #container sees volume at /data
      name: dynamic-storage       #refers to volume defined below
  volumes:                        #defines storage volumes available to Pod
  - name: dynamic-storage
    persistentVolumeClaim:        #attaches a PVC to Pod
      claimName: pvc-dynamic      #refers to PVC we created earlier with 'storageClassName: standard'
```
- Because the StorageClass uses `WaitForFirstConsumer`, this Pod is the “first consumer” that triggers Kubernetes to provision and bind a new PV dynamically.

### Apply Pod
```bash
#Deploy the Pod into our cluster.
kubectl apply -f pvc-dynamic-pod.yaml

#Confirm with:
kubectl get pods
kubectl get pvc
kubectl get pv
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(836).png)

### Write data
```bash
#Generate data inside the mounted volume.
kubectl exec pvc-dynamic-pod -- sh -c "echo FirstRun >> /data/message.txt"

#Verify with:
kubectl exec pvc-dynamic-pod -- cat /data/message.txt
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(842).png)

### Delete and recreate Pod
```bash
#Remove the Pod and redeploy it to test persistence.
kubectl delete pod pvc-dynamic-pod

kubectl apply -f pvc-dynamic-pod.yaml

kubectl exec pvc-dynamic-pod -- sh -c "echo SecondRun >> /data/message.txt"
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(847).png)

### Verify persistence
```bash
#Check that data from both Pods exists.
kubectl exec pvc-dynamic-pod -- cat /data/message.txt
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(846).png)

Both entries are present, confirming the dynamically provisioned PVC retained data across Pod deletions - identical behavior to the statically provisioned PVC.

## 7. Clean Up
### Delete all Pods
```bash
kubectl delete pod --all
```
(Run this only in the namespace where we created our test Pods, e.g. `default`, to avoid deleting system Pods.)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(849).png)

### Delete PVCs
```bash
kubectl delete pvc --all
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(851).png)

### Check PVs
```bash
kubectl get pv
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(852).png)

### What happens to PVs after PVC deletion?
| PV type | Reclaim policy | Behavior after PVC deleted |
|---|---|---|
| Dynamic (Task 6) | `Delete` | PV is **automatically deleted** |
| Static `pv-static` | `Retain` | PV stays in `Released` state - must be deleted manually |

### Delete the remaining PV manually
```bash
kubectl delete pv pv-static

# Final verification
kubectl get pv
kubectl get pvc
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b28b387b3f91e36c2f206344683d1675338f75d9/2026/day-55/Screenshots/Screenshot%20(854).png)

Everything should now be clear except for built-in system resources.
### Reclaim policy summary
| Policy | When PVC is deleted | Data fate |
|---|---|---|
| `Delete` | PV and underlying storage are removed | Data deleted |
| `Retain` | PV stays in `Released` state | Data preserved until admin cleans up |
| `Recycle` | Deprecated - basic scrub then makes PV available again | Data wiped |


