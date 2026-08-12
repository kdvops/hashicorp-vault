# HashiCorp Vault on k3s with Argo CD

This repository contains a GitOps-friendly deployment of HashiCorp Vault for k3s using:

- Vault OSS `2.0.3`
- Integrated Raft storage
- `StatefulSet` with 1 replica
- Argo CD `Application`
- k3s overlay with direct `NodePort` access
- Basic network policies

## Repository layout

```text
argocd/
  vault-application.yaml
  vault-userpass-bootstrap-application.yaml
k8s/
  base/
    configmap.yaml
    kustomization.yaml
    networkpolicy-allow-dns.yaml
    networkpolicy-allow-internal.yaml
    networkpolicy-default-deny.yaml
    namespace.yaml
    pdb.yaml
    service-headless.yaml
    service.yaml
    serviceaccount.yaml
    statefulset.yaml
  overlays/
    k3s/
      kustomization.yaml
      service-patch.yaml
      statefulset-patch.yaml
    userpass-bootstrap/
      kustomization.yaml
```

## Design choices

- Uses integrated Raft instead of Consul to keep the stack smaller and easier to operate on k3s.
- Leaves the initial `vault operator init` and unseal flow as a manual step so no bootstrap secrets are committed to Git.
- Includes a GitOps bootstrap Job for `userpass` users that reads secrets from SealedSecrets.
- Exposes Vault directly through `NodePort` so you can access it with the IP of a k3s node.
- Uses the k3s `local-path` storage class in the overlay.
- Keeps the initial deployment single-node so direct IP access works reliably without standby redirects.

## Before syncing

Update these files with your real values:

- `k8s/overlays/k3s/statefulset-patch.yaml`
  - adjust storage size, resources, and storage class if needed
- `k8s/overlays/k3s/service-patch.yaml`
  - adjust `nodePort` if `30200` is already in use in your cluster

## Deploy with Argo CD

Apply the Argo CD application:

```bash
kubectl apply -f argocd/vault-application.yaml
```

Or apply the overlay directly with kubectl:

```bash
kubectl apply -k k8s/overlays/k3s
```

## Access

The service is exposed as `NodePort`.

- UI and API: `http://<NODE_IP>:30200`

Check it with:

```bash
kubectl -n vault get svc vault
```

## Initial bootstrap

After the pod is running, initialize Vault once:

```bash
kubectl -n vault exec -it vault-0 -- vault operator init
```

Save the unseal keys and root token in a secure offline location.

Then unseal the pod:

```bash
kubectl -n vault exec -it vault-0 -- vault operator unseal
```

Login:

```bash
kubectl -n vault exec -it vault-0 -- vault login
```

Validate status:

```bash
kubectl -n vault exec -it vault-0 -- vault status
```

## GitOps userpass bootstrap

This repository now includes a separate Argo CD Application that can upsert a Vault `userpass` account such as `hermes` when you are ready to run it.

Apply it only after the SealedSecrets exist:

```bash
kubectl apply -f argocd/vault-userpass-bootstrap-application.yaml
```

### Required sealed secrets

Create these as `SealedSecret` resources in the `vault` namespace:

- `vault-bootstrap-token`
  - key: `token`
  - value: a Vault token with permission to manage `auth/userpass`
- `vault-userpass-hermes`
  - key: `password`
  - value: the `userpass` password for `hermes`

### Example generation flow

```bash
kubectl -n vault create secret generic vault-bootstrap-token \
  --from-literal=token='YOUR_ADMIN_TOKEN' \
  --dry-run=client -o yaml | kubeseal --format yaml > vault-bootstrap-token.sealedsecret.yaml

kubectl -n vault create secret generic vault-userpass-hermes \
  --from-literal=password='YOUR_PASSWORD' \
  --dry-run=client -o yaml | kubeseal --format yaml > vault-userpass-hermes.sealedsecret.yaml
```

### Behavior

The bootstrap Job:
- waits for Vault to be reachable,
- ensures the `userpass` auth method exists,
- writes/updates `auth/userpass/users/hermes`,
- applies the policies configured in the Job (`admin` by default).

Update `HERMES_POLICIES` in the Job if the user should map to an additional policy from your Vault setup.

## Recommended next steps

- Scale to 3 replicas only when you are ready to add a stable access layer for HA traffic.
- Configure auto-unseal with a KMS or HSM-backed mechanism when you are ready for production hardening.
- Add auth methods and policies for your workloads, for example Kubernetes auth.
- Restrict `NodePort` access with firewall rules or a private network path.





##########################################





####  hacer login 
export VAULT_ADDR="http://10.0.0.100:30200"
vault login

####  crear policy admin
cat > /tmp/admin.hcl <<'EOF'
# Administración general de Vault
path "*" {
  capabilities = ["create", "read", "update", "delete", "list", "patch", "sudo"]
}
EOF


#### aplicar policy admin
vault policy write admin /tmp/admin.hcl

#### leer policy
vault policy read admin

#### listar todas las policy
vault auth list

#### habilitar policy userpass | si no esta activo
#vault auth enable userpass

#### Crear usuario
vault write auth/userpass/users/hermes \
  password='TuPasswordSegura'

#### Crear usuario y asignar a policy en 1 solo paso 

#### vault write auth/userpass/users/hermes \
####  password='TuPasswordSegura' \
####  token_policies='admin'
  
#### leer detalles de usuario   
vault read auth/userpass/users/hermes

#### asignar policy a usuario
vault write auth/userpass/users/hermes \
  token_policies="admin"


#### hacer login usando userpass
vault login -method=userpass username=hermes

#### ver detalles de usuario logueado
vault token lookup
