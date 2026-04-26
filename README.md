# Task Manager DevOps Project

Full-stack task manager with React frontend, FastAPI backend, and Supabase as external database/auth service.

This repository is configured for local DevOps workflow using Docker, Jenkins, and Minikube (no cloud provider required).

## Stack

- Frontend: React + Vite + Nginx
- Backend: FastAPI + Uvicorn
- External service: Supabase
- Containers: Docker
- CI/CD: Jenkins
- Orchestration: Kubernetes on Minikube

## Project Structure

```text
mini_project/
|-- backend/
|-- frontend/
|-- k8s/
|   |-- namespace.yaml
|   |-- configmap.yaml
|   |-- backend.yaml
|   `-- frontend.yaml
|-- Jenkinsfile
|-- docker-compose.yml
|-- .env.example
`-- README.md
```

## Prerequisites

- Docker
- Minikube
- kubectl
- Jenkins (with a Linux-capable agent that has Docker, kubectl, and Minikube access)
- Supabase project URL and service role key

## Local Environment Variables

1. Copy sample file:

```bash
cp .env.example .env
```

2. Update at minimum:

- SUPABASE_URL
- SUPABASE_SERVICE_ROLE_KEY
- JWT_SECRET

## Dockerize Services

Both apps are already dockerized:

- Backend image build context: backend/
- Frontend image build context: frontend/

Quick local container test:

```bash
docker compose up --build -d
```

App URL: http://localhost:5173

Stop:

```bash
docker compose down
```

## Minimal Minikube Deploy

Use these files under k8s/:

- namespace.yaml: namespace mini-project
- configmap.yaml: only CORS_ORIGINS
- backend.yaml: backend Deployment + ClusterIP Service
- frontend.yaml: frontend Deployment + NodePort Service (30080)

Frontend proxies /api to backend service inside the cluster.

Manual deploy:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl -n mini-project create secret generic task-manager-secrets \
  --from-literal=SUPABASE_URL="<your-url>" \
  --from-literal=SUPABASE_SERVICE_ROLE_KEY="<your-key>" \
  --from-literal=JWT_SECRET="<your-jwt-secret>" \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
```

Access app:

```bash
minikube service frontend -n mini-project --url
```

## Jenkins CI/CD Pipeline

Pipeline file: Jenkinsfile

### What it does

1. Checkout source
2. Build backend and frontend Docker images
3. Load images into Minikube image cache
4. Create/update Kubernetes secret and apply manifests
5. Update Deployment images to current build tag
6. Wait for rollout and print cluster status

### Jenkins credentials required

Create these Jenkins credentials as Secret text:

- supabase-url
- supabase-service-role-key
- jwt-secret

### Create Jenkins job

1. New Item -> Pipeline
2. Set Pipeline script from SCM
3. SCM: Git, point to this repository
4. Script Path: Jenkinsfile
5. Build Now

After success, open the frontend using:

```bash
minikube service frontend -n mini-project --url
```

## Useful Commands

```bash
# Check workload
kubectl -n mini-project get all

# Logs
kubectl -n mini-project logs deployment/backend
kubectl -n mini-project logs deployment/frontend

# Restart deployment
kubectl -n mini-project rollout restart deployment/backend
kubectl -n mini-project rollout restart deployment/frontend

# Delete deployment
kubectl delete namespace mini-project
```
