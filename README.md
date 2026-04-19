# Blog Platform – DevSecOps on AWS EKS

A production-oriented DevSecOps implementation of a full-stack blog application deployed on AWS EKS Auto Mode, with infrastructure provisioned through Terraform, a multi-stage security pipeline via GitHub Actions, and GitOps-based continuous delivery using ArgoCD.

## Tech Stack

| Layer | Tools |
|---|---|
| Infrastructure | AWS EKS Auto Mode, VPC, EBS, ALB, provisioned with Terraform |
| CI/CD | GitHub Actions (7-stage DevSecOps pipeline) |
| Security Scanning | Trivy, Checkov, Hadolint, npm audit |
| GitOps | ArgoCD |
| Application | React (frontend), Node.js/Express (backend), PostgreSQL |
| Ingress | AWS ALB Ingress Controller with HTTPS via ACM and Route 53 |

---

## Repository Structure

```
├── frontend/               # React (Vite) frontend
├── backend/                # Node.js Express API
├── terraform/              # AWS infrastructure as code
│   ├── main.tf             # EKS Auto Mode and VPC
│   ├── variables.tf
│   ├── output.tf
│   └── terraform.tfvars
├── k8s_manifest/           # Kubernetes manifests (split by component)
│   ├── 00-namespace/
│   ├── 01-storage/         # gp3 EBS StorageClass and PVC
│   ├── 02-secrets/         # Database credentials via Kubernetes Secrets
│   ├── 03-database/        # PostgreSQL deployment and service
│   ├── 04-backend/         # Node.js deployment and service
│   ├── 05-frontend/        # React/Nginx deployment and service
│   ├── 06-ingress/         # AWS ALB Ingress and IngressClass
│   └── 07-network/         # NetworkPolicies for pod-level isolation
├── .github/
│   └── workflows/
│       └── pipeline.yaml   # DevSecOps CI/CD pipeline
├── deploy/                 # EC2 bare-metal deployment scripts
└── docker-compose.yaml     # Local development
```

---

## Infrastructure

Provisioned with Terraform using the official AWS EKS and VPC modules:

- **EKS Auto Mode** with managed node pools (general-purpose and system), eliminating manual node group management
- **VPC** with 3 public and 3 private subnets across 3 availability zones, using a single NAT Gateway
- **EBS gp3** encrypted persistent storage for PostgreSQL (10Gi, Retain reclaim policy)
- **KMS** encryption enabled for cluster secrets

```bash
cd terraform/
terraform init
terraform apply
```

---

## CI/CD Pipeline

A 7-stage DevSecOps pipeline triggered on every push and pull request:

```
Stage 1: Lint            ESLint on frontend and backend
Stage 2: SCA             npm audit for high and critical vulnerabilities
Stage 3: IaC Scan        Checkov against Terraform and Kubernetes manifests
Stage 4: Build           Docker images built and pushed to GHCR
Stage 5: Image Scan      Trivy for OS and library CVEs at CRITICAL/HIGH severity
Stage 6: Dockerfile      Hadolint for Dockerfile best practice violations
Stage 7: Manifest Update Image tags updated in k8s_manifest/ and committed
```

Manifest commits trigger ArgoCD to automatically synchronize the cluster state.

---

## Kubernetes Deployment

```bash
kubectl apply -f k8s_manifest/00-namespace/
kubectl apply -f k8s_manifest/01-storage/
kubectl apply -f k8s_manifest/02-secrets/
kubectl apply -f k8s_manifest/03-database/
kubectl apply -f k8s_manifest/04-backend/
kubectl apply -f k8s_manifest/05-frontend/
kubectl apply -f k8s_manifest/06-ingress/
kubectl apply -f k8s_manifest/07-network/
```

### Security Configuration

- **NetworkPolicies** restrict pod-level traffic: the database accepts connections from the backend only, and the backend accepts connections from the frontend only
- **Kubernetes Secrets** manage database credentials with least-privilege access
- **Non-root containers** with dropped Linux capabilities applied to all workloads
- **ReadOnlyRootFilesystem** enabled where the runtime permits

---

## Ingress and DNS

Traffic is routed through the AWS ALB Ingress Controller:

```
Internet → ALB (HTTPS 443)
           ├── /api/*  → backend service (port 5000)
           └── /*      → frontend service (port 80)
```

- HTTPS termination via AWS Certificate Manager (ACM)
- Automatic HTTP to HTTPS redirection
- DNS routing managed via Route 53

---

## Application

A full-stack blog platform supporting post creation, editing, deletion, and commenting. The React frontend is served via Nginx and proxies all `/api` requests to the Node.js backend. Data is persisted in PostgreSQL using EBS-backed storage.

### Local Development

```bash
docker-compose up
```

Frontend runs on `http://localhost:3000` and backend on `http://localhost:5000`.

---

## Architecture

```
                    ┌──────────────────────────────────────┐
                    │          AWS EKS Auto Mode           |
                    │                                      │
Internet → ALB ─────┤──► Frontend (React/Nginx)            │
           HTTPS    │          │                           │
                    │          │ /api/*                    │
                    │          ▼                           │
                    │     Backend (Node.js/Express)        │
                    │          │                           │
                    │          ▼  ClusterIP only           │
                    │     Database (PostgreSQL + EBS)      │
                    └──────────────────────────────────────┘
```
