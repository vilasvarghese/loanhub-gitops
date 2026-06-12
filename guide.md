# LoanHub DevOps Project — Complete Step-by-Step Guide

Full production deployment of a Java + React app on AWS EKS using a GitOps workflow.
Stack: GitHub (polyrepo) → Docker → GitHub Actions CI → Terraform (AWS EKS + RDS + ECR) → Kustomize → ArgoCD.

---

## Architecture Overview

```
Developer pushes code
        │
        ▼
GitHub Actions CI (per repo)
  ├── Build & test (JDK 21 / Node 20)
  ├── Docker build --platform linux/amd64
  ├── Trivy image scan
  ├── Push to ECR
  └── Bump image tag in loanhub-gitops
        │
        ▼
ArgoCD (running in EKS) watches loanhub-gitops
        │
        ▼
EKS Cluster (ap-south-1)
  ├── loanhub namespace
  │   ├── backend  (Spring Boot :8080, context /api)
  │   ├── frontend (nginx :8080)
  │   └── ingress-nginx → AWS ELB
  ├── external-secrets namespace (ESO)
  └── argocd namespace
        │
AWS Services
  ├── ECR  (loanhub-backend, loanhub-frontend)
  ├── RDS  (PostgreSQL 16)
  └── Secrets Manager (DB password)
```

## Polyrepo Layout

| Repo | Purpose |
|------|---------|
| `raj-pro/loanhub-backend` | Spring Boot API |
| `raj-pro/loanhub-frontend` | React + Vite SPA |
| `raj-pro/loanhub-infra` | Terraform (VPC, EKS, RDS, ECR, GitHub OIDC) |
| `raj-pro/loanhub-gitops` | Kustomize manifests + ArgoCD apps |

---

## Phase 1 — Git & GitHub Scaffolding

### 1.1 Create four public repos under `raj-pro`

Create on GitHub (UI or gh CLI):
- `loanhub-backend`
- `loanhub-frontend`
- `loanhub-infra`
- `loanhub-gitops`

### 1.2 Push code to each repo

```sh
# Backend
cd backend
git init && git add -A
git commit -m "feat: initial Spring Boot backend"
git remote add origin https://github.com/raj-pro/loanhub-backend.git
git push -u origin main

# Frontend
cd ../frontend
git init && git add -A
git commit -m "feat: initial React frontend"
git remote add origin https://github.com/raj-pro/loanhub-frontend.git
git push -u origin main

# Infra
cd ../infra
git init && git add -A
git commit -m "feat: initial Terraform infra"
git remote add origin https://github.com/raj-pro/loanhub-infra.git
git push -u origin main

# GitOps
cd ../gitops
git init && git add -A
git commit -m "feat: initial Kustomize manifests"
git remote add origin https://github.com/raj-pro/loanhub-gitops.git
git push -u origin main
```

> **Common error:** `rejected — non-fast-forward` on frontend push.
> Fix: `git pull --rebase origin main` then push again.

---

## Phase 2 — Containerization

### 2.1 Backend Dockerfile (multi-stage, distroless)

```dockerfile
# backend/Dockerfile
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /build
COPY pom.xml .
COPY src ./src
RUN mvn -B -DskipTests package

FROM gcr.io/distroless/java21-debian12:nonroot
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

> **Critical:** Always build with `--platform linux/amd64` when deploying to EKS from an Apple Silicon Mac:
> ```sh
> docker build --platform linux/amd64 -t loanhub-backend .
> ```
> Without this, EKS nodes (x86_64) get an ARM image and return `not found` when pulling — misleading error.

### 2.2 Frontend Dockerfile (nginx-unprivileged)

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_BASE_URL=/api
RUN npm run build

FROM nginxinc/nginx-unprivileged:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

### 2.3 nginx.conf — must use port 8080 (unprivileged image cannot bind 80)

```nginx
server {
    listen 8080;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> **Common error:** nginx fails to start with `bind() to 0.0.0.0:80 failed (13: Permission denied)`.
> Fix: change `listen 80` → `listen 8080` in nginx.conf AND update docker-compose.yml port mapping from `:80` → `:8080`.

### 2.4 docker-compose.yml port mapping

```yaml
frontend:
  ports:
    - "${FRONTEND_PORT:-3000}:8080"   # ← 8080 not 80
```

---

## Phase 3 — CI Pipeline (GitHub Actions)

### 3.1 Backend CI (`backend/.github/workflows/ci.yml`)

Key jobs:
1. `build-test` — JDK 21 Temurin, `mvn verify`
2. `checkstyle` — code style gate
3. `owasp` — dependency vulnerability scan
4. `sonarcloud` — static analysis
5. `build-image` — `docker build --platform linux/amd64`
6. `trivy-scan` — image CVE scan
7. `push-and-deploy` — ECR push + bump image tag in gitops repo (main branch only)

### 3.2 GitHub Secrets required (set per repo)

| Secret | Value | Repos |
|--------|-------|-------|
| `AWS_ROLE_ARN` | Output from `terraform output AWS_ROLE_ARN` | backend, frontend |
| `ECR_BACKEND_URI` | Output from `terraform output ECR_BACKEND_URI` | backend |
| `ECR_FRONTEND_URI` | Output from `terraform output ECR_FRONTEND_URI` | frontend |
| `SONAR_TOKEN` | From SonarCloud | backend |
| `SNYK_TOKEN` | From Snyk | frontend |
| `GITOPS_DEPLOY_KEY` | SSH key with write access to loanhub-gitops | backend, frontend |

### 3.3 OIDC — no static AWS credentials in CI

The CI uses OIDC to assume an IAM role. Never add `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` as GitHub secrets. The GitHub OIDC provider Terraform module creates the trust relationship.

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ap-south-1
```

---

## Phase 4 — Terraform Infrastructure (AWS)

### 4.1 Prerequisites

```sh
brew install terraform awscli
terraform --version   # must be >= 1.5.0
aws configure         # set access key, secret, region ap-south-1
```

> **Common error:** `Error: Invalid required_providers constraint "~> 1.7.0"` if local Terraform is 1.5.x.
> Fix: change `required_version = ">= 1.7.0"` → `">= 1.5.0"` in `versions.tf`.

### 4.2 Module structure

```
infra/
├── main.tf          # wires all modules
├── variables.tf
├── outputs.tf
├── versions.tf
└── modules/
    ├── vpc/         # VPC, subnets, NAT gateway
    ├── eks/         # EKS cluster, node group, OIDC, IRSA roles
    ├── rds/         # PostgreSQL 16, Secrets Manager
    ├── ecr/         # Two ECR repos + lifecycle policies
    └── github-oidc/ # GitHub OIDC provider + CI IAM role
```

### 4.3 Apply

```sh
cd infra
terraform init
terraform plan
terraform apply
```

> **Common error:** `Error: configuring Terraform AWS Provider: no valid credential sources found`.
> Fix: run `aws configure` first, or set `AWS_PROFILE` env var.

### 4.4 EBS CSI driver — IRSA required

The `aws-ebs-csi-driver` addon needs its own IAM role or it hangs for 20 minutes then times out.

```hcl
# In eks module — required or addon creation times out
resource "aws_eks_addon" "ebs_csi" {
  cluster_name             = aws_eks_cluster.this.name
  addon_name               = "aws-ebs-csi-driver"
  service_account_role_arn = aws_iam_role.ebs_csi.arn   # ← mandatory
}
```

> **Common error:** `Error: waiting for EKS Add-On (loanhub-dev:aws-ebs-csi-driver) create: timeout — 20m0s`.
> Fix: create the IRSA role with `AmazonEBSCSIDriverPolicy` and pass it via `service_account_role_arn`.

> **Common error:** `409 ResourceInUseException` during re-apply after timeout.
> Fix: run `terraform apply` again (the addon was actually created; the import race is self-healing).

### 4.5 Capture outputs after apply

```sh
terraform output AWS_ROLE_ARN=github_ci_role_arn        # → set as GitHub secret
terraform output ECR_BACKEND_URI     # → set as GitHub secret
terraform output ECR_FRONTEND_URI    # → set as GitHub secret
terraform output eks_cluster_name    # → used in kubeconfig
terraform output rds_host            # → used in gitops configmap
terraform output eso_role_arn        # → annotate ESO service account
terraform output rds_db_secret_arn   # → used in ExternalSecret
```

Define github repo secrets for frontend for 
- AWS_ROLE_ARN= Terraform output of github_ci_role_arn
- ECR_FRONTEND_URI = Terraform out put of ECR_FRONTEND_URI

Define github repo secrets for backend for 
- AWS_ROLE_ARN= Terraform output of github_ci_role_arn
- ECR_BACKEND_URI = Terraform out put of ECR_BACKEND_URI

change [backend configmap.yaml](https://github.com/vilasvarghese/loanhub-gitops/blob/main/base/backend/configmap.yaml)
        edit the url from terraform output


Create the Token
Step 1: Generate a GitHub Personal Access Token (PAT)
1. Go to https://github.com/settings/tokens?type=beta (Fine-grained tokens)
2. Click "Generate new token"
3. Configure it:
Setting	Value
Token name	loanhub-gitops-token
Expiration	90 days (or custom)
Resource owner	vilasvarghese
Repository access	Only select repositories → select loanhub-gitops
	4	Under Permissions → Repository permissions, set:
Permission	Access
Contents	Read and write
Metadata	Read-only (auto-selected)
	5	Click "Generate token"
1. Copy the token immediately (you won't see it again)
Step 2: Add it as a GitHub Secret
Do this in both loanhub-frontend and loanhub-backend repos:
1. Go to your repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Set:
    * Name: GITOPS_TOKEN
    * Value: paste the token from Step 1
4. Click "Add secret"
Step 3: Re-run the workflow

        

### 4.6 Connect kubectl to EKS

```sh
aws eks update-kubeconfig \
  --name $(terraform output -raw eks_cluster_name) \
  --region ap-south-1
kubectl get nodes   # should show 2 Ready nodes
```

---

## Phase 5 — Kubernetes Manifests (Kustomize)

### 5.1 Directory layout

```
gitops/
├── apps/
│   └── loanhub.yaml          # ArgoCD Application
├── base/
│   ├── backend/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── hpa.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── clustersecretstore.yaml
│   │   └── externalsecret.yaml
│   ├── frontend/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── ingress/
│       ├── kustomization.yaml
│       └── ingress.yaml
└── overlays/
    ├── dev/
    └── prod/
        ├── kustomization.yaml
        ├── backend/replicas.yaml
        └── frontend/replicas.yaml
```

> **Common error:** Kustomize security error when `ingress.yaml` is placed as a standalone file outside a subdirectory.
> Fix: move ingress to `base/ingress/ingress.yaml` with its own `kustomization.yaml` and reference it in the overlay.

### 5.2 Ingress — no rewrite, Prefix path type

```yaml
# base/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: loanhub
  namespace: loanhub
  # NO rewrite-target annotation — Spring Boot handles /api context path itself
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 8080
```

> **Common error:** `GET /api/products 404` even though pods are running.
> Cause: `nginx.ingress.kubernetes.io/rewrite-target: /$2` strips `/api/` before forwarding, but Spring Boot context path is `/api` so it expects `/api/products`, not `/products`.
> Fix: remove the rewrite annotation and use `pathType: Prefix` with path `/api`.

> **Common error:** Overlay `ingress-host.yaml` patch re-adds `host: loanhub.example.com` and ArgoCD selfHeal keeps reverting manual kubectl patches.
> Fix: remove the ingress-host.yaml patch file and its entry from `overlays/prod/kustomization.yaml`.

### 5.3 Backend deployment — liveness probe

```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 15
  failureThreshold: 3
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 45
  periodSeconds: 10
  failureThreshold: 3
```

> **Common error:** Backend pods crash-loop with exit code 143 (SIGTERM). Spring Boot takes ~38 seconds to start.
> Cause 1: `initialDelaySeconds` too low — liveness probe fires before app is ready and kills it.
> Cause 2: `httpGet` probe hits `/api/actuator/health/liveness` but Spring Actuator is not in `pom.xml` → 404 → 3 failures → SIGTERM.
> Fix: use `tcpSocket` probes (just checks the port is open) with `initialDelaySeconds: 60`.

### 5.4 External Secrets Operator

Install ESO via Helm:
```sh
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

Annotate the ESO service account with the IRSA role ARN:
```sh
ESO_ROLE_ARN=$(cd infra && terraform output -raw eso_role_arn)
kubectl annotate serviceaccount external-secrets \
  -n external-secrets \
  eks.amazonaws.com/role-arn="$ESO_ROLE_ARN" \
  --overwrite
kubectl rollout restart deployment/external-secrets -n external-secrets
```

> **Common error:** `ESO_ROLE=$(terraform output -raw <full ARN string>)` — user passes the ARN itself as the output name.
> Fix: `terraform output -raw eso_role_arn` (the output name, not the value).

ClusterSecretStore:
```yaml
# base/backend/clustersecretstore.yaml
apiVersion: external-secrets.io/v1      # ← v1, NOT v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

> **Common error:** `external-secrets.io/ExternalSecret v1beta1 not found, v1 is installed`.
> Fix: change `apiVersion: external-secrets.io/v1beta1` → `external-secrets.io/v1` in both `clustersecretstore.yaml` and `externalsecret.yaml`.

---

## Phase 6 — ArgoCD GitOps CD

### 6.1 Install ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose UI (LoadBalancer or port-forward)
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get initial admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### 6.2 ArgoCD Application manifest

```yaml
# gitops/apps/loanhub.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loanhub
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/raj-pro/loanhub-gitops
    targetRevision: main
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: loanhub
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Apply it:
```sh
kubectl apply -f gitops/apps/loanhub.yaml
```

> **Common error:** Two separate ArgoCD Application resources both pointing to `overlays/prod` cause resource conflicts.
> Fix: use a single Application named `loanhub` that covers the full overlay.

### 6.3 Force sync (when ArgoCD polls too slowly)

```sh
# Hard refresh — forces re-fetch from git
kubectl annotate application loanhub -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite

# Trigger sync manually
kubectl -n argocd patch application loanhub \
  --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {"revision": "HEAD"}}}'
```

> **Common error:** `kubectl patch ingress` immediately reverts — ArgoCD `selfHeal: true` is working correctly and enforcing git state.
> Fix: always commit the change to git first; ArgoCD will apply it. Never fight selfHeal with manual kubectl patches.

### 6.4 Check sync status

```sh
kubectl get application loanhub -n argocd
kubectl describe application loanhub -n argocd | grep -A 5 "Message:"
```

---

## Common Errors Reference

| Error | Cause | Fix |
|-------|-------|-----|
| Lombok `TypeTag::UNKNOWN` on JDK 26 | Lombok 1.18.30 doesn't support JDK 23+ | Bump to `1.18.46` in `pom.xml`, add `annotationProcessorPaths` |
| Mockito can't mock concrete class | JDK 26 bytecode incompatibility | Build with JDK 21 Temurin (`JAVA_HOME` override) |
| Spring context fails at startup (`lambda Converter`) | Spring's `ApplicationConversionService` scans for `@Bean` converters; lambda erases generics | Remove `@Bean` from `firebaseAuthenticationConverter()`, make it `private` |
| `ImagePullBackOff` with `not found` on EKS | Image built for ARM on Apple Silicon, EKS is x86_64 | Add `--platform linux/amd64` to every `docker build` |
| Backend pods exit code 143 (crash-loop) | Liveness probe kills pod before Spring Boot finishes starting | Use `tcpSocket` probes with `initialDelaySeconds: 60` |
| `GET /api/products 404` in browser | nginx ingress rewrite strips `/api/` but Spring Boot context path expects it | Remove `rewrite-target` annotation; use `pathType: Prefix` |
| `v1beta1 not found` for ExternalSecret | ESO v0.9+ uses `v1` not `v1beta1` | Change `apiVersion: external-secrets.io/v1beta1` → `v1` |
| ArgoCD app OutOfSync / kubectl reverts | `selfHeal: true` enforces git state — manual patches are overwritten | Commit the fix to git, then force a hard refresh in ArgoCD |
| EBS CSI 20-minute timeout | Addon needs IRSA role to call EC2 APIs | Add `service_account_role_arn` to `aws_eks_addon.ebs_csi` |
| Terraform `409 ResourceInUseException` | Race between destroy+create of addons | Re-run `terraform apply` — self-healing |

---

## Day-2 Operations

### Check pod health

```sh
kubectl get pods -n loanhub
kubectl logs -n loanhub -l app=backend --tail=100
kubectl describe pod -n loanhub <pod-name>
```

### Connect local terminal to EKS

```sh
aws eks update-kubeconfig --name loanhub-dev --region ap-south-1
kubectl config current-context   # verify
kubectl get nodes                 # should show Ready nodes
```

### Manual image push (skip CI)

```sh
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <account>.dkr.ecr.ap-south-1.amazonaws.com

docker build --platform linux/amd64 -t <ECR_BACKEND_URI>:latest backend/
docker push <ECR_BACKEND_URI>:latest

kubectl rollout restart deployment/backend -n loanhub
```

### Force ArgoCD to pick up latest git commit

```sh
kubectl annotate application loanhub -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

### Tear down all AWS resources

```sh
cd infra
terraform destroy
```

---

## Demo Logins

Seeded automatically by Spring Boot `DataInitializer` on first start:

| Role | Email | Password |
|------|-------|----------|
| Customer | raj.nayan@scaler.com | password123 |
| Officer | officer@hdfcland.com | password123 |
| Admin | admin@hdfcland.com | password123 |
