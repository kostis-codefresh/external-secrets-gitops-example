# Argo CD with Vault and the External secret operator demo

Example of using External Secrets with Vault and Argo CD. The demo application is a Go web server that reads database credentials from a mounted Kubernetes secret and automatically reloads them when they change—no pod restart required.

Read the full blog at https://codefresh.io/blog/gitops-secrets-with-argo-cd-hashicorp-vault-and-the-external-secret-operator/

---

## How it works

```
HashiCorp Vault  →  External Secrets Operator  →  Kubernetes Secret  →  Application
```

ESO polls Vault every 15 seconds and writes values into a native Kubernetes `Secret`. The app mounts that secret as a file and uses fsnotify to reload it without restarting.

---

## 1. Prerequisites

- A Kubernetes cluster with `cluster-admin` permissions
- `kubectl` and `helm` (v3)

---

## 2. Set up Vault

Install Vault in dev mode (root token = `root`, pre-unsealed):

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com && helm repo update
kubectl create namespace vault
helm install vault hashicorp/vault --namespace vault --set "server.dev.enabled=true"
```

Exec into the pod and configure it:

```bash
kubectl exec -it vault-0 -n vault -- /bin/sh
```

```sh
vault login root

# Write the demo credentials
vault kv put secret/mysql_credentials \
  url="mysql.example.com:3306" \
  username="my_demo_user" \
  password="my_demo_password"

# Enable Kubernetes auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"

# Policy and role for ESO
vault policy write eso-read-policy - <<EOF
path "secret/data/mysql_credentials" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/demo \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-read-policy \
  ttl=24h

exit
```

---

## 3. Set up the External Secrets Operator

```bash
helm repo add external-secrets https://charts.external-secrets.io && helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace
```

Apply the `ClusterSecretStore` (connects ESO to Vault using the `demo` Kubernetes auth role):

```bash
kubectl apply -f manifests/vault-integration/secretstore.yml
kubectl get clustersecretstore vault-backend  # should show READY: True
```

---

## 4. Deploy the app with Argo CD

Point an Argo CD application at `manifests/app` in this repo. That directory contains the Deployment, Service, and the `ExternalSecret` that creates the `mysql-credentials` Kubernetes secret.

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-secrets-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <YOUR_REPO_URL>
    targetRevision: main
    path: manifests/app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

Once synced, verify and access the app:

```bash
kubectl get externalsecret my-db-credentials  # STATUS: SecretSynced
kubectl port-forward svc/gitops-secrets-service 8080:8080
```

Open http://localhost:8080 to see the current credentials.

---

## 5. Rotate a secret and watch the app update itself

Update the secret in Vault:

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/mysql_credentials \
  url="mysql.example.com:3306" \
  username="rotated_user" \
  password="new_super_secret_password"
```

Within ~15 seconds the app reloads automatically—no restart, no Argo CD sync:

```bash
kubectl logs -f -l app=gitops-secrets-app
# Config file changed: /secrets/credentials
# Username is rotated_user
# Password is new_super_secret_password
```

Refresh `http://localhost:8080` to confirm.
