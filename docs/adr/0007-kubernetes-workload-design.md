# ADR 0007: Kubernetes workload design

Status: Accepted
Date: 2026-11-11

## Context

The API must run safely and recover by itself, on a node as small as 2 GiB of memory.

## Decision

- One namespace per environment (clinicflow-dev).
- Hardened pods: non-root user 10001, seccomp RuntimeDefault, no privilege
  escalation, read-only root filesystem, all capabilities dropped, and an
  emptyDir at /tmp.
- Liveness probe on /health restarts a stuck container. Readiness probe on /ready
  removes a pod from traffic when the database is unreachable.
- Requests 50m CPU and 64Mi memory, limits 200m and 256Mi.
- HPA scales on 70 percent CPU of the request, from 2 to a maximum of 5 replicas.
- A PodDisruptionBudget keeps at least one pod available.
- Rolling updates with maxUnavailable 0, so a bad release cannot remove working pods.
- Network policies: default deny for ingress, API reachable from its namespace
  and kube-system, Postgres reachable only from pods labeled app=clinicflow-api.
- Manifests use Kustomize, which kubectl and Argo CD both understand.
- The lab Postgres lives in lab/ and is not synced. Secrets are created with
  kubectl and never stored in Git.

## Alternatives considered

- Helm: more flexible, but heavier than this project needs.
- Storing secrets in Git: unsafe. External Secrets with SSM Parameter Store is planned.

## Consequences

- Experiments confirmed the design. Killing a pod was replaced at once. With the
  database down the API pods went not Ready but were not restarted. A bad image
  left the old pods serving. The network policy blocked an unlabeled pod from
  Postgres while an identical labeled pod connected.
- Under load, CPU reached about 378 percent of the request and the HPA scaled
  from 2 to 4, with demand above the cap at the peak.
- A test pod that carried the API label was counted by the HPA and the Service.
  Labels are an interface, so tests must not reuse application labels.
- Small requests make utilization percentages look large. They must be sized
  together with the node.
