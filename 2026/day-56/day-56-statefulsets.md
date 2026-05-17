# Kubernetes StatefulSets
A hands-on guide to understanding why stateful applications need more than a Deployment, and how StatefulSets provide stable identity, ordered lifecycle management, and persistent per-pod storage.
 
## Table of Contents
- [Understand the Problem - Why Not a Deployment?](#1-understand-the-problem)
- [Create a Headless Service](#2-create-a-headless-service)
- [Create a StatefulSet](#3-create-a-statefulset)
- [Stable Network Identity](#4-stable-network-identity)
- [Stable Storage - Data Survives Pod Deletion](#5-stable-storage--data-survives-pod-deletion)
- [Ordered Scaling](#6-ordered-scaling)
- [Clean Up](#7-clean-up)

## 1. Understand the Problem
Before building a StatefulSet, this exercise demonstrates what breaks when we try to run a stateful workload as a regular Deployment.
 
### Manifest - `nginx-deployment.yaml`
```yaml
apiVersion: apps/v1               #apps/v1 is stable API for workloads like Deployments, ReplicaSets, StatefulSets
kind: Deployment                  #defines resource type, Deployment manages Pods via ReplicaSets, ensuring scaling and self‑healing
metadata:
  name: nginx-deployment          #unique name of Deployment resource
spec:                             #desired state of Deployment
  replicas: 3                     #ensures 3 Pods are always running
  selector:                       #tells Deployment which Pods it manages
    matchLabels:
      app: nginx                  #must match labels in Pod template below
  template:                       #pod blueprint used by Deployment
    metadata:
      labels:
        app: nginx                #must match selector above (app: nginx)
    spec:
      containers:                 #defines workload inside each Pod
      - name: nginx               #logical name for container
        image: nginx:latest       #docker image pulled from registry (Nginx latest version)
        ports:
        - containerPort: 80       #declares container listens on port 80 (HTTP) - this helps Services target Pod correctly
```

```bash
kubectl apply -f nginx-deployment.yaml
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(856).png)

Pod names will look like `nginx-deployment-7f9d8c9b4f-abcde` - random suffixes, different every time.

### Delete a pod and observe what replaces it
```bash
kubectl delete pod <pod-name>
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(858).png)

The replacement pod appears with a **completely different random suffix**. The original identity is gone.
 
### Why does this break stateful applications?
 
Databases and clustered systems (MySQL, PostgreSQL, Kafka, Zookeeper) require every node to have a **stable, predictable identity**. They use hostnames and DNS names to recognize each other, form quorums, and replicate data. When a node rejoins the cluster with a new name, the cluster does not recognize it - replication breaks, quorums fail, and data consistency is at risk.
 
### Deployment vs StatefulSet - side by side
 
| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random (`app-xxxxx-yyyyy`) | Stable, ordered (`app-0`, `app-1`, `app-2`) |
| Startup order | All pods start simultaneously | Ordered: `pod-0` → `pod-1` → `pod-2` |
| Storage | Shared PVC or ephemeral | Each pod gets its own dedicated PVC |
| Network identity | No stable hostname | Stable DNS entry per pod |
| Use case | Stateless web servers, APIs | Databases, message queues, caches |
 
```bash
# Clean up before moving on
kubectl delete deployment nginx-deployment
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(862).png)

## 2. Create a Headless Service
A StatefulSet requires a **Headless Service** to create stable DNS entries for each pod individually.

A normal Service assigns a single `ClusterIP` and load-balances traffic across all matching pods - callers never know which pod they hit. A Headless Service (`clusterIP: None`) skips the load balancer entirely and instead creates individual DNS records for each pod, allowing them to be addressed directly.
### Manifest - `web-headless-service.yaml`
```yaml
apiVersion: v1            #defines which Kubernetes API group/version this resource belongs to, v1 is core API group for basic resources like Pods, Services, ConfigMaps, etc
kind: Service             #specifies resource type, Service provides stable networking for Pods, abstracting away their ephemeral IPs
metadata:                 #identification details for resource
  name: web-headless      #service will be called web-headless, this name is used in DNS resolution (<pod>.<service>.<namespace>.svc.cluster.local)
spec:                     #defines desired configuration of Service
  clusterIP: None         #makes this a Headless Service - instead of load‑balancing behind a single ClusterIP, K8s creates individual DNS records for each Pod - this is required for StatefulSets so each pod can be addressed directly
  selector:               #determines which Pods this Service targets  
    app: web              #matches Pods with label 'app=web' - this must align with labels defined in our StatefulSet Pod template
  ports:                  #defines how Service exposes Pods
  - port: 80              #Service listens on port 80
    name: http            #logical name for port (useful for referencing in other manifests, e.g., probes)
```
```bash
kubectl apply -f web-headless-service.yaml
kubectl get svc web-headless
```
Example output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(863).png)

`CLUSTER-IP: None` confirms this is a Headless Service. Rather than a single stable IP, Kubernetes will create individual DNS records for each pod managed by the StatefulSet.
 
### Normal Service vs Headless Service
| | Normal Service | Headless Service |
|---|---|---|
| `clusterIP` | Assigned automatically | `None` |
| DNS result | Single IP (the Service VIP) | One IP per pod |
| Traffic routing | Load-balanced | Direct to individual pods |
| Required for StatefulSet | ✗ | ✓ |

## 3. Create a StatefulSet
### Manifest - `web-statefulset.yaml`
```yaml
apiVersion: apps/v1                               #uses stable API group for workload controllers like Deployments, ReplicaSets, and StatefulSets
kind: StatefulSet                                 #declares this resource is a StatefulSet, designed for stateful applications needing stable identity, ordered startup, and persistent storage
metadata:
  name: web                                       #StatefulSet will be called 'web'
spec:
  serviceName: web-headless                       #links StatefulSet to a Headless Service (web-headless) - this ensures each pod gets a stable DNS entry like 'web-0.web-headless.default.svc.cluster.local'
  replicas: 3                                     #Specifies 3 pods should be created (web-0, web-1, web-2)
  selector:                                       #defines which pods belong to this StatefulSet
    matchLabels:
      app: web                                    #must match labels in the pod template below (app: web)
  template:                                       #blueprint for pods created by StatefulSet
    metadata:
      labels:
        app: web                                  #labels (app: web) must match selector above
    spec:
      containers:                                 #defines workload inside each pod
      - name: nginx                               #logical name for container
        image: nginx:latest                       #pulls latest Nginx image
        ports:
        - containerPort: 80                       #exposes HTTP port 80
        volumeMounts:                             #mounts a persistent volume (web-data) at '/usr/share/nginx/html' so each pod has its own storage for serving content
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:                           #defines how PVCs are created for each pod
  - metadata:
      name: web-data                              #PVCs will be named web-data-web-0, web-data-web-1, etc.
    spec:
      accessModes: ["ReadWriteOnce"]              #each PVC can be mounted by a single node at a time
      resources:
        requests:
          storage: 100Mi                          #each pod gets 100Mi of persistent storage
```
> `serviceName: web-headless` must match the Headless Service name exactly. This is what links the StatefulSet to its DNS identity.
 
> `volumeClaimTemplates` is unique to StatefulSets - it acts as a PVC blueprint. Kubernetes automatically creates one PVC per pod from this template.
 
```bash
kubectl apply -f web-statefulset.yaml
 
# Watch pod creation in real time
kubectl get pods -l app=web -w
```
Pods are created **sequentially** --> `web-0` must be `Ready` before `web-1` starts, and `web-1` must be `Ready` before `web-2` starts.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(866).png)

### Verify per-pod PVCs were created
```bash
kubectl get pvc
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(870).png)
 
Each pod has its own dedicated PVC - `web-0` always uses `web-data-web-0`, `web-1` always uses `web-data-web-1`, and so on. This binding is permanent and survives pod deletion.

## 4. Stable Network Identity
Each StatefulSet pod receives a predictable DNS name following this pattern:
 
```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```
 
For our StatefulSet:
 
```
web-0.web-headless.default.svc.cluster.local
web-1.web-headless.default.svc.cluster.local
web-2.web-headless.default.svc.cluster.local
```
 
### Verify DNS resolution
 
```bash
# Launch a temporary BusyBox pod for DNS testing
kubectl run tmp --rm -it --image=busybox --restart=Never -- sh
 
# Inside the pod, run:
nslookup web-0.web-headless.default.svc.cluster.local
nslookup web-1.web-headless.default.svc.cluster.local
nslookup web-2.web-headless.default.svc.cluster.local
exit
```
 
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(873).png)

```bash
# In another terminal, check the actual pod IPs
kubectl get pods -o wide -l app=web
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(877).png)

The IPs returned by `nslookup` should match the pod IPs shown in `kubectl get pods -o wide`. This confirms that each pod has a stable, individually addressable DNS entry - something a Deployment cannot provide.
 
### Why this matters for clustered applications
| Scenario | Deployment | StatefulSet |
|---|---|---|
| Pod restarts with same name | ✗ No - new random suffix | ✓ Yes - same ordinal name |
| Pod has stable DNS entry | ✗ No | ✓ Yes |
| Peer nodes can find each other | ✗ No | ✓ Yes |
| Safe for database clustering | ✗ No | ✓ Yes |

## 5. Stable Storage - Data Survives Pod Deletion
### Write unique data into each pod
```bash
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"
```
 
### Verify data before deletion
```bash
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-2 -- cat /usr/share/nginx/html/index.html
```
 
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(882).png)
 
### Delete a pod and verify data survives
```bash
kubectl delete pod web-0
 
# Watch until web-0 is recreated and Ready
kubectl get pods -l app=web -w
 
# Check the data on the recreated pod
kubectl exec web-0 -- cat /usr/share/nginx/html/index.html
```
 
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(887).png)
 
The recreated `web-0` automatically reattached to `web-data-web-0` - the same PVC as before. The data is intact.
 
### Deployment vs StatefulSet storage behavior
| | Deployment (emptyDir) | StatefulSet (PVC per pod) |
|---|---|---|
| Data after pod restart | Lost | Preserved |
| Data after pod deletion | Lost | Preserved |
| Each pod has own storage | ✗ No | ✓ Yes |
| Pod reattaches to same storage | ✗ No | ✓ Yes |

## 6. Ordered Scaling
StatefulSets enforce strict ordering during both scale-up and scale-down operations.
### Scale up to 5 replicas
```bash
kubectl scale statefulset web --replicas=5
kubectl get pods -l app=web -w
```
Pods are created in order - `web-3` starts only after `web-2` is `Ready`, and `web-4` starts only after `web-3` is `Ready`:
 
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(889).png)

### Scale down to 3 replicas
```bash
kubectl scale statefulset web --replicas=3
kubectl get pods -l app=web -w
```
Pods terminate in **reverse order** - `web-4` first, then `web-3`. The original three pods (`web-0`, `web-1`, `web-2`) remain.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(891).png)

### Check PVCs after scale-down
```bash
kubectl get pvc
```
All 5 PVCs are still present - including `web-data-web-3` and `web-data-web-4` for the deleted pods:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(893).png)

Kubernetes deliberately retains PVCs after scale-down. If we scale back up to 5, `web-3` and `web-4` will reattach to their original PVCs with all data intact.
 
### Scaling behavior summary
| Operation | Order | PVCs affected |
|---|---|---|
| Scale up | Ascending (`web-3` → `web-4`) | New PVCs created per pod |
| Scale down | Descending (`web-4` → `web-3`) | PVCs **retained**, not deleted |
| Scale back up | Ascending | Pods reattach to existing PVCs |

## 7. Clean Up
```bash
# Delete the StatefulSet
kubectl delete statefulset web
 
# Delete the Headless Service
kubectl delete svc web-headless
 
# Check PVCs - they are still present
kubectl get pvc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(897).png)

Deleting the StatefulSet does **not** automatically delete its PVCs. This is intentional - Kubernetes protects stateful data from accidental deletion. We must remove PVCs explicitly:
 
```bash
kubectl delete pvc --all
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(899).png)

> Run this only in our test namespace (e.g., `default`) to avoid deleting PVCs used by other workloads.
 
```bash
# Verify everything is removed
kubectl get statefulset
kubectl get svc
kubectl get pvc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/3e408d7a0f3f592179da3dc383fe903f1bc9217c/2026/day-56/Screenshots/Screenshot%20(901).png)

### Why Kubernetes does not auto-delete PVCs
| Resource | Auto-deleted with StatefulSet? | Reason |
|---|---|---|
| Pods | ✓ Yes | Pods are managed replicas, safe to remove |
| PVCs | ✗ No | Data protection - accidental deletion could be unrecoverable |
| PVs (dynamic) | Depends on reclaim policy | `Delete` policy removes PV; `Retain` keeps it |
 
This safety-first design reflects a core Kubernetes principle: **compute is replaceable, data is not**.