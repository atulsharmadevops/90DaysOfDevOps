# Kubernetets - Metrics Server and Horizontal Pod Autoscaler (HPA)
A hands-on guide to installing the Metrics Server, observing real-time resource usage, and configuring the Horizontal Pod Autoscaler to scale workloads automatically based on CPU utilization.
 
## Table of Contents
- [Install the Metrics Server](#1-install-the-metrics-server)
- [Explore kubectl top](#2-explore-kubectl-top)
- [Create a Deployment with CPU Requests](#3-create-a-deployment-with-cpu-requests)
- [Create an HPA (Imperative)](#4-create-an-hpa-imperative)
- [Generate Load and Watch Autoscaling](#5-generate-load-and-watch-autoscaling)
- [Create an HPA from YAML (Declarative)](#6-create-an-hpa-from-yaml-declarative)
- [Clean Up](#7-clean-up)

## 1. Install the Metrics Server
The Metrics Server collects CPU and memory usage from each node's kubelet and exposes it to the Kubernetes API. It is required for `kubectl top` and for HPA to function.
 
### Check if it is already installed
```bash
kubectl get pods -n kube-system | grep metrics-server
```
If we see a pod with `STATUS=Running`, the Metrics Server is already active. If there is no output, install it using one of the methods below.
 
### Install on minikube
```bash
minikube addons enable metrics-server
```
 
### Install on kind or kubeadm
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(976).png)

On local clusters we may encounter TLS errors because the kubelet uses self-signed certificates. Patch the Deployment to add `--kubelet-insecure-tls`:

Using `kubectl patch` is more reliable than `kubectl edit` - use a full args replacement to be certain:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/args",
    "value": [
      "--cert-dir=/tmp",
      "--secure-port=10250",
      "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
      "--kubelet-use-node-status-port",
      "--metric-resolution=15s",
      "--kubelet-insecure-tls"
    ]
  }
]'
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(977).png)

Verify the args were applied
```bash
kubectl get deploy metrics-server -n kube-system -o jsonpath='{.spec.template.spec.containers[0].args}' | tr ',' '\n'
```
- We should see `--kubelet-insecure-tls` in the output.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(978).png)

### Force clean rollout & check status
```bash
# Force a rollout to clear the stale old pod
kubectl rollout restart deployment/metrics-server -n kube-system

# Watch until only ONE pod is Running 1/1
kubectl get pods -n kube-system -w | grep metrics-server
```
- Wait until we see 1/1 Running - this may take 30–60 seconds.
- `STATUS` should be `Running`
- `RESTARTS` should be `0`

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(984).png)

### Test `kubectl top` commands
```bash
#Confirm Metrics Server is collecting and serving metrics.
kubectl top nodes
kubectl top pods -A
```
- If we see CPU and memory usage values, Metrics Server is working

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(987).png)

## 2. Explore kubectl top
### Check Node Resource Usage
```bash
kubectl top nodes
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(988).png)

- Shows CPU cores and memory usage for each node.
- This is actual usage, not the requests/limits we set in manifests.

### Check Pod Resource Usage (All Namespaces)
```bash
kubectl top pods -A
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(989).png)

- Lists every pod across all namespaces with real-time CPU and memory consumption.
- Useful for spotting which workloads are consuming the most resources.

### Sort Pods by CPU Usage
```bash
kubectl top pods -A --sort-by=cpu
```
- Displays pods sorted by CPU usage (highest at the bottom/top depending on our terminal).
- This helps us quickly identify the most CPU-intensive pod.

Example output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(990).png)

 
### Requests vs actual usage - an important distinction
| Value | Source | Meaning |
|---|---|---|
| **Requests / Limits** | Our YAML manifests | What we configured as minimum / maximum |
| **Usage** (`kubectl top`) | Metrics Server | What the pod is actually consuming right now |
 
The Metrics Server polls each kubelet approximately every 15 seconds, so `kubectl top` values are near-real-time, not instantaneous.

### Which pod is using the most CPU right now?

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(990).png)

- Here, `kube-apiserver-devops-cluster-control-plane` → 24m CPU - is the top CPU consumer.

## 3. Create a Deployment with CPU Requests
### Manifest - `php-apache-deployment.yaml`
```yaml
apiVersion: apps/v1                            #tells K8s which API group/version to use - apps/v1 is stable API for Deployments, ReplicaSets, StatefulSets
kind: Deployment                               #defines resource type, Deployment manages Pods via ReplicaSets, ensuring scaling and self‑healing
metadata:
  name: php-apache                             #unique name of Deployment
spec:                                          #desired state of Deployment
  replicas: 1                                  #ensures one Pod is always running
  selector:                                    #tells Deployment which Pods it manages - must match Pod template labels below
    matchLabels:
      app: php-apache
  template:                                    #pod blueprint used by Deployment
    metadata:
      labels:
        app: php-apache                        #must match selector above (app: php-apache)
    spec:
      containers:                              #defines workload inside each Pod
      - name: php-apache                       #logical name for container
        image: registry.k8s.io/hpa-example     #docker image pulled from registry (CPU‑intensive PHP‑Apache server used for HPA demos)
        resources:                             #defines resource requests/limits
          requests:
            cpu: 200m                          #Pod requests 200 milliCPU (0.2 cores)
                                              #This is critical: HPA uses this request to calculate utilization percentages, without it HPA shows <unknown> and cannot scale
```
> **CPU requests are mandatory for HPA.** Without `resources.requests.cpu`, the HPA cannot calculate utilization percentage and will display `TARGETS: <unknown>` indefinitely. This is the most common HPA setup mistake.

### Apply the Deployment
```bash
kubectl apply -f php-apache-deployment.yaml
```
Expose the Deployment as a Service
```bash
kubectl expose deployment php-apache --port=80
```
- This creates a ClusterIP Service named `php-apache` on port 80.
- The load generator in later tasks will use this Service.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(993).png)

### Verify Pod Status
```bash
kubectl get pods -l app=php-apache
```
- Ensure the pod is in Running state.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(994).png)

### Check Current CPU Usage
```bash
kubectl top pods -l app=php-apache
```
- This shows actual CPU usage of the pod.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(995).png)

At idle, CPU usage will be low (e.g., `5m`). The load generator in Task 5 will push this above the HPA threshold.

## 4. Create an HPA (Imperative)
### Create the HPA
```bash
kubectl autoscale deployment php-apache --cpu=50% --min=1 --max=10
```
| Parameter | Value | Meaning |
|---|---|---|
| `--cpu-percent=50` | 50% | Scale up when average CPU exceeds 50% of the requested 200m (= 100m) |
| `--min=1` | 1 | Minimum replica count - never scale below this |
| `--max=10` | 10 | Maximum replica count - never scale above this |

### Check HPA Status
```bash
kubectl get hpa
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(996).png)

`TARGETS: <unknown>` is normal for the first ~30 seconds while the Metrics Server collects its first data points.

### Describe the HPA for Details
```bash
kubectl describe hpa php-apache
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(998).png)

- Shows metrics, scaling events, and conditions.
- The `TARGETS` column may show `<unknown>` initially - this is normal until Metrics Server provides data (usually ~30 seconds).

### Understand Scaling Behavior
- HPA checks metrics every 15 seconds.
- If average CPU usage > 50% of requests --> scales up.
- If average CPU usage < 50% --> scales down (with a 5‑minute stabilization delay).


### What does the TARGETS column show?

Example after metrics arrive:
```Code
NAME         REFERENCE                   TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache       60%/50%   1         10        2          1m
```
Meaning: Current average CPU = 60%, target = 50% --> HPA scaled replicas up to 2.

This scales up when average CPU exceeds 50% of requests, and down when it drops below.

## 5. Generate Load and Watch Autoscaling
### Start a Load Generator Pod
```bash
kubectl run load-generator --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(999).png)

- This creates a one‑off BusyBox pod that continuously sends HTTP requests to the `php-apache` Service.
- The repeated requests simulate CPU load on the PHP‑Apache container.

### Watch the HPA in Action
```bash
kubectl get hpa php-apache --watch
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1005).png)

- Observe the `TARGETS` column and `REPLICAS` count.
- Over 1–3 minutes, CPU usage climbs above 50% of the requested 200m.
- HPA increases replicas to balance load.

### Stop the Load Generator
```bash
kubectl delete pod load-generator
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1008).png)

- This stops the continuous requests.
- CPU usage will drop, but scale‑down is slow (default 5‑minute stabilization window).
- We don’t need to wait for scale‑down in this exercise.

### HPA scaling lifecycle 
| Phase | Trigger | Delay |
|---|---|---|
| Scale up | Avg CPU > target % | Immediate (no stabilization) |
| Scale down | Avg CPU < target % | 5-minute stabilization window |
| Minimum floor | Always | Replicas never drop below `--min` |
| Maximum ceiling | Always | Replicas never exceed `--max` |

### How many replicas did HPA scale to under load?

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1006).png)

- HPA scaled replicas to 10 when CPU usage exceeded 50%.

## 6. Create an HPA from YAML (Declarative)
The imperative `kubectl autoscale` command works for simple CPU-based scaling. The declarative `autoscaling/v2` API unlocks multiple metrics, memory-based scaling, and fine-grained control over scaling speed.

### Delete the Imperative HPA
```bash
kubectl delete hpa php-apache
```
- Removes the HPA created in Task 4.
- Ensures we don’t have conflicting autoscalers.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1009).png)

### Declarative HPA Manifest `php-apache-hpa.yaml`:
```yaml
apiVersion: autoscaling/v2                  #uses autoscaling/v2 API, which supports CPU, memory, custom metrics, and advanced scaling behaviors - this is recommended version for modern HPA manifests
kind: HorizontalPodAutoscaler               #defines resource type - HPA automatically adjusts number of replicas in a target workload based on observed metrics
metadata:
  name: php-apache                          #unique name of this HPA resource
spec:
  scaleTargetRef:                           #defines which workload this HPA will scale - here it targets 'php-apache' Deployment created earlier
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1                            #1 pod (always at least one running)
  maxReplicas: 10                           #10 pods (never exceed this limit)
  metrics:                                  #defines what metric to monitor
  - type: Resource                          #uses built‑in resource metrics
    resource:
      name: cpu                             #monitors CPU usage.
      target:
        type: Utilization                   #target is a percentage of requested CPU
        averageUtilization: 50              #scale to keep average CPU usage at ~50% of requested value (200m) - If usage rises to 100m (50% of 200m), HPA keeps replicas steady - If usage rises to 150m (75%), HPA scales up
  behavior:                                 #fine‑grained control over scaling speed
    scaleUp:
      stabilizationWindowSeconds: 0         #scale up immediately when threshold exceeded (no delay)
    scaleDown:
      stabilizationWindowSeconds: 300       #wait 300 seconds (5 minutes) before scaling down, to avoid flapping
```
### Key fields explained
 
| Field | Value | Effect |
|---|---|---|
| `apiVersion: autoscaling/v2` | v2 | Supports CPU, memory, and custom metrics |
| `scaleTargetRef` | `php-apache` Deployment | The workload HPA will scale |
| `averageUtilization: 50` | 50% | Same threshold as the imperative command |
| `scaleUp.stabilizationWindowSeconds: 0` | 0s | Scale up immediately when threshold is exceeded |
| `scaleDown.stabilizationWindowSeconds: 300` | 300s | Wait 5 minutes before scaling down |

### Apply the Manifest
```bash
kubectl apply -f php-apache-hpa.yaml
```

### Verify HPA
```bash
kubectl describe hpa php-apache
```
- Shows metrics, scaling events, and behavior configuration.
- Confirm that the behavior section is applied.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1011).png)

### Imperative vs declarative HPA 
| | `kubectl autoscale` | `autoscaling/v2` YAML |
|---|---|---|
| Supports CPU scaling | ✓ | ✓ |
| Supports memory scaling | ✗ | ✓ |
| Custom metrics | ✗ | ✓ |
| Scaling behavior control | ✗ | ✓ |
| Version-controlled | ✗ | ✓ |
| Recommended for production | ✗ | ✓ |

### What does the behavior section control?
- It controls **how quickly the HPA scales up or down**:
    - Scale‑up --> immediate (no stabilization).
    - Scale‑down --> delayed by 300 seconds to avoid flapping.

`autoscaling/v2` supports multiple metrics and fine-grained scaling behavior that the imperative command cannot configure.

## 7. Clean Up

```bash
# Delete the HPA
kubectl delete hpa php-apache
 
# Delete the Service
kubectl delete svc php-apache
 
# Delete the Deployment
kubectl delete deployment php-apache
 
# Delete the load generator pod (if still present)
kubectl delete pod load-generator
 
# Verify everything is removed
kubectl get hpa
kubectl get svc
kubectl get deployments
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/4eb4f0382eec5b5cf521f7bf4f443d65600a7481/2026/day-58/Screenshots/Screenshot%20(1012).png)

No HPA, Service, Deployment, or load-generator pod should remain. The Metrics Server continues running in `kube-system` - it is a cluster-level component, not part of this exercise's cleanup.