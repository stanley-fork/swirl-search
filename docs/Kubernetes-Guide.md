---
layout: default
title: Kubernetes Deployment
nav_order: 23
---

<details markdown="block">
  <summary>
    Table of Contents
  </summary>
  {: .text-delta }
- TOC
{:toc}
</details>

<span class="big-text">Kubernetes Deployment Guide</span><br/><span class="med-text">Production Scalability with Kubernetes</span>

---

# Overview

SWIRL can be deployed on Kubernetes for production scalability. The platform is containerized and ready for cloud-native deployments across on-premises and cloud environments (AWS, Azure, GCP). The `swirl-infra/` directory in the repository contains example manifests for rapid deployment.

Kubernetes deployments provide:
- **Auto-scaling** based on CPU and memory usage
- **High availability** with multiple pod replicas
- **Service discovery** for inter-pod communication
- **Rolling updates** with zero downtime
- **Persistent volumes** for databases and uploads
- **Health monitoring** with liveness and readiness probes

# Prerequisites

Before deploying SWIRL on Kubernetes, ensure you have:

- **Kubernetes cluster** version 1.27 or higher
- **kubectl** configured to access your cluster
- **PostgreSQL database** (managed service or in-cluster)
- **Redis instance** (managed service or in-cluster)
- **Container registry access** to SWIRL Enterprise images
- **TLS certificates** (for production ingress via cert-manager and Let's Encrypt)
- **Sufficient cluster resources**: Minimum 16 CPU and 32 GB RAM
- **Persistent volumes** provisioner available in cluster
- **Ingress controller** installed (nginx-ingress or cloud provider default)

{: .warning }
**SWIRL Enterprise License Required**  
The Kubernetes deployment uses SWIRL Enterprise container images. You must have a valid SWIRL license and container registry credentials.

# Architecture

The SWIRL Kubernetes architecture consists of the following pods and services:

## SWIRL Application Pod

The primary Django application container with integrated Celery workers:
- **Runs Django web server** on port 8000
- **Includes 5 Celery worker processes** for async task execution
- **Uses PostgreSQL** for data persistence
- **Uses Redis** for message broker and caching
- **Resource allocation:** 8 CPU cores, 10 GB memory (production minimum)

## Redis Pod

Message broker and caching layer:
- **Port:** 6379
- **Use:** Celery task queue, session storage, caching
- **Resource allocation:** 2 CPU cores, 4 GB memory

## Tika Pod

Document text extraction service:
- **Port:** 9998
- **Image:** `apache/tika:latest`
- **Use:** OCR and text extraction from PDF, Office documents, and images
- **Resource allocation:** 4 CPU cores, 8 GB memory

## Topic Text Matcher Pod

Passage detection and semantic matching for RAG:
- **Port:** 7029
- **Image:** `swirlai/swirl-integrations:topic-text-matcher`
- **Use:** Identifies relevant passages in document collections for RAG workflows
- **Resource allocation:** 8 CPU cores, 10 GB memory

## PostgreSQL

Relational database for SWIRL metadata:
- **Deployment:** External managed service (recommended for production) or in-cluster StatefulSet
- **Port:** 5432
- **Databases required:** Create `swirl` database before deployment
- **Backups:** Essential for production workloads

# Configuration

SWIRL Kubernetes deployments use a `ConfigMap` to manage environment variables. Key configuration settings:

## Host and Protocol

```yaml
ALLOWED_HOSTS: "*.example.com,swirl.example.com"
PROTOCOL: "https"  # Always use https in production
DEBUG: "False"
SECRET_KEY: "<generate-strong-secret>"
```

## Database Configuration

```yaml
SQL_ENGINE: "django.db.backends.postgresql"
SQL_DATABASE: "swirl"
SQL_USER: "swirl"
SQL_PASSWORD: "<secure-password>"  # Store in Secret, not ConfigMap
SQL_HOST: "postgres.default.svc.cluster.local"
SQL_PORT: "5432"
```

## Celery and Messaging

```yaml
CELERY_BROKER_URL: "redis://redis:6379/0"
CELERY_RESULT_BACKEND: "redis://redis:6379/1"
```

## External Services

```yaml
TIKA_SERVER_ENDPOINT: "http://tika:9998"
SWIRL_TEXT_SUMMARIZATION_URL: "http://topic-text-matcher:7029"
```

## SWIRL Features

```yaml
SWIRL_LICENSE: "<your-enterprise-license>"
SWIRL_EXPLAIN: "True"
```

Store sensitive data (passwords, tokens) in Kubernetes Secrets:

```bash
kubectl create secret generic swirl-secrets \
  --from-literal=sql_password=<password> \
  --from-literal=secret_key=<generated-key> \
  -n swirl
```

# Deployment Steps

## 1. Create Namespace

```bash
kubectl create namespace swirl
```

## 2. Store Secrets

```bash
kubectl create secret generic swirl-secrets \
  --from-literal=sql_password=<your-password> \
  --from-literal=secret_key=<your-secret-key> \
  -n swirl

kubectl create secret docker-registry swirl-registry \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  -n swirl
```

## 3. Apply Manifests

Deploy Redis, PostgreSQL (if in-cluster), Tika, and Topic Text Matcher services:

```bash
kubectl apply -f redis-deployment.yaml -n swirl
kubectl apply -f tika-deployment.yaml -n swirl
kubectl apply -f topic-text-matcher-deployment.yaml -n swirl
```

If using an in-cluster PostgreSQL:

```bash
kubectl apply -f postgres-deployment.yaml -n swirl
```

## 4. Apply SWIRL Deployment

```bash
kubectl apply -f swirl-deployment.yaml -n swirl
```

## 5. Verify Pods Are Running

```bash
kubectl get pods -n swirl
kubectl logs -f deployment/swirl -n swirl
```

## 6. Run Database Migrations

SWIRL includes an init container that automatically runs migrations on startup:

```bash
kubectl logs -f deployment/swirl -n swirl | grep -i "migration\|completed"
```

Or manually run migrations:

```bash
kubectl exec -it deployment/swirl -n swirl -- python manage.py migrate
```

## 7. Verify Health

```bash
kubectl logs -f deployment/swirl -n swirl | grep -i "health\|ready"
```

# Ingress and TLS

## Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

## Create ClusterIssuer for Let's Encrypt

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

Apply the issuer:

```bash
kubectl apply -f cluster-issuer.yaml
```

## Create Ingress with TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: swirl-ingress
  namespace: swirl
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - swirl.example.com
    secretName: swirl-tls
  rules:
  - host: swirl.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: swirl
            port:
              number: 8000
```

Apply the ingress:

```bash
kubectl apply -f swirl-ingress.yaml -n swirl
```

Verify TLS certificate:

```bash
kubectl get certificate -n swirl
kubectl describe certificate swirl-tls -n swirl
```

# Scaling

## Horizontal Pod Autoscaling

Create an HorizontalPodAutoscaler to scale SWIRL replicas based on resource usage:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: swirl-hpa
  namespace: swirl
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: swirl
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

Apply the HPA:

```bash
kubectl apply -f swirl-hpa.yaml -n swirl
```

Monitor HPA status:

```bash
kubectl get hpa -n swirl -w
```

## Resource Limits and Requests

Each SWIRL pod requires:

```yaml
resources:
  requests:
    cpu: "4"
    memory: "6Gi"
  limits:
    cpu: "8"
    memory: "10Gi"
```

Tika pod requires:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

Topic Text Matcher pod requires:

```yaml
resources:
  requests:
    cpu: "4"
    memory: "6Gi"
  limits:
    cpu: "8"
    memory: "10Gi"
```

## Node Autoscaling

Enable Cluster Autoscaler on your cloud provider:

**AWS (EKS):**
```bash
# Enable autoscaling on your node group
aws eks update-nodegroup-config \
  --cluster-name <cluster-name> \
  --nodegroup-name <nodegroup-name> \
  --scaling-config minSize=3,maxSize=20,desiredSize=5
```

**Azure (AKS):**
```bash
# Enable autoscaling on node pool
az aks nodepool update \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20
```

**GCP (GKE):**
```bash
# Update node pool with autoscaling
gcloud container node-pools update <nodepool-name> \
  --cluster <cluster-name> \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 20
```

# Health Checks

SWIRL includes health check endpoints for Kubernetes probes.

## Readiness Probe

```yaml
readinessProbe:
  httpGet:
    path: /health/celery/
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

## Liveness Probe

```yaml
livenessProbe:
  httpGet:
    path: /health/celery/
    port: 8000
  initialDelaySeconds: 60
  periodSeconds: 30
  timeoutSeconds: 5
  failureThreshold: 3
```

Check probe status:

```bash
kubectl describe pod <pod-name> -n swirl | grep -A 10 "Readiness\|Liveness"
```

# Supporting Services

## Apache Tika

Tika is required for document text extraction and OCR.

**Deployment manifest snippet:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tika
  namespace: swirl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tika
  template:
    metadata:
      labels:
        app: tika
    spec:
      containers:
      - name: tika
        image: apache/tika:latest
        ports:
        - containerPort: 9998
        resources:
          requests:
            cpu: "2"
            memory: "4Gi"
          limits:
            cpu: "4"
            memory: "8Gi"
        livenessProbe:
          httpGet:
            path: /tika
            port: 9998
          initialDelaySeconds: 30
          periodSeconds: 30
```

## Topic Text Matcher

Topic Text Matcher provides semantic passage detection for RAG workflows.

**Deployment manifest snippet:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topic-text-matcher
  namespace: swirl
spec:
  replicas: 1
  selector:
    matchLabels:
      app: topic-text-matcher
  template:
    metadata:
      labels:
        app: topic-text-matcher
    spec:
      containers:
      - name: topic-text-matcher
        image: swirlai/swirl-integrations:topic-text-matcher
        ports:
        - containerPort: 7029
        resources:
          requests:
            cpu: "4"
            memory: "6Gi"
          limits:
            cpu: "8"
            memory: "10Gi"
        env:
        - name: PORT
          value: "7029"
        livenessProbe:
          httpGet:
            path: /health
            port: 7029
          initialDelaySeconds: 60
          periodSeconds: 30
```

# Azure-Specific Notes

## AKS Node Pool Configuration

For Azure Kubernetes Service (AKS), create appropriately sized node pools:

```bash
# Create a node pool for SWIRL workloads
az aks nodepool add \
  --cluster-name <cluster-name> \
  --name swirl-pool \
  --vm-set-type VirtualMachineScaleSets \
  --node-vm-size Standard_D8s_v3 \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20 \
  --resource-group <resource-group>
```

## Azure File Share for Uploads

Configure Azure File Share for persistent uploads storage:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: swirl-uploads-pv
  namespace: swirl
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: azure-storage-secret
    shareName: swirl-uploads
    readOnly: false
  storageClassName: azurefile

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: swirl-uploads-pvc
  namespace: swirl
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 500Gi
```

Create the storage secret:

```bash
kubectl create secret generic azure-storage-secret \
  --from-literal=azurestorageaccountname=<account-name> \
  --from-literal=azurestorageaccountkey=<account-key> \
  -n swirl
```

Mount the share in the SWIRL deployment:

```yaml
volumes:
- name: uploads
  azureFile:
    secretName: azure-storage-secret
    shareName: swirl-uploads
    readOnly: false
```

# Monitoring and Logging

## View Logs

```bash
# View SWIRL application logs
kubectl logs -f deployment/swirl -n swirl

# View Redis logs
kubectl logs -f deployment/redis -n swirl

# View Tika logs
kubectl logs -f deployment/tika -n swirl

# View Topic Text Matcher logs
kubectl logs -f deployment/topic-text-matcher -n swirl
```

## Check Pod Status

```bash
# List all SWIRL pods
kubectl get pods -n swirl -o wide

# Describe a specific pod for events and status
kubectl describe pod <pod-name> -n swirl

# Watch pod status changes
kubectl get pods -n swirl -w
```

## Access SWIRL

Once the ingress is active with TLS certificates issued:

```bash
# Check ingress status
kubectl get ingress -n swirl

# Get the external IP or hostname
kubectl get ingress swirl-ingress -n swirl -o wide
```

Visit: `https://swirl.example.com`

# Troubleshooting

## Pod Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n swirl

# Check logs
kubectl logs <pod-name> -n swirl

# Check resource availability
kubectl describe nodes
```

## Database Connection Issues

```bash
# Verify PostgreSQL is accessible
kubectl run -it --rm debug --image=postgres:15 --restart=Never -- \
  psql -h postgres.default.svc.cluster.local -U swirl -d swirl

# Test from within SWIRL pod
kubectl exec -it deployment/swirl -n swirl -- \
  python manage.py dbshell
```

## Redis Connection Issues

```bash
# Test Redis connectivity
kubectl run -it --rm debug --image=redis:latest --restart=Never -- \
  redis-cli -h redis:6379 ping

# Check Redis info
kubectl exec -it deployment/redis -n swirl -- \
  redis-cli info
```

## Celery Workers Not Processing Tasks

```bash
# Check Celery worker status
kubectl exec -it deployment/swirl -n swirl -- \
  celery -A swirl_server inspect active

# View Celery logs
kubectl logs -f deployment/swirl -n swirl | grep -i celery
```

---

For additional support, refer to the [SWIRL documentation](https://swirl.today) or contact your SWIRL Enterprise support team.
