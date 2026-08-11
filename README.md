# Axion-UI — Industrial IoT Condition Monitoring Platform

End-to-end DevOps lab: **5 microservices** consolidated into a monorepo with
**GitHub Actions CI/CD**, **Trivy image scanning**, **SonarQube code quality**,
**GHCR image registry**, **Azure Kubernetes Service (AKS)**, and **ArgoCD GitOps**.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Repository Structure](#repository-structure)
3. [Prerequisites](#prerequisites)
4. [Lab 1 — Run Locally (Docker Compose)](#lab-1--run-locally-docker-compose)
5. [Lab 2 — CI/CD Pipeline (GitHub Actions)](#lab-2--cicd-pipeline-github-actions)
6. [Lab 3 — Deploy to AKS with ArgoCD](#lab-3--deploy-to-aks-with-argocd)
7. [Verify the Deployment](#verify-the-deployment)
8. [Troubleshooting](#troubleshooting)
9. [Useful Commands](#useful-commands)

---

## Architecture

```
Refinery devices (simulated)
    |
    v
axion-data-simulator  ──POST──►  axion-ingestion-service  ──►  PostgreSQL 17
                                                              ▲
                                                              │
        Browser ──► NGINX Ingress ──► axion-ui (Nginx) ──/api/*──► axion-telemetry-query-service
                  (axion.santansre.shop)
```

| Service | Purpose | Stack |
|---------|---------|-------|
| **axion-ui** | Dashboard frontend + API reverse proxy | React + Vite + TypeScript, served by Nginx |
| **axion-telemetry-query-service** | Read API for dashboard charts | FastAPI + asyncpg |
| **axion-ingestion-service** | Write API for telemetry ingestion | FastAPI + asyncpg |
| **axion-data-simulator** | Generates synthetic telemetry data | Python worker |
| **axion-database-schema** | Database migrations | PostgreSQL SQL scripts |
| **PostgreSQL** | Persistent data store | postgres:17-alpine (StatefulSet) |

---

## Repository Structure

```
Axion-UI/
├── .github/workflows/ci.yml          # GitHub Actions CI/CD (matrix over 5 services)
├── axion-ui/                          # Frontend (React + Nginx)
│   ├── Dockerfile
│   ├── nginx.conf
│   └── helm/axion-ui/                 # Helm chart
├── axion-telemetry-query-service/     # Query API
│   ├── Dockerfile
│   ├── main.py
│   └── helm/...
├── axion-ingestion-service/           # Ingestion API
│   ├── Dockerfile
│   ├── main.py
│   └── helm/...
├── axion-data-simulator/              # Data simulator
│   ├── Dockerfile
│   └── helm/...
├── axion-database-schema/             # SQL migrations
│   └── Dockerfile
├── gitops/
│   ├── argocd/
│   │   ├── project.yaml               # ArgoCD AppProject
│   │   ├── app-of-apps.yaml           # Parent Application
│   │   └── apps/*.yaml                # One Application per service
│   └── db/
│       ├── values.yaml                # Postgres config
│       └── templates/statefulset.yaml # Postgres StatefulSet
├── docker-compose.yml                 # Local development
├── .env.example                       # Environment variables
└── README.md                          # This file
```

---

## Prerequisites

### For Local Development (Lab 1)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- Git

### For CI/CD (Lab 2)
- GitHub account with the repository forked or pushed
- GitHub PAT (Personal Access Token) with `read:packages` and `write:packages` scopes
- (Optional) SonarQube server + token for code quality scans

### For Kubernetes Deployment (Lab 3)
- Azure CLI (`az`) installed and logged in
- `kubectl` installed
- `helm` installed
- `argocd` CLI installed
- An AKS cluster (Standard_D2s_v7 or larger recommended)

---

## Lab 1 — Run Locally (Docker Compose)

### Step 1: Clone the repository

```bash
git clone https://github.com/santandevops01012025/Axion-UI.git
cd Axion-UI
```

### Step 2: Copy the environment file

```bash
cp .env.example .env
```

### Step 3: Start all services

```bash
docker compose up --build
```

This starts 6 containers:
- PostgreSQL database (port 5432)
- Schema migration (runs once, then exits)
- Telemetry query service (port 8000)
- Ingestion service (port 8001)
- Data simulator (sends data every 5 seconds)
- UI dashboard (port 8080)

### Step 4: Open the dashboard

Open your browser to **http://localhost:8080**

### Step 5: Test the APIs

```bash
# Health checks
curl http://localhost:8000/health        # Query service
curl http://localhost:8001/health        # Ingestion service

# Query telemetry data
curl http://localhost:8000/devices

# Ingest a test reading
curl -X POST http://localhost:8001/api/v1/telemetry/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "deviceId": "AX-TEST-001",
    "deviceType": "PUMP",
    "refineryRegion": "NORTH_PLANT",
    "timestamp": "2026-01-01T12:00:00",
    "metrics": {"temperature": 85.0, "vibration": 6.5, "current": 30.0}
  }'
```

### Step 6: Stop the stack

```bash
docker compose down          # stop containers
docker compose down -v       # stop and delete data volume
```

---

## Lab 2 — CI/CD Pipeline (GitHub Actions)

The pipeline in `.github/workflows/ci.yml` runs for every push to `main` and
processes all 5 services in parallel via a matrix strategy.

### What the pipeline does

```
Push to main
    |
    v
SonarQube scan (code quality gate)  ──►  Build Docker image  ──►  Trivy scan (security gate)
                                                                        |
                                                                        v
                                                              Push to ghcr.io/santandevops01012025/<service>:latest
                                                                        |
                                                                        v
                                                              ArgoCD detects new image  ──►  Rollout to AKS
```

### Step 1: Configure GitHub Secrets

Go to your repo **Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|--------|-------|
| `SONAR_HOST_URL` | Your SonarQube server URL (e.g., `http://sonarqube.example.com`) |
| `SONAR_TOKEN` | SonarQube authentication token |

### Step 2: Enable SonarQube (optional)

Go to **Settings → Secrets and variables → Actions → Variables** and create:

| Variable | Value |
|----------|-------|
| `ENABLE_SONARQUBE` | `true` |

If not set, the SonarQube step is skipped and the pipeline proceeds directly to build.

### Step 3: Push code to trigger CI

```bash
git add .
git commit -m "feat: initial commit"
git push origin main
```

### Step 4: Watch the pipeline

Go to **Actions** tab in GitHub to watch all 10 jobs (5 SonarQube + 5 Build/Trivy/Push).

### What each pipeline step does

1. **SonarQube Scan** — Static code analysis. Blocks the build if the quality gate fails.
2. **Docker Build** — Multi-stage build for each service.
3. **Trivy Gate** — Scans the image for CRITICAL vulnerabilities. Fails the pipeline if any are found.
4. **Trivy SARIF Upload** — Uploads scan results to GitHub Security tab (informational).
5. **GHCR Push** — Pushes the image to `ghcr.io/santandevops01012025/<service>:latest`.

### Understanding the Trivy security scan

- Uses `format: table` with `exit-code: 1` and `severity: CRITICAL` as the gate
- SARIF upload is non-blocking (`if: always()`) for GitHub Security visibility
- Known CVEs without fixes are listed in each service's `.trivyignore` file
- Standard industry practice: only block on CRITICAL severity

---

## Lab 3 — Deploy to AKS with ArgoCD

### Prerequisites

```bash
# Install required CLIs
# Azure CLI: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
# kubectl:   az aks install-cli
# helm:      https://helm.sh/docs/intro/install/
# argocd:    curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64

# Login to Azure
az login
```

### Step 1: Create the AKS cluster

```bash
# Create a resource group
az group create \
  --name rg-compute-dev-eastus \
  --location eastus

# Create the AKS cluster
az aks create \
  --resource-group rg-compute-dev-eastus \
  --name aks-enterprise-dev-eastus \
  --node-count 2 \
  --node-vm-size Standard_D2s_v7 \
  --generate-ssh-keys \
  --enable-addons monitoring
```

### Step 2: Connect to the cluster

```bash
az aks get-credentials \
  --resource-group rg-compute-dev-eastus \
  --name aks-enterprise-dev-eastus \
  --overwrite-existing

# Verify connection
kubectl get nodes
```

### Step 3: Install ArgoCD

```bash
# Create the argocd namespace
kubectl create namespace argocd

# Install ArgoCD via Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=LoadBalancer \
  --set configs.params.server\.insecure=true

# Wait for ArgoCD to be ready
kubectl get pods -n argocd -w

# Get the ArgoCD external IP
kubectl get svc argocd-server -n argocd

# Get the initial admin password
argocd admin initial-password -n argocd
```

### Step 4: Log in to ArgoCD

```bash
# Login (accept the insecure prompt)
argocd login <ARGOCD_EXTERNAL_IP> \
  --username admin \
  --password <INITIAL_PASSWORD> \
  --insecure
```

Open the ArgoCD UI at `http://<ARGOCD_EXTERNAL_IP>` in your browser.

### Step 5: Create the axion namespace

```bash
kubectl create namespace axion
```

### Step 6: Create the database secret

```bash
kubectl -n axion create secret generic axion-db-secret \
  --from-literal=POSTGRES_USER=axion_user \
  --from-literal=POSTGRES_PASSWORD='P@ssw01rd@123' \
  --from-literal=POSTGRES_DB=axion_db

kubectl -n axion create secret generic axion-db-credentials \
  --from-literal=DATABASE_URL='postgresql://axion_user:P%40ssw01rd%40123@axion-postgres:5432/axion_db'
```

### Step 7: Register the Git repo in ArgoCD

```bash
argocd repo add https://github.com/santandevops01012025/Axion-UI.git \
  --username <YOUR_GITHUB_USERNAME> \
  --password <YOUR_GITHUB_PAT> \
  --upsert
```

### Step 8: Apply the ArgoCD project

```bash
kubectl apply -n argocd -f gitops/argocd/project.yaml
```

### Step 9: Deploy the app-of-apps

```bash
kubectl apply -n argocd -f gitops/argocd/app-of-apps.yaml
```

This creates all child Applications which ArgoCD will sync automatically:

```bash
# Watch the applications sync
argocd app list
argocd app get axion-app-of-apps

# Watch pods come up
kubectl get pods -n axion -w
```

### Step 10: Verify all pods are running

```bash
kubectl get pods -n axion -o wide
```

Expected output:
```
NAME                                             READY   STATUS    RESTARTS   AGE
axion-data-simulator-xxxxx                       1/1     Running   0          5m
axion-ingestion-service-xxxxx                    1/1     Running   0          5m
axion-postgres-0                                 1/1     Running   0          5m
axion-telemetry-query-service-xxxxx              1/1     Running   0          5m
axion-ui-xxxxx                                   1/1     Running   0          5m
```

### Step 11: Install NGINX Ingress Controller

```bash
# Add the ingress-nginx Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install the ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Wait for the external IP to be assigned
kubectl get svc -n ingress-nginx ingress-nginx-controller -w

# Note the EXTERNAL-IP (e.g., 48.206.108.209)
```

### Step 12: Configure the health probe annotation

The Standard Load Balancer creates health probes using path `/` by default. The
ingress controller returns 404 on `/` which causes probes to fail and blocks all
traffic. Fix this by setting the probe path to `/healthz`:

```bash
kubectl annotate svc ingress-nginx-controller -n ingress-nginx \
  "service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path=/healthz" \
  --overwrite
```

Wait 2-3 minutes for the Standard LB to reconfigure the probes.

### Step 13: Create the Ingress resource

```bash
kubectl apply -f gitops/argocd/apps/axion-ingress.yaml
```

This creates an Ingress rule that routes traffic from `axion.santansre.shop`
to the `axion-ui` service on port 80.

### Step 14: Add NSG rules for HTTP traffic

```bash
# Get the MC resource group name
MC_RG=$(az aks show \
  --resource-group rg-compute-dev-eastus \
  --name aks-enterprise-dev-eastus \
  --query "nodeResourceGroup" -o tsv)

# Get the NSG name
NSG_NAME=$(az network nsg list \
  --resource-group $MC_RG \
  --query "[0].name" -o tsv)

# Allow HTTP (port 80) from any source
az network nsg rule create \
  --resource-group $MC_RG \
  --nsg-name $NSG_NAME \
  --name AllowHTTP \
  --priority 100 \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow \
  --direction Inbound \
  --description "Allow HTTP for Ingress Controller"

# Allow Azure LB health probes
az network nsg rule create \
  --resource-group $MC_RG \
  --nsg-name $NSG_NAME \
  --name AllowHealthProbe \
  --priority 101 \
  --destination-port-ranges 32012 32047 \
  --protocol Tcp \
  --source-address-prefixes AzureLoadBalancer \
  --access Allow \
  --direction Inbound \
  --description "Allow Azure LB health probes to ingress nodePorts"
```

### Step 15: Configure DNS

Point your domain to the ingress external IP:

| Type | Host | Value | TTL |
|------|------|-------|-----|
| A | axion.santansre.shop | 48.206.108.209 | 300 |

```bash
# Verify DNS resolution
nslookup axion.santansre.shop
```

### Step 16: Verify external access

```bash
# Test with curl (add Host header to match ingress rule)
curl -H "Host: axion.santansre.shop" http://48.206.108.209/

# Test the API proxy
curl -H "Host: axion.santansre.shop" http://48.206.108.209/api/devices

# Open in browser
# http://axion.santansre.shop
```

---

## Verify the Deployment

### Test via the public ingress (recommended)

Open **http://axion.santansre.shop** in your browser. You should see the
Axion Dashboard login page.

**Login credentials** (hardcoded in `Login.tsx`):

| Field | Value |
|-------|-------|
| Email | `info@devopsinsiders.com` |
| Password | `P@ssw01rd@123` |

> Note: This is hardcoded client-side authentication (not secure for
> production). For a lab/demo environment it works, but for real use you'd
> want proper backend auth.

### Test the API from the command line

```bash
# Devices endpoint (through ingress)
curl -H "Host: axion.santansre.shop" http://48.206.108.209/api/devices

# Dashboard summary
curl -H "Host: axion.santansre.shop" http://48.206.108.209/api/dashboard/summary
```

### Test all services from inside the cluster

```bash
# Find a running pod to exec into
POD=$(kubectl get pod -n axion -l app.kubernetes.io/name=axion-data-simulator -o jsonpath='{.items[0].metadata.name}')

# Health checks
kubectl exec $POD -n axion -- python3 -c "import urllib.request; print(urllib.request.urlopen('http://axion-telemetry-query-service:8000/health').read().decode())"
kubectl exec $POD -n axion -- python3 -c "import urllib.request; print(urllib.request.urlopen('http://axion-ingestion-service:8000/health').read().decode())"

# Dashboard summary
kubectl exec $POD -n axion -- python3 -c "import urllib.request,json; print(json.dumps(json.loads(urllib.request.urlopen('http://axion-telemetry-query-service:8000/dashboard/summary').read().decode()), indent=2))"

# List all devices
kubectl exec $POD -n axion -- python3 -c "import urllib.request,json; data=json.loads(urllib.request.urlopen('http://axion-telemetry-query-service:8000/devices').read().decode()); print(f'{len(data)} devices found')"

# Test the UI
kubectl exec $POD -n axion -- python3 -c "import urllib.request; r=urllib.request.urlopen('http://axion-ui:80/'); print(f'UI status: {r.status}, length: {len(r.read())} bytes')"

# Test the nginx API proxy (UI -> backend)
kubectl exec $POD -n axion -- python3 -c "import urllib.request; print(urllib.request.urlopen('http://axion-ui:80/api/health').read().decode())"
kubectl exec $POD -n axion -- python3 -c "import urllib.request,json; print(json.dumps(json.loads(urllib.request.urlopen('http://axion-ui:80/api/dashboard/summary').read().decode()), indent=2))"
```

### Test via port-forward (alternative)

```bash
# Terminal 1: UI dashboard
kubectl port-forward svc/axion-ui 8080:80 -n axion
# Open http://localhost:8080

# Terminal 2: Query API
kubectl port-forward svc/axion-telemetry-query-service 8000:8000 -n axion
curl http://localhost:8000/health

# Terminal 3: Ingestion API
kubectl port-forward svc/axion-ingestion-service 8001:8000 -n axion
curl http://localhost:8001/health
```

### Test the database

```bash
kubectl exec axion-postgres-0 -n axion -- pg_isready -U axion_user -d axion_db
kubectl exec axion-postgres-0 -n axion -- psql -U axion_user -d axion_db -c "SELECT count(*) FROM telemetry"
kubectl exec axion-postgres-0 -n axion -- psql -U axion_user -d axion_db -c "\dt"
```

---

## Troubleshooting

### Pod stuck in CrashLoopBackOff

```bash
# Check logs
kubectl logs <pod-name> -n axion --tail=50

# Common causes:
# - Database not ready (connection refused) → wait for postgres pod
# - nginx permission denied → check Dockerfile USER directive
# - Missing secret → verify axion-db-secret exists
```

### Pod stuck in Pending

```bash
# Check why it can't schedule
kubectl describe pod <pod-name> -n axion

# Common causes:
# - Insufficient CPU → scale up node pool or reduce resource requests
# - Too many pods → node max-pods limit reached (default 30)
# - PVC not bound → check PersistentVolumeClaim status
```

### Pod in ImagePullBackOff

```bash
# Check the event
kubectl describe pod <pod-name> -n axion | grep -A5 Events

# Common causes:
# - Image not in registry → check ghcr.io/santandevops01012025/<service>
# - Missing imagePullSecret → create ghcr-pull secret
# - Wrong tag → verify image tag in helm values
```

### ArgoCD app stuck OutOfSync

```bash
# Force sync
argocd app sync <app-name>

# Check what's different
argocd app diff <app-name>
```

### ArgoCD app stuck Progressing

```bash
# Check the app status
argocd app get <app-name>

# Check pod events
kubectl describe pod <pod-name> -n axion
```

### Database not accepting connections

```bash
# Check postgres pod
kubectl logs axion-postgres-0 -n axion --tail=20

# Check if the secret exists
kubectl get secret axion-db-secret -n axion

# Check if PVC is bound
kubectl get pvc -n axion
```

### External access times out (ingress)

```bash
# 1. Check ingress resource exists
kubectl get ingress -n axion

# 2. Check ingress controller external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller

# 3. Verify the health probe annotation
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.metadata.annotations}'

# Expected: service.beta.kubernetes.io/azure-load-balancer-health-probe-request-path=/healthz

# 4. Check LB probe path
az network lb probe list \
  --resource-group MC_rg-compute-dev-eastus_aks-enterprise-dev-eastus_eastus \
  --lb-name kubernetes \
  --query "[].{name:name, path:requestPath}" -o table

# 5. Verify NSG rules
az network nsg rule list \
  --resource-group MC_rg-compute-dev-eastus_aks-enterprise-dev-eastus_eastus \
  --nsg-name aks-agentpool-10099729-nsg \
  --query "[?direction=='Inbound'].{name:name, priority:priority, dstPort:destinationPortRange}" -o table

# 6. Test from inside cluster
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  curl -s -o /dev/null -w "%{http_code}" http://localhost/

# 7. Test with Host header from your machine
curl -v -H "Host: axion.santansre.shop" http://<EXTERNAL-IP>/
```

**Common cause:** The Standard LB health probe checks path `/` by default. The
ingress controller returns 404 on `/`, so probes fail and the LB drops all
traffic. The fix is the annotation in Step 12.

### Ingress returns 404 for API routes

```bash
# Check that the ingress backend points to axion-ui service
kubectl get ingress axion-ingress -n axion -o yaml

# Verify nginx proxy_pass has trailing slash (strips /api/ prefix)
# The nginx.conf should have: proxy_pass http://axion-telemetry-query-service:8000/;
```

---

## Useful Commands

### Cluster Management

```bash
# Get all resources in the axion namespace
kubectl get all -n axion

# Watch pods in real time
kubectl get pods -n axion -w

# Get pod resource usage
kubectl top pods -n axion

# Check node resources
kubectl describe node <node-name> | grep -A10 "Allocated resources"
```

### ArgoCD

```bash
# List all apps
argocd app list

# Get app details
argocd app get <app-name>

# Sync an app
argocd app sync <app-name>

# View app logs
argocd app logs <app-name>

# Rollback
argocd app rollback <app-name> <revision>
```

### CI/CD

```bash
# Trigger a manual workflow run
gh workflow run ci.yml

# Check recent runs
gh run list --limit 5

# Watch a run
gh run watch <run-id>

# View logs for a run
gh run view <run-id> --log
```

### Database

```bash
# Connect to postgres
kubectl exec -it axion-postgres-0 -n axion -- psql -U axion_user -d axion_db

# Common psql commands
\dt              # list tables
\d telemetry     # describe table
SELECT count(*) FROM telemetry;
SELECT DISTINCT device_id FROM telemetry;
```

### Cleanup

```bash
# Delete the ingress resource
kubectl delete ingress axion-ingress -n axion

# Delete the axion namespace (removes all workloads)
kubectl delete namespace axion

# Delete the ArgoCD apps
kubectl delete -n argocd -f gitops/argocd/app-of-apps.yaml
kubectl delete -n argocd -f gitops/argocd/project.yaml

# Uninstall the ingress controller
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx

# Uninstall ArgoCD
helm uninstall argocd -n argocd
kubectl delete namespace argocd

# Delete the AKS cluster
az aks delete \
  --resource-group rg-compute-dev-eastus \
  --name aks-enterprise-dev-eastus \
  --yes --no-wait

# Delete the resource group
az group delete --name rg-compute-dev-eastus --yes --no-wait
```

---

## Secrets Reference

### GitHub Actions Secrets

| Secret | Purpose |
|--------|---------|
| `SONAR_HOST_URL` | SonarQube server URL |
| `SONAR_TOKEN` | SonarQube authentication token |
| `GITHUB_TOKEN` | Auto-provided by GitHub Actions |

### GitHub Actions Variables

| Variable | Purpose |
|----------|---------|
| `ENABLE_SONARQUBE` | Set to `true` to enable SonarQube scans |

### Kubernetes Secrets

| Secret | Namespace | Keys |
|--------|-----------|------|
| `axion-db-secret` | axion | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` |
| `axion-db-credentials` | axion | `DATABASE_URL` |

---

## GHCR Images

All images are published to: `ghcr.io/santandevops01012025/<service>:latest`

| Image | Description |
|-------|-------------|
| `ghcr.io/santandevops01012025/axion-ui` | Frontend (React + Nginx) |
| `ghcr.io/santandevops01012025/axion-telemetry-query-service` | Query API |
| `ghcr.io/santandevops01012025/axion-ingestion-service` | Ingestion API |
| `ghcr.io/santandevops01012025/axion-data-simulator` | Data simulator |
| `ghcr.io/santandevops01012025/axion-database-schema` | Database schema |

---

## License

Internal — Axion DevOps Lab
