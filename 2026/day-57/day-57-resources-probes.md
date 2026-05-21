# Kubernetes - Resource Requests, Limits, and Probes
A hands-on guide to controlling how much compute a container can use, what happens when it exceeds those limits, and how Kubernetes detects and responds to unhealthy containers.
 
## Table of Contents
- [Resource Requests and Limits](#1-resource-requests-and-limits)
- [OOMKilled - Exceeding Memory Limits](#2-oomkilled--exceeding-memory-limits)
- [Pending Pod - Requesting Too Much](#3-pending-pod--requesting-too-much)
- [Liveness Probe](#4-liveness-probe)
- [Readiness Probe](#5-readiness-probe)
- [Startup Probe](#6-startup-probe)
- [Clean Up](#7-clean-up)

## 1. Resource Requests and Limits
Kubernetes uses two mechanisms to control how much CPU and memory a container can consume:
- **Requests** - the guaranteed minimum. The scheduler uses this value to decide which node to place the pod on.
- **Limits** - the maximum allowed. The kubelet enforces this at runtime - CPU is throttled, memory violations kill the container.

### Manifest - `resource-pod.yaml`
```yaml
apiVersion: v1                                #specifies which version of K8s API we’re using, v1 is stable core API group for basic resources like Pods, Services, ConfigMaps
metadata:                                     #provides identifying information about object
  name: resource-pod                          #unique name for this Pod in namespace
spec:                                         #defines desired state of object - for Pods, it describes containers, volumes, and policies
  containers:                                 #list of containers that run inside Pod
  - name: busybox                             #identifier for container
    image: busybox                            #lightweight Linux utility image, often used for testing
    command: ["sh", "-c", "sleep 3600"]       #overrides container’s default entrypoint - here: run a shell (sh), execute sleep 3600 → container stays alive for 1 hour
    resources:                                #defines how much CPU/memory container asks for and is allowed to use
      requests:
        cpu: "100m"                           #0.1 CPU core guaranteed
        memory: "128Mi"                       #'128 MiB' RAM guaranteed
      limits:
        cpu: "250m"                           #max 0.25 CPU core      
        memory: "256Mi"                       #max 256 MiB RAM
                                              #Kubelet enforces these at runtime. Since requests < limits → QoS class = Burstable.
```
### Units reference
| Resource | Unit | Example | Equivalent |
|---|---|---|---|
| CPU | millicores | `100m` | 0.1 CPU core |
| CPU | cores | `1` | 1 full CPU core |
| Memory | mebibytes | `128Mi` | ~134 MB |
| Memory | gibibytes | `1Gi` | ~1.07 GB |
 
```bash
kubectl apply -f resource-pod.yaml
kubectl get pods
 
# Inspect resource allocation and QoS class
kubectl describe pod resource-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(909).png)

Look for the `Resources` section under `Containers`, and the `QoS Class` field near the bottom of the output.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(910).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(911).png)

### Quality of Service (QoS) classes
Kubernetes assigns a QoS class to every pod based on how its resource fields are set:
| QoS Class | Condition | Eviction priority |
|---|---|---|
| **Guaranteed** | `requests == limits` for all containers | Last to be evicted |
| **Burstable** | `requests < limits` (this pod) | Middle priority |
| **BestEffort** | No requests or limits set | First to be evicted |
 
Since this pod has `requests < limits`, its QoS class is **Burstable**. If requests and limits were equal, it would be **Guaranteed**.
 
> The scheduler uses **requests** for placement decisions. The kubelet enforces **limits** at runtime. A pod can use resources beyond its request (up to its limit) when the node has spare capacity.

## 2. OOMKilled — Exceeding Memory Limits
This exercise demonstrates what happens when a container tries to allocate more memory than its limit allows.
 
### Manifest - `oom-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: stress
    image: polinux/stress                                             #utility image that generates artificial CPU/memory load
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]       #overrides default entrypoint - runs 'stress' tool with arguments - '--vm 1' → start one memory worker - '--vm-bytes 200M' → allocate 200 MB memory - '--vm-hang 1' → keep allocation alive. This deliberately exceeds memory limit.
    resources:                                                        #defines runtime constraints
      limits:
        memory: "100Mi"                                               #container can use at most 100 MiB RAM. Since command tries to allocate 200 MB, kubelet enforces limit and kills container.
```
| Field | Value | Meaning |
|---|---|---|
| `--vm-bytes 200M` | 200 MB | Memory the process tries to allocate |
| `limits.memory` | 100Mi | Maximum the container is allowed |
| Result | OOMKilled | Linux OOM killer terminates the process |
 
```bash
kubectl apply -f oom-pod.yaml
 
# Watch the status change in real time
kubectl get pods -w
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(917).png)

The pod starts, immediately exceeds its memory limit, and transitions to `CrashLoopBackOff`.
 
```bash
# Inspect the termination details
kubectl describe pod oom-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(924).png)

Look for:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(922).png)
 
```bash
kubectl logs oom-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(928).png)

### CPU vs Memory - how limits are enforced
| Resource | Over-limit behavior | Container killed? |
|---|---|---|
| CPU | Throttled - process slows down | ✗ No |
| Memory | OOMKilled - process terminated immediately | ✓ Yes |
 
Exit code `137` = `128 + 9` (SIGKILL). This is the standard exit code for any process killed by the Linux OOM killer.

## 3. Pending Pod — Requesting Too Much
This exercise shows what happens when a pod requests more resources than any node in the cluster can provide.
 
### Manifest - `pending-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "sleep 3600"]       #overrides container’s default entrypoint - runs a shell (sh) that executes sleep 3600 → container stays alive for 1 hour
    resources:                                #defines how much CPU/memory container asks for
      requests:
        cpu: "100"                            #requests 100 CPU cores (far beyond what a single node has)
        memory: "128Gi"                       #requests 128 GiB RAM (also unrealistic for most clusters)
                                              #scheduler tries to place Pod but finds no node with enough resources
```
| Request | Value | Why it fails |
|---|---|---|
| `cpu: "100"` | 100 cores | Far exceeds any single node's capacity |
| `memory: "128Gi"` | 128 GiB | Far exceeds any standard node's RAM |
 
```bash
kubectl apply -f pending-pod.yaml
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(930).png)

The pod stays in `Pending` indefinitely - the scheduler cannot find a node that satisfies the request.
 
```bash
kubectl describe pod pending-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(933).png)

Scroll to the `Events` section. We will see:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(935).png)
 
### What this demonstrates
The pod is never scheduled - it does not consume any resources and does not start. This is a safe failure mode. The scheduler event message tells us exactly why placement failed, making it straightforward to diagnose over-provisioned requests.

## 4. Liveness Probe
A **liveness probe** detects containers that are running but stuck or unhealthy. If the probe fails beyond the threshold, Kubernetes restarts the container.
 
### Manifest - `liveness-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "touch /tmp/healthy; sleep 30; rm /tmp/healthy; sleep 3600"]       #overrides container’s default entrypoint - steps executed :- 'touch /tmp/healthy' → create a file marking container as healthy - 'sleep 30' → wait 30 seconds - 'rm /tmp/healthy' → delete file - 'sleep 3600' → keep container alive for 1 hour
    livenessProbe:                                                                           #defines how K8s checks if container is still alive
      exec:                                                                                  #runs a command inside container
        command: ["cat", "/tmp/healthy"]                                                     #tries to read file
      periodSeconds: 5                                                                       #probe runs every 5 seconds
      failureThreshold: 3                                                                    #after 3 consecutive failures, K8s restarts container
                                                                                             #Since file is deleted after 30s, probe fails → container restarted once.
```
### What this pod does
```
t=0s   → /tmp/healthy created → probe passes ✓
t=30s  → /tmp/healthy deleted → probe starts failing
t=45s  → 3 consecutive failures (3 × 5s) → container restarted
```
 
```bash
kubectl apply -f liveness-pod.yaml
kubectl get pods
 
# Watch status changes in real time
kubectl get pod liveness-pod -w
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(942).png)

After ~45 seconds, the pod will restart. Check the restart count:
```bash
kubectl describe pod liveness-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(945).png)

Look for `Restart Count: 1` under the `Containers` section.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(946).png)

### Liveness probe configuration fields
| Field | Value | Meaning |
|---|---|---|
| `periodSeconds` | 5 | Check every 5 seconds |
| `failureThreshold` | 3 | Restart after 3 consecutive failures |
| Time to restart | 15s after file deleted | 3 failures × 5s = 15s |

## 5. Readiness Probe
A **readiness probe** controls whether a pod receives traffic. Probe failure removes the pod from Service endpoints - but the container is **not restarted**. This is the key distinction from a liveness probe.
 
### Manifest - `readiness-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
  labels:
    app: readiness          #adds a label to Pod - this is critical because Services use labels to select Pods - without labels, 'kubectl expose' cannot create a Service that targets Pod
spec:
  containers:
  - name: nginx
    image: nginx
    readinessProbe:         #defines how K8s checks if container is ready to serve traffic
      httpGet:              #sends an HTTP GET request
        path: /             #checks root path of Nginx server
        port: 80            #probes container’s port 80
      periodSeconds: 5      #probe runs every 5 seconds
                            #if probe fails, Pod is marked 'Not Ready' and removed from Service endpoints, but container is not restarted
```
```bash
kubectl apply -f readiness-pod.yaml
 
# Expose the pod as a Service
kubectl expose pod readiness-pod --port=80 --name=readiness-svc
 
# Verify the pod is ready and registered as an endpoint
kubectl get endpoints readiness-svc
kubectl get pods
```
The pod shows `1/1 READY` and its IP appears under endpoints.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(950).png)

### Break the readiness probe
```bash
# Delete the file that Nginx serves — probe will now fail
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html
```
Wait approximately 15 seconds, then check:
```bash
kubectl get pods                    # Shows 0/1 READY
kubectl get endpoints readiness-svc # Endpoints list is now empty
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(956).png)

```bash
kubectl describe pod readiness-pod
```
The events show repeated readiness probe failures. There are **no restart events** - the container is still running, just not receiving traffic.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(958).png)

### Liveness vs Readiness - key differences
| | Liveness Probe | Readiness Probe |
|---|---|---|
| Purpose | Detect stuck/deadlocked containers | Detect containers not ready to serve traffic |
| On failure | Container is **restarted** | Pod is **removed from Service endpoints** |
| Container stopped | ✓ Yes | ✗ No |
| Use case | Detect hangs, infinite loops | Detect slow startup, temporary unavailability |

## 6. Startup Probe
A **startup probe** gives slow-starting containers extra time to initialize. While the startup probe is running, liveness and readiness probes are **disabled** - preventing premature restarts of applications that take a long time to boot.
 
### Manifest - `startup-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: slow-container
    image: busybox
    command: ["sh", "-c", "sleep 20 && touch /tmp/started && sleep 3600"]     #steps executed :- 'sleep 20' → simulate slow startup (20 seconds) - 'touch /tmp/started' → create a file once startup is complete - 'sleep 3600' → keep container alive for 1 hour
    startupProbe:                                                             #special probe that runs during container startup
      exec:                                                                   #runs a command inside container
        command: ["cat", "/tmp/started"]                                      #checks if file exists
      periodSeconds: 5                                                        #probe runs every 5 seconds
      failureThreshold: 12                                                    #allows 12 failures before killing container → 12 × 5s = 60s budget, this ensures container has enough time (20s) to finish startup before liveness checks begin
    livenessProbe:                                                            #runs after startup probe succeeds
      exec:
        command: ["cat", "/tmp/started"]                                      #Cchecks same file every 5 seconds to ensure container remains healthy - if file disappears later, K8s restarts container
      periodSeconds: 5
```
### Startup timeline
```
t=0s   → Container starts, /tmp/started does not exist yet
t=0-20s → Startup probe checks every 5s → fails (file missing)
t=20s  → sleep 20 completes → /tmp/started created
t=20s  → Startup probe passes ✓ → liveness probe takes over
```
Total startup budget: `failureThreshold × periodSeconds` = `12 × 5s` = **60 seconds**.
```bash
kubectl apply -f startup-pod.yaml
kubectl get pods -w
 
# Inspect probe activity
kubectl describe pod startup-pod
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(964).png)

### Effect of `failureThreshold` on startup budget
| `failureThreshold` | Budget | Result |
|---|---|---|
| `12` | 60s | Startup completes at ~20s - pod stays healthy ✓ |
| `2` | 10s | Budget exhausted before startup completes - container killed ✗ |
 
> Set `failureThreshold` generously for applications with variable startup times. A tight threshold causes unnecessary restarts during normal initialization.
 
### All three probes - when to use which
| Probe | Triggers | On failure | Disabled during startup probe? |
|---|---|---|---|
| **Startup** | Once, at container start | Container restarted | N/A |
| **Liveness** | Continuously after startup | Container restarted | ✓ Yes |
| **Readiness** | Continuously after startup | Removed from endpoints | ✓ Yes |

## 7. Clean Up
```bash
# Delete all pods created in this exercise
kubectl delete pod resource-pod oom-pod pending-pod liveness-pod readiness-pod startup-pod
 
# Delete the readiness Service
kubectl delete svc readiness-svc
 
# Verify cleanup
kubectl get pods
kubectl get svc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/73250fa92dc982376ec13de1a7b99761a9c85e26/2026/day-57/Screenshots/Screenshot%20(967).png)

> Using `kubectl delete pod --all` removes every pod in the namespace, not just the ones from this exercise. Use it only if we are certain the namespace contains no other workloads.
 
Only system pods and the default `kubernetes` Service should remain after cleanup.
After cleanup, only system Pods and default Services remain.
