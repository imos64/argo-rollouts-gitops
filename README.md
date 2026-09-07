# Argo Rollouts GitOps

[![Validate](https://github.com/imos64/argo-rollouts-gitops/actions/workflows/validate.yml/badge.svg)](https://github.com/imos64/argo-rollouts-gitops/actions/workflows/validate.yml)

**Canary and blue/green progressive delivery.** Argo Rollouts v1.10.0 with manual-promotion blue/green delivery plus an independent canary example. Argo Rollouts reconciles Rollout resources; Argo CD or Flux can deliver those resources from Git.

Independent deployment examples maintained by imos64. Upstream software remains maintained by its respective authors. This collection does not claim a measured ranking or that every tool independently performs continuous Git reconciliation.

## Local validation

Requirements: Linux amd64, Python 3.12+, Helm 3.21.3 and Make. CLI downloads are checksum-verified and placed in ignored `.tools/`. Internet access is needed for tools and Kubernetes schemas. Docker is needed for werf's container smoke test.

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements-dev.txt
make validate
```

`make render` refreshes the committed controller installation preview. CI has read-only repository permissions and no deployment credentials. It checks artifact checksums, exact rendering, strict Kubernetes 1.35.0 schemas, pinned upstream custom-resource schemas and delivery guardrails. Schema validation does not execute CEL, admission webhooks or controllers, and does not establish compatibility with every cluster version.

## Blue/green delivery with manual promotion

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f rendered/install.yaml
kubectl -n argo-rollouts rollout status deployment/argo-rollouts
kubectl apply -f bootstrap/namespace.yaml
kubectl apply -f apps/demo/workload.yaml
kubectl -n rollouts-demo get rollouts,pods,services
```

The active and preview Services start with the same app selector; the controller manages their ReplicaSet hash selection. `autoPromotionEnabled: false` pauses an update before switching active traffic. Inspect the preview endpoint and controller status before promoting with a matching Argo Rollouts kubectl plugin. The first deployment bootstraps the active revision; change the Pod template through Git to exercise an actual update and promotion pause.

## Separate canary example

```bash
kubectl apply -f variants/canary/namespace.yaml
kubectl apply -f variants/canary/workload.yaml
```

This independent namespace uses four replicas, weights 25/50/100 and manual pauses. Without a traffic-router integration, weights are approximated using replica counts; they do not guarantee exact request percentages or account for sticky/long-lived connections. Add a supported router and real analysis metrics for production canaries.

Use Argo CD or Flux to deliver Rollout resources, while Argo Rollouts owns their ReplicaSets and traffic selectors. Exclude controller-managed Service selector mutations from any conflicting GitOps reconciliation. No independent Deployment should own the same Pod selector. Revert the Git change or use the Rollouts abort/undo workflow, then reconcile Git so an aborted revision is not immediately reapplied. Schema checks do not establish live promotion or traffic correctness.

## Operations and provenance

Local validation was completed on 2026-09-07. No GitOps controller, cloud resource, external cluster registration or live delivery pipeline was deployed. See [validation scope](docs/validation.md), [controller ownership](docs/ownership.md), and [official upstream documentation](https://argoproj.github.io/argo-rollouts/).

Exact upstream URLs and artifact SHA-256 checksums are recorded in `package.json`. Controller versions follow the pinned upstream release/chart; demo workload images are digest-pinned. Review image digests, release notes and CRD lifecycle during upgrades. Upstream charts/manifests retain their own licensing; repository-authored integration files use MIT. Secrets and private site files must remain outside this public repository.
