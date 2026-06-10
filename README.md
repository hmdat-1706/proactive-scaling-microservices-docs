# ⚡ Proactive Autoscaling for Microservices on Kubernetes

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![K3s](https://img.shields.io/badge/K3s-FFC61C?style=for-the-badge&logo=k3s&logoColor=black)
![ArgoCD](https://img.shields.io/badge/Argo%20CD-1e0b3e?style=for-the-badge&logo=argo&logoColor=#d16044)
![Argo Rollouts](https://img.shields.io/badge/Argo%20Rollouts-1e0b3e?style=for-the-badge&logo=argo&logoColor=#4cb4e6)
![KEDA](https://img.shields.io/badge/KEDA-2F6FE0?style=for-the-badge&logo=kubernetes&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-%23F46800.svg?style=for-the-badge&logo=grafana&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=Prometheus&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232671E5.svg?style=for-the-badge&logo=githubactions&logoColor=white)
![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?style=for-the-badge&logo=mlflow&logoColor=white)
![k6](https://img.shields.io/badge/k6-7D64FF?style=for-the-badge&logo=k6&logoColor=white)

## 📖 Project Overview

This project implements a **Proactive Autoscaling Platform** for microservices deployed on a K3s Kubernetes cluster. Unlike traditional reactive scaling (HPA) that responds *after* traffic spikes, this system uses an **AI-powered Prophet model** to **predict future traffic** and scale services *before* demand increases.

The entire platform is managed through **GitOps (ArgoCD)** and provisioned via **Ansible automation**, enabling full cluster bootstrap to a production-ready state with minimal manual actions required.

### Key Highlights
- **AI-Driven Proactive Scaling** — KEDA polls a FastAPI prediction endpoint to pre-scale services before traffic spikes occur
- **GitOps (ArgoCD App-of-Apps)** — Single bootstrap point for the entire infrastructure with automated synchronization. Kustomize manages `base`/`overlays` per environment: Prophet supports Dev/Prod, Boutique targets Production with Blue/Green
- **Zero-Downtime CD** — Blue/Green deployment strategy powered by Argo Rollouts
- **DevSecOps Pipeline** — GitHub Actions with Trivy vulnerability scanning, yamllint, kubeconform, Gitleaks, SAST and Flake8
- **Zero Plaintext Secrets** — Bitnami Sealed Secrets with automated certificate retrieval and encryption via Ansible
- **Full Observability & Alerting** — kube-prometheus-stack with custom Traefik RPS metrics, PrometheusRules, and automated Slack notifications for critical alerts

## 🏗️ Architecture

![Enterprise Architecture Blueprint](./docs/images/diagram.png)

## 📂 Repository Structure

```text
├── .github/workflows/           # CI/CD GitHub Actions pipelines
├── apps/
│   ├── boutique/                # Google Online Boutique microservices (Kustomize)
│   └── prophet/                 # AI Scaler (FastAPI, CronJobs, MLflow, models)
├── infra/
│   ├── ansible/                 # K3s & ArgoCD cluster bootstrap automation
│   ├── argocd/                  # GitOps App-of-Apps definitions
│   ├── autoscaling/             # KEDA ScaledObjects rules
│   ├── ingress/                 # Traefik Ingress rules & ServiceMonitor for metrics
│   └── monitoring/              # Prometheus, Grafana, Alertmanager & Sealed Secrets configs
└── load-test/                   # K6 load testing scripts
```

## ⚙️ Quick Start — Full Cluster Bootstrap

### Prerequisites
- 2 Ubuntu VMs (Master: 4GB RAM, Worker: 8GB RAM) with SSH access
- Ansible installed on your control machine
- GitHub account with a PAT token for GHCR access
- Slack Webhook URL for Alertmanager

### 1. Configure Inventory

Edit `infra/ansible/inventories/production/hosts` with your VM IPs:
```ini
[master]
<MASTER_IP>

[worker]
<WORKER_IP>

[k3s_cluster:children]
master
worker
```

### 2. Run the Playbook

```bash
cd infra/ansible
ansible-playbook -i inventories/production/hosts site.yaml
```

The playbook will:
1. Install base packages (`curl`) on all nodes
2. Provision K3s control plane + install `kubeseal` CLI
3. Join worker node to the K3s cluster
4. Install ArgoCD + apply App-of-Apps manifest (which auto-deploys Helm charts: KEDA, Sealed Secrets, kube-prometheus-stack, Argo Rollouts)
5. Prompt for credentials → wait for ArgoCD to sync Sealed Secrets controller → encrypt and apply GHCR, Grafana, and Slack Webhook secrets → set ArgoCD admin password

### 3. Verify Deployment

```bash
# Check ArgoCD applications
k3s kubectl get applications -n argocd

# Check all pods
k3s kubectl get pods -A

# Access services (add to /etc/hosts)
# <MASTER_IP> web.local grafana.local argocd.local api.local mlflow.local rollouts.local
```

## 🤖 How Proactive Scaling Works

Unlike traditional reactive scaling (which responds *after* a spike), this system continuously forecasts future traffic and pre-scales services *before* demand arrives.

```mermaid
graph TD
    subgraph TRAFFIC ["1. Live Traffic"]
        User(["Users / Load Generator"])
        Ingress["Traefik Ingress"]
    end

    subgraph OBSERVE ["2. Observability & Data Collection"]
        Prometheus[("Prometheus")]
        Alertmanager["Alertmanager\n(Slack Notifications)"]
        DataIngest["Data Ingestion CronJob\n(Daily)"]
        PV[("Shared Data Lake\n(Persistent Volume)")]
    end

    subgraph MLOPS ["3. MLOps Pipeline"]
        Retrain["Model Retrain CronJob\n(Weekly)"]
        MLflow["MLflow Model Registry"]
        Prophet["Prophet Server\n(FastAPI /api/forecast)"]
    end

    subgraph SCALE ["4. Proactive Autoscaling"]
        KEDA["KEDA Operator\n(metrics-api + CPU triggers)"]
        HPA["Horizontal Pod Autoscaler"]
        Pods["Microservice Pods"]
    end

    User -->|"HTTP Requests"| Ingress
    Ingress -->|"Route to"| Pods
    Ingress -..->|"Scrape RPS Metrics"| Prometheus
    Prometheus -->|"Fire Alerts"| Alertmanager
    Prometheus -->|"Query Historical Data"| DataIngest
    DataIngest -->|"Append Dataset"| PV
    PV -->|"Load Training Data"| Retrain
    Retrain -->|"Register New Model"| MLflow
    MLflow -->|"Load Latest Model"| Prophet
    Prophet -->|"Expose Forecasted RPS"| KEDA
    KEDA -->|"Manage Desired Replicas"| HPA
    HPA -->|"Scale UP before traffic hits"| Pods
```

## 🗂️ AI Serving Infrastructure

The AI forecasting component is treated as **just another workload** on the cluster — packaged, deployed, and lifecycle-managed entirely through Kubernetes primitives and GitOps.

```mermaid
graph LR
    subgraph K8S ["Kubernetes Cluster (boutique namespace)"]
        direction TB
        CJ1["⏱ Data Ingestion CronJob\n(Daily)\nQueries Prometheus → appends CSV"]
        CJ2["⏱ Model Retrain CronJob\n(Weekly)\nRetrains Prophet on latest CSV"]
        PV[("PersistentVolume\nShared data & model storage")]
        MLflow["MLflow Deployment\nModel version registry"]
        Prophet["Prophet Deployment\nFastAPI · /api/forecast\nReturns predicted RPS"]
    end

    subgraph INFRA ["Infrastructure Layer"]
        Prometheus[("Prometheus")]
        KEDA["KEDA\nScaledObject · metrics-api trigger\npollInterval: 30s"]
    end

    Prometheus --> |"Historical RPS data"| CJ1
    CJ1 --> |"Append dataset"| PV
    PV --> |"Load training data"| CJ2
    CJ2 --> |"Save model artifact"| PV
    CJ2 --> |"Log run & register version"| MLflow
    Prophet -->|"GET /api/forecast\n→ predicted_rps for next 15 min"| KEDA
```

## 🔐 Security Design

| Layer | Implementation |
|-------|---------------|
| **Container Registry** | Sealed Secret (`ghcr_sealed.yaml`) — asymmetric encryption |
| **Grafana Admin** | Sealed Secret via Ansible `vars_prompt` — no plaintext in Git |
| **ArgoCD Admin** | bcrypt-hashed password via Ansible `vars_prompt` — patched at provision time |
| **Alertmanager Slack Webhook** | Sealed Secret via Ansible `vars_prompt` — automated routing |
| **Container Runtime** | Non-root user (UID 1000) + Pod `securityContext` |
| **Secret Detection** | Gitleaks scans all commits on PR — blocks credential leaks |
| **SAST** | Bandit scans Python source on every push — blocks insecure patterns |
| **Container Vulnerabilities** | Trivy blocks `CRITICAL`/`HIGH` CVEs before image is pushed |
| **Code Quality** | Flake8 (Python syntax) + yamllint + Kubeconform (K8s schemas) |

## 🔄 CI/CD Pipeline & GitOps Workflow

The full lifecycle is fully automated. The CI pipeline checks PRs and builds images on merge, updating Kustomize Overlays. ArgoCD then detects drift and applies changes using Blue/Green deployments for mission-critical services.

```mermaid
graph TD
    Dev(["Developer / Admin"])

    subgraph BOOTSTRAP ["Phase 1: Infrastructure Bootstrap"]
        Ansible["Ansible Playbook\n(site.yaml)"]
        K3s["K3s Cluster\n+ ArgoCD v3.4.3"]
    end

    subgraph CI ["Phase 2: CI Pipeline (GitHub Actions)"]
        Code["GitHub - Source Code"]
        Audit["Audit Job\nGitleaks · Bandit · yamllint\nKubeconform"]
        Build["Build & Push Job\nDockerBuildx · Trivy scan"]
        GHCR["GHCR\nImage Registry"]
        GitOps["Auto-commit\nKustomize image tag update"]
    end

    subgraph CD ["Phase 3: CD Pipeline (GitOps)"]
        Manifests["GitHub - K8s Manifests\n(Kustomize dev/prod overlays)"]
        Argo["ArgoCD Controller\n(App-of-Apps)"]
        Rollouts["Argo Rollouts\n(Blue/Green · manual promotion)"]
        Pods["Microservices Pods"]
    end

    Dev -->|"1. ansible-playbook site.yaml"| Ansible
    Ansible -->|"2. Provision VMs + bootstrap"| K3s

    Dev -->|"3. git push / PR"| Code
    Code -->|"4. Trigger on push/PR"| Audit
    Audit -->|"5. Pass"| Build
    Build -->|"6. Push image"| GHCR
    Build -->|"7. Commit new tag"| GitOps
    GitOps --> Manifests

    Argo -->|"8. Watch for drift"| Manifests
    Argo -->|"9. Sync"| Rollouts
    Rollouts -->|"10. Blue/Green deploy"| Pods
    Pods -..->|"11. Pull image"| GHCR
```

## 📊 Monitoring Access

| Service | URL | Credentials |
|---------|-----|-------------|
| Boutique Shop | `http://web.local` | — |
| Grafana | `http://grafana.local` | Set during playbook run |
| ArgoCD | `http://argocd.local` | `admin` / Set during playbook run |
| Argo Rollouts | `http://rollouts.local` | — |
| AI Forecast API | `http://api.local/api/forecast` | — |
| MLflow | `http://mlflow.local` | — |

## 🛠️ Tech Stack

| Category | Technology | Version |
|----------|-----------|--------|
| **Orchestration** | K3s (lightweight Kubernetes) | `v1.33.1+k3s1` |
| **GitOps / CD** | ArgoCD (App-of-Apps pattern) | `v3.4.3` |
| **Progressive Delivery** | Argo Rollouts (Blue/Green) | `2.41.0` |
| **Autoscaling** | KEDA (metrics-api + CPU triggers) | `2.18.0` |
| **AI/ML** | Facebook Prophet, MLflow, FastAPI | `1.3.0`, `3.12.0`, `0.115.x` |
| **CI Pipeline** | GitHub Actions, Docker Buildx, Trivy, Bandit, Gitleaks | — |
| **Manifest Management** | Kustomize (base/overlays), Kubeconform | — |
| **Monitoring & Alerting** | kube-prometheus-stack (Prometheus, Grafana, Alertmanager + Slack) | `75.12.0` |
| **Secret Management** | Bitnami Sealed Secrets | `2.18.6` |
| **IaC / Provisioning** | Ansible | — |
| **Ingress** | Traefik (K3s built-in) | — |
| **Load Testing** | K6 | — |

## 📝 Known Limitations

- **No TLS:** `.local` domains use HTTP only (cert-manager + Let's Encrypt would be the production path)
- **No NetworkPolicy:** All pods can communicate freely within the cluster namespace
- **Frozen MLOps Loop:** The Retrain CronJob demonstrates the MLOps architecture end-to-end, but the AI Server deliberately uses a frozen pre-trained model. The load generator (Boutique's synthetic traffic) lacks real-world seasonality and sufficient volume to produce a model that improves on retraining — this is a known data constraint of the lab environment, not a gap in the pipeline design.
- **Static `.local` DNS:** Requires a manual `/etc/hosts` entry on the client machine; a proper DNS resolver (e.g., CoreDNS external or a wildcard entry) would remove this step.

---

*This project was developed as a Major Project (Đồ án chuyên ngành) at UIT, focusing on modern DevOps Engineering practices — proactive AI-driven scaling, GitOps, DevSecOps, and observability abilities.*
