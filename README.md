# 🎓 Thesis: Containerized Deployment to Kubernetes with CI/CD

> **Case study for the Kubernetes thesis project** — demonstrating platform engineering, DevOps automation, and cloud-native deployment skills. The source code is private; this repo documents the architecture, decisions, and technical depth.

## 📋 Overview

A practical DevOps thesis: take a PHP/MySQL web application → Dockerize → package as a Helm chart → deploy on **Azure Kubernetes Service (AKS)** → automate database backup/restore to cloud storage → build a full CI/CD pipeline.

**Doctor Appointment Management System** — a real-world web app with persistent database state, used as the vehicle to demonstrate production-grade Kubernetes deployment patterns.

## 🏗️ Architecture

```
                    ┌──────────────────────────────────┐
                    │        Azure AKS Cluster         │
                    │                                   │
  Internet ──▶ Ingress (nginx)                          │
                    │         ┌───────────────────┐     │
                    │         │  PHP App (2 pods)  │     │
                    ├────────▶│  PHP 7.4 + Apache  │     │
                    │         │  NodePort: 30080   │     │
                    │         └────────┬──────────┘     │
                    │                  │                 │
                    │         ┌────────▼──────────┐     │
                    │         │  MySQL (1 pod)    │     │
                    │         │  ClusterIP: 3306  │     │
                    │         │  PVC: 10Gi        │     │
                    │         └────────┬──────────┘     │
                    │                  │                 │
                    └──────────────────┼─────────────────┘
                                       │
                              ┌────────▼──────────┐
                              │  CronJob (5h)     │
                              │  mysqldump        │
                              │  → Azure Blob     │
                              └───────────────────┘
```

## 🔧 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Application** | PHP 7.4, Apache, MySQL |
| **Containerization** | Docker (multi-stage) |
| **Orchestration** | Kubernetes (AKS) |
| **Packaging** | Helm v2 |
| **Networking** | nginx Ingress, NodePort |
| **Storage** | PersistentVolumeClaim (Azure) |
| **Backup** | CronJob → Azure Blob Storage |
| **CI/CD** | GitHub Actions |

## 🔄 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
Trigger: push to main
Steps:
  1. Build Docker image (php:7.4-apache)
  2. Push to DockerHub (umerazim/my-php-app:latest)
  3. Helm upgrade --install to AKS (via kubeconfig secret)
```

**Pipeline flow:**
```
git push → GitHub Actions → Docker Build → DockerHub → Helm Deploy → AKS
```

## 📦 Helm Chart Structure

```
helm-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── php-deployment.yaml      # 2 replicas, rolling update
    ├── php-service.yaml         # NodePort 30080
    ├── mysql-deployment.yaml    # 1 replica, PVC-backed
    ├── mysql-service.yaml       # ClusterIP 3306
    ├── pvc-mysql.yml            # 10Gi PersistentVolumeClaim
    ├── ingress.yml              # nginx Ingress (php.toolkit.mobi)
    ├── cronjob-backup.yaml      # mysqldump → Azure Blob (every 5h)
    ├── load-db-from-azure.yaml  # One-time restore on first deploy
    └── azure-blob-secret.yaml   # K8s Secret for Azure credentials
```

## 🗄️ Backup & Restore Strategy

### Automated Backup (CronJob)
- Runs every 5 hours via Kubernetes CronJob
- Executes `mysqldump` against the MySQL pod
- Uploads timestamped + `latest.sql` backup to Azure Blob Storage
- Retains complete backup history

### First-Deploy Restore (Job)
- Uses Helm's `Release.Revision` conditional: `{{ if eq .Release.Revision 1 }}`
- Only executes on the **first Helm install** — skips on upgrades
- Downloads `latest.sql` from Azure Blob and restores the database
- Ensures the app starts with seed data on initial deployment

> **Pattern:** This "restore-on-first-deploy" approach handles stateful app bootstrapping cleanly — production data is preserved across upgrades while new environments get seeded automatically.

## 🎯 Key Engineering Decisions

### 1. Helm for Packaging
Chose Helm over raw YAML manifests for:
- Templated values (image tags, replicas, ingress host)
- Release management (install/upgrade/rollback lifecycle)
- Conditional restore logic in templates

### 2. CronJob for Database Backup
Kubernetes-native CronJob instead of external backup service:
- Runs in-cluster, no external dependencies
- Azure Blob provides durable storage
- Timestamped backups enable point-in-time recovery

### 3. nginx Ingress
- Single entry point for external traffic
- Path-based routing capability
- TLS termination support (extensible)

### 4. NodePort + Ingress Layering
- NodePort provides cluster-level access for debugging
- Ingress provides production-grade routing on top
- Separation of concerns: internal access vs external routing

## 📊 Deployment Metrics

| Metric | Value |
|--------|-------|
| PHP replicas | 2 (HA) |
| MySQL replicas | 1 (stateful) |
| Backup frequency | Every 5 hours |
| PVC size | 10Gi |
| CI/CD deploy time | ~3 minutes |
| Release strategy | Rolling update |

## 💡 Lessons Learned

1. **Stateful workloads on K8s** require careful consideration of storage, backup, and restore — unlike stateless services
2. **Helm templating** enables clean separation of config from code, making multi-environment deploys straightforward
3. **CI/CD for Kubernetes** needs kubeconfig management — base64-encoded secrets work but are a starting point, not an endpoint
4. **Database migrations** and seed data should be handled as a first-class concern in the deployment pipeline
5. **CronJob images** should be pre-built (not installing packages on every run) for reliability and speed

## 🔗 Related

- **Full thesis** (university submission) available upon request
- Architecture approach applicable to any LAMP-stack → Kubernetes migration

---

**Author:** Muhammad Umer | [GitHub](https://github.com/uby174) | [LinkedIn](https://www.linkedin.com/in/muhammad-umer-/)