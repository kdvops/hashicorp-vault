# agents.md — hashicorp-vault

## Repository identity
- **Repo:** `kdvops/hashicorp-vault`
- **Purpose:** GitOps-style HashiCorp Vault deployment for k3s using Argo CD.
- **Primary manifests:** Kubernetes base + k3s overlay + Argo CD Application.

## Stack and runtime
- **Workload:** HashiCorp Vault OSS `2.0.3`
- **Kubernetes pattern:** `StatefulSet` with integrated Raft storage
- **Exposed service:** `NodePort` on `8200` → `30200`
- **Storage class:** `local-path` in the k3s overlay
- **Config source:** `ConfigMap` with `vault.hcl`

## Deployment topology
- `argocd/vault-application.yaml`
  - Argo CD `Application` named `vault`
  - source repo: this repository
  - source path: `k8s/overlays/k3s`
  - destination namespace: `vault`
- `argocd/vault-userpass-bootstrap-application.yaml`
  - optional Argo CD `Application` for the `userpass` bootstrap Job
  - source path: `k8s/overlays/userpass-bootstrap`
- `k8s/base/`
  - namespace, service account, config map, headless service, service, statefulset, PDB, network policies
- `k8s/overlays/k3s/`
  - patches the service and statefulset for k3s defaults
- `k8s/overlays/userpass-bootstrap/`
  - deploys only the userpass bootstrap ConfigMap + Job

## Operational dependencies
- Requires Argo CD and a target Kubernetes cluster.
- Designed for k3s with `local-path` storage.
- Vault is expected to be initialized and unsealed manually after first start.

## Local bootstrap / deploy
- Apply the Argo CD Application:
  - `kubectl apply -f argocd/vault-application.yaml`
- Or apply the overlay directly:
  - `kubectl apply -k k8s/overlays/k3s`

## Access pattern
- Vault UI/API: `http://<NODE_IP>:30200`
- Kubernetes service: `vault` in namespace `vault`
- Headless service for raft/internal traffic: `vault-internal`

## Secret / credential locations
- No bootstrap secrets are committed to the repo.
- Initial Vault unseal keys and root token are generated manually during `vault operator init` and must be stored offline.
- GitOps `userpass` bootstrap expects two SealedSecrets in the `vault` namespace:
  - `vault-bootstrap-token` with key `token`
  - `vault-userpass-kalcala` with key `password`
- Any future auth methods, policies, or K8s auth config should live in separate manifests or external secret management, not in plaintext here.

## Repo-specific hazards
- `StatefulSet` defaults assume a single replica and integrated Raft.
- `NodePort` `30200` may need changing if it conflicts with another service in the cluster.
- Storage size and `local-path` assumptions are in `k8s/overlays/k3s/statefulset-patch.yaml`.
- Vault is not auto-initialized or auto-unsealed.
- NetworkPolicies are restrictive; internal/DNS access is explicitly allowed.

## Verification
- Render/apply the overlay:
  - `kubectl apply -k k8s/overlays/k3s`
- Check namespace and workload:
  - `kubectl -n vault get pods,svc,statefulset`
- Validate Vault health once initialized/unsealed:
  - `kubectl -n vault exec -it vault-0 -- vault status`
