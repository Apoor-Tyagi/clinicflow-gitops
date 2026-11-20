# ADR 0011: Observability with Prometheus, Grafana and SLO alerts

Status: Accepted
Date: 2026-11-20

## Context

The API was deployable but not observable. Job-ready operations work needs
metrics, dashboards and alerts tied to a service level objective, and the stack
has to fit a laptop cluster and a small EC2 node.

## Decision

- kube-prometheus-stack, chart version pinned to 93.6.0, delivered by an Argo CD
  Application with ServerSideApply (the CRDs are large).
- 6 hours of retention and no persistent volume, because the data is disposable.
- The control-plane scrapers (scheduler, controller manager, etcd, kube-proxy)
  are off, because k3s does not expose them the standard way.
- Grafana admin credentials come from a Secret created with kubectl, never from Git.
- Monitoring configuration lives in platform/monitoring with its own Application,
  so the AWS node's API Application does not depend on Prometheus CRDs.
- A ServiceMonitor scrapes /metrics through a named Service port. The network
  policy allows the monitoring namespace to reach the API on port 8000.
- SLOs on /api/*: 99.9 percent availability and p95 latency under 300 ms. The
  kubelet probes /health and /ready constantly, so they are excluded.
- Burn-rate alerts use windows of 1h/5m and 3h/30m, because 6 hours of retention
  cannot support the 3-day windows from the usual guidance.
- The dashboard is a ConfigMap in Git, loaded by the Grafana sidecar, which is
  restricted to the monitoring namespace.

## Alternatives considered

- Hosted monitoring: less to run, but it adds cost and an account.
- Prometheus and Grafana installed by hand: faster, but not reproducible.
- CloudWatch only: native to AWS, but it does not show Kubernetes internals.

## Consequences

- Measured memory for the stack: about 1.2 GiB (Grafana 624 Mi, Prometheus
  377 Mi, the rest about 215 Mi). A t3.small had 278 MiB free, so monitoring
  needs a t3.medium on the node.
- Grafana restarted in a loop on first start. Its start was CPU-heavy and slow,
  and the liveness probe killed it before it listened. A startup probe fixed it.
- Baseline: 20 virtual users for 3 minutes gave about 535 requests per second,
  0 percent errors and p95 91 ms. It ran through one kubectl port-forward, so it
  measures the API and the tunnel, not autoscaling.
- The path label holds the raw URL, so unknown paths create new series. A route
  template would fix that.
- The alerts have not fired yet. Fault injection is planned to prove they work.
