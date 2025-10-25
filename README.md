Voting App — Kubernetes Deployment

This repository contains a Kubernetes deployment of a simple voting application composed of multiple microservices. The manifests are written in production-style YAML files (one service per file where applicable). The README explains the architecture, the deployment order you should follow, important environment variables used by the manifests, and basic verification commands.

Project architecture and data flow

vote (frontend): the user-facing voting UI. This service depends on Redis for storing/queuing votes before they are processed.

redis: in-memory data store used by the vote app and the worker.

postgres (service named db): persistent relational database that stores final vote counts.

worker: background processor that reads queued votes from Redis and writes them into Postgres.

result: dashboard service that reads aggregated results from Postgres and displays them.

Data flow (step-by-step):

Users interact with the vote frontend.

vote writes incoming votes to Redis.

worker reads queued votes from Redis, processes them, and writes the results into Postgres (db).

result reads from Postgres and exposes the aggregated vote counts.

Service names used by the manifests:

Redis service name: redis

Postgres service name: db

Vote service name: vote

Worker deployment name: worker-deployment

Result deployment name: result-deployment

Deployment order (recommended)

To ensure proper startup and avoid connection errors, follow this order when deploying to a Kubernetes cluster:

Deploy data stores first:

Redis

Postgres

These two can be applied together because both are independent of the application components.

Deploy application components:

Deploy the vote frontend next. It depends on the Redis service for immediate operation.

Deploy the worker. The worker requires access to both Redis and Postgres.

Deploy the result dashboard last. It reads from Postgres to show results.

Example commands (apply manifests in the repository folder voteapp/ or apply files individually):

Apply both databases together:

kubectl apply -f voteapp/redis.yml -f voteapp/postgres.yml


Apply frontend and worker and result:

kubectl apply -f voteapp/vote.yml -f voteapp/worker.yml -f voteapp/result.yml


Or apply the whole folder (order of resource creation depends on Kubernetes; follow the recommended order above if you want to guarantee start sequence):

kubectl apply -f voteapp/

Basic readiness / verification commands

Check deployments and rollout status:

kubectl get deployments
kubectl rollout status deployment/redis-deployment
kubectl rollout status deployment/postgres-deployment
kubectl rollout status deployment/vote-deployment
kubectl rollout status deployment/worker-deployment
kubectl rollout status deployment/result-deployment


Check pods and services:

kubectl get pods
kubectl get svc
kubectl get pods -l app=redis
kubectl get pods -l app=vote


Inspect logs when troubleshooting:

kubectl logs deployment/vote-deployment
kubectl logs deployment/worker-deployment
kubectl logs deployment/result-deployment

Environment variables used by the manifests

The manifests already include the environment variables required by each service. If you customize images or change service names, make sure these variables are updated accordingly.

Vote frontend (vote Deployment)

REDIS_HOST — service name for Redis (value in manifests: redis)

Worker (worker Deployment)

REDIS_HOST — service name for Redis (value in manifests: redis)

DB_HOST — service name for Postgres (value in manifests: db)

POSTGRES_USER

POSTGRES_PASSWORD

POSTGRES_DB

Result (result Deployment)

DB_HOST — service name for Postgres (value in manifests: db)

POSTGRES_USER

POSTGRES_PASSWORD

POSTGRES_DB

Postgres container (initialization)

POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB — used by the Postgres container at startup to initialize the database.

Note: The exact variables and values are present in the YAML manifests. If you replace container images or change secret/ConfigMap usage, move sensitive values into Kubernetes Secrets rather than inline environment variables.

Folder structure
voteapp/
 ├── redis.yml
 ├── postgres.yml
 ├── vote.yml
 ├── result.yml
 └── worker.yml

Security and production considerations (short list)

Move credentials out of plain YAML into Kubernetes Secrets before using this in a shared or production cluster.

Use resource requests/limits for each container to avoid noisy-neighbor issues.

Add liveness and readiness probes to each Deployment for robust orchestration.

Consider using PersistentVolumeClaims for Postgres storage in any non-ephemeral environment.

Use NetworkPolicies if you need to restrict traffic between components.

Troubleshooting tips

If vote cannot reach Redis, confirm the Redis service exists (kubectl get svc) and check the REDIS_HOST name in the vote Deployment.

If the worker cannot write to Postgres, confirm DB_HOST, Postgres credentials, and that Postgres accepts connections from the cluster.

Use kubectl describe pod <pod-name> to inspect events and errors for container start failures.
