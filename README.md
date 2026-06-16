# Multicloud Gitops

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

[Live build status](https://validatedpatterns.io/ci/?pattern=mcgitops)

## Start Here

If you've followed a link to this repository, but are not really sure what it contains
or how to use it, head over to [Multicloud GitOps](https://validatedpatterns.io/patterns/multicloud-gitops/)
for additional context and installation instructions

## Rationale

The goal for this pattern is to:

* Use a GitOps approach to manage hybrid and multi-cloud deployments across both public and private clouds.
* Enable cross-cluster governance and application lifecycle management.
* Securely manage secrets across the deployment.

## Secrets backend switch

ESO backend is controlled by one value in `values-global.yaml`:

```yaml
global:
  secretStore:
    backend: vault        # or kubernetes
```

HashiCorp Vault **always** stays deployed (`vault` application in `values-hub.yaml`). Switching backend only changes which `ClusterSecretStore` the `openshift-external-secrets` chart renders and where `make load-secrets` stores material.

| Backend | ClusterSecretStore | Secret store name for ExternalSecrets | `make load-secrets` target |
|---------|-------------------|---------------------------------------|----------------------------|
| `vault` (default) | `vault-backend` | `vault-backend` | Vault paths under `hub/` (e.g. `secret/data/global/config-demo`) |
| `kubernetes` | `kubernetes-backend` | `kubernetes-backend` | Kubernetes `Secret` objects in `validated-patterns-secrets` |

Backend-specific store metadata (`values-eso-backend-vault.yaml` / `values-eso-backend-kubernetes.yaml`) is merged via `clusterGroup.sharedValueFiles` and sets `secretStore.name` for pattern charts. These files contain **no secret values** and are safe to commit.

Never commit `values-secret*` files with real credentials. Use `make load-secrets` with `values-secret.yaml` as the primary secret loader per [VP secrets documentation](https://validatedpatterns.io/learn/secrets-management-in-the-validated-patterns-framework/).

### Switch to kubernetes backend

1. Set the toggle (either edit `values-global.yaml` or run `make secrets-backend-kubernetes`).
2. Commit and push so Argo CD picks up the change.
3. Sync the `openshift-external-secrets` application.
4. Run `make load-secrets` to populate secrets as Kubernetes `Secret` objects in `validated-patterns-secrets`.
5. Verify:

```bash
oc get clustersecretstore kubernetes-backend
oc get role,rolebinding -n validated-patterns-secrets
```

Vault remains running but unused by ESO until you switch back.

### Switch to vault backend

1. Set `global.secretStore.backend: vault` (or `make secrets-backend-vault`).
2. Commit, push, and sync `openshift-external-secrets`.
3. Run `make load-secrets` (configures Vault K8s auth and loads secrets into Vault).
4. Verify:

```bash
oc get clustersecretstore vault-backend
```

### ExternalSecret store reference

When authoring `ExternalSecret` manifests, match the active backend:

```yaml
spec:
  secretStoreRef:
    name: vault-backend       # when backend=vault
    kind: ClusterSecretStore
```

```yaml
spec:
  secretStoreRef:
    name: kubernetes-backend  # when backend=kubernetes
    kind: ClusterSecretStore
```

Pattern charts that use the clustergroup `secretStore.name` value pick up the correct name automatically via the overlay files.

### RoleBinding patch

`openshift-external-secrets` 0.0.4 creates a `Role` in `validated-patterns-secrets` for the kubernetes backend but not the matching `RoleBinding`. The local `charts/eso-rbac` chart supplies that binding when `backend=kubernetes`.

## OpenShift MCP Server (GitOps)

Two local charts deploy the [MCP Lifecycle Operator](https://github.com/openshift/mcp-lifecycle-operator) and an `MCPServer` on the hub:

| Argo application | Chart | Purpose |
|------------------|-------|---------|
| `mcp-lifecycle-operator` | `charts/mcp-lifecycle-operator` | CRD, operator deployment (sync-wave -10) |
| `mcp-server` | `charts/mcp-server` | MCPServer CR, ConfigMap, Route, RBAC (sync-wave -5) |

Requires OCP 4.22+ with **TechPreviewNoUpgrade**. Default route: `https://mcp-server.<hubClusterDomain>/mcp`.

Configure image, route, and RBAC in `charts/mcp-server/values.yaml`. Scope access via `rbac.clusterRoleName` (lab default: `cluster-admin`; use `view` or a custom ClusterRole for production).

Verify after Argo sync:

```bash
oc get pods -n mcp-lifecycle-operator-system
oc get mcpserver -n openshift-mcp-server
curl -sk https://mcp-server.apps.ocp.lab/healthz
```
