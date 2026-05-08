# Shared Keycloak

The shared Keycloak instance for the cluster. Operator-managed (Quay's official Keycloak Operator, vendored in `../keycloak-operator/`).

## Files

| File | Purpose |
|---|---|
| `db-cluster.yaml` | CloudNativePG `Cluster` named `keycloak-db` (database for Keycloak) |
| `instance.yaml` | `Keycloak` CR — server config, hostname, DB ref, image pin |
| `ingress.yaml` | Traefik `Ingress` with cert-manager TLS for `auth-stage.vivavida.life` |
| `kustomization.yaml` | Base aggregator |
| `overlays/{stage,prod}/kustomization.yaml` | Per-env patches (prod patches commented until prod migration) |

## Required SealedSecrets

The user authors these per-cluster (sealed-secrets keys are cluster-bound). All in the `keycloak` namespace. Files land in `overlays/<env>/secrets/<name>.yaml` and must be added to the `resources:` list in `overlays/<env>/secrets/kustomization.yaml` for Argo CD to pick them up.

### `keycloak-db-credentials`
Postgres credentials for the CNPG-managed `keycloak-db` cluster.

```bash
kubectl create secret generic keycloak-db-credentials \
  --namespace keycloak \
  --from-literal=username=keycloak \
  --from-literal=password="$(openssl rand -base64 24)" \
  --dry-run=client -o yaml \
| kubeseal --controller-name=sealed-secrets-controller \
           --controller-namespace=sealed-secrets \
           --context beck-stage-admin@beck-stage \
           --format yaml \
> /Users/beck/src/PlatformAppsForge/platform/keycloak/overlays/stage/secrets/keycloak-db-credentials.yaml
```

### `keycloak-client-secrets`
Plaintext client secrets for each consuming app's OIDC client. Mounted as env vars on the Keycloak pod via `envFrom: secretRef: keycloak-client-secrets`. The realm import CRs use `${env:VAR}` substitution to reference them.

Keys must match the `${env:VAR}` references in the realm CRs:

| Key | Used by realm CR | Mirror sealed-secret in app namespace |
|---|---|---|
| `CONECTA_CLIENT_SECRET` | `conecta-life-deploy/environments/<env>/realms/conecta.yaml` | `conecta` ns: `conecta-oidc/client-secret` |
| `DEVEXFORGE_PORTAL_CLIENT_SECRET` | (only if devexforge-portal stops being public) | (n/a today) |

For the conecta value, use **the same plaintext** that's already sealed in `conecta-life-deploy/environments/stage/secrets/conecta-oidc.yaml`. Two sealed secrets, one plaintext value, sealed twice for two namespaces. (Future improvement: External Secrets Operator or Reflector to mirror cross-namespace from a single source.)

```bash
kubectl create secret generic keycloak-client-secrets \
  --namespace keycloak \
  --from-literal=CONECTA_CLIENT_SECRET='<same value as conecta-oidc/client-secret>' \
  --dry-run=client -o yaml \
| kubeseal --controller-name=sealed-secrets-controller \
           --controller-namespace=sealed-secrets \
           --context beck-stage-admin@beck-stage \
           --format yaml \
> /Users/beck/src/PlatformAppsForge/platform/keycloak/overlays/stage/secrets/keycloak-client-secrets.yaml
```

### Initial admin credentials (Keycloak admin console)
The Keycloak Operator auto-creates an `admin` user with a generated password the first time the instance starts. Retrieve it via:

```bash
kubectl --context beck-stage-admin@beck-stage \
  -n keycloak get secret keycloak-initial-admin -o jsonpath='{.data.password}' | base64 -d
```

Treat this as a break-glass credential — the day-to-day realm config flows through `KeycloakRealmImport` CRs, not the admin console.

## What this does NOT contain

Realm configuration. Each consuming app owns its realm declaratively in its own deploy repo:

- `~/src/DevExForge-deploy/environments/<env>/realms/teams.yaml` — DevExForge's `teams` realm
- `~/src/conecta-life-deploy/environments/<env>/realms/conecta.yaml` — Conecta's `conecta` realm

Each is an independent ArgoCD `Application` with `argocd.argoproj.io/sync-wave: "20"`, applied after the Keycloak instance (wave 15) is Healthy.
