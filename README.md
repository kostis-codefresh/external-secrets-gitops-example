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
- `kubectl` and Argo CD installed in the `argocd` namespace

---

## 2. Set up Vault

Install Vault in dev mode (root token = `root`, pre-unsealed) via Argo CD:

```bash
argocd app create vault \
--project default \
--repo https://helm.releases.hashicorp.com \
--helm-chart vault \
--revision 0.28.0 \
--sync-policy auto \
--sync-option CreateNamespace=true \
--parameter server.dev.enabled=true \
--dest-namespace vault \
--dest-server https://kubernetes.default.svc
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
argocd app create eso \
--project default \
--repo https://charts.external-secrets.io \
--helm-chart external-secrets \
--revision 0.9.19 \
--sync-policy auto \
--sync-option CreateNamespace=true \
--dest-namespace external-secrets \
--dest-server https://kubernetes.default.svc
```

Apply the `ClusterSecretStore` (connects ESO to Vault using the `demo` Kubernetes auth role):

```bash
argocd app create vault-secret-store \
--project default \
--repo https://github.com/kostis-codefresh/external-secrets-gitops-example.git \
--path "./manifests/vault-integration" \
--sync-policy auto \
--dest-namespace external-secrets \
--dest-server https://kubernetes.default.svc


kubectl get clustersecretstore vault-backend  # should show READY: True
```

---

## 4. Deploy the app with Argo CD

Point an Argo CD application at `manifests/app` in this repo. That directory contains the Deployment, Service, and the `ExternalSecret` that creates the `mysql-credentials` Kubernetes secret.

```bash
argocd app create my-secret-app \
--project default \
--repo https://github.com/kostis-codefresh/external-secrets-gitops-example.git \
--path "./manifests/app" \
--sync-policy auto \
--dest-namespace default \
--dest-server https://kubernetes.default.svc
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
