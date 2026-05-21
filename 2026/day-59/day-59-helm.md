# Kubernetes Package Manager - HELM
A hands-on guide to installing Helm, deploying applications from public chart repositories, customizing releases with values, upgrading and rolling back, and authoring our own chart from scratch.
 
## Table of Contents
- [Install Helm](#1-install-helm)
- [Add a Repository and Search](#2-add-a-repository-and-search)
- [Install a Chart](#3-install-a-chart)
- [Customize with Values](#4-customize-with-values)
- [Upgrade and Rollback](#5-upgrade-and-rollback)
- [Create Your Own Chart](#6-create-our-own-chart)
- [Clean Up](#7-clean-up)

## 1. Install Helm
### Install Helm on Linux
```bash
#Use the official curl script to fetch Helm binaries.
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1013).png)

### Verify Helm Installation
```bash
#Confirm Helm is installed and accessible.
helm version   #shows client version

helm env       #prints environment variables
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1015).png)

### Core concepts
| Term | Definition |
|---|---|
| **Chart** | A package of Kubernetes manifest templates - the unit of distribution in Helm |
| **Release** | A specific installation of a chart in our cluster - one chart can be installed multiple times as separate releases |
| **Repository** | A collection of charts, analogous to a package registry like npm or pip |
| **Values** | Configuration parameters that customize a chart's templates at install time |

## 2. Add a Repository and Search

### Add Bitnami Repository
```bash
#Run the following command to add Bitnami’s Helm chart repository:
helm repo add bitnami https://charts.bitnami.com/bitnami
```
- This registers Bitnami as a chart source in our local Helm configuration. Bitnami maintains 300+ actively maintained charts covering popular applications like NGINX, PostgreSQL, Redis, and more.

### Update Repository
```bash
#Refresh our local cache of available charts:
helm repo update
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1017).png)

- Always run this before searching or installing to ensure we have the latest chart versions.

### Search for Charts
```bash
#Search for NGINX charts:
helm search repo nginx

#Search all Bitnami charts:
helm search repo bitnami
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1018).png)

- This lists every chart available in the Bitnami repo, including metadata like version and application.

### How many charts does Bitnami have?  
```bash
helm search repo bitnami | wc -l
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1019).png)

- This will return the number of charts listed. As of May 2026, Bitnami maintains 300+ charts in its repository. 

## 3. Install a Chart
### Install NGINX Chart
```bash
helm install my-nginx bitnami/nginx
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1046).png)

- Helm generates and applies a complete set of Kubernetes manifests automatically - Deployment, ReplicaSet, Pod, and Service - without writing a single line of YAML.

### Inspect Kubernetes Resources
```bash
#Check what was created:
kubectl get all
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1047).png)

| Resource | Purpose |
|---|---|
| Deployment | Manages the desired Pod count |
| ReplicaSet | Ensures the correct number of replicas |
| Pod(s) | Running NGINX containers |
| Service | Exposes the Pods |

### Inspect Helm Release
```bash
helm list                        # all installed releases
helm status my-nginx             # Pod, Service details, and release notes
helm get manifest my-nginx       # the exact YAML Helm applied to the cluster
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1048).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1049).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1050).png)

### How many Pods are running?  
```bash
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1051).png)

- By default, Bitnami’s NGINX chart deploys 1 Pod (replicaCount = 1).

### What Service type was created?  
```bash
kubectl get svc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1052).png)

- Default is ClusterIP (internal service).

## 4. Customize with Values
Helm charts expose configurable parameters through a `values.yaml` file. We can override any default value at install time.
 
### View all default values for a chart
```bash
#Check the default configuration for the Bitnami NGINX chart:
helm show values bitnami/nginx
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1053).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1053-A).png)

- This prints all configurable parameters (replicaCount, service type, resources, etc.).

### Inline Overrides
```bash
#Install a release with custom values directly via --set:
helm install custom-nginx bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1054).png)

- This overrides the default replica count (1 → 3) and service type (ClusterIP → NodePort).
- Good for quick one-off overrides. Not recommended for complex configurations or version-controlled environments.

### Create a Values File `custom-values.yaml`:
```yaml
replicaCount: 3

service:
  type: NodePort

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```
- This allows us to manage overrides in a reusable file.

### Install with Values File
```bash
#Deploy another release using the file:
helm install file-nginx bitnami/nginx -f custom-values.yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1055).png)

### Inspect Overrides
```bash
#Check what values were applied:
helm get values file-nginx
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1056).png)

- Add `--all` to see defaults + overrides:
    ```bash
    helm get values file-nginx --all
    ```
    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1057).png)

### Verification
```bash
kubectl get pods    # 3 replicas running
kubectl get svc     # Service type: NodePort
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1058).png)

Expected:
- Pods: 3 replicas running.
- Service type: NodePort (instead of ClusterIP).

### `--set` vs values file 
| Method | Best for | Version control friendly |
|---|---|---|
| `--set` | Quick tests, single overrides | ✗ No |
| `-f values.yaml` | Multi-key configs, production | ✓ Yes |

## 5. Upgrade and Rollback
Helm tracks every change to a release as a numbered revision, making upgrades and rollbacks straightforward.
### Upgrade Release
```bash
#Increase replicas from 1 → 5:
helm upgrade my-nginx bitnami/nginx --set replicaCount=5
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1060).png)

- This applies new values to the existing release.
- Helm updates the Deployment manifest and triggers Kubernetes to scale Pods accordingly.

### Check Release History
```bash
#View history:
helm history my-nginx
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1061).png)

We’ll see revisions:
- Revision 1 → initial install (replicaCount=1)
- Revision 2 → upgrade (replicaCount=5)

### Rollback Release
```bash
#Rollback to revision 1:
helm rollback my-nginx 1
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1062).png)

- This reverts the release to its original state (replicaCount=1).
- Helm does not overwrite revision 2 - it creates a **new revision (3)** that matches the state of revision 1. The full history is always preserved.

### Verify Rollback
```bash
#Check history again:
helm history my-nginx
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1063).png)

Expected output:
- Revision 1 → Install (replicaCount=1)
- Revision 2 → Upgrade (replicaCount=5)
- Revision 3 → Rollback to revision 1 (replicaCount=1)

### Verification
```bash
kubectl get pods
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1064).png)

Expected:
- After upgrade → 5 Pods running.
- After rollback, the pod count returns to 1.

### Revision lifecycle 
| Action | Command | Revision created | Pod count |
|---|---|---|---|
| Install | `helm install` | 1 | 1 |
| Upgrade | `helm upgrade --set replicaCount=5` | 2 | 5 |
| Rollback | `helm rollback my-nginx 1` | 3 | 1 |

**How many revisions after rollback?** → 3 revisions (install, upgrade, rollback).

## 6. Create Our Own Chart
### Scaffold a new chart
```bash
helm create my-app
```
This generates the following directory structure:
```
my-app/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default configurable parameters
└── templates/          # Kubernetes manifests with Go template syntax
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    └── ...
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1066).png)

### Explore Directory
Key files:
- **Chart.yaml** → name, version, description
- **values.yaml** → configurable parameters
- **templates/deployment.yaml** → Deployment manifest with Go template syntax like:
```yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1067).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1068).png)

At install time, Helm renders these templates by substituting `.Values.*` with entries from `values.yaml` (or our overrides).

### Edit Values
Open `values.yaml` and set:
```yaml
replicaCount: 3

image:
  repository: nginx
  tag: 1.25
```

### Validate Chart
```bash
helm lint my-app
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1069).png)

- Ensures chart structure and syntax are valid.

### Preview Templates
Render manifests without installing:
```bash
helm template my-release ./my-app
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1070).png)

- Shows the YAML that Helm would apply.

Always run `helm lint` and `helm template` before installing a new or modified chart. `helm template` is especially useful for reviewing exactly what will be applied to the cluster.

### Install Chart
Deploy your chart:
```bash
# Install with 3 replicas
helm install my-release ./my-app
kubectl get pods    # 3 pods running
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1071).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1072).png)

- Creates resources with 3 replicas of NGINX:1.25.

### Upgrade Release
Scale replicas to 5:
```bash
# Upgrade to 5 replicas
helm upgrade my-release ./my-app --set replicaCount=5
kubectl get pods    # 5 pods running
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1073).png)

### Chart development workflow 
```
helm create my-app
       │
       ▼
Edit values.yaml and templates/
       │
       ▼
helm lint my-app          (validate)
       │
       ▼
helm template my-release ./my-app   (preview)
       │
       ▼
helm install my-release ./my-app    (deploy)
       │
       ▼
helm upgrade / rollback as needed
```

## 7. Clean Up
### Uninstall Releases
```bash
#Remove each release we installed:
helm uninstall my-nginx
helm uninstall custom-nginx
helm uninstall file-nginx
helm uninstall my-release
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1076).png)

- Add `--keep-history` if we want to retain release history for auditing:
    ```bash
    helm uninstall my-nginx --keep-history
    ```

### Remove Chart Directory & Values File
```bash
#Delete our custom chart and values file:
rm -rf my-app
rm custom-values.yaml
```

### Verify Cleanup
```bash
#Check that no releases remain:
helm list
kubectl get all
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/670401ffe865f18562f314d16093539dabf350b2/2026/day-59/Screenshots/Screenshot%20(1077).png)

`helm list` should return an empty table. Only system resources in `kube-system` should remain.

### Verification
- **Does `helm list` show zero releases?** → Yes, after uninstalling all releases.
- Chart directory and values file removed from our workspace.
