# CI/CD Pipeline → EKS with ArgoCD

A production-style GitOps pipeline: push code → GitHub Actions builds & pushes Docker image to ECR → ArgoCD auto-syncs Helm chart to EKS.

---

## Architecture

```
Developer pushes code
        │
        ▼
┌─────────────────────┐
│   GitHub Actions    │
│  1. Lint & Test     │
│  2. Docker Build    │
│  3. Push to ECR     │
│  4. Update Helm tag │
└────────┬────────────┘
         │ git push values.yaml
         ▼
┌─────────────────────┐
│   GitHub Repo       │  ◄─── ArgoCD watches this
│   helm/myapp/       │
│   values.yaml       │
└────────┬────────────┘
         │ detects change
         ▼
┌─────────────────────┐
│      ArgoCD         │
│  Auto-sync to EKS   │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   AWS EKS Cluster   │
│  Deployment + HPA   │
│  Service + Ingress  │
└─────────────────────┘
```

---

## Project Structure

```
cicd-eks-argocd/
├── app/                          # Flask application
│   ├── app.py
│   └── requirements.txt
├── Dockerfile                    # Multi-stage build
├── .github/
│   └── workflows/
│       └── ci-cd.yml             # GitHub Actions pipeline
├── helm/
│   └── myapp/                    # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml           # ← auto-updated by CI
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── hpa.yaml
│           └── ingress.yaml
├── argocd/
│   └── application.yaml          # ArgoCD Application manifest
└── terraform/
    ├── modules/
    │   ├── eks/                  # EKS + IAM module
    │   └── ecr/                  # ECR + lifecycle policy
    └── environments/
        └── dev/                  # Dev environment entry point
```

---

## Prerequisites

- AWS CLI configured (`aws configure`)
- `kubectl`, `helm`, `terraform` installed
- ArgoCD installed on your EKS cluster

---

## Setup Guide

### Step 1 — Provision Infrastructure

```bash
cd terraform/environments/dev

# Fill in your values
cp terraform.tfvars.example terraform.tfvars

terraform init
terraform plan
terraform apply
```

Update your kubeconfig after cluster is ready:
```bash
aws eks update-kubeconfig --region ap-south-1 --name my-eks-cluster-dev
```

### Step 2 — Install ArgoCD on EKS

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### Step 3 — Register the ArgoCD Application

```bash
# Replace <YOUR_USERNAME> in argocd/application.yaml first
kubectl apply -f argocd/application.yaml
```

ArgoCD will now watch your repo and auto-deploy on every change to `helm/myapp/`.

### Step 4 — Add GitHub Actions Secrets

In your GitHub repo → Settings → Secrets → Actions:

| Secret Name          | Value                              |
|----------------------|------------------------------------|
| `AWS_ACCESS_KEY_ID`  | Your IAM user access key           |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user secret key        |
| `AWS_ACCOUNT_ID`     | Your 12-digit AWS account ID       |
| `GH_PAT`             | GitHub PAT with `repo` write scope |

### Step 5 — Push Code and Watch the Pipeline

```bash
git add .
git commit -m "feat: initial app"
git push origin main
```

**Watch the flow:**
1. GitHub Actions tab → CI/CD Pipeline runs
2. ECR → new image tag appears
3. ArgoCD UI → deployment syncs automatically
4. `kubectl get pods -n myapp` → pods rolling update

---

## Key Concepts Demonstrated

| Concept | Where |
|---------|-------|
| Multi-stage Docker build | `Dockerfile` |
| GitHub Actions matrix | `.github/workflows/ci-cd.yml` |
| Trivy container scanning | CI job 2 |
| GitOps via ArgoCD | `argocd/application.yaml` |
| Helm templating | `helm/myapp/templates/` |
| HPA autoscaling | `helm/myapp/templates/hpa.yaml` |
| Terraform modules | `terraform/modules/` |
| Remote state (S3 + DynamoDB) | `terraform/environments/dev/main.tf` |

---

## Certifications This Maps To

- ✅ AWS Certified AI Practitioner — ECR, EKS, IAM
- ✅ Terraform for DevOps — module structure, remote state
- ✅ Docker Certification — multi-stage Dockerfile
- ✅ GitHub Actions — full CI/CD pipeline
# cicd-eks-argocd
