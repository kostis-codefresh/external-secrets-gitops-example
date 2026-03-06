# Argo CD with Vault and the External secret operator demo

Example of using External Secrets with Vault and Argo CD. The demo application is a Go web server that reads database credentials from a mounted Kubernetes secret and automatically reloads them when they change—no pod restart required.

Read the full blog at https://codefresh.io/blog/gitops-secrets-with-argo-cd-hashicorp-vault-and-the-external-secret-operator/

---

## How it works

```
HashiCorp Vault  →  External Secrets Operator  →  Kubernetes Secret  →  Application
```

1. Vault holds the actual secret values (database credentials).
2. The External Secrets Operator (ESO) pulls those values from Vault every 15 seconds and writes them into a native Kubernetes `Secret`.
3. The application mounts that Kubernetes `Secret` as a file and watches it for changes using [fsnotify](https://github.com/fsnotify/fsnotify). When the file changes, the application reloads the values without restarting.

---

## 1. Prerequisites

### Kubernetes cluster

Any Kubernetes cluster works (kind, k3s, GKE, EKS, AKS, etc.). You need `cluster-admin` permissions to install the operators.

### CLI tools

| Tool | Purpose |
|------|---------|
| `kubectl` | Manage Kubernetes resources |
| `helm` (v3) | Install Vault and ESO |
| `vault` | Interact with the Vault API from your workstation |
| `argocd` | (Optional) Manage Argo CD from the CLI |

### Fork or clone this repository

ESO and Argo CD will reference manifests stored in Git, so you need a publicly accessible (or Argo CD–accessible) Git remote:

```bash
git clone https://github.com/kostis-codefresh/external-secrets-gitops-example.git
cd external-secrets-gitops-example
```

---

## 2. Set up Vault

### Install Vault with Helm

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace vault

helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true"
```

> **Note:** `server.dev.enabled=true` starts Vault in development mode with a root token of `root` and an already-unsealed, in-memory storage backend. This is fine for demos but **never use dev mode in production**.

Wait for the Vault pod to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=vault -n vault --timeout=120s
```

### Create the database credentials secret in Vault

Open a shell inside the Vault pod and configure it:

```bash
kubectl exec -it vault-0 -n vault -- /bin/sh
```

Inside the pod:

```sh
# Authenticate (dev mode root token)
vault login root

# Enable the KV v2 secrets engine at path "secret" (already enabled in dev mode)
vault secrets enable -path=secret kv-v2 2>/dev/null || true

# Write the MySQL credentials
vault kv put secret/mysql_credentials \
  url="mysql.example.com:3306" \
  username="my_demo_user" \
  password="my_demo_password"

# Verify
vault kv get secret/mysql_credentials
```

### Enable Kubernetes authentication

ESO needs to authenticate against Vault using a Kubernetes service account token. Still inside the Vault pod:

```sh
# Enable the Kubernetes auth method
vault auth enable kubernetes

# Configure it to talk to the in-cluster Kubernetes API
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
```

### Create a Vault policy for ESO

```sh
vault policy write eso-read-policy - <<EOF
path "secret/data/mysql_credentials" {
  capabilities = ["read"]
}
EOF
```

### Create a Vault role for ESO

The role binds the Vault policy to the ESO service account in Kubernetes. ESO is installed in the `external-secrets` namespace (see next section) and its main service account is named `external-secrets`:

```sh
vault write auth/kubernetes/role/demo \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=eso-read-policy \
  ttl=24h
```

Exit the pod shell:

```sh
exit
```

---

## 3. Set up the External Secrets Operator

### Install ESO with Helm

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

Wait for ESO to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=external-secrets \
  -n external-secrets --timeout=120s
```

### Apply the ClusterSecretStore

The `ClusterSecretStore` tells ESO how to connect to Vault. It uses Kubernetes authentication with the `demo` role created above:

```bash
kubectl apply -f manifests/vault-integration/secretstore.yml
```

The store points to `http://vault.vault:8200`—the Vault service using the format `<service-name>.<namespace>`.

Verify the store is ready:

```bash
kubectl get clustersecretstore vault-backend
```

You should see `READY: True`.

---

## 4. Deploy the app with Argo CD

### Install Argo CD (if not already installed)

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=180s
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### Create the Argo CD application

Replace `<YOUR_REPO_URL>` with the URL of your fork:

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

Argo CD will sync the three manifests in `manifests/app/`:

- `deployment.yml` — the Go application that mounts the `mysql-credentials` secret.
- `service.yml` — a ClusterIP service on port 8080.
- `external-secret-db.yml` — the `ExternalSecret` resource that tells ESO to pull credentials from Vault into the `mysql-credentials` Kubernetes secret.

### Verify the deployment

Check that the `ExternalSecret` has synced successfully:

```bash
kubectl get externalsecret my-db-credentials
```

You should see `READY: True` and `STATUS: SecretSynced`.

Check that the Kubernetes secret was created:

```bash
kubectl get secret mysql-credentials
```

Check that the application pod is running:

```bash
kubectl get pods -l app=gitops-secrets-app
```

Port-forward to view the running application:

```bash
kubectl port-forward svc/gitops-secrets-service 8080:8080
```

Open http://localhost:8080 in your browser. You will see the database connection details currently read from the secret.

---

## 5. Rotate a secret and watch the app update itself

This is the key part of the demo. ESO polls Vault every **15 seconds** (as configured in `external-secret-db.yml`). When it detects a change, it updates the Kubernetes secret, which updates the mounted file. The application watches the file with fsnotify and reloads without restarting.

### Update the secret in Vault

```bash
kubectl exec -it vault-0 -n vault -- vault kv put secret/mysql_credentials \
  url="mysql.example.com:3306" \
  username="rotated_user" \
  password="new_super_secret_password"
```

### Watch it propagate automatically

Watch the pod logs to see the application detect the file change and reload:

```bash
kubectl logs -f -l app=gitops-secrets-app
```

Within about 15 seconds you will see output like:

```
Config file changed: /secrets/credentials
Reading configuration from /secrets/credentials
Connection string is mysql.example.com:3306
Username is rotated_user
Password is new_super_secret_password
```

Refresh http://localhost:8080 and the new credentials will be displayed.

This is the full GitOps secrets rotation flow:

1. A secret value changes in Vault (the source of truth).
2. ESO detects the change and updates the Kubernetes `Secret`.
3. The mounted file inside the pod is updated by kubelet.
4. The application reloads from the file automatically.

All without touching the Git repository or triggering an Argo CD sync.
