# ADR 0008: GitOps with Argo CD

Status: Accepted
Date: 2026-11-12

## Context

Deploying with kubectl apply leaves the real cluster state in a terminal history.
CI should not hold credentials for the cluster.

## Decision

- Argo CD v3.6.1 runs in the cluster and pulls from this repository.
- Auto-sync, prune and self-heal are on, so Git is the source of truth.
- ignoreDifferences on /spec/replicas, because the HPA owns the replica count.
- Argo manages the apps folder only. The lab Postgres and the secrets are outside Git.
- Two Applications: clinicflow-api-dev for the local cluster, and
  clinicflow-api-aws for the EC2 node. The AWS one uses a Kustomize overlay that
  pins the ECR image by commit SHA.
- The repository is public and holds no secrets, so Argo needs no credentials.
- The first Application is applied once by hand (the bootstrap). An app-of-apps
  root Application is planned.

## Alternatives considered

- CI running kubectl apply: needs cluster credentials in GitHub.
- Flux: similar idea, but Argo CD has a UI that makes demos clearer.

## Consequences

- Argo polls Git about every 3 minutes unless refreshed manually.
- A manual edit in the cluster was reverted by self-heal. A deleted Service was recreated.
- Deploying a new image means changing the tag in the overlay through a pull
  request. Automating that is a later step.
- On a 2 GiB node, three Argo components (dex, notifications, ApplicationSet)
  are scaled to zero to save memory.
