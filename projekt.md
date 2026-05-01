# CLAUDE.md — Secure Event Ticketing Platform

## What is this project

University project for the course Uvod u DevOps - DevSecOps at Sveuciliste Algebra Bernays, Zagreb 2026.
The goal is to build and deliver a secure multi-tier application through the full DevOps/DevSecOps cycle.
That means: local containerized development first, then production deployment on Kubernetes/OpenShift.

App GitHub repo: https://github.com/matej-basic/devops-project-app


## Application services

The app is called Secure Event Ticketing Platform and has five services.

frontend - web UI, runs on port 3000
api - REST API, runs on port 8080
worker - background job processor, consumes jobs from Redis queue
postgres - PostgreSQL database, needs a persistent volume
redis - used as a queue and cache between api and worker

API endpoints:
GET /healthz returns {"status":"ok","service":"api"}
GET /tickets/orders returns list of all orders
POST /tickets/orders creates a new order, queues it via Redis, worker processes it and saves to postgres


## Environment variables

Never hardcode secrets. Use .env locally and Kubernetes Secrets in production.

Required variables:

POSTGRES_DB=ticketing
POSTGRES_USER=appuser
POSTGRES_PASSWORD=changeme
DATABASE_URL=postgresql://appuser:changeme@postgres:5432/ticketing
REDIS_URL=redis://redis:6379
PORT=8080
API_URL=http://api:8080
NEXT_PUBLIC_API_URL=http://localhost:8080


## Part 1 - Local developer environment using Docker or Podman Compose

The goal is that any developer can clone the repo and run the entire stack locally with one command.

Required deliverables for part 1:
- compose.yaml defining all five services
- Containerfile or Dockerfile for each custom service (frontend, api, worker)
- .env.example with all variables listed above but with placeholder values
- docs/README.md explaining how to start and stop the stack

Containerfile rules that apply to every service:
- use multi-stage build, separate the build stage from the runtime stage
- use a minimal runtime base image like node:20-alpine or python:3.12-slim
- never use latest tag on base images, always pin a version
- run the app as a non-root user inside the container, create a user called appuser

compose.yaml must:
- define all five services: frontend, api, worker, postgres, redis
- use a named volume for postgres data persistence
- load variables from .env file
- enable hot-reload for frontend and api where it makes sense
- make sure services start in the right order using depends_on

Useful local commands:
docker compose up -d
docker compose logs -f
docker compose down
docker compose down -v
curl http://localhost:8080/healthz


## Part 2 - Production deployment on Kubernetes or OpenShift

The goal is to have all services running in a proper orchestrated production environment.

Required deliverables for part 2:
- k8s/ directory containing all manifests, or a Helm chart
- docs/deployment.md explaining how to deploy to production
- docs/runbook.md with incident procedures for three scenarios: database crash, bad image tag, wrong secret
- security/trivy-report.json with the image vulnerability scan output

Kubernetes manifest requirements:
- Deployment for each service: frontend, api, worker, postgres, redis
- Service object for each deployment
- ConfigMap for non-sensitive configuration
- Secret for sensitive values like passwords and connection strings
- Ingress for external access to frontend and api
- Liveness probe and readiness probe on api, frontend, and worker
- Resource requests and limits defined on every deployment
- NetworkPolicy to restrict which services can talk to each other
- ServiceAccount with minimal RBAC permissions

Useful kubectl commands:
kubectl apply -f k8s/
kubectl get pods
kubectl get services
kubectl logs -f deployment/api
kubectl set image deployment/api api=registry/image:new-tag
kubectl rollout status deployment/api
kubectl rollout undo deployment/api
trivy image registry/image:tag


## CI/CD with GitHub Actions

Create .github/workflows/ci.yaml with these steps in order:
1. lint and test the code
2. build the container images
3. scan images with Trivy, fail the pipeline if critical vulnerabilities are found
4. push images to container registry with a proper tag (use git sha or version, never latest in production)
5. deploy to Kubernetes

Quality gate: the pipeline must not push or deploy if any step fails.


## Security principles to follow throughout the project

Never put secrets in code or Dockerfiles.
Always run Trivy scan before pushing an image.
Use non-root user in every container.
Pin base image versions.
Use Kubernetes Secrets for production credentials.
Apply NetworkPolicy so services only accept traffic from services that need to reach them.
Use a ServiceAccount with only the permissions the service actually needs.
Document every security decision briefly in comments or in the runbook.


## Grading breakdown (100 points total)

I1 - Container vs VM comparison, service selection, architecture design: 16 points
I2 - Secure image building, minimal images, non-root, scanning, tagging policy: 16 points
I3 - Automated CI build and test, image registry publishing, reproducible delivery: 17 points
I4 - Security checks in CI/CD pipeline, no hardcoded secrets, DevSecOps tooling: 17 points
I5 - Realistic incident scenarios, root cause analysis, runbook quality: 17 points
I6 - Kubernetes deploy, probes, resources, ingress, RBAC, rolling update and rollback: 17 points


## Reference links

Kubernetes overview: https://kubernetes.io/docs/concepts/overview/
Kubernetes Secrets: https://kubernetes.io/docs/concepts/configuration/secret/
Network Policies: https://kubernetes.io/docs/concepts/services-networking/network-policies/
Trivy image scanner: https://trivy.dev/latest/
GitHub Actions docs: https://docs.github.com/en/actions
App source code: https://github.com/matej-basic/devops-project-app
