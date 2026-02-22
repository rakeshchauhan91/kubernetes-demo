# MyApi

A **.NET 8 Minimal API** containerized with Docker and deployed to **Kubernetes** (Docker Desktop). Connects to **SQL Server** running as a Docker container. Includes full local development setup, Kubernetes manifests, and deployment workflow.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
  - [1. Clone & Build](#1-clone--build)
  - [2. Start SQL Server](#2-start-sql-server)
  - [3. Run Locally (without Docker)](#3-run-locally-without-docker)
  - [4. Run with Docker](#4-run-with-docker)
  - [5. Deploy to Kubernetes](#5-deploy-to-kubernetes)
- [Environment Configuration](#environment-configuration)
- [API Endpoints](#api-endpoints)
- [Kubernetes Resources](#kubernetes-resources)
- [Updating the API](#updating-the-api)
- [Troubleshooting](#troubleshooting)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | .NET 8 (Minimal API) |
| ORM | Entity Framework Core 8 |
| Database | SQL Server 2022 (Docker container) |
| Containerization | Docker Desktop (Windows 11) |
| Orchestration | Kubernetes 1.31 (Docker Desktop — kubeadm) |
| Ingress | NGINX Ingress Controller |
| Config | Kubernetes ConfigMap + Secret |

---

## Project Structure

```
sampleapi/
├── MyApi/
│   ├── Program.cs                  # App entry point, routes, DbContext
│   ├── MyApi.csproj
│   ├── appsettings.json            # Default config (connection string)
│   ├── appsettings.Development.json
│   └── Dockerfile                  # Multi-stage build
├── k8s/
│   ├── 1-configmap.yaml            # Non-secret env vars
│   ├── 2-secret.yaml               # DB credentials (do not commit!)
│   ├── 3-deployment.yaml           # Pod spec, replicas, probes
│   ├── 4-service.yaml              # LoadBalancer + NodePort + ClusterIP
│   └── 5-ingress.yaml              # NGINX ingress rules (api.local)
├── docker-compose.yml              # SQL Server container
├── deploy.ps1                      # One-command deploy script
├── kind-multinode.yaml             # Multi-node Kind cluster config
└── .gitignore
```

---

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Docker Desktop | Latest | [docker.com](https://www.docker.com/products/docker-desktop/) |
| Kubernetes | 1.31+ (kubeadm mode) | Enable in Docker Desktop Settings |
| .NET SDK | 8.0+ | [dotnet.microsoft.com](https://dotnet.microsoft.com/download) |
| kubectl | Latest | Bundled with Docker Desktop |
| Helm | 3.x | `winget install Helm.Helm` |

### Docker Desktop Kubernetes Setup

```
Docker Desktop → Settings → Kubernetes
→ Enable Kubernetes
→ Provisioning method: Kubeadm   ← important (not Kind)
→ Apply & Restart
```

Verify:
```powershell
kubectl config use-context docker-desktop
kubectl get nodes
# NAME             STATUS   ROLES           AGE
# docker-desktop   Ready    control-plane   Xm
```

---

## Getting Started

### 1. Clone & Build

```powershell
git clone <your-repo-url>
cd sampleapi

# Restore dependencies
cd MyApi
dotnet restore
```

### 2. Start SQL Server

SQL Server runs as a Docker container with a persistent named volume.

```powershell
# From project root (where docker-compose.yml lives)
docker-compose up -d

# Wait ~30 seconds, then create the database
docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd `
  -S localhost -U sa -P "YourStr0ngPassword!" `
  -Q "CREATE DATABASE MyApiDb"

# Verify
docker-compose ps
# NAME        STATUS
# sqlserver   Up (healthy)  ✅
```

Connect via SSMS:
```
Server:   localhost,1433
Auth:     SQL Server Authentication
Login:    sa
Password: YourStr0ngPassword!
```

### 3. Run Locally (without Docker)

```powershell
cd MyApi
dotnet run

# API available at:
# http://localhost:5000
# http://localhost:5000/swagger
```

### 4. Run with Docker

```powershell
# Build the image
docker build -t myapi:latest ./MyApi

# Run container (SQL Server must be running via docker-compose)
docker run -d -p 8080:8080 `
  --name myapicontainer `
  -e ASPNETCORE_ENVIRONMENT=Development `
  -e "ConnectionStrings__DefaultConnection=Server=host.docker.internal,1433;Database=MyApiDb;User Id=sa;Password=YourStr0ngPassword!;TrustServerCertificate=True" `
  myapi:latest

# View logs
docker logs myapicontainer -f

# Test
curl http://localhost:8080/health

# Open Swagger
# http://localhost:8080/swagger
```

> **Note:** `host.docker.internal` resolves to your Windows host machine from inside any container — this is how the API container reaches the SQL Server container on port 1433.

### 5. Deploy to Kubernetes

#### One-time setup — NGINX Ingress Controller

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml

kubectl wait --namespace ingress-nginx `
  --for=condition=ready pod `
  --selector=app.kubernetes.io/component=controller `
  --timeout=120s
```

#### One-time setup — hosts file

Open Notepad as Administrator and add to `C:\Windows\System32\drivers\etc\hosts`:
```
127.0.0.1  api.local
```

#### Deploy

```powershell
# Build image (Kubernetes uses Docker's local image cache directly)
docker build -t myapi:latest ./MyApi

# Apply manifests in order
kubectl apply -f k8s/1-configmap.yaml
kubectl apply -f k8s/2-secret.yaml
kubectl apply -f k8s/3-deployment.yaml
kubectl apply -f k8s/4-service.yaml
kubectl apply -f k8s/5-ingress.yaml

# Watch pods come up
kubectl get pods -w
# NAME                      READY   STATUS    RESTARTS
# my-api-xxx-yyy            1/1     Running   0        ✅
# my-api-xxx-zzz            1/1     Running   0        ✅
```

#### Test all access methods

```powershell
# LoadBalancer (port 80)
curl http://localhost/health

# NodePort (port 30080)
curl http://localhost:30080/health

# Ingress (api.local → needs hosts file entry)
curl http://api.local/health

# Swagger UI (open in browser)
# http://localhost/swagger
# http://api.local/swagger
```

---

## Environment Configuration

Configuration is injected via Kubernetes ConfigMap and Secret — no hardcoded values in the image.

### ConfigMap (`k8s/1-configmap.yaml`)

```yaml
data:
  ASPNETCORE_ENVIRONMENT: Production
  DB_HOST: host.docker.internal
  DB_PORT: "1433"
  DB_NAME: MyApiDb
```

### Secret (`k8s/2-secret.yaml`)

```yaml
stringData:
  DB_USER: sa
  DB_PASSWORD: YourStr0ngPassword!
```

> ⚠️ `k8s/2-secret.yaml` is in `.gitignore`. Never commit secrets to Git.

### Connection String (assembled in deployment.yaml)

```
Server=host.docker.internal,1433;Database=MyApiDb;User Id=sa;Password=...;TrustServerCertificate=True
```

### Local Development (`appsettings.Development.json`)

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost,1433;Database=MyApiDb;User Id=sa;Password=YourStr0ngPassword!;TrustServerCertificate=True"
  }
}
```

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Health check — returns `{ "status": "healthy" }` |
| `GET` | `/swagger` | Swagger UI (interactive API docs) |
| `GET` | `/products` | Get all products |
| `POST` | `/products` | Create a product |
| `GET` | `/products/{id}` | Get product by ID |
| `PUT` | `/products/{id}` | Update product |
| `DELETE` | `/products/{id}` | Delete product |

### Sample Requests

```powershell
# Health check
curl http://localhost/health

# Get all products
curl http://localhost/products

# Create a product
curl -X POST http://localhost/products `
  -H "Content-Type: application/json" `
  -d '{"name": "Widget", "price": 9.99}'
```

---

## Kubernetes Resources

### Overview

```
Ingress (api.local)
    └── Service: LoadBalancer  (localhost:80)
    └── Service: NodePort      (localhost:30080)
    └── Service: ClusterIP     (internal only)
            └── Deployment (2 replicas)
                    └── Pod 1 → myapi:latest → host.docker.internal:1433 → SQL Server
                    └── Pod 2 → myapi:latest → host.docker.internal:1433 → SQL Server
```

### Resource Summary

| Resource | Name | Purpose |
|---|---|---|
| Deployment | `my-api` | Runs 2 pod replicas, rolling updates |
| Service | `myapi-service-api` | LoadBalancer — `localhost:80` |
| Service | `my-api-nodeport` | NodePort — `localhost:30080` |
| Service | `my-api-clusterip` | ClusterIP — internal pod-to-pod |
| Ingress | `my-ingress` | Routes `api.local` → service |
| ConfigMap | `app-config` | Non-sensitive environment config |
| Secret | `app-secret` | DB credentials |

### Useful kubectl Commands

```powershell
# See all resources
kubectl get all

# Watch pods
kubectl get pods -w

# View logs
kubectl logs deployment/my-api --tail=50
kubectl logs deployment/my-api -f           # stream live

# Shell into a pod
kubectl exec -it <pod-name> -- /bin/sh

# Check environment variables
kubectl exec <pod-name> -- printenv

# Port-forward for quick testing
kubectl port-forward deployment/my-api 8080:8080

# Scale up/down
kubectl scale deployment/my-api --replicas=4

# View events (useful for debugging)
kubectl get events --sort-by=.lastTimestamp
```

---

## Updating the API

Use the deploy script for consistent builds and deploys:

```powershell
# deploy.ps1 — from project root
param([string]$Tag = "latest")

$image = "myapi:$Tag"

Write-Host "Building $image..." -ForegroundColor Cyan
docker build -t $image ./MyApi

Write-Host "Updating deployment..." -ForegroundColor Cyan
kubectl set image deployment/my-api my-api=$image

Write-Host "Waiting for rollout..." -ForegroundColor Cyan
kubectl rollout status deployment/my-api

Write-Host "Done!" -ForegroundColor Green
```

```powershell
# Usage
.\deploy.ps1                  # builds myapi:latest
.\deploy.ps1 -Tag 1.2         # builds myapi:1.2

# Rollback if something breaks
kubectl rollout undo deployment/my-api

# View rollout history
kubectl rollout history deployment/my-api
```

---

## Troubleshooting

### Pods not starting

```powershell
# Check pod status
kubectl describe pod <pod-name>
# Look at Events section at the bottom

# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous   # if pod crashed
```

| Error | Cause | Fix |
|---|---|---|
| `ErrImageNeverPull` | Image not in Docker cache | `docker build -t myapi:latest ./MyApi` |
| `CrashLoopBackOff` | App crashing on start | Check logs — usually bad connection string |
| `Pending` | No resources on node | `kubectl describe pod` → check Events |
| `OOMKilled` | Exceeded memory limit | Increase `resources.limits.memory` |

### Cannot connect to SQL Server

```powershell
# 1. Is SQL Server running?
docker-compose ps

# 2. Is port 1433 reachable from cluster?
kubectl run -it --rm nettest --image=busybox --restart=Never `
  -- nc -zv host.docker.internal 1433
# Expected: open ✅

# 3. Check connection string in pod
kubectl exec <pod-name> -- printenv | findstr "Connection"
```

### Swagger not showing

```powershell
# Check environment — Swagger may be disabled in Production
kubectl exec deployment/my-api -- printenv ASPNETCORE_ENVIRONMENT

# Enable Swagger in Production (Program.cs):
# app.UseSwagger();
# app.UseSwaggerUI();
# (move outside the if IsDevelopment block)
```

### Ingress 404

```powershell
# Check hosts file has: 127.0.0.1 api.local
Get-Content C:\Windows\System32\drivers\etc\hosts | Select-String "api.local"

# Check NGINX ingress controller is running
kubectl get pods -n ingress-nginx

# Check ingress rules
kubectl describe ingress my-ingress
```

### HTTPS Redirect Issue

If running the container directly with `docker run` and getting connection refused:

```powershell
# Disable HTTPS redirect for containers
# In Program.cs:
if (app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();   # only in dev
}

# Or override via env var
docker run -e ASPNETCORE_ENVIRONMENT=Development ...
```

---

## SQL Server Management

```powershell
# Start SQL Server
docker-compose up -d

# Stop SQL Server (data is preserved in volume)
docker-compose down

# View SQL Server logs
docker-compose logs sqlserver -f

# Connect via sqlcmd
docker exec -it sqlserver /opt/mssql-tools/bin/sqlcmd `
  -S localhost -U sa -P "YourStr0ngPassword!"

# Backup database
docker exec sqlserver /opt/mssql-tools/bin/sqlcmd `
  -S localhost -U sa -P "YourStr0ngPassword!" `
  -Q "BACKUP DATABASE MyApiDb TO DISK='/var/opt/mssql/backup/MyApiDb.bak'"
```

> **Data persistence:** SQL Server data is stored in a Docker named volume (`sqlserver-data`). Data survives `docker-compose down` but is lost with `docker-compose down -v`. Never use `-v` unless you want to reset the database.

---

## .gitignore

```gitignore
# Build output
bin/
obj/

# Secrets — never commit
k8s/2-secret.yaml
.env
docker-compose.override.yml

# IDE
.vs/
*.user
.vscode/

# Docker
*.dockerignore
```

---

## Architecture

```
Windows 11 Machine
│
├── Docker Desktop
│   │
│   ├── sqlserver (container)
│   │   ├── SQL Server 2022
│   │   └── sqlserver-data (named volume) ← data persists here
│   │
│   └── Kubernetes Cluster (kubeadm, 1 node)
│       ├── NGINX Ingress Controller
│       │   └── Routes api.local → myapi-service
│       ├── my-api Pod 1 (myapi:latest)
│       │   └── connects to host.docker.internal:1433
│       └── my-api Pod 2 (myapi:latest)
│           └── connects to host.docker.internal:1433
│
└── Your Browser / curl
    ├── http://localhost/swagger        (LoadBalancer)
    ├── http://localhost:30080/health   (NodePort)
    └── http://api.local/health         (Ingress)
```
