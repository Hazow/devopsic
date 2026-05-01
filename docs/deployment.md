# Deployment Guide — Secure Event Ticketing Platform

## Prerequisites

- `kubectl` configured and pointing to your cluster (`kubectl get nodes` works)
- Access to `ghcr.io/hazow` container registry (images are public)
- Cluster has an nginx Ingress controller installed

## Step 1 — Create the Secret with real credentials

Never apply `k8s/secret.yaml` directly — it contains a placeholder. Create the Secret manually:

```bash
kubectl create secret generic ticketing-secret \
  --from-literal=POSTGRES_USER=ticketing_user \
  --from-literal=POSTGRES_PASSWORD=<your-strong-password> \
  --from-literal=POSTGRES_DB=ticketing
```

Verify it was created:
```bash
kubectl get secret ticketing-secret
```

## Step 2 — Apply all manifests

```bash
kubectl apply -f k8s/
```

This creates in order:
- ConfigMap (non-sensitive config)
- ServiceAccount (minimal RBAC)
- PersistentVolumeClaim (postgres storage)
- Deployments for postgres, redis, api, frontend, worker
- Services for each deployment
- Ingress (external access)
- NetworkPolicies (traffic restrictions)

## Step 3 — Verify all pods are Running

```bash
kubectl get pods
```

Expected output (all pods Running):
```
NAME                        READY   STATUS    RESTARTS   AGE
api-xxx                     1/1     Running   0          1m
frontend-xxx                1/1     Running   0          1m
worker-xxx                  1/1     Running   0          1m
postgres-xxx                1/1     Running   0          1m
redis-xxx                   1/1     Running   0          1m
```

## Step 4 — Configure Ingress hostname

Edit `k8s/ingress.yaml` and replace `ticketing.example.com` with your actual domain, then re-apply:

```bash
kubectl apply -f k8s/ingress.yaml
```

## Step 5 — Verify the application

```bash
# Health check
curl http://<your-domain>/api/healthz
# Expected: {"status":"ok","service":"api"}

# List events
curl http://<your-domain>/api/events

# Purchase a ticket
curl -X POST http://<your-domain>/api/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"test@example.com","quantity":1}'
```

---

## Rolling update (deploy a new image)

The CI pipeline handles this automatically on push to `main`. To do it manually:

```bash
# Replace IMAGE_TAG with the git SHA from CI (e.g. git-abc1234)
kubectl set image deployment/api api=ghcr.io/hazow/devopsic-api:IMAGE_TAG
kubectl set image deployment/frontend frontend=ghcr.io/hazow/devopsic-frontend:IMAGE_TAG
kubectl set image deployment/worker worker=ghcr.io/hazow/devopsic-worker:IMAGE_TAG

# Wait for rollout to complete
kubectl rollout status deployment/api
kubectl rollout status deployment/frontend
kubectl rollout status deployment/worker
```

## Rollback

```bash
# Roll back all three services to the previous version
kubectl rollout undo deployment/api
kubectl rollout undo deployment/frontend
kubectl rollout undo deployment/worker

# Verify rollback succeeded
kubectl rollout status deployment/api
kubectl describe deployment/api | grep Image
```

## Tear down

```bash
# Remove all resources but keep the PVC (data preserved)
kubectl delete -f k8s/

# Remove everything including database data
kubectl delete -f k8s/
kubectl delete pvc postgres-data
```
