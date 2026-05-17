# Kubernetes ConfigMaps and Secrets
A hands-on guide to externalizing configuration and sensitive data from application containers using ConfigMaps and Secrets.
 
## Table of Contents
- [Create a ConfigMap from Literals](#1-create-a-configmap-from-literals)
- [Create a ConfigMap from a File](#2-create-a-configmap-from-a-file)
- [Use ConfigMaps in a Pod](#3-use-configmaps-in-a-pod)
- [Create a Secret](#4-create-a-secret)
- [Use Secrets in a Pod](#5-use-secrets-in-a-pod)
- [Live ConfigMap Updates](#6-update-a-configmap-and-observe-propagation)
- [Clean Up](#7-clean-up)

## 1. Create a ConfigMap from Literals
### Create the ConfigMap
Run:
```bash
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_PORT=8080
```
- `--from-literal` - adds a key/value pair directly from the command line.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(757).png)

### Inspect the ConfigMap
Check details:
```bash
kubectl describe configmap app-config
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(758).png)

### YAML view:
```bash
kubectl get configmap app-config -o yaml
```
- `-o yaml` - outputs the resource definition in YAML format (instead of the default table view).

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(761).png)

## 2. Create a ConfigMap from a File
### Write the Nginx Config
Create a file named `default.conf` with a `/health` endpoint:
```nginx
server {                    #defines server block in Nginx, each block describes how Nginx should handle requests for a particular domain, IP, or port
  listen 80;                #tells Nginx to listen for incoming HTTP requests on port 80 (the default port for HTTP), this means if we visit http://<server-ip>/, Nginx will process the request using this block
  location /health {        #defines location block inside the server, it matches requests with the path /health - http://<server-ip>/health will be handled here.
    return 200 'healthy';   #immediately returns an HTTP 200 OK response, this is a simple way to create a health check endpoint for monitoring systems (like Kubernetes readiness/liveness probes).
  }
}
```
### Create the ConfigMap
Run:
```bash
kubectl create configmap nginx-config --from-file=default.conf=default.conf
```
- This command is creating a **ConfigMap from a file** instead of literals.
- This means the contents of the file `default.conf` will be stored under the key `default.conf` inside the ConfigMap.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(764).png)

| Argument | Meaning |
|---|---|
| `nginx-config` | Name of the ConfigMap |
| `default.conf` (left of `=`) | Key name used inside the ConfigMap |
| `default.conf` (right of `=`) | Local file to read the value from |

### Inspect the ConfigMap
```bash
kubectl get configmap nginx-config -o yaml
```
### Expected snippet:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(765).png)

The file contents are stored under the `data` section. When mounted into a Pod, the key name (`default.conf`) becomes the filename at the mount path.

## 3. Use ConfigMaps in a Pod
There are two ways to consume a ConfigMap in a Pod: as environment variables or as a mounted volume file.
### Method A - Inject as Environment Variables
`busybox-configmap.yaml` uses `envFrom` to pull all keys from `app-config` directly into the container's environment:
```yaml
apiVersion: v1                #uses core Kubernetes API group, version v1
kind: Pod
metadata:
  name: busybox-configmap     #pod’s name in cluster
spec:                         #defines how Pod should run
  containers:
  - name: busybox             #container’s name
    image: busybox            #uses lightweight BusyBox image
    command: ["sh", "-c", "echo APP_ENV=$APP_ENV && echo APP_DEBUG=$APP_DEBUG && echo APP_PORT=$APP_PORT && sleep 3600"]  #overrides default command - runs a shell (sh -c) that - prints values of environment variables [APP_ENV, APP_DEBUG, and APP_PORT] - then sleeps for 3600 seconds (1 hour) to keep container alive.
    envFrom:
    - configMapRef:           #pulls environment variables from a ConfigMap
        name: app-config
```
When we run:
```bash
kubectl apply -f busybox-configmap.yaml
kubectl logs busybox-configmap
```
We should see:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(767).png)

### Method B - Mount as a Volume File
`nginx-configmap.yaml` mounts the `nginx-config` ConfigMap into the container's filesystem at `/etc/nginx/conf.d`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configmap                   #pod’s name in cluster
spec:
  containers:
  - name: nginx                           #container’s name
    image: nginx                          #uses official Nginx image
    volumeMounts:                         #mounts volume into container’s filesystem
    - name: nginx-config-volume           #refers to volume defined below
      mountPath: /etc/nginx/conf.d        #inside container this directory will contain files from ConfigMap - since our ConfigMap has a key default.conf, Kubernetes will create a file /etc/nginx/conf.d/default.conf with contents of our Nginx config
  volumes:                                #defines volumes available to Pod
  - name: nginx-config-volume             #matches volumeMount above
    configMap:                            #source of volume is a ConfigMap
      name: nginx-config                  #refers to ConfigMap we created earlier with - kubectl create configmap nginx-config --from-file=default.conf=default.conf.
```
### Verification
Apply both manifests:
```bash
kubectl apply -f nginx-configmap.yaml
```
Test the `/health` endpoint:
```bash
kubectl exec nginx-configmap -- curl -s http://localhost/health
```
Expected output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(769).png)

### Comparison
| Method | How | Best for |
|---|---|---|
| `envFrom` / `env` | Keys become environment variables | Simple key-value settings |
| Volume mount | Keys become files at a mount path | Config files, multi-line content |

## 4. Create a Secret
### Create the Secret
Run:
```bash
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cureP@ssw0rd
```
- `generic` - indicates we’re creating a generic secret (key/value pairs)
- `db-credentials` - name of Secret

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(771).png)

### Inspect the Secret
```bash
kubectl get secret db-credentials -o yaml
```
### Expected snippet:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(772).png)

Values are **base64-encoded**, not encrypted. Decode any value to confirm:
```bash
echo 'czNjdXJlUEBzc3cwcmQ=' | base64 --decode
```
Output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(773).png)

### ConfigMap vs Secret - what is actually different?
| Property | ConfigMap | Secret |
|---|---|---|
| Storage format | Plain text | Base64-encoded |
| Encrypted at rest | No (by default) | Optional (via EncryptionConfiguration) |
| Node storage | Regular disk | `tmpfs` (in-memory, not written to disk) |
| Access control | Standard RBAC | RBAC with stricter conventions |
| Intended for | Non-sensitive config | Credentials, tokens, certificates |
 
> **Important:** Base64 is encoding, not encryption. Anyone with access to the cluster can decode a Secret value. The real security benefits come from RBAC separation, `tmpfs` storage on nodes, and optional encryption at rest - not from the encoding itself.

## 5. Use Secrets in a Pod
This Pod demonstrates both consumption methods simultaneously - `DB_USER` is injected as an environment variable, and the full Secret is mounted as a volume.
### Manifest - `secret-pod.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod                              #Pod’s name
spec:                                           #defines how Pod should run
  containers:
  - name: busybox                               #container’s name
    image: busybox                              #uses lightweight BusyBox image
    command: ["sh", "-c", "echo DB_USER=$DB_USER && cat /etc/db-credentials/DB_PASSWORD && sleep 3600"]
    env:                                        #defines environment variables for container
    - name: DB_USER                             #environment variable name inside container
      valueFrom:
        secretKeyRef:                           #pulls value from a Secret
          name: db-credentials                  #refers to Secret we created
          key: DB_USER                          #uses DB_USER key from that Secret
    volumeMounts:                               #mounts a volume into container’s filesystem
    - name: secret-volume                       #refers to volume defined below
      mountPath: /etc/db-credentials            #inside container, this directory will contain files from Secret
      readOnly: true                            #ensures container cannot modify Secret files
  volumes:                                      #defines volumes available to Pod
  - name: secret-volume                         #matches volumeMount above
    secret:                                     #source of volume is a Secret
      secretName: db-credentials                #refers to Secret we created earlier
```
### Apply and Verify
Apply the manifest:
```bash
kubectl apply -f secret-pod.yaml
```

Check logs for the environment variable:
```bash
kubectl logs secret-pod
```
Expected:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(777).png)

Inspect mounted files:
```bash
kubectl exec secret-pod -- ls /etc/db-credentials
```
Output:
```Code
DB_USER
DB_PASSWORD
```
Read file contents:
```bash
kubectl exec secret-pod -- cat /etc/db-credentials/DB_PASSWORD
```
Output:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(779).png)

> Kubernetes automatically decodes base64 values when mounting Secrets as files. The container always sees the original plaintext value.

## 6. Update a ConfigMap and Observe Propagation
This exercise demonstrates a key behavioral difference between the two consumption methods: **volume-mounted ConfigMaps update automatically; environment variables do not**.
### Create the ConfigMap
Initialize a ConfigMap with a simple key-value pair.
```bash
kubectl create configmap live-config --from-literal=message=hello
```
Confirm with:
```bash
kubectl get configmap live-config -o yaml
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(783).png)

### Manifest - `live-config-pod.yaml`
The Pod mounts `live-config` as a volume and reads the `message` file every 5 seconds:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: live-config-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "while true; do cat /etc/live/message; sleep 5; done"]    #overrides default command - runs a shell (sh -c) that - continuously prints contents of /etc/live/message every 5 seconds - keeps container alive indefinitely with while true
    volumeMounts:               #mounts volume into container’s filesystem
    - name: live-volume         #refers to volume defined below
      mountPath: /etc/live      #inside container this directory will contain files from ConfigMap - configMap key message will appear as a file /etc/live/message
  volumes:                      #defines volumes available to Pod
  - name: live-volume           #matches volumeMount above
    configMap:                  #source of volume is a ConfigMap
      name: live-config         #refers to ConfigMap we created earlier with - kubectl create configmap live-config --from-literal=message=hello.
```
Apply it:
```bash
kubectl apply -f live-config-pod.yaml
```
### Observe Initial Output
Check logs:
```bash
kubectl logs -f live-config-pod
```
Expected output every 5 seconds:

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(787).png)

### Update the ConfigMap
Patch the value:
```bash
kubectl patch configmap live-config --type merge -p '{"data":{"message":"world"}}'
```
Within **30–60 seconds**, the Pod logs will automatically switch from `hello` to `world` - no Pod restart required.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(788).png)

### Propagation behavior
| Consumption method | Updates automatically? | Requires Pod restart? |
|---|---|---|
| Volume mount | ✓ Yes (within ~30–60s) | No |
| Environment variable (`envFrom` / `env`) | ✗ No | Yes |
 
This is why dynamic configuration (feature flags, messages, tunable parameters) should be mounted as files rather than injected as environment variables.

## 7. Clean Up
### Delete Pods
```bash
#Remove all Pods we created for ConfigMaps and Secrets:
kubectl delete pod busybox-configmap
kubectl delete pod nginx-configmap
kubectl delete pod secret-pod
kubectl delete pod live-config-pod

#Or simply:
kubectl delete pod --all
#(Be careful: this deletes every Pod in the namespace, not just the ones from this exercise.)
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(789).png)

### Delete ConfigMaps
```bash
#Remove the ConfigMaps we created:
kubectl delete configmap app-config
kubectl delete configmap nginx-config
kubectl delete configmap live-config
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(791).png)

### Delete Secrets
```bash
#Remove the Secret:
kubectl delete secret db-credentials
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(792).png)

### Verification
```bash
#Check that everything is gone:
kubectl get pods
kubectl get configmaps
kubectl get secrets
```

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/0dd603acc669ba3860f379d5f74e9d5b370c14e5/2026/day-54/Screenshots/Screenshot%20(793).png)

Only the built-in `kube-root-ca.crt` ConfigMap and `default-token` Secret (if present) should remain in the default namespace.