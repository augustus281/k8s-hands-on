# Kubernetes Gateway API: A Hands-On Guide to Modern Traffic Management

## Prerequisites:

- A running Kubernetes cluster (currently using minikube)
- `kubectl` configured to access your cluster
- `helm` package manager installed

## Step by step:

### Step 1: Install Gateway API CRDs

```shell
# Install Gateway API Standard Channel (GA and Beta resources)
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

Verify the installation:

```shell
kubectl get crd | grep gateway
```

### Step 2: Deploy the Sample Application

```shell
kubectl apply -f deployment/
```

### Step 3: Install Traefik with Gateway API Support

Traefik will serve as our Gateway Controller. 

**Add Traefik Helm Repository**

```shell
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

**Configure Traefik Values**: Create a values file to enable Gateway API support

**Install Traefik**

```shell
helm install traefik traefik/traefik \
  --values traefik/values.yaml \
  --namespace traefik \
  --create-namespace
```

Verify the installation:

```shell
kubectl get pods -n traefik
kubectl get svc -n traefik
```

### Step 4: Create Gateway API Resources

**Create GatewayClass**
The GatewayClass defines which controller will handle our Gateway

**Create Gateway**
The Gateway defines the network entry points

Apply the Gateway resources:

```shell
kubectl apply -f gateway-class.yaml
kubectl apply -f gateway.yaml
```

Verify the Gateway status:

```shell
kubectl get gateway
kubectl describe gateway gateway-api
```

### Step 5: Implement Hostname-Based Routing

```shell
kubectl apply -f http-route-by-hostname.yaml
```

### Step 6: Implement Path-Based Routing (Exact Match)

```shell
kubectl apply -f httproute-by-path-exact.yaml
```

### Step 7: Implement URL Rewriting
Create an HTTPRoute with URL rewriting capabilities

```shell
kubectl apply -f httproute-by-path-rewrite.yaml
```

### Step 8: Configure Local Testing with /etc/hosts

**Get the Gateway External IP**
For kind cluster, get the node IP:

```shell
# Get kind cluster node IP
kubectl get nodes -o wide
# Or get the service details
kubectl get svc -n traefik traefik
```

**Update `/etc/hosts` File**

Edit your `/etc/hosts` file:

```shell
sudo nano /etc/hosts
```

Add the following line (replace with your actual kind node IP):

```shell
127.0.0.1       demo.apigateway.com
```