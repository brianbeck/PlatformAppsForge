# Keycloak Operator

Vendored from [keycloak/keycloak-k8s-resources](https://github.com/keycloak/keycloak-k8s-resources) tag `26.0.5`.

## Files

| File | Purpose |
|---|---|
| `crds/keycloaks.yaml` | `Keycloak` CRD — server instance |
| `crds/keycloakrealmimports.yaml` | `KeycloakRealmImport` CRD — declarative realm config |
| `operator.yaml` | ServiceAccount, ClusterRoles, RoleBindings, Service, Deployment |

The operator is namespace-scoped to `keycloak`. The Argo CD ApplicationSet creates this namespace via `syncOptions: [CreateNamespace=true]`.

Operator image: `quay.io/keycloak/keycloak-operator:26.0.5`.

## Updating to a newer Keycloak version

```bash
cd /tmp && mkdir kc-vendor && cd kc-vendor
TAG=26.x.y
for f in keycloaks.k8s.keycloak.org-v1.yml keycloakrealmimports.k8s.keycloak.org-v1.yml kubernetes.yml; do
  curl -fsSL "https://raw.githubusercontent.com/keycloak/keycloak-k8s-resources/$TAG/kubernetes/$f" -o "$f"
done
cp keycloaks.k8s.keycloak.org-v1.yml ~/src/PlatformAppsForge/platform/keycloak-operator/crds/keycloaks.yaml
cp keycloakrealmimports.k8s.keycloak.org-v1.yml ~/src/PlatformAppsForge/platform/keycloak-operator/crds/keycloakrealmimports.yaml
cp kubernetes.yml ~/src/PlatformAppsForge/platform/keycloak-operator/operator.yaml
```

Then bump the `Keycloak` CR's `spec.image: quay.io/keycloak/keycloak:<TAG>` in `platform/keycloak/instance.yaml`.
