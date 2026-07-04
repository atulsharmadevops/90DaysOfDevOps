# Capstone: Deploy WordPress + MySQL on Kubernetes
A full-stack capstone project that brings together namespaces, Secrets, ConfigMaps, StatefulSets, Deployments, Services, resource limits, health probes, HPA, and Helm - all in a single production-style deployment.
 
## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Create the Namespace](#1-create-the-namespace-day-52)
- [Deploy MySQL](#2-deploy-mysql-days-54-56)
- [Deploy WordPress](#3-deploy-wordpress-days-52-54-57)
- [Expose WordPress](#4-expose-wordpress-day-53)
- [Test Self-Healing and Persistence](#5-test-self-healing-and-persistence)
- [Set Up HPA](#6-set-up-hpa-day-58)
- [Bonus — Compare with Helm](#7-bonus-compare-with-helm-day-59)
- [Clean Up and Reflect](#8-clean-up-and-reflect)

## Architecture Overview
```
                        ┌─────────────────────────────────────┐
                        │         capstone namespace          │
                        │                                     │
 Browser / curl         │   ┌──────────────────────────┐      │
 ──────────────────────►│   │  wordpress (NodePort:    │      │
       :30080           │   │   30080)                 │      │
                        │   └────────────┬─────────────┘      │
                        │                │                    │
                        │   ┌────────────▼─────────────┐      │
                        │   │  WordPress Deployment    │      │
                        │   │  (2 replicas + HPA)      │      │
                        │   │  + Liveness/Readiness    │      │
                        │   │    Probes                │      │
                        │   └────────────┬─────────────┘      │
                        │                │ envFrom            │
                        │   ┌────────────▼─────────────┐      │
                        │   │  ConfigMap + Secret      │      │
                        │   │  (DB host, credentials)  │      │
                        │   └────────────┬─────────────┘      │
                        │                │                    │
                        │   ┌────────────▼─────────────┐      │
                        │   │  mysql (Headless Svc)    │      │
                        │   └────────────┬─────────────┘      │
                        │                │                    │
                        │   ┌────────────▼─────────────┐      │
                        │   │  MySQL StatefulSet       │      │
                        │   │  (mysql-0)               │      │
                        │   │  + PVC (1Gi)             │      │
                        │   └──────────────────────────┘      │
                        └─────────────────────────────────────┘
```
## Concepts used in this capstone
 
| # | Concept | Day |
|---|---|---|
| 1 | Namespace | Day 52 |
| 2 | Secret | Day 54 |
| 3 | ConfigMap | Day 54 |
| 4 | PersistentVolumeClaim | Day 55 |
| 5 | StatefulSet | Day 56 |
| 6 | Headless Service | Day 56 |
| 7 | Deployment | Day 52 |
| 8 | NodePort Service | Day 53 |
| 9 | Resource Requests & Limits | Day 57 |
| 10 | Liveness & Readiness Probes | Day 57 |
| 11 | Horizontal Pod Autoscaler | Day 58 |
| 12 | Helm | Day 59 |

## 1. Create the Namespace (Day 52)
### Create the capstone namespace
Namespaces logically isolate resources in Kubernetes, helping us keep this capstone deployment separate.
```bash
kubectl create namespace capstone
```
- This registers a new namespace named **capstone** in our cluster
- Namespaces are useful for grouping related workloads

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1484).png)

### Set capstone as default
Switching our current context ensures all subsequent commands apply to the capstone namespace without repeating `-n capstone`.
```bash
kubectl config set-context --current --namespace=capstone
```
- This modifies our kubeconfig context to default to capstone - no need to append `-n capstone` to every command.
- Saves time and avoids mistakes when applying manifests

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1485).png)

### Verify namespace creation
Confirm that the namespace exists and is set as default before moving on.
```bash
kubectl get namespaces  #should list capstone

kubectl config view --minify | grep namespace:  #should show capstone
```
- This ensures all future resources will be scoped correctly

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1492).png)

All subsequent manifests (Secrets, StatefulSets, Deployments) will now be created inside this namespace automatically.

## 2. Deploy MySQL (Days 54-56)
MySQL is a stateful workload. It requires stable pod identity, persistent storage, and stable DNS - all provided by a StatefulSet backed by a Headless Service.
 
### Step 1 - Secret (`mysql-secret.yaml`)
`stringData` lets us write plaintext in the YAML. Kubernetes base64-encodes the values automatically on apply.
```yaml
apiVersion: v1                      #tells K8s which API group/version to use - v1 is stable core API for basic resources like Pods, Services, ConfigMaps, and Secrets
kind: Secret                        #stores sensitive data (passwords, tokens, keys) in base64‑encoded form.
metadata:
  name: mysql-secret                #gives this Secret unique name in namespace - Pods, Deployments, or StatefulSets will reference this name when they need DB credentials
stringData:                         #convenience field - we can write plain text values, and K8s will automatically encode them into base64 when storing
  MYSQL_ROOT_PASSWORD: rootpass     #root account password for MySQL
  MYSQL_DATABASE: wordpress         #database WordPress will use
  MYSQL_USER: wpuser                #non‑root user for WordPress
  MYSQL_PASSWORD: wppass            #password for that user
```
```bash
#Apply
kubectl apply -f mysql-secret.yaml

#Verify
kubectl get secret mysql-secret
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1495).png)

### Step 2 - Headless Service (`mysql-service.yaml`)
`clusterIP: None` creates individual DNS records per pod instead of a single load-balanced IP - required for StatefulSet stable identity.
```yaml
apiVersion: v1
kind: Service           #provides networking and DNS for Pods
metadata:
  name: mysql           #WordPress deployment will reference this name when connecting to database
spec:
  clusterIP: None       #makes this a Headless Service - instead of giving a single stable ClusterIP, K8s creates DNS records for each Pod (mysql-0.mysql, mysql-1.mysql, etc.) - required for StatefulSets
  ports:                #defines which port Service exposes
    - port: 3306        #default MySQL port
      name: mysql       #naming port 'mysql' is optional but useful for clarity and when referencing in other manifests
  selector:             #tells Service which Pods to route traffic to
    app: mysql          #it matches Pods with the label 'app=mysql' - in our StatefulSet, each MySQL Pod has this label, so the Service knows where to send connections
```
```bash
#Apply
kubectl apply -f mysql-service.yaml

#Verify
kubectl get svc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1498).png)

### Step 3 - StatefulSet (`mysql-statefulset.yaml`)
This guarantees stable pod identity and persistent storage.
```yaml
apiVersion: apps/v1                           #API group - required for controllers like Deployments, StatefulSets, and DaemonSets
kind: StatefulSet                             #StatefulSet - manages Pods with stable names (mysql-0, mysql-1), ordered startup, and persistent storage
metadata:
  name: mysql                                 #names StatefulSet 'mysql' - used in pod naming (mysql-0) and PVC naming
spec:
  serviceName: mysql                          #associates StatefulSet with Headless Service (mysql) - ensures Pods get DNS entries like 'mysql-0.mysql'
  replicas: 1                                 #runs one MySQL Pod (mysql-0) - for WordPress, single DB instance is enough
  selector:
    matchLabels:
      app: mysql                              #matches Pods with label 'app=mysql' - ensures StatefulSet manages correct Pods
  template:
    metadata:
      labels:
        app: mysql                            #applies label 'app=mysql' to Pods created by this StatefulSet - must match selector above
    spec:
      containers:                             #defines MySQL container
      - name: mysql
        image: mysql:8.0                      #pulls official MySQL 8.0 image
        envFrom:
        - secretRef:
            name: mysql-secret                #loads environment variables from Secret 'mysql-secret' - provides 'MYSQL_ROOT_PASSWORD', 'MYSQL_DATABASE', 'MYSQL_USER', 'MYSQL_PASSWORD'.
        resources:                            #defines resource requests and limits - prevents MySQL from consuming too much cluster capacity
          requests:                           #minimum guaranteed (250m CPU, 512Mi RAM)
            cpu: 250m
            memory: 512Mi
          limits:                             #maximum allowed (500m CPU, 1Gi RAM)
            cpu: 500m
            memory: 1Gi
        ports:
        - containerPort: 3306
          name: mysql                         #exposes MySQL on port 3306 inside container - named 'mysql' for clarity
        volumeMounts:                         #mounts persistent storage
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql           #this is where MySQL stores its data files - ensures persistence across pod restarts
  volumeClaimTemplates:                       #defines a PersistentVolumeClaim (PVC) template
  - metadata:
      name: mysql-persistent-storage          #each Pod gets its own PVC named 'mysql-persistent-storage-mysql-0'
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi                        #requests 1Gi of storage, with access mode ReadWriteOnce (mounted by one node at a time) - ensures MySQL data persists even if Pod is deleted
```
```bash
#Apply
kubectl apply -f mysql-statefulset.yaml

#Verify
kubectl get pods -l app=mysql
kubectl get pvc
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1500).png)

###  Test MySQL
Exec into the pod and check the database:
```bash
kubectl exec -it mysql-0 -- mysql -u wpuser -pwppass -e "SHOW DATABASES;"
```
- Verify: Output should include `wordpress` in the database list.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1503).png)

## 3. Deploy WordPress (Days 52, 54, 57)
WordPress is a stateless workload - it reads configuration from environment variables and connects to MySQL for persistence. Two replicas run behind a Service, with liveness and readiness probes ensuring only healthy pods receive traffic.
 
### Step 1 - ConfigMap (`wordpress-config.yaml`)
Non-sensitive connection settings go in a ConfigMap, not a Secret.
```yaml
apiVersion: v1
kind: ConfigMap                                                     #ConfigMap - stores non‑sensitive key‑value pairs (unlike Secrets, which store sensitive data)
metadata:
  name: wordpress-config                                            #names ConfigMap 'wordpress-config' - WordPress Deployment will reference this name to load DB connection details
data:                                                               #contains key‑value pairs
  WORDPRESS_DB_HOST: mysql-0.mysql.capstone.svc.cluster.local:3306 #points WordPress to MySQL Pod - 'mysql-0.mysql.capstone.svc.cluster.local:3306' is fully qualified DNS name of first MySQL Pod (mysql-0) via Headless Service (mysql) in capstone namespace - port 3306 is default MySQL port
  WORDPRESS_DB_NAME: wordpress                                     #specifies database WordPress should use 'wordpress' - this must match database created by MySQL using Secret
```
```bash
#Apply
kubectl apply -f wordpress-config.yaml

#Verify
kubectl get configmap wordpress-config
```

> `mysql-0.mysql.capstone.svc.cluster.local` is the stable DNS name provided by the Headless Service for the `mysql-0` StatefulSet pod.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1506).png)

### Step 2 - Deployment (`wordpress-deployment.yaml`)
This runs 2 replicas, pulls DB credentials from the Secret, and includes probes.
```yaml
apiVersion: apps/v1                         #uses apps/v1 API group, required for Deployments, StatefulSets, etc
kind: Deployment                            #defines this resource as a Deployment, which manages ReplicaSets and Pods, ensuring desired number of replicas are running
metadata:
  name: wordpress                           #names deployment 'wordpress' - pods created will inherit this name prefix
spec:
  replicas: 2                               #runs 2 WordPress Pods for redundancy and load balancing
  selector:
    matchLabels:
      app: wordpress                        #ensures deployment manages Pods with label 'app=wordpress'
  template:
    metadata:
      labels:
        app: wordpress                      #applies 'app=wordpress' label to Pods created by this deployment - must match selector
    spec:
      containers:                           #defines WordPress container
      - name: wordpress
        image: wordpress:latest             #pulls latest WordPress image from Docker Hub
        envFrom:
        - configMapRef:
            name: wordpress-config          #loads environment variables from ConfigMap 'wordpress-config' (DB host and DB name)
        env:                                #loads sensitive values from Secret 'mysql-secret'
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_USER
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: MYSQL_PASSWORD
        resources:                          #defines resource requests and limits
          requests:                         #minimum guaranteed (250m CPU, 512Mi RAM)
            cpu: 250m
            memory: 512Mi
          limits:                           #maximum allowed (500m CPU, 1Gi RAM)
            cpu: 500m
            memory: 1Gi
        ports:                              #exposes WordPress on port 80 inside container
        - containerPort: 80
        startupProbe:                       #checks '/wp-admin/install.php' during installation phase
          httpGet:
            path: /wp-admin/install.php
            port: 80
          failureThreshold: 30              #allows up to 5 minutes for WordPress to initialize before marking pod unhealthy
          periodSeconds: 10
          timeoutSeconds: 5
        livenessProbe:                      #ensures container is alive by checking '/wp-login.php'
          httpGet:
            path: /wp-login.php
            port: 80
          initialDelaySeconds: 60
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3               #if it fails 3 times in a row - K8s restarts pod
        readinessProbe:                     #ensures pod is ready to serve traffic - pod is added to Service only when this probe passes
          httpGet:
            path: /wp-login.php             #checks '/wp-login.php' after 30s, every 5s
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 5
```
```bash
#Apply
kubectl apply -f wordpress-deployment.yaml

#Check pods
kubectl get pods -l app=wordpress
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1509).png)

Check probes:
```bash
kubectl describe pod <wordpress-pod-name> | grep -A3 Liveness
```
- Expect: HTTP GET `/wp-login.php` on port 80.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1510).png)

### Configuration injection summary
| Environment Variable | Source | Method |
|---|---|---|
| `WORDPRESS_DB_HOST` | ConfigMap | `envFrom.configMapRef` |
| `WORDPRESS_DB_NAME` | ConfigMap | `envFrom.configMapRef` |
| `WORDPRESS_DB_USER` | Secret | `env.valueFrom.secretKeyRef` |
| `WORDPRESS_DB_PASSWORD` | Secret | `env.valueFrom.secretKeyRef` |

## 4. Expose WordPress (Day 53)
### Create the NodePort Service `wordpress-service.yaml`:
This exposes WordPress on port `30080` of each node.
```yaml
apiVersion: v1
kind: Service             #service - provides stable networking and DNS for Pods
metadata:
  name: wordpress         #names Service wordpress - this name is used for DNS resolution inside cluster 'wordpress.capstone.svc.cluster.local'
spec:
  type: NodePort          #exposes Service on a port of each Node (between 30000–32767) - allows external access from outside cluster using '<NodeIP>:<NodePort>'
  selector:
    app: wordpress        #routes traffic to Pods with label 'app=wordpress' - matches Pods created by our WordPress deployment
  ports:                  #Defines how traffic flows
    - port: 80            #service’s internal port
      targetPort: 80      #container port inside WordPress Pods
      nodePort: 30080     #external port on each Node
```
```bash
#Apply
kubectl apply -f wordpress-service.yaml

#Verify
kubectl get svc wordpress
```
- Expect: `TYPE=NodePort`, `PORT(S)=80:30080/TCP`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1576).png)

### Access WordPress
Minikube
```bash
minikube service wordpress -n capstone
```
- Opens browser with WordPress setup page.

Kind
```bash
kubectl port-forward --address 0.0.0.0 svc/wordpress 8080:80 -n capstone
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1515).png)

- Add `--address 0.0.0.0` - without it, port-forward only binds to `localhost` on the EC2 instance, which is unreachable from our browser even if port 8080 is open in the security group.

  Then:
  - Open port `8080` (not `30080`) in our EC2 security group.

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1517).png)

- Visit `http://<ec2-public-ip>:8080`.

### Complete the WordPress Setup Wizard 
- #### Site Title - `My Capstone Project` (It appears in the browser tab and at the top of pages)
- #### Username - `atulsharma`
- #### Password - Store it safely (this is how we log in to /wp-admin)
- #### Your Email - `s.atul47@yahoo.in`
- #### Search Engine Visibility (checkbox)
  - If checked, WordPress discourages search engines from indexing our site.
  - For our capstone project, we can leave it **unchecked**.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1522).png)

### Create a Test Blog Post
#### Log in to WordPress Admin

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1523).png)

#### Navigate to Posts
- In the left sidebar, click **Posts** → **Add New**.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1525).png)

- Write a Test Post
  - Title: `Persistence Test Post`
  - Content: `This is a test post to verify persistence across pod deletions.`
  - **Publish** the Post

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1529).png)

- Click **Publish** → **Publish** again to confirm.
- Our post is now live.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1530).png)

### Verify on Frontend
- Visit:
  ```Code
  http://<EC2_Public_IP>:30080
  ```
  - We should see our test post listed on the homepage.

## 5. Test Self-Healing and Persistence
### Test 1 - WordPress pod self-healing (Deployment)
Pick one of the WordPress pods and delete it:
```bash
kubectl get pods -l app=wordpress
kubectl delete pod <wordpress-pod-name>
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1531).png)

Verify:
```bash
kubectl get pods -l app=wordpress -w
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1533).png)

The deleted pod is replaced within seconds with a new pod (different suffix). The WordPress site remains accessible throughout - the Service routes traffic to the surviving replica while the replacement starts.

### Test 2 - MySQL pod persistence (StatefulSet)
```bash
#Delete the single MySQL pod
kubectl delete pod mysql-0

#Verify
kubectl get pods -l app=mysql -w
```
The StatefulSet recreates `mysql-0` with the same name and automatically reattaches it to the same PVC. All database data is preserved.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1535).png)

### Verify persistence
After `mysql-0` recovers, reload WordPress in browser and log in. The test blog post we created should still be there - confirming that persistent storage survived the pod deletion.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1536).png)

### Self-healing behavior comparison
| Component | Controller | Pod name after recreation | Data preserved |
|---|---|---|---|
| WordPress | Deployment | New random suffix | N/A (stateless) |
| MySQL | StatefulSet | Same (`mysql-0`) | ✓ Yes (via PVC) |

## 6. Set Up HPA (Day 58)
The HPA automatically scales the WordPress Deployment between 2 and 10 replicas based on CPU utilization.
 
### Manifest - `wordpress-hpa.yaml`
Target the WordPress Deployment, set CPU utilization to 50%, with min 2 and max 10 replicas.
```yaml
apiVersion: autoscaling/v2          #v2 supports CPU, memory, and custom metrics - unlike v1 which only supports CPU
kind: HorizontalPodAutoscaler       #HPA - automatically adjusts number of replicas in a Deployment, StatefulSet, or ReplicaSet
metadata:
  name: wordpress-hpa               #names HPA 'wordpress-hpa' - this name is used when checking status 'kubectl get hpa wordpress-hpa'
spec:
  scaleTargetRef:                   #defines target resource to scale
    apiVersion: apps/v1             #here it points to WordPress Deployment (apps/v1, Deployment, wordpress) - HPA will increase/decrease replicas of this Deployment
    kind: Deployment
    name: wordpress
  minReplicas: 2                    #ensures at least 2 WordPress Pods are always running
  maxReplicas: 10                   #allows scaling up to 10 Pods when load increases
  metrics:                          #defines scaling metric
  - type: Resource                  #uses a built‑in resource metric
    resource:
      name: cpu                     #monitors CPU usage
      target:
        type: Utilization           #uses percentage of requested CPU
        averageUtilization: 50      #scales Pods so that average CPU usage stays around 50% of requested CPU - if usage > 50%, HPA adds replicas - if usage < 50%, HPA reduces replicas (but never below 2)
```
```bash
#Apply
kubectl apply -f wordpress-hpa.yaml

#Check the HPA resource
kubectl get hpa
```
Expected output:
| Field | Value | Meaning |
|---|---|---|
| `minReplicas` | 2 | Never scale below 2 replicas |
| `maxReplicas` | 10 | Never scale above 10 replicas |
| `averageUtilization` | 50% | Scale up when avg CPU > 50% of the 250m request (= 125m) |

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1540).png)

### Get Complete Picture
```bash
kubectl get all -n capstone
```
This shows:
- WordPress Deployment with 2 replicas
- MySQL StatefulSet with 1 pod
- Services (WordPress NodePort, MySQL headless)
- HPA resource attached to WordPress

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1542).png)

### Verification
- HPA lists correct min/max values.
- Target utilization shows `50%`.
- WordPress pods remain at 2 until load increases.

## 7. (Bonus) Compare with Helm (Day 59)
This optional task installs the same WordPress + database stack using a Helm chart, then compares the experience against the manual YAML approach.

### Create a Separate Namespace
Keep Helm resources isolated from our capstone namespace.
```bash
kubectl create namespace helm-test

#Verify
kubectl get ns
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1545).png)

### Install WordPress via Helm
Use the Bitnami chart, which bundles WordPress + MariaDB.
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install wp-helm bitnami/wordpress -n helm-test
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1549).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1551).png)

### Verify:
```bash
kubectl get all -n helm-test
```
Helm creates the Deployment, StatefulSet (MariaDB), Services, PVCs, Secrets, and ConfigMaps automatically.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1553).png)

### Manual YAML vs Helm — side by side
| Dimension | Manual YAML (`capstone`) | Helm (`helm-test`) |
|---|---|---|
| Files to manage | ~8 individual manifests | 1 `helm install` command |
| Configuration control | Full — every field explicit | Via `--set` or `values.yaml` |
| Resource tuning | Directly in YAML | Requires knowing chart value names |
| Probe customization | Full control | Depends on chart support |
| Version control | ✓ Manifests are files | ✓ Chart + values file |
| Upgrade / rollback | Manual `kubectl apply` | `helm upgrade` / `helm rollback` |
| Best for | Learning, full control | Speed, standardized stacks |

### Verification: Count resources in each namespace with
```bash
kubectl get all -n capstone
kubectl get all -n helm-test
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1555).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1557).png)

| Aspect | Manual (``capstone``) | Helm (``helm-test``) |
| --- | --- | --- |
| **Setup** | Multiple YAML files applied manually | One ``helm ``install`` command |
| **Database** | MySQL StatefulSet + Secret | MariaDB StatefulSet auto‑created |
| **Service Type** | NodePort (fixed port 30080) | LoadBalancer (external IP pending) |
| **Scaling** | Explicit HPA defined | Replica count in values.yaml (HPA optional) |
| **Persistence** | PVC manually defined | PVC auto‑created by chart |
| **Transparency** | Full control, educational | Abstracted, faster, production‑ready |
| **Best Practices** | You decide | Bitnami pre‑configures |

### Clean Up Helm Deployment
```bash
helm uninstall wp-helm -n helm-test
kubectl delete namespace helm-test
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1560).png)

Verify:
```bash
kubectl get ns
```
- Expect: `helm-test` removed.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1561).png)

## 8. Clean Up and Reflect
### Review everything before deletion
 
```bash
kubectl get all -n capstone
kubectl get pvc -n capstone
kubectl get secret -n capstone
kubectl get configmap -n capstone
kubectl get hpa -n capstone
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1563).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1569).png)

### Delete the entire namespace
 
```bash
kubectl delete namespace capstone
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1570).png)

Deleting the namespace removes every resource inside it - Pods, Deployments, StatefulSets, Services, ConfigMaps, Secrets, and HPAs - in a single command.
 
> **Note:** PVCs are also deleted when the namespace is removed. In production, always back up database data before deleting a namespace.
 
### Reset the default namespace
 
```bash
kubectl config set-context --current --namespace=default
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1571).png)

### Final verification
 
```bash
kubectl get ns
# Expect: only default, kube-system, kube-public, kube-node-lease
 
kubectl get pods -A
# Expect: only system pods (coredns, etcd, kube-apiserver, etc.)
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/f4cedb40113e7cd40cb15e162e6b2d22eaa3341e/2026/day-60/Screenshots/Screenshot%20(1574).png)


### What this capstone covered
 
Every manifest in this project maps directly to a concept introduced in Days 50–59:
 
```
Namespace (Day 52)
  └── Secret + ConfigMap (Day 54)
        └── PVC via volumeClaimTemplates (Day 55)
              └── StatefulSet + Headless Service (Day 56)
                    └── Deployment + NodePort Service (Day 52, 53)
                          └── Resource Limits + Probes (Day 57)
                                └── HPA (Day 58)
                                      └── Helm comparison (Day 59)
```
 
Each layer builds on the previous one - from storage and identity, through configuration and exposure, to reliability and scale.