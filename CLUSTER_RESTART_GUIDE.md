# How to Start Your EKS Cluster Again

## Before You Start (Check These First)

- [ ] Supabase project is active (not paused) → https://supabase.com/dashboard
- [ ] MongoDB Atlas cluster is running → https://cloud.mongodb.com
- [ ] Confluent Kafka cluster is active → https://confluent.cloud
- [ ] Resend API key works → https://resend.com/api-keys
- [ ] Domain `koushikpanda.online` hasn't expired → AWS Route53 console
- [ ] ACM certificate is still valid → AWS Certificate Manager console
- [ ] Your local secret files still exist (check below)

### Your secret files (ONLY on your Mac, NOT in GitHub):
```
~/Desktop/full stack/deployment/auth-service/secret.yaml
~/Desktop/full stack/deployment/catalog-service/secret.yaml
~/Desktop/full stack/deployment/billing-service/secret.yaml
~/Desktop/full stack/deployment/notification-service/secret.yaml
~/Desktop/full stack/deployment/websocket-service/secret.yaml
```
If any are missing, you need to recreate the credentials.

---

## Step 1: Create EKS Cluster (15-20 min wait)

```bash
eksctl create cluster \
  --name microservices \
  --region us-east-1 \
  --nodegroup-name microservice-nodes \
  --node-type t3.micro \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed
```

Wait for it to finish. Then verify:
```bash
kubectl get nodes
```
You should see 2 nodes in `Ready` state.

---

## Step 2: Install NGINX Ingress (USE YOUR LOCAL deploy.yaml!)

```bash
cd ~/Desktop/full\ stack/deployment
kubectl apply -f deploy.yaml
```

**DO NOT use the remote generic manifest.** Your `deploy.yaml` has your ACM SSL certificate:
`arn:aws:acm:us-east-1:660235343221:certificate/1484a7f8-d46b-40f3-8ae7-67b999b8a79e`

Wait for it to be ready:
```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s
```

---

## Step 3: Get New Load Balancer URL (SAVE THIS!)

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Copy this URL. You need it for DNS in Step 7.

---

## Step 4: Free Up Pod Slots

t3.micro can only run 4 pods per node. Free up space:
```bash
kubectl scale deployment metrics-server -n kube-system --replicas=1
```

---

## Step 5: Deploy All Services (from LOCAL folder!)

```bash
cd ~/Desktop/full\ stack/deployment

kubectl apply -f auth-service/
kubectl apply -f catalog-service/
kubectl apply -f billing-service/
kubectl apply -f notification-service/
kubectl apply -f websocket-service/
```

**Deploy from LOCAL directory, NOT from GitHub clone.** GitHub doesn't have your secret.yaml files.

---

## Step 6: Verify Everything is Running

```bash
kubectl get pods
```

Wait until all pods show `Running` (2-3 min).

**If pods are stuck in Pending:**
```bash
# Option 1: Add more nodes
eksctl scale nodegroup --cluster=microservices --nodes=3 --name=microservice-nodes --region us-east-1

# Option 2: Reduce replicas to 1 each
kubectl scale deployment auth-service --replicas=1
kubectl scale deployment catalog-service --replicas=1
kubectl scale deployment billing-service --replicas=1
kubectl scale deployment notification-service --replicas=1
kubectl scale deployment websocket-service --replicas=1
```

Check ingresses:
```bash
kubectl get ingress
```
You should see: auth-service-ingress, tenant-ingress, user-ingress, catalog-service-ingress, billing-service-ingress, notification-service-ingress, websocket-service-ingress

---

## Step 7: Update DNS in Route53

1. Go to **AWS Console → Route53 → Hosted Zones → koushikpanda.online**
2. Find the `api` A record
3. Update it to point to the **new Load Balancer URL** from Step 3
4. If it's an A record with Alias, select the new NLB from dropdown
5. Save changes

---

## Step 8: Test

Wait 2-3 minutes for DNS, then:
```bash
curl https://api.koushikpanda.online/api/auth/auth/self
```

---

## Troubleshooting

### Pods stuck in Pending
t3.micro = max 4 pods per node. You have system pods eating slots.
```bash
kubectl describe pod <pod-name>
# Then scale up nodes or reduce replicas (see Step 6)
```

### Pods in CrashLoopBackOff
```bash
kubectl logs <pod-name>
```
Common reasons:
- **Supabase paused** → Go to Supabase dashboard and resume the project
- **Kafka expired** → Check Confluent Cloud, may need new credentials
- **MongoDB down** → Check Atlas cluster at https://cloud.mongodb.com
- **Secret missing** → You deployed from GitHub instead of local folder

### HTTPS not working
- Did you use YOUR `deploy.yaml`? (not the remote one)
- Is ACM certificate still valid? Check AWS Certificate Manager
- Is NLB listening on port 443?

### Can't connect to API
```bash
nslookup api.koushikpanda.online
kubectl get ingress
kubectl get pods -n ingress-nginx
```

---

## Your Config Reference

| Service | Port | Image |
|---------|------|-------|
| auth-service | 5501 | koushik172/auth-service:build-25 |
| catalog-service | 5502 | koushik172/catalog-service:build-6 |
| billing-service | 5503 | koushik172/billing-service:build-8 |
| notification-service | 5504 | koushik172/notification-service:build-8 |
| websocket-service | 5505 | koushik172/web-socket-service:build-3 |

## Secrets Summary

| Service | What's in secret.yaml |
|---------|----------------------|
| auth-service | DB_PASSWORD, REFRESH_TOKEN_SECRET, ADMIN_PASSWORD, RSA PRIVATE_KEY |
| catalog-service | MONGO_URI, AWS S3 keys, Kafka credentials |
| billing-service | MONGO_URI, Stripe keys, Kafka credentials |
| notification-service | MONGO_URI, Resend SMTP password, Kafka credentials |
| websocket-service | MONGO_URI, Kafka credentials |

## Domains

| Domain | Purpose |
|--------|---------|
| api.koushikpanda.online | API Gateway (points to NLB) |
| admin.koushikpanda.online | Admin Dashboard |
| pizza.koushikpanda.online | Client App |

## GitHub Repos

- Deployment: https://github.com/KoushikPanda1729/microservice_deployment
- Auth: https://github.com/KoushikPanda1729/auth-service
- Catalog: https://github.com/KoushikPanda1729/catalog-service
- Billing: https://github.com/KoushikPanda1729/billing-service
- Notification: https://github.com/KoushikPanda1729/notification-service
- Websocket: https://github.com/KoushikPanda1729/websocket-service

## Quick Commands

```bash
kubectl get nodes                    # Check cluster
kubectl get pods                     # Check all pods
kubectl get svc                      # Check services
kubectl get ingress                  # Check ingresses
kubectl logs <pod-name>              # Check pod logs
kubectl rollout restart deployment/<name>  # Restart a service
```
