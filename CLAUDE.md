# CLAUDE.md — PlatformAppsForge

Orientation file for Claude Code sessions in this repo.

## What this is

App-layer shared services that sit on top of [PlatformForge](https://github.com/brianbeck/PlatformForge). Reusable building blocks for any application running on the cluster — databases, identity, secret management — installed once and consumed by many.

## What's here

| Service | Purpose | Wave |
|---|---|---|
| `cloudnative-pg` | PostgreSQL operator (HA, PITR, declarative `Cluster` CRDs) | 05 |

## Layout convention

Same as PlatformForge:

```
PlatformAppsForge/
├── platform/<service>/
│   ├── base-values.yaml
│   └── overlays/{stage,prod}/values.yaml
└── argocd/waves-{stage,prod}/<wave>-<domain>/<service>.yml
```

ApplicationSets are checked in to this repo. They use `repoURL: https://github.com/brianbeck/PlatformAppsForge.git`.

## Sync waves used here

- **05** — Operators (CloudNativePG today; CNPG and similar must precede any Cluster CR).
- **15 (planned)** — Identity (Keycloak, when it migrates from DevExForge).
- **25 (planned)** — Secret management (Vault/OpenBao).

## Applying

```bash
kubectl --context beck-stage-admin@beck-stage \
  apply -f argocd/waves-stage/05-operators/cloudnative-pg.yml

kubectl --context beck-prod-admin@beck-prod \
  apply -f argocd/waves-prod/05-operators/cloudnative-pg.yml
```

## Public/private split — not adopted yet

PlatformForge uses `PlatformForge` (public) + `platformforge-env` (private) to keep cluster-specific config out of the public repo. PlatformAppsForge does **not** have this split today because there are no env-specific secrets here yet.

**Adopt the split when** Vault/OpenBao or Keycloak land here, since they bring real env config (Vault unseal keys, Keycloak DB connection strings, OIDC discovery URLs) that should not be public. At that time:

1. Create `platformappsforge-env` (private).
2. Move `platform/<svc>/overlays/<env>/values.yaml` and any future SealedSecrets there.
3. Update each ApplicationSet's `$values` source to point at the private repo.

## TODO

1. Push to GitHub as `brianbeck/PlatformAppsForge`. Confirm Argo CD has the appropriate repo credentials (PAT, if private).
2. Apply the CloudNativePG ApplicationSet to stage; verify `cnpg-system` namespace and operator pod ready.
3. Repeat for prod once stage is green.
4. **Future:** add Keycloak under `platform/keycloak/...` (wave 15). Migrate the `teams` realm from DevExForge.
5. **Future:** add Vault/OpenBao under `platform/vault/...` (wave 25). Adopt the public/private split at that point.
