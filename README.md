# clinicflow-gitops

What runs in the cluster for
[ClinicFlow](https://github.com/Apoor-Tyagi/clinicflow-infra), kept in sync by
Argo CD. Git is the source of truth: a change here reaches the cluster through a
pull request, and manual edits in the cluster are reverted by self-heal. The
project overview and decision records live in the infra repository.

## Layout

| Path | What it is |
|---|---|
| `apps/clinicflow-api/base/` | Deployment, Service, HPA, PodDisruptionBudget and network policies |
| `apps/clinicflow-api/overlays/aws-dev/` | The AWS overlay, which pins the ECR image by commit SHA |
| `argocd/applications/` | Argo CD Applications for the API and the monitoring stack |
| `argocd/aws/` | The Application that the AWS node applies at boot |
| `platform/monitoring/` | ServiceMonitor, SLO rules and the Grafana dashboard |
| `lab/` | A throwaway PostgreSQL for local clusters. Not synced |
| `docs/adr/` | Decisions about the workload, GitOps and observability |

## Rules this repository follows

- No secrets in Git. Passwords and connection strings are created in the cluster.
- The workload is hardened: non-root, read-only root filesystem, dropped
  capabilities, probes, resource limits and default-deny network policies.
- Monitoring configuration lives apart from the API, so the AWS node's API
  Application does not depend on Prometheus CRDs.

## Deploying a new image

Change `newTag` in `apps/clinicflow-api/overlays/aws-dev/kustomization.yaml` to a
commit SHA that exists in ECR, and open a pull request. The ECR lifecycle policy
keeps about 20 pushes, so a pinned tag that is never refreshed will eventually
expire. See
[ADR 0006](https://github.com/Apoor-Tyagi/clinicflow-app/blob/main/docs/adr/0006-ci-and-image-publishing.md).

## Local cluster

The local cluster is k3d with Argo CD, using the `clinicflow-api-dev` and
monitoring Applications. Reach the interfaces with `kubectl port-forward`. None of
them is exposed publicly.
