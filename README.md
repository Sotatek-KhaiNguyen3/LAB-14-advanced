# Lab 2: Notes App - Helm + Vault + Ingress

## Tổng quan

Lab 2 triển khai ứng dụng Notes App trên Kubernetes sử dụng các công nghệ enterprise-grade:

| Component | Công nghệ | Mục đích |
|-----------|-----------|----------|
| Package Manager | **Helm** | Quản lý deployment như một package |
| Secrets | **HashiCorp Vault** | Centralized secrets management |
| Routing | **Ingress NGINX** | HTTP routing với hostname |
| Database | **MySQL StatefulSet** | Persistent storage |

---

## Kiến trúc

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
      /    │   Frontend    │    /api   │    Backend    │
           │   (React)     │           │   (Node.js)   │
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

## Điểm khác biệt chính so với triển khai truyền thống

### 1. Helm Chart thay vì YAML files riêng lẻ

**Trước (kubectl + YAML):**
```bash
kubectl apply -f frontend.yaml
kubectl apply -f backend.yaml
kubectl apply -f mysql.yaml
kubectl apply -f secret.yaml
```

**Sau (Helm):**
```bash
helm install notes-app ./notes-app-chart
```

**Lợi ích:**
- Một lệnh deploy toàn bộ stack
- Values có thể override: `helm install --set backend.replicas=5`
- Versioning và rollback: `helm rollback notes-app 1`
- Templating: DRY (Don't Repeat Yourself)

---

### 2. Vault thay vì Kubernetes Secrets

**Trước (K8s Secret / SealedSecret):**
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: notes-secret
      key: DB_PASSWORD
```

**Sau (Vault Agent Injection):**
```yaml
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "notes-app"
  vault.hashicorp.com/agent-inject-secret-creds: "secret/data/notes-app"
```

**Lợi ích:**

| Feature | K8s Secret | Vault |
|---------|------------|-------|
| Storage | etcd (base64) | Encrypted server |
| Access Control | RBAC | Fine-grained policies |
| Rotation | Manual | Automatic TTL |
| Audit | Limited | Full audit log |
| Dynamic Secrets | No | Yes (DB credentials) |
| UI Management | No | Yes |

---

### 3. Ingress thay vì NodePort

**Trước (NodePort):**
```
http://<node-ip>:30001  → Frontend
http://<node-ip>:30002  → Backend API
```

**Sau (Ingress):**
```
http://notes.local/     → Frontend
http://notes.local/api  → Backend API
```

**Lợi ích:**
- URL thân thiện với hostname
- Path-based routing (`/`, `/api`)
- Single entry point
- SSL/TLS termination
- Load balancing tích hợp

---

## Quick Start

### Prerequisites
- Minikube
- Helm 3.x
- kubectl

### Deploy

```bash
# 1. Start Minikube
minikube start --driver=docker

# 2. Enable Ingress
minikube addons enable ingress

# 3. Install Vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault --create-namespace \
  --set "server.dev.enabled=true" \
  --set "injector.enabled=true"

# 4. Configure Vault secrets (xem demo2.md Step 5)

# 5. Deploy Notes App
helm install notes-app ./notes-app-chart

# 6. Access
minikube tunnel
# Thêm "127.0.0.1 notes.local" vào hosts file
# Truy cập: http://notes.local
```

---

## Cấu trúc Helm Chart

```
notes-app-chart/
├── Chart.yaml              # Metadata
├── values.yaml             # Default values
└── templates/
    ├── backend-deployment.yaml
    ├── backend-service.yaml
    ├── frontend-deployment.yaml
    ├── frontend-service.yaml
    ├── mysql-statefulset.yaml
    ├── mysql-service.yaml
    ├── mysql-configmap.yaml
    ├── ingress.yaml
    └── serviceaccount.yaml
```

---

## Vault Integration Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      POD STARTUP                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Pod created with Vault annotations                      │
│          │                                                  │
│          ▼                                                  │
│  2. Vault Injector detects annotations                      │
│          │                                                  │
│          ▼                                                  │
│  3. Init container (vault-agent-init) added                 │
│          │                                                  │
│          ▼                                                  │
│  4. Vault Agent authenticates via ServiceAccount            │
│          │                                                  │
│          ▼                                                  │
│  5. Secrets fetched and written to /vault/secrets/          │
│          │                                                  │
│          ▼                                                  │
│  6. App container starts, sources secrets                   │
│          │                                                  │
│          ▼                                                  │
│  7. Sidecar keeps secrets refreshed (TTL renewal)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Useful Commands

```bash
# Helm
helm list                           # List releases
helm upgrade notes-app ./notes-app-chart  # Upgrade
helm rollback notes-app 1           # Rollback
helm uninstall notes-app            # Delete

# Vault
kubectl port-forward svc/vault -n vault 8200:8200  # Access UI
kubectl exec vault-0 -n vault -- vault kv get secret/notes-app

# Debug
kubectl get pods
kubectl logs <pod> -c vault-agent-init
kubectl describe pod <pod>
```
