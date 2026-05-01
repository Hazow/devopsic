# Runbook — Secure Event Ticketing Platform

This document describes detection, diagnosis, and remediation steps for the three most likely production incidents.

---

## Incident 1 — Database crash (PostgreSQL pod fails)

### Symptoms
- `GET /tickets/orders` returns `500 Internal Server Error`
- `GET /readyz` returns `{"status":"not-ready","error":"..."}`
- Monitoring alert: postgres pod not Running
- Users report they cannot see their ticket orders

### Diagnosis

```bash
# 1. Check pod status
kubectl get pods
# Expected symptom: postgres pod shows CrashLoopBackOff, Error, or Pending

# 2. Read postgres logs
kubectl logs deployment/postgres --previous
# Look for: data directory corruption, out-of-disk, OOM kill

# 3. Check events for the pod
kubectl describe pod -l app=postgres
# Look for: OOMKilled, FailedMount, insufficient storage

# 4. Check PersistentVolumeClaim
kubectl get pvc
kubectl describe pvc postgres-data
# Look for: Pending, Lost, or no available storage
```

### Remediation

**Case A — Pod crashed but data is intact (PVC still Bound):**
```bash
# Restart the postgres deployment
kubectl rollout restart deployment/postgres

# Wait for it to become ready
kubectl rollout status deployment/postgres

# Verify the API recovers
kubectl logs -f deployment/api
curl http://<ingress>/readyz
```

**Case B — Data corruption (pod starts but queries fail):**
```bash
# 1. Scale down api and worker to stop writes
kubectl scale deployment/api --replicas=0
kubectl scale deployment/worker --replicas=0

# 2. Restore from the most recent backup
kubectl exec -it deployment/postgres -- \
  psql -U ticketing_user -d ticketing -c "\dt"

# If backup exists (restore via pg_restore):
kubectl cp backup.dump postgres-pod:/tmp/backup.dump
kubectl exec -it deployment/postgres -- \
  pg_restore -U ticketing_user -d ticketing /tmp/backup.dump

# 3. Scale api and worker back up
kubectl scale deployment/api --replicas=1
kubectl scale deployment/worker --replicas=1
```

**Case C — PVC Lost (data unrecoverable):**
```bash
# 1. Delete the broken PVC and recreate
kubectl delete pvc postgres-data

# 2. Re-apply the postgres manifest (creates a fresh PVC)
kubectl apply -f k8s/

# 3. The database schema will be recreated from init.sql on first start
# Data from before the incident is lost — inform stakeholders
```

### Prevention
- Configure automated PostgreSQL backups (e.g., pg_dump CronJob to object storage)
- Set resource limits on the postgres pod to prevent OOM kills
- Monitor PVC usage and alert before disk is full
- Test restore procedure quarterly

---

## Incident 2 — Bad image tag deployed (application broken after deploy)

### Symptoms
- `GET /healthz` returns `502 Bad Gateway` or times out
- Pods show `CrashLoopBackOff` or `ImagePullBackOff` immediately after deploy
- New orders cannot be placed; existing orders are not processed
- GitHub Actions deploy job succeeded but app is broken

### Diagnosis

```bash
# 1. Check pod status — look for CrashLoopBackOff or ImagePullBackOff
kubectl get pods

# 2. Identify the bad image tag
kubectl describe deployment/api | grep Image
# Example output: Image: ghcr.io/hazow/devopsic-api:git-badf00d

# 3. Read crash logs
kubectl logs deployment/api --previous
# Look for: MODULE_NOT_FOUND, SyntaxError, missing env variable

# 4. Check image pull errors
kubectl describe pod -l app=api
# Look for: ErrImagePull, ImagePullBackOff, manifest not found
```

### Remediation

```bash
# 1. Immediate rollback to the last known good deployment
kubectl rollout undo deployment/api
kubectl rollout undo deployment/frontend
kubectl rollout undo deployment/worker

# 2. Wait for rollback to complete
kubectl rollout status deployment/api
kubectl rollout status deployment/frontend
kubectl rollout status deployment/worker

# 3. Verify the service is healthy again
curl http://<ingress>/healthz
# Expected: {"status":"ok","service":"api"}

curl http://<ingress>/readyz
# Expected: {"status":"ready"}

# 4. Check which image is now running (confirm it is the previous good tag)
kubectl describe deployment/api | grep Image

# 5. Investigate the bad commit before re-deploying
git log --oneline -5
```

### Prevention
- The CI pipeline runs `node --check` and `npm audit` before building — catches syntax errors and critical dependency vulnerabilities
- Trivy scan blocks push of images with CRITICAL vulnerabilities
- Liveness and readiness probes on all pods ensure Kubernetes only routes traffic to healthy instances
- Always deploy to a staging environment before production
- Tag every image with the git SHA so broken deploys are immediately traceable to the exact commit

---

## Incident 3 — Wrong secret in production (API cannot connect to database)

### Symptoms
- `GET /readyz` returns `{"status":"not-ready","error":"password authentication failed for user \"ticketing_user\""}`
- `POST /tickets/purchase` returns `500` — orders queue but are never processed
- Worker logs show repeated `password authentication failed` errors
- Incident typically occurs right after a secret rotation or a fresh cluster setup

### Diagnosis

```bash
# 1. Check readiness probe failures
kubectl describe deployment/api
# Look for: Readiness probe failed, Liveness probe failed

# 2. Read api logs
kubectl logs deployment/api
# Look for: password authentication failed, ECONNREFUSED

# 3. Read worker logs
kubectl logs deployment/worker
# Look for: password authentication failed for user

# 4. Verify what is currently in the Secret (base64-decoded)
kubectl get secret ticketing-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
echo ""
# Compare this value with what postgres actually expects

# 5. Check what password postgres was initialized with
kubectl exec -it deployment/postgres -- \
  psql -U ticketing_user -c "SELECT 1;"
# If this also fails, the postgres pod itself has a different password
```

### Remediation

```bash
# 1. Patch the Secret with the correct password
kubectl create secret generic ticketing-secret \
  --from-literal=POSTGRES_USER=ticketing_user \
  --from-literal=POSTGRES_PASSWORD=<correct-password> \
  --from-literal=POSTGRES_DB=ticketing \
  --dry-run=client -o yaml | kubectl apply -f -

# 2. Restart api and worker to pick up the new Secret values
kubectl rollout restart deployment/api
kubectl rollout restart deployment/worker

# 3. Wait for rollout
kubectl rollout status deployment/api
kubectl rollout status deployment/worker

# 4. Verify recovery
curl http://<ingress>/readyz
# Expected: {"status":"ready"}

kubectl logs deployment/worker
# Should show: Worker started and waiting for jobs...

# 5. If postgres itself was also re-initialized with a different password,
#    update the postgres Secret and restart postgres too
kubectl rollout restart deployment/postgres
```

### Prevention
- Never store real credentials in k8s/secret.yaml committed to git — always use `REPLACE_BEFORE_APPLY` as a placeholder
- Use a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager) for automated secret rotation
- After any secret rotation, immediately run `kubectl rollout restart` on all affected deployments
- The readiness probe on `/readyz` catches this within seconds — configure alerting on readiness probe failures
- Document the correct credentials in a secure password manager accessible to the on-call team, not in git

---

## Quick reference — useful kubectl commands

```bash
# Overview of all pods
kubectl get pods

# Follow logs of a service
kubectl logs -f deployment/api
kubectl logs -f deployment/worker

# Rollback a deployment
kubectl rollout undo deployment/api

# Check rollout history
kubectl rollout history deployment/api

# Force restart (picks up new Secrets or ConfigMaps)
kubectl rollout restart deployment/api

# Check what image is running
kubectl describe deployment/api | grep Image

# Check events (useful for diagnosing ImagePullBackOff, OOMKilled)
kubectl describe pod -l app=api

# Open a shell in a running pod
kubectl exec -it deployment/api -- sh

# Check resource usage
kubectl top pods
```
