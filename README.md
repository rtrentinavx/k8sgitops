Purpose
This file gives concise, project-specific guidance for AI coding agents working on this repo (k8sgitops). Focus on discoverable patterns, important files, and concrete commands to be immediately productive.

**Big Picture**
- **Repo role:** A GitOps repo for deploying an `nginx` app via ArgoCD using Kustomize overlays.
- **Deployment flow:** Author manifests under `nginx/base` and environment customizations under `nginx/overlays/*`; ArgoCD reads `applications/nginx.yaml` (points to `nginx/overlays/dev`).
- **Why this structure:** base holds canonical resources (Deployment, Service, ConfigMap generator). Overlays change runtime knobs (namespace, namePrefix, replicas, service type).

**Key files / entrypoints**
- `applications/nginx.yaml` : ArgoCD Application manifest. It targets `path: nginx/overlays/dev` and `namespace: nginx-dev`.
- `nginx/base/kustomization.yaml` : central kustomize config. Contains `configMapGenerator`, `images` override, and `commonLabels` (includes `managed-by: argocd`).
- `nginx/base/deployment.yaml` and `nginx/base/service.yaml` : canonical Deployment and Service definitions.
- `nginx/base/nginx.conf` : served via a generated ConfigMap and mounted with `subPath` into the container.
- `nginx/overlays/dev/kustomization.yaml` : overlays that add `namePrefix: dev-`, switch namespace to `nginx-dev`, reduce replicas and patch Service `type` to `ClusterIP`.

**Commands & workflows (concrete examples)**
- Render the dev manifest locally: `kustomize build nginx/overlays/dev` or `kubectl kustomize nginx/overlays/dev`.
- Apply dev manifests locally (useful for iterative testing): `kubectl apply -k nginx/overlays/dev`.
- Inspect what ArgoCD will deploy: open `applications/nginx.yaml` or run `kustomize build nginx/overlays/dev` and inspect output.
- Port-forward the (overlay-prefixed) service for local testing: `kubectl port-forward svc/dev-nginx 8080:80 -n nginx-dev` then `curl http://localhost:8080/health`.

**Project-specific conventions & patterns**
- Overlays use `namePrefix` (e.g., `dev-`) so resources in `overlays/dev` become `dev-<name>`; code should account for the prefix when referencing names in examples/tests.
- Config maps are generated via `configMapGenerator` and mounted using `subPath` to place `nginx.conf` at `/etc/nginx/nginx.conf` (see `deployment.yaml`).
- Image updates are done by editing `nginx/base/kustomization.yaml` under `images: - name: nginx newTag: ...`.
- `commonLabels` include `managed-by: argocd`; follow that labeling when adding cross-cutting resources.

**Integration points & infra notes**
- ArgoCD: `applications/nginx.yaml` is the source of truth for automatic sync. `syncPolicy` sets `CreateNamespace=true`, `ApplyOutOfSyncOnly=true`, and auto-prune/self-heal.
- Cloud LB hint: `service.yaml` attempts to set a GCP LB annotation `cloud.google.com/load-balancer-type: "Network"` (note: file uses singular `annotation`, see "Gotchas").
