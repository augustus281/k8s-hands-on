# KGATEWAY

## Testing KGATEWAY with a Simple Application

We will deploy a simple app (httpbin) to test KGATEWAY.

### 1️⃣ Deploy the test application

```bash
kubectl apply -f https://raw.githubusercontent.com/kgateway-dev/kgateway/refs/heads/v2.1.x/examples/httpbin.yaml

namespace/httpbin created
serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created
```

### 2️⃣ Verify that the pod is running

```bash
kubectl -n httpbin get pods
```

### 3️⃣ Verify that the HTTPRoute is applied successfully

```bash
kubectl get httproute/httpbin -n httpbin -o yaml
```

### 4️⃣ Get the gateway address

#### Option A: If using a cloud cluster with LoadBalancer support

```bash
export INGRESS_GW_ADDRESS=$(kubectl get svc -n kgateway-system http -o=jsonpath="{.status.loadBalancer.ingress[0].ip}")
echo $INGRESS_GW_ADDRESS
```

#### Option B: If using Minikube / Docker driver (macOS / Windows)

**Use port-forward to expose the gateway locally**

```bash
kubectl port-forward svc/http 8080:8080 -n kgateway-system
export INGRESS_GW_ADDRESS=localhost
```

⚠️ Make sure the Host header does not include the port

```bash
curl -i http://$INGRESS_GW_ADDRESS:8080/headers -H "Host: www.example.com"
```

**Notes:**
- On Minikube with Docker driver, running minikube service http --url requires the terminal to remain open.
- Using kubectl port-forward is usually simpler and works across all environments. 
- Always ensure the Host header matches the HTTPRoute rules.