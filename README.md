# argotest

Opinionated Argo CD repo that deploys a small stack (Prometheus, Fluent Bit, OpenTelemetry, and a hello-world app) to multiple clusters using an ApplicationSet generator and Kustomize overlays.

## Overview
- Control-plane: Argo CD runs in a single cluster (prod in this example: `kind-kind`).
- Root apps: An `ApplicationSet` scans `overlays/*` and creates one root `Application` per environment (e.g., `prod`, `test`).
- Children: Each root app renders Kustomize overlays which produce Argo CD `Application` resources for the actual apps (Prometheus, Fluent Bit, etc.).
- Multi-cluster: Overlays patch each child `Application` to target a specific registered cluster by `spec.destination.name`.

## Repository Layout
- `applicationset.yaml`: Defines the generator and template for root Applications.
- `base/`: Kustomize base that contains child Argo CD `Application` manifests in `base/apps/*`.
- `overlays/`: One folder per environment; includes the base and applies patches (e.g., cluster destinations, per-env tweaks).
- `manifests/`: Raw Kubernetes/Kustomize manifests that some apps deploy (e.g., `opentelemetry`, `hello-world`).

## ApplicationSet (Generator + Template)
File: `applicationset.yaml:1`
- Generator: Git directories generator
  - `spec.generators[0].git.directories: overlays/*` — creates one output per directory under `overlays/`.
  - `repoURL: https://github.com/ragnarhulskamp/argotest.git`, `revision: main` — points to this repo/branch.
- Template: Becomes a root `Application` per environment
  - `metadata.name: '{{path.basename}}'` — names like `prod`, `test`.
  - `spec.source.path: '{{path}}'` — the overlay dir (e.g., `overlays/prod`).
  - `spec.destination.server: https://kubernetes.default.svc` — roots live in the same cluster as Argo CD.
  - `syncPolicy.automated.prune/selfHeal: true` — auto-sync with pruning.

What it does: The ApplicationSet controller reconciles `applicationset.yaml`, discovers `overlays/prod` and `overlays/test`, and creates two root Applications (`prod`, `test`), each sourcing from its overlay path.

Prerequisite: Ensure the Argo CD ApplicationSet controller and CRDs are installed alongside Argo CD.

## Overlays (Kustomize)
Files:
- `overlays/prod/kustomization.yaml:1`
- `overlays/test/kustomization.yaml:1`

Common elements:
- `resources: [../../base]` — include the base, which renders child Argo `Application` resources from `base/apps/*`.
- `patchesJson6902` — JSON6902 patches applied to those child `Application` resources.

Key patches:
- Multi-cluster targeting by cluster name:
  - Replace `spec.destination.server` with `spec.destination.name` so Argo deploys to a named, registered cluster.
  - Prod overlay sets `name: kind-kind`.
  - Test overlay sets `name: kind-test`.
- Per-app customization (example):
  - Test overlay adds `spec.source.kustomize.namePrefix: test-` to the `hello-world` app only.

Why patch by name: Using `destination.name` is resilient to API server URL changes and matches `argocd cluster add <context-name>`.

## Base Children (Apps of Apps)
Files:
- `base/kustomization.yaml:1` — includes all child apps.
- `base/apps/*.yaml` — each is an Argo CD `Application`:
  - `kube-prometheus-stack.yaml` (Helm chart)
  - `fluent-bit.yaml` (Helm chart)
  - `opentelemetry-operator.yaml` (Helm chart)
  - `opentelemetry-collector.yaml` (local manifests)
  - `hello-world.yaml` (local manifests)

These child Applications are what actually install software into the destination clusters. The overlays only patch their destinations and, when needed, per-app Kustomize options.

## Hello-World Example (Per-env Names)
Files:
- `manifests/hello-world/` — simple Deployment + Service.
- `base/apps/hello-world.yaml:1` — Argo app sourcing the Kustomize directory.
- `overlays/test/kustomization.yaml:1` — adds `namePrefix: test-` specifically to the `hello-world` app via JSON6902 patch.

Behavior:
- Prod: resources named `hello-world` (no prefix), deployed to `kind-kind`.
- Test: resources named `test-hello-world`, deployed to `kind-test`.

## Multi-Cluster Setup
Register clusters with Argo CD from your workstation (kubectl context must point to the cluster):
- `argocd cluster add kind-kind`
- `argocd cluster add kind-test` (when created)

Apply the ApplicationSet to the Argo CD control-plane cluster (e.g., kind-kind):
- `kubectl apply -n argocd -f applicationset.yaml`

Argo CD will render root apps for `prod` and `test`. `prod` syncs immediately; `test` will report "cluster not found" until `kind-test` is registered.

## Add a New Environment
1) Create a new overlay directory under `overlays/` (copy one of the existing ones).
2) Set its destination cluster by patching `spec.destination.name` in `patchesJson6902`.
3) Commit and push — the ApplicationSet will detect the new folder and create a new root app automatically.

## Useful Commands
- List managed clusters: `argocd cluster list`
- Check Applications: `kubectl get applications.argoproj.io -n argocd`
- Build overlays locally (preview): `kustomize build overlays/prod`

## Notes
- File `base/core-app/app-of-apps.yaml` is a legacy single root app reference and is not used when `applicationset.yaml` is in place. You can remove or repoint it if you want a classic single app-of-apps instead of ApplicationSet.
- Helm chart versions: some are pinned; consider pinning all once stable.
