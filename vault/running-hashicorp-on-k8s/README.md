# Running HashiCorp Vault on Kubernetes

## Installing Vault with Helm

### Prerequisites

To complete the exercise, you’ll need:

- a running `kind` cluster
- `kubectl` installed
- `helm` installed

First, add the HashiCorp Helm repository:

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

Install Vault:

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dev.enabled=true
```

This creates the `vault` namespace automatically. As we discussed in the previous article, server **dev** mode is suitable for testing purposes.

Check that Vault is running:

```bash
kubectl get pods -n vault

NAME                                   READY   STATUS    RESTARTS   AGE
vault-0                                1/1     Running   0          115s
```

## Enabling Kubernetes Authentication

Vault does not trust Kubernetes automatically, so we must enable Kubernetes authentication first.

```bash
kubectl exec -n vault vault-0 -- vault auth enable kubernetes
```

Verify it:

```bash
kubectl exec -n vault vault-0 -- vault auth list
```

## Creating a Test Secret

Let’s store a simple secret for a demo application:

```bash
kubectl exec -n vault vault-0 -- vault kv put secret/my-app/config \
  username="demo-user" \
  password="demo-password"
```

Verify it:

```bash
kubectl exec -n vault vault-0 -- vault kv get secret/my-app/config
```

## Creating a Policy for the App

Vault never gives access without a policy.

This policy allows read access only to secrets under `secret/my-app`:

```bash
kubectl exec -n vault -i vault-0 -- vault policy write my-app - <<EOF
path "secret/data/my-app/*" {
  capabilities = ["read"]
}
EOF
```

```bash
kubectl exec -n vault vault-0 -- vault policy read my-app
```

## Connecting Vault to the Kubernetes API

Vault must verify the ServiceAccount tokens. To do that, it calls the Kubernetes API and checks whether the token is valid and which Service Account it belongs to.

For this, Vault needs:

- a token reviewer JWT
- the Kubernetes API address
- the cluster CA certificate

Configure it like this:

```bash
kubectl exec -n vault -i vault-0 -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert="$(kubectl exec -n vault vault-0 -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)"
```

What this does:

- Tells Vault how to connect to the Kubernetes API.
- `token_reviewer_jwt`: Vault's own service account token to call K8S API.
- `kubernetes_ca_cert`: Certificate to trust the k8s API server

Now Vault can validate that pods are who they claim to be.

Verify the config:

```bash
kubectl exec -n vault vault-0 -- vault read auth/kubernetes/config
```

## Creating an Application Identity

Create a dedicated namespace for the app:

```bash
kubectl create namespace app
```

Create a ServiceAccount:

```bash
kubectl create serviceaccount my-app -n app
```

## Mapping Kubernetes Identity to Vault Policy

Let’s create the Kubernetes Auth Role:

```bash
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app \
  bound_service_account_namespaces=app \
  policies=my-app \
  ttl=1h
```

What this does:

- Creates a Vault role named `my-app` that maps to the Kubernetes Service Account `my-app`.
- Only pods using the ServiceAccount `my-app` in the namespace `app` can assume this role.
- Grants the `my-app` policy to the authenticated pods.
- `ttl=1h`: Vault tokens expire after 1 hour. It provides dynamic, temporary access.

This mapping is the core of Kubernetes auth.

## Running a Test Pod

Let’s deploy a pod that uses this ServiceAccount:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: vault-test
  namespace: app
spec:
  serviceAccountName: my-app
  containers:
    - name: app
      image: curlimages/curl:8.5.0
      command: ["sleep", "3600"]
EOF
```

## Authenticating to Vault From the Pod

Exec into the pod:

```bash
kubectl exec -n app -it vault-test -- sh
```

Every pod gets a ServiceAccount token mounted automatically. Map the token into a TOKEN variable:

```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

Now send that token to Vault:

```bash
LOGIN=$(curl -s --request POST \
  --data '{"jwt": "'"$TOKEN"'", "role": "my-app"}' \
  http://vault.vault.svc:8200/v1/auth/kubernetes/login)
```

If authentication succeeds, Vault returns a client token. Extract it:

```bash
VAULT_TOKEN=$(echo "$LOGIN" | sed -n 's/.*"client_token":"\([^"]*\)".*/\1/p')
```

## Reading the Secret

Inside the vault-test pod, use the Vault token to read the secret:

```bash
curl -s \
  --header "X-Vault-Token: $VAULT_TOKEN" \
  http://vault.vault.svc:8200/v1/secret/data/my-app/config
```