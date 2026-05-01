# PlatformAppsForge

Shared **app-layer** services that sit on top of [PlatformForge](https://github.com/brianbeck/PlatformForge). Managed by Argo CD,
organized with the same conventions as PlatformForge (`platform/<svc>/...` +
sync-wave ApplicationSets in `argocd/waves-<env>/`).

PlatformForge provides cluster infrastructure (ingress, observability, policy, secrets controller). PlatformAppsForge provides reusable application-layer services (databases, identity, secret management) that any app on the cluster can consume. Individual applications then have their own pair of repos (e.g. `conecta-life` + `conecta-life-deploy`).

## What's here

| Service | Purpose | Wave |
|---|---|---|
| `cloudnative-pg` | PostgreSQL operator (HA, PITR, declarative `Cluster` CRDs) for any app that needs a managed DB | 05 |

## What's not here (yet)

- **Keycloak** — currently deployed by DevExForge at `keycloak-{env}.brianbeck.net`. A future task promotes it to a shared service here.
- **Vault / OpenBao** — scheduled before vivavida.life handles real patient data. Will host transit encryption keys and dynamic DB credentials.
- **External Secrets Operator** — already an optional service in PlatformForge; enable it from there rather than deploying a second copy here.

## Layout

```
PlatformAppsForge/
├── platform/<service>/
│   ├── base-values.yaml
│   └── overlays/{stage,prod}/values.yaml
└── argocd/waves-{stage,prod}/<wave>-<domain>/<service>.yml
```

## Applying

```bash
# Stage
kubectl --context beck-stage-admin@beck-stage \
  apply -f argocd/waves-stage/05-operators/cloudnative-pg.yml

# Prod
kubectl --context beck-prod-admin@beck-prod \
  apply -f argocd/waves-prod/05-operators/cloudnative-pg.yml
```

The ApplicationSets reference `https://github.com/brianbeck/PlatformAppsForge.git`. Push this repo to GitHub before applying, or rewrite the `repoURL` to a local path for offline testing.

## Public/private split — not yet adopted

PlatformForge uses a `PlatformForge` (public) + `platformforge-env` (private) split to keep cluster-specific config out of the public repo. PlatformAppsForge does **not** need this split today — its only contents are operator base values with no env-specific secrets. Revisit when Vault/OpenBao and Keycloak land here and bring real env config; at that point introduce `platformappsforge-env`.
