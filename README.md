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
│   ├── 2-secrets.yaml               # DB credentials (do not commit!)
│   ├── 3-deployment.yaml           # Pod spec, replicas, probes
│   ├── 4-Service.yaml              # LoadBalancer + NodePort + ClusterIP
│   └── 5-ingress.yaml              # NGINX ingress rules (api.local)
│   └── 6-nodeport.yaml             # testing kubernetes node
│   └── 5-slusterip.yaml             
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
kubectl apply -f k8s/ 
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
 
 
