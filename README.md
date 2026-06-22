# Home Lab

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A [Validated Patterns](https://validatedpatterns.io/) hub deployment (`global.pattern: home-lab`) derived from [Multicloud GitOps](https://validatedpatterns.io/patterns/multicloud-gitops/). Upstream pattern docs and install flow still apply for bootstrap and day‑zero setup.

## Start Here

For general Validated Patterns concepts and installation, see [Multicloud GitOps](https://validatedpatterns.io/patterns/multicloud-gitops/).

This repo trims the upstream pattern to a single hub cluster group and adds local GitOps charts for External Secrets and the OpenShift MCP server.

## Hub applications

Argo CD applications are defined in `values-hub.yaml`:

| Application | Source | Purpose |
|-------------|--------|---------|
| `openshift-external-secrets` | `charts/openshift-external-secrets` | ClusterSecretStore, ESO RBAC, operator config |
| `mcp-lifecycle-operator` | `charts/mcp-lifecycle-operator` | MCP Lifecycle Operator CRD and deployment (sync-wave -10) |
| `mcp-server` | `charts/mcp-server` | MCPServer CR, ConfigMap, Route, RBAC (sync-wave -5) |

The upstream `hashicorp-vault` application is **commented out** in `values-hub.yaml`. Use the kubernetes secrets backend (default) or point ESO at an external Vault via chart values (see `charts/openshift-external-secrets/README.md`).

Local charts (formerly prefixed `home-lab-*`):

```
charts/
├── mcp-lifecycle-operator/
├── mcp-server/
└── openshift-external-secrets/
```

## Secrets backend switch

ESO backend is controlled by `global.secretStore.backend` in `values-global.yaml`:

```yaml
global:
  secretStore:
    backend: kubernetes   # or vault
```

Default in this repo: `kubernetes`.

Two committed overlay pairs keep chart references in sync with the toggle:

| Files | Merged via | Sets |
|-------|------------|------|
| `values-eso-backend-{vault,kubernetes}.yaml` | `clusterGroup.sharedValueFiles` | `secretStore.name` for pattern charts |
| `values-eso-operator-backend-{vault,kubernetes}.yaml` | `openshift-external-secrets` `extraValueFiles` | `global.secretStore.backend` for the local ESO chart |

These files contain **no secret values** and are safe to commit.

Never commit `values-secret.yaml` with real credentials. Use `make load-secrets` with `values-secret.yaml` (from `values-secret.yaml.template`) per [VP secrets documentation](https://validatedpatterns.io/learn/secrets-management-in-the-validated-patterns-framework/).

| Backend | ClusterSecretStore | Secret store name for ExternalSecrets | `make load-secrets` target |
|---------|-------------------|---------------------------------------|----------------------------|
| `kubernetes` (default) | `kubernetes-backend` | `kubernetes-backend` | Kubernetes `Secret` objects in `validated-patterns-secrets` |
| `vault` | `vault-backend` | `vault-backend` | Vault paths under `hub/` (requires in-cluster or external Vault) |

Makefile helpers:

```bash
make secrets-backend-kubernetes   # set backend to kubernetes
make secrets-backend-vault        # set backend to vault
```

### Switch to kubernetes backend

1. Set the toggle (edit `values-global.yaml` or run `make secrets-backend-kubernetes`).
2. Commit and push so Argo CD picks up the change.
3. Sync the `openshift-external-secrets` application.
4. Run `make load-secrets` to populate secrets as Kubernetes `Secret` objects in `validated-patterns-secrets`.
5. Verify:

```bash
oc get clustersecretstore kubernetes-backend
oc get role -n validated-patterns-secrets
```

### Switch to vault backend

1. Set `global.secretStore.backend: vault` (or `make secrets-backend-vault`).
2. Deploy or configure a Vault endpoint (uncomment the `vault` application in `values-hub.yaml`, or set `ocpExternalSecrets.vault.externalAddress` in chart values).
3. Commit, push, and sync `openshift-external-secrets`.
4. Run `make load-secrets` (configures Vault K8s auth and loads secrets into Vault when using framework-managed Vault).
5. Verify:

```bash
oc get clustersecretstore vault-backend
```

### ExternalSecret store reference

When authoring `ExternalSecret` manifests, match the active backend:

```yaml
spec:
  secretStoreRef:
    name: kubernetes-backend  # when backend=kubernetes
    kind: ClusterSecretStore
```

```yaml
spec:
  secretStoreRef:
    name: vault-backend       # when backend=vault
    kind: ClusterSecretStore
```

Pattern charts that use the clustergroup `secretStore.name` value pick up the correct name automatically via `values-eso-backend-*.yaml`.

## OpenShift MCP Server (GitOps)

Two local charts deploy the [MCP Lifecycle Operator](https://github.com/openshift/mcp-lifecycle-operator) and an `MCPServer` on the hub (see table above).

Requires OCP 4.22+ with **TechPreviewNoUpgrade**. Default route: `https://mcp-server.<hubClusterDomain>/mcp`.

Configure image, route, and RBAC in `charts/mcp-server/values.yaml`. Scope access via `rbac.clusterRoleName` (lab default: `cluster-admin`; use `view` or a custom ClusterRole for production).

Verify after Argo sync:

```bash
oc get pods -n mcp-lifecycle-operator-system
oc get mcpserver -n openshift-mcp-server
curl -sk https://mcp-server.apps.ocp.lab/healthz
```
