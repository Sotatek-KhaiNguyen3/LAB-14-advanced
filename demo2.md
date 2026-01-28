# Lab 2: Notes App với Helm + Vault + Ingress

## Tech Stack
- **Helm** - Package manager cho Kubernetes
- **HashiCorp Vault** - Secrets management
- **Ingress NGINX** - Traffic routing
- **StatefulSet** - MySQL database

---

## Architecture

```
                         ┌─────────────────┐
        notes.local      │    Ingress      │
       ─────────────────►│    (NGINX)      │
                         └────────┬────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
           ┌───────────────┐           ┌───────────────┐
      /    │   frontend    │    /api   │ nodejs-backend│
           │   (Service)   │           │   (Service)   │
           └───────────────┘           └───────┬───────┘
                                               │
                    ┌──────────────────────────┤
                    │                          │
                    ▼                          ▼
           ┌───────────────┐           ┌───────────────┐
           │     MySQL     │           │     Vault     │
           │ (StatefulSet) │           │   (Secrets)   │
           └───────────────┘           └───────────────┘
```

---

## So sánh Lab 1 vs Lab 2

| Feature | Lab 1 | Lab 2 |
|---------|-------|-------|
| Deployment | kubectl + YAML | Helm chart |
| Secrets | Sealed Secrets | HashiCorp Vault |
| Routing | NodePort | Ingress |
| Config | ConfigMap | ConfigMap (Helm templated) |

---

# STEP-BY-STEP DEPLOYMENT

## Step 1: Start Minikube

```bash
minikube start --driver=docker

# Verify
kubectl get nodes
```

---

## Step 2: Load Images vào Minikube

```bash
minikube image load nodejstodolist-frontend
minikube image load nodejstodolist-nodejs-backend

# Verify
minikube image list | grep -E "frontend|backend"
```

---

## Step 3: Enable Ingress Addon

```bash
minikube addons enable ingress

# Verify (đợi khoảng 1 phút)
kubectl get pods -n ingress-nginx
```

---

## Step 4: Cài đặt Vault

### 4.1 Thêm Helm repo

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### 4.2 Cài Vault trong dev mode (cho lab)

```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true" \
  --set "injector.enabled=true"

# Verify
kubectl get pods -n vault
```

### 4.3 Đợi Vault sẵn sàng

```bash
kubectl wait --for=condition=Ready pod/vault-0 -n vault --timeout=120s
```

---

## Step 5: Cấu hình Vault Secrets

### 5.1 Exec vào Vault pod

```bash
kubectl exec -it vault-0 -n vault -- /bin/sh
```

### 5.2 Tạo secrets (trong Vault shell)

```bash
# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Tạo secrets cho Notes App
vault kv put secret/notes-app \
  DB_PASSWORD="AppPassword123!" \
  DB_NAME="notesdb" \
  KEY="todolist"

# Verify
vault kv get secret/notes-app
```

### 5.3 Cấu hình Kubernetes Auth

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Cấu hình auth với Kubernetes API
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# Tạo policy cho Notes App
vault policy write notes-app - <<EOF
path "secret/data/notes-app" {
  capabilities = ["read"]
}
EOF

# Tạo role cho Notes App
vault write auth/kubernetes/role/notes-app \
  bound_service_account_names=notes-app \
  bound_service_account_namespaces=default \
  policies=notes-app \
  ttl=24h

# Thoát khỏi Vault shell
exit
```

---

## Step 6: Cài đặt Helm (nếu chưa có)

```bash
# Windows
winget install Helm.Helm

# Hoặc Chocolatey
choco install kubernetes-helm

# Verify
helm version
```

---

## Step 7: Deploy Notes App bằng Helm

```bash
# Từ folder lab 2
helm install notes-app ./notes-app-chart

# Hoặc upgrade nếu đã có
helm upgrade --install notes-app ./notes-app-chart
```

---

## Step 8: Verify Deployment

```bash
# Xem tất cả resources
kubectl get all

# Xem pods (bao gồm Vault sidecar)
kubectl get pods

# Xem Ingress
kubectl get ingress

# Xem logs Vault Agent (trong app pods, không phải vault-agent-injector)
kubectl logs <nodejs-backend-pod> -c vault-agent-init
kubectl logs mysql-0 -c vault-agent-init
```

---

## Step 9: Truy cập App

### Windows + Docker Driver

```bash
# Cách 1: Minikube tunnel (Recommended)
minikube tunnel

# Thêm vào file hosts (C:\Windows\System32\drivers\etc\hosts)
# 127.0.0.1  notes.local

# Truy cập: http://notes.local
```

```bash
# Cách 2: Port forward Ingress
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80

# Truy cập với Host header
curl -H "Host: notes.local" http://localhost:8080
```

### Linux/macOS

```bash
# Lấy IP minikube
minikube ip

# Thêm vào /etc/hosts
# <minikube-ip>  notes.local

# Truy cập: http://notes.local
```

---

## Step 10: Test API

```bash
# Test qua Ingress
curl -H "Host: notes.local" http://localhost:8080/api/notes

# Kết quả mong đợi:
# {"success":true,"data":[...],"message":"Notes retrieved successfully"}
```

---

# VAULT CONCEPTS

## Vault Agent Sidecar Injector

Khi pod có annotation `vault.hashicorp.com/agent-inject: "true"`:

1. **Init Container**: Vault Agent khởi động, authenticate với Vault
2. **Sidecar Container**: Liên tục renew token và update secrets
3. **Secret File**: Được mount tại `/vault/secrets/<secret-name>`

```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "notes-app"
  vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/notes-app"
```

## So sánh Sealed Secrets vs Vault

| Feature | Sealed Secrets | Vault |
|---------|---------------|-------|
| Storage | Encrypted in Git | Central server |
| Rotation | Manual | Automatic |
| Audit | Git history | Built-in audit log |
| Dynamic secrets | No | Yes |
| Complexity | Low | High |
| Production | Small teams | Enterprise |

---

# TROUBLESHOOTING

## Vault Agent không inject secrets

```bash
# Kiểm tra Vault injector
kubectl get pods -n vault | grep injector

# Xem logs injector
kubectl logs -n vault -l app.kubernetes.io/name=vault-agent-injector

# Kiểm tra ServiceAccount
kubectl get sa notes-app
```

## Pods stuck ở Init

```bash
# Xem logs init container
kubectl logs <pod-name> -c vault-agent-init

# Kiểm tra Vault role
kubectl exec -it vault-0 -n vault -- vault read auth/kubernetes/role/notes-app
```

## Ingress không hoạt động

```bash
# Kiểm tra Ingress controller
kubectl get pods -n ingress-nginx

# Xem logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Kiểm tra Ingress rules
kubectl describe ingress notes-app-ingress
```

## MySQL không kết nối được

```bash
# Kiểm tra secrets đã inject chưa
kubectl exec <mysql-pod> -- cat /vault/secrets/db-creds

# Xem logs MySQL
kubectl logs mysql-0
```

---

# FILE STRUCTURE

```
lab 2/
├── notes-app-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── mysql-configmap.yaml
│       ├── mysql-statefulset.yaml
│       ├── mysql-service.yaml
│       ├── backend-deployment.yaml
│       ├── backend-service.yaml
│       ├── frontend-deployment.yaml
│       ├── frontend-service.yaml
│       ├── ingress.yaml
│       └── serviceaccount.yaml
└── demo2.md
```

---

# CLEANUP

```bash
# Xóa Notes App
helm uninstall notes-app

# Xóa Vault
helm uninstall vault -n vault
kubectl delete namespace vault

# Xóa Ingress addon
minikube addons disable ingress

# Xóa cluster
minikube delete
```
