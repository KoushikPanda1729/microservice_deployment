# Kubernetes Deployment Guide

Complete guide for deploying microservices on AWS EKS.

---

## Table of Contents

1. [Cluster Setup](#1-cluster-setup)
2. [Basic kubectl Commands](#2-basic-kubectl-commands)
3. [Debugging Commands](#3-debugging-commands)
4. [ConfigMap & Secrets](#4-configmap--secrets)
5. [Deployment](#5-deployment)
6. [Ingress Controller](#6-ingress-controller)
7. [Scaling Nodes](#7-scaling-nodes)
8. [ArgoCD Setup](#8-argocd-setup)
9. [Testing & Verification](#9-testing--verification)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Cluster Setup

### Create EKS Cluster
```bash
eksctl create cluster \
  --name microservices \
  --version 1.30 \
  --region us-east-1 \
  --nodegroup-name microservice-nodes \
  --node-type t3.micro \
  --nodes 2 \
  --timeout 60m
```

### Verify Cluster
```bash
kubectl get nodes
kubectl cluster-info
```

---

## 2. Basic kubectl Commands

### View Resources
```bash
# Get all resources
kubectl get all

# Get pods
kubectl get pods
kubectl get pods -o wide                    # Show node info
kubectl get pods -A                         # All namespaces
kubectl get pods -n <namespace>             # Specific namespace

# Get nodes
kubectl get nodes
kubectl get nodes -o wide

# Get services
kubectl get svc

# Get deployments
kubectl get deployment

# Get namespaces
kubectl get ns

# Get ingress
kubectl get ingress

# Get configmaps and secrets
kubectl get configmap
kubectl get secret
```

### Apply & Delete Resources
```bash
# Apply configuration
kubectl apply -f <file.yaml>
kubectl apply -f <directory>/              # Apply all files in directory

# Delete resources
kubectl delete -f <file.yaml>
kubectl delete deployment <name>
kubectl delete service <name>
kubectl delete pod <name>
kubectl delete pod <name> --force --grace-period=0   # Force delete
```

---

## 3. Debugging Commands

### Check Pod Status
```bash
# Describe pod (shows events and errors)
kubectl describe pod <pod-name>
kubectl describe pod <pod-name> | tail -20   # Last 20 lines (events)

# View logs
kubectl logs <pod-name>
kubectl logs -f <pod-name>                   # Follow/stream logs
kubectl logs <pod-name> --previous           # Previous container logs
kubectl logs -f -l app=auth-service          # All pods with label
```

### Check Node Status
```bash
kubectl describe nodes
kubectl describe nodes | grep -A 10 "Allocated resources"
kubectl describe nodes | grep -E "Capacity:|Allocatable:|pods:"
```

### Check All Pods Across Namespaces
```bash
kubectl get pods -A -o wide
kubectl get pods -A | grep -v Completed | wc -l   # Count running pods
```

---

## 4. ConfigMap & Secrets

### Create ConfigMap
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
data:
  PORT: "5501"
  NODE_ENV: "production"
  DB_HOST: "your-db-host"
  DB_PORT: "5432"
  DB_NAME: "postgres"
  DB_USERNAME: "postgres"
```

### Create Secret
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-service-secret
type: Opaque
stringData:
  DB_PASSWORD: "your-password"
  REFRESH_TOKEN_SECRET: "your-secret"
  PRIVATE_KEY: |
    -----BEGIN PRIVATE KEY-----
    ...
    -----END PRIVATE KEY-----
```

### Apply ConfigMap & Secret
```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
```

### View ConfigMap & Secret
```bash
kubectl get configmap auth-service-config -o yaml
kubectl get secret auth-service-secret -o yaml
kubectl get secret auth-service-secret -o jsonpath="{.data.DB_PASSWORD}" | base64 -d
```

---

## 5. Deployment

### Deployment with ConfigMap & Secret
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  labels:
    app: auth-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
        - name: auth-service
          image: koushik172/auth-service:build-25
          ports:
            - containerPort: 5501
          envFrom:
            - configMapRef:
                name: auth-service-config
            - secretRef:
                name: auth-service-secret

---

apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
  ports:
    - protocol: TCP
      port: 5501
      targetPort: 5501
```

### Deployment Commands
```bash
# Apply deployment
kubectl apply -f deployment.yaml

# Scale deployment
kubectl scale deployment auth-service --replicas=2

# Restart deployment (picks up new config)
kubectl rollout restart deployment auth-service

# Check rollout status
kubectl rollout status deployment auth-service

# Rollback deployment
kubectl rollout undo deployment auth-service
```

---

## 6. Ingress Controller

### Install NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/aws/deploy.yaml
```

### Verify Installation
```bash
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Ingress Configuration
```yaml
# auth-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-service-ingress
  annotations:
    kubernetes.io/ingress.class: nginx

    # CORS Configuration
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, PATCH, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"

    # Rewrite Target
    nginx.ingress.kubernetes.io/rewrite-target: /auth/$2
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api/auth(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: auth-service
                port:
                  number: 5501
```

### Get Load Balancer URL
```bash
kubectl get svc -n ingress-nginx
# Look for EXTERNAL-IP of ingress-nginx-controller
```

---

## 7. Scaling Nodes

### Scale Node Group
```bash
# Scale to desired number of nodes
eksctl scale nodegroup \
  --cluster=microservices \
  --name=microservice-nodes \
  --nodes=<desired> \
  --nodes-max=<max> \
  --nodes-min=1 \
  --region=us-east-1

# Examples:
eksctl scale nodegroup --cluster=microservices --name=microservice-nodes --nodes=3 --nodes-max=4 --region=us-east-1
eksctl scale nodegroup --cluster=microservices --name=microservice-nodes --nodes=5 --nodes-max=6 --region=us-east-1
eksctl scale nodegroup --cluster=microservices --name=microservice-nodes --nodes=6 --nodes-max=7 --region=us-east-1
```

### Check Nodegroup Status
```bash
eksctl get nodegroup --cluster microservices --region us-east-1 --name microservice-nodes
```

### Watch Nodes
```bash
kubectl get nodes -w
```

### Node Pod Limits (t3.micro = 4 pods per node)
| Nodes | Total Pod Slots |
|-------|-----------------|
| 2 | 8 |
| 3 | 12 |
| 5 | 20 |
| 6 | 24 |

---

## 8. ArgoCD Setup

### Create Namespace
```bash
kubectl create namespace argocd
```

### Install ArgoCD
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Check Pods
```bash
kubectl get pods -n argocd
```

### Get Admin Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Port Forward to Access UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Access ArgoCD
- **URL:** https://localhost:8080
- **Username:** admin
- **Password:** (from command above)

---

## 9. Testing & Verification

### Port Forward for Local Testing
```bash
kubectl port-forward service/auth-service 5501:5501
```

### Test with cURL
```bash
# Health check
curl http://localhost:5501/health

# Via Ingress (Load Balancer)
curl http://<LOAD-BALANCER-URL>/api/auth/health

# Login test
curl -X POST http://<LOAD-BALANCER-URL>/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"Admin@123456"}'
```

### Test Inside Cluster
```bash
kubectl run test --rm -it --image=curlimages/curl --restart=Never -- curl http://auth-service:5501/health
```

---

## 10. Troubleshooting

### Pod Stuck in Pending
```bash
# Check why
kubectl describe pod <pod-name>

# Common causes:
# 1. Not enough pod slots (node limit)
# 2. Not enough CPU/memory
# 3. Node selector/affinity issues

# Solutions:
# - Scale up nodes
# - Scale down other pods
kubectl scale deployment metrics-server -n kube-system --replicas=0
kubectl scale deployment coredns -n kube-system --replicas=1
```

### Pod in CrashLoopBackOff
```bash
# Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Common causes:
# 1. Missing environment variables
# 2. Database connection issues
# 3. Application errors
```

### Network Unreachable (IPv6 Issue)
```bash
# Use Supabase Session Pooler URL instead of direct connection
# Change DB_HOST from:
#   db.xxx.supabase.co (IPv6)
# To:
#   aws-0-us-east-1.pooler.supabase.com (IPv4)
```

### Clean Up Everything
```bash
# Delete all in namespace
kubectl delete all --all -n default

# Delete specific resources
kubectl delete -f deployment.yaml
kubectl delete configmap auth-service-config
kubectl delete secret auth-service-secret
kubectl delete ingress auth-service-ingress

# Delete entire cluster
eksctl delete cluster --name microservices --region us-east-1
```

---

## File Structure

```
deployment/
├── auth-service/
│   ├── configmap.yaml      # Non-sensitive configuration
│   ├── secret.yaml         # Sensitive data (passwords, keys)
│   ├── deployment.yaml     # Deployment + Service
│   └── auth-ingress.yaml   # Ingress configuration
└── README.md               # This file
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Get pods | `kubectl get pods` |
| Get logs | `kubectl logs <pod>` |
| Describe pod | `kubectl describe pod <pod>` |
| Apply config | `kubectl apply -f <file>` |
| Delete resource | `kubectl delete -f <file>` |
| Scale deployment | `kubectl scale deployment <name> --replicas=N` |
| Restart deployment | `kubectl rollout restart deployment <name>` |
| Port forward | `kubectl port-forward svc/<name> <local>:<remote>` |
| Scale nodes | `eksctl scale nodegroup --cluster=<cluster> --name=<ng> --nodes=N` |

---

## Author

Created during Kubernetes learning session.
