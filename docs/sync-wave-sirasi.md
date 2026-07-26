# Argo CD Sync Wave Order

> Quick reference only. The source of truth is the
> `argocd.argoproj.io/sync-wave` annotation in each manifest.

## Order

| Wave | Resources | Status |
| ---: | --- | --- |
| `0` | `dev`, `openbao`, and `external-secrets` namespaces; AppProjects | Current |
| `10` | `platform-openbao-storage`; `apps` ApplicationSet | Current |
| `20` | `platform-openbao` | Current |
| `30` | `platform-external-secrets` | Current |
| `40` | `platform-secretstores` / `ClusterSecretStore` | Current |
| `50` | `platform-external-secrets-resources` / `ExternalSecret` resources | Current |
| `60` | `platform-local-storage` / shared StorageClass and Redis PV | Planned in Phase 6 |
| `70` | `platform-redis` / Redis Helm chart | Planned in Phase 6 |

```text
Namespaces and AppProjects
  → OpenBao storage
  → OpenBao
  → External Secrets Operator
  → ClusterSecretStore
  → ExternalSecret resources
  → Redis storage
  → Redis
```

## Rules

- Lower wave numbers sync first.
- Resources in the same wave must not depend on one another.
- A sync wave controls apply order; it does not prove that a workload is ready.
- Before using Redis, verify `ClusterSecretStore/openbao` is `Ready=True` and
  `ExternalSecret/redis-auth` is `SecretSynced=True`.
- Keep increments of ten for future dependencies.
- Waves `60` and `70` do not exist in the live cluster until Phase 6 is applied.

## Check the repository

```bash
rg -n -C 2 'argocd.argoproj.io/sync-wave' argocd
```
