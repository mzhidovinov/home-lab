# ESO backend E2E (GitOps)

Single `ExternalSecret` deployed via Argo CD (`home-lab-eso-e2e` app). Backend-specific `secretStoreRef` and `remoteRef` come from clustergroup `sharedValueFiles` overlays — not hardcoded in the chart.

## How it works

| Layer | Source |
|-------|--------|
| Backend toggle | `global.secretStore.backend` in `values-global.yaml` |
| Store name | `values-eso-backend-{vault\|kubernetes}.yaml` → `secretStore.name` |
| Remote key | Same overlay → `e2eTest.remoteRef.key` |
| ExternalSecret | `charts/home-lab-eso-e2e/templates/externalsecret.yaml` (one template) |

## Seed test secret (kubernetes backend)

```bash
oc create secret generic config-demo -n validated-patterns-secrets \
  --from-literal=secret=from-kubernetes-backend-gitops \
  --dry-run=client -o yaml | oc apply -f -
```

## Verify (kubernetes backend active)

```bash
oc get clustersecretstore kubernetes-backend
oc get role,rolebinding -n validated-patterns-secrets | grep external-secrets
oc get externalsecret,secret -n eso-e2e-test
oc get secret e2e-projection -n eso-e2e-test -o jsonpath='{.data.secret}' | base64 -d; echo
# expect: from-kubernetes-backend-gitops
```

## Cleanup

```bash
oc delete secret config-demo -n validated-patterns-secrets
# Remove app from values-hub.yaml or set e2eTest.enabled: false to stop projecting
```
