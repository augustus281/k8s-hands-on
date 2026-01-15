# Kubernetes Horizontal Pod Autoscaler

Configure and test Horizontal Pod Autoscaler to automatically scale applications based on CPU and memory usage.

## Verify Cluster Setup and Metrics Server

### Commands to Run

```bash
# Display general information about the Kubernetes cluster
# Shows the API server and core services
kubectl cluster-info

# List all nodes in the cluster
# -o wide shows additional details such as intern IP, OS, and container runtime
kubectl get nodes -o wide

NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
minikube   Ready    control-plane   89d   v1.32.0   192.168.49.2   <none>        Ubuntu 22.04.5 LTS   6.12.54-linuxkit   docker://27.4.1

# Install Metrics Server in the cluster
# Metrics Server collects CPU and memory usage
# Required for `kubectl top` and Horizontal Pod Autoscaler (HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check the status of the metrics-server deployment
# Ensures the deployment exists in the kube-system namespace
kubectl get deployment metrics-server -n kube-system

# If using Minikube, enable the built-in Metrics Server addon
minikube addons enable metrics-server

# Wait until the metrics-server deployment becomes available
# --timeout=300s means wait up to 5 minutes
kubectl wait --for=condition=available --timeout=300s deployment/metrics-server -n kube-system
```

### Why you must run `minikube addons enable metrics-server`

**1. Minikube does not enable Metrics Server by default**

- Metrics Server is **optional** in Minikube to keep the cluster lightweight.
- Without it, commands like:

```bash
kubectl top nodes
kubectl top pods
```

**will not work**.

**2. The Minikube addon installs a Minikube-compatible Metrics Server**

The addon:

- Deploys Metrics Server with **correct flags** for Minikube
- Automatically sets:
    + `--kubelet-insecure-tls`
    + `--kubelet-preferred-address-types=InternalIP`
- These flags are often **required in local environments** because kubelets use self-signed certificates.

---

## Deploy Sample Application with Resource Limits
### Commands to Run

```bash
kubectl apply -f hpa-demo-app.yaml

kubectl get deployment php-apache

kubectl get pods -l run=php-apache

kubectl get service php-apache
```

---

## Configure Horizontal Pod Autoscaler

Create HPA configuration to automatically scale based on CPU utilization.

### Commands to Run

```bash
kubectl apply -f hpa-config.yaml

kubectl get hpa php-apache-hpa

kubectl describe hpa php-apache-hpa
```

---

## Monitor and Analyze Scaling Behavior

Observe the autoscaling behavior and analyze the scaling events.

### Commands to Run

```bash
# Watch the HPA in real time to observe scaling decisions
# -w keeps the command running and updates on changes
# & runs the command in the background
kubectl get hpa php-apache-hpa -w &

# Continuously monitor pods managed by the php-apache deployment
# This helps verify when replicas are scaled up or down.
watch 'kubectl get pods -l run=php-apache'

# Display detailed information about the HPA
# Includes current metrics, scaling conditions, and recent events
kubectl describe hpa php-apache-hpa

# List cluster events related to the Horizontal Pod Autoscaler
# Sorted by timestamp to show the most recent scaling actions
kubectl get events --sort-by='.lastTimestamp' | grep -i horizontal
```

---

## Test Scale-Down Behavior

Remove load and observe how HPA scales down the application.

### Commands to Run

```bash
# Delete the load-generator deployment to stop generating traffic
# This removes CPU/memory load from the application
kubectl delete deployment load-generator

# Print a message indicating that the load has been removed
# and the system is now waiting for scale-down
echo 'Load removed, waiting for scale-down...'

# Wait for 5 minutes (300 seconds)
# This allows the HPA scale-down stabilization window to expire
sleep 300

# Check the current state of the HPA after the cooldown period
# Verifies whether replicas have been scaled down
kubectl get hpa php-apache-hpa

# List the pods managed by the php-apache deployment
# Confirms the actual number of running replicas
kubectl get pods -l run=php-apache

# Show detailed HPA information
# Includes scale-down decisions, current metrics, and recent events
kubectl describe hpa php-apache-hpa
```