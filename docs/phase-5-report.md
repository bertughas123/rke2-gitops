# Phase 5 Report - OpenBao and External Secrets Operator

Status: completed.

Date recorded: 2026-07-25

## Result

Phase 5 completed the local secret-management path for the RKE2 GitOps lab.

The phase introduced:

- persistent local storage for OpenBao;
- a standalone OpenBao deployment;
- KV v2 and Kubernetes authentication for External Secrets Operator;
- External Secrets Operator and its CRDs;
- a namespace-restricted `ClusterSecretStore`;
- a harmless test `ExternalSecret`;
- Argo CD repository credential management through OpenBao and ESO;
- a controlled read-only GitHub Deploy Key handover and rotation.

The resulting flow is:

```text
ExternalSecret
      |
      v
ClusterSecretStore
      |
      v
External Secrets Operator
      |
      v
OpenBao KV v2
      |
      v
Kubernetes Secret
      |
      v
Argo CD or an application
```

Secret values are not stored in Git. Git contains only the declarative
resources, pinned component versions, authentication references and remote
secret paths required to reproduce the integration.

## Component Versions

| Component | Chart version | Image or application version |
| --- | --- | --- |
| RKE2 | Not installed by this phase | `v1.36.2+rke2r1` |
| Argo CD | Not installed by this phase | `v3.4.5` |
| OpenBao | `0.28.5` | `2.6.0` |
| External Secrets Operator | `2.8.0` | `2.8.0` |

The OpenBao and ESO versions were pinned instead of following an unbounded
latest version. Their required container images were checked for Linux ARM64
support before the deployment path was accepted.

## Platform Ownership Model

Phase 5 changed the platform ownership model prepared during Phase 4.

The application layer continues to use:

```text
argocd/appsets/apps-appset.yaml
```

The `apps` ApplicationSet discovers directories under:

```text
apps-values/dev/*
```

Platform components no longer use the generic platform ApplicationSets. The
following files were removed:

```text
argocd/appsets/platform-helm-appset.yaml
argocd/appsets/platform-manifests-appset.yaml
```

OpenBao, ESO, storage and secret resources have different dependency and
lifecycle requirements. They are therefore managed through explicit Argo CD
Applications:

| Application | Purpose | Sync wave |
| --- | --- | --- |
| `platform-openbao-storage` | StorageClass and local PersistentVolume | `10` |
| `platform-openbao` | OpenBao Helm release | `20` |
| `platform-external-secrets` | ESO Helm release and CRDs | `30` |
| `platform-secretstores` | OpenBao ClusterSecretStore | `40` |
| `platform-external-secrets-resources` | Test and repository ExternalSecrets | `50` |

All five Applications:

- exist in the `argocd` namespace;
- use the `platform` AppProject;
- enable automated prune and self-heal;
- use the `Prune=confirm` sync option;
- do not use the Argo CD resources finalizer;
- do not use `CreateNamespace=true`.

Only `platform-external-secrets` uses:

```text
ServerSideApply=true
```

OpenBao and the raw-manifest Applications use normal apply behavior.

## Namespaces

Phase 5 added Git-managed namespace manifests for:

```text
openbao
external-secrets
```

Both namespace resources use sync wave `0` and `Prune=confirm`.

Namespace creation remains the responsibility of `bootstrap-root`; the child
platform Applications do not create their destination namespaces implicitly.

## OpenBao Local Storage

The RKE2 VM contains the local storage directory:

```text
/var/lib/rancher/rke2/storage/openbao
```

The prepared directory ownership and mode are:

```text
UID: 100
GID: 1000
mode: 0700
```

Git manages the storage resources through:

```text
platform-manifests/openbao-storage/storageclass.yaml
platform-manifests/openbao-storage/persistentvolume.yaml
```

Storage configuration:

| Item | Value |
| --- | --- |
| StorageClass | `openbao-local` |
| Provisioner | `kubernetes.io/no-provisioner` |
| Volume binding mode | `WaitForFirstConsumer` |
| PersistentVolume | `openbao-data` |
| Capacity | `5Gi` |
| Access mode | `ReadWriteOnce` |
| Reclaim policy | `Retain` |
| Local path | `/var/lib/rancher/rke2/storage/openbao` |
| Node affinity | `rke2-server` |

The PersistentVolume also carries `Prune=confirm`.

This storage protects OpenBao data across Pod replacement and normal restarts.
It does not provide production-grade replication, node failover or protection
against deletion of the Multipass VM.

## OpenBao Deployment

OpenBao is deployed from the official Helm repository:

```text
https://openbao.github.io/openbao-helm
```

The Argo CD Application uses a multi-source model:

```text
OpenBao chart 0.28.5
        +
platform-values/openbao/values.yaml
        |
        v
openbao namespace
```

The final OpenBao profile is intentionally small and local:

- standalone mode enabled;
- HA mode disabled;
- development mode disabled;
- public Ingress disabled;
- injector disabled;
- CSI provider disabled;
- UI exposed only through a `ClusterIP` Service;
- file storage under `/openbao/data`;
- `5Gi` `openbao-local` persistent storage;
- audit storage disabled for the initial lab;
- TLS disabled only inside this isolated local lab.

Pinned image:

```text
quay.io/openbao/openbao:2.6.0
```

Resource profile:

| Resource | Request | Limit |
| --- | --- | --- |
| CPU | `100m` | `500m` |
| Memory | `256Mi` | `512Mi` |

Security settings include:

- `runAsNonRoot: true`;
- `runAsUser: 100`;
- `runAsGroup: 1000`;
- `fsGroup: 1000`;
- `seccompProfile: RuntimeDefault`;
- `allowPrivilegeEscalation: false`;
- all Linux capabilities dropped.

The liveness probe remains disabled in the initial profile so that an
initialized but sealed OpenBao instance does not enter a restart loop. The
readiness probe represents whether the service is ready to answer client
requests.

## Initialize, Unseal and Persistence Boundary

OpenBao was initialized once and unsealed before the KV and authentication
configuration was created.

The initialization output was treated as sensitive. Root tokens and
unseal/recovery material were stored outside Git and were not copied into
documentation or command output used by this report.

This lab does not configure auto-unseal.

After an OpenBao Pod or VM restart:

- persisted data remains on the local volume;
- OpenBao may return in a sealed state;
- data cannot be read until OpenBao is unsealed;
- ESO cannot refresh values while OpenBao is sealed;
- existing Kubernetes Secrets normally remain present during a temporary
  provider outage.

`deletionPolicy: Retain` is not a backup system. It prevents automatic deletion
of a target Kubernetes Secret for the relevant provider-side deletion case, but
it does not restore lost OpenBao data.

RKE2 etcd snapshots protect Kubernetes and control-plane state. They do not back
up the OpenBao data stored inside the PersistentVolume.

## OpenBao Authentication Model

The following OpenBao configuration was prepared outside Git:

- a KV v2 mount at `secret/`;
- a minimum read/list policy for the paths needed by this phase;
- the Kubernetes authentication method at `kubernetes`;
- an authentication role named `external-secrets`;
- a role binding restricted to the `external-secrets` ServiceAccount in the
  `external-secrets` namespace;
- the audience value `vault`.

OpenBao performs the Kubernetes TokenReview operation through its own
`openbao` ServiceAccount:

```text
server.authDelegator.enabled: true
```

ESO does not receive `system:auth-delegator`:

```text
systemAuthDelegator: false
```

ESO instead requires permission to request a short-lived token for the
referenced ServiceAccount. The rendered ESO RBAC includes the required
`serviceaccounts/token` create permission.

## External Secrets Operator

ESO is deployed from:

```text
https://charts.external-secrets.io
```

Pinned image:

```text
ghcr.io/external-secrets/external-secrets:v2.8.0
```

The Helm release installs:

- the ESO controller;
- the webhook;
- the certificate controller;
- the required External Secrets CRDs.

The deployment uses one replica for each controller component and a
resource-constrained local profile.

Controller resources:

| Resource | Request | Limit |
| --- | --- | --- |
| CPU | `50m` | `250m` |
| Memory | `128Mi` | `256Mi` |

Webhook and certificate controller resources:

| Resource | Request | Limit |
| --- | --- | --- |
| CPU | `25m` | `100m` |
| Memory | `64Mi` | `128Mi` |

The required CRDs were installed before the `ClusterSecretStore` and
`ExternalSecret` resources were introduced. This preserved the dependency order
instead of relying only on sync-wave numbers.

## ClusterSecretStore

The shared store is:

```text
ClusterSecretStore/openbao
```

Provider configuration:

| Item | Value |
| --- | --- |
| ESO API version | `external-secrets.io/v1` |
| Provider type | `vault` |
| OpenBao address | `http://openbao.openbao.svc.cluster.local:8200` |
| KV path | `secret` |
| KV version | `v2` |
| Kubernetes auth mount | `kubernetes` |
| Auth role | `external-secrets` |
| ServiceAccount | `external-secrets/external-secrets` |
| Audience | `vault` |
| Refresh interval | `1m` |

ESO uses the Vault-compatible provider schema because OpenBao exposes the
compatible API used by this integration.

The store is restricted with namespace conditions. It may only be referenced
from:

```text
external-secrets
argocd
```

This prevents arbitrary namespaces from using the cluster-scoped store.

## Harmless Test ExternalSecret

The first end-to-end validation used:

```text
ExternalSecret/external-secrets/phase5-test
```

Contract:

| Item | Value |
| --- | --- |
| Remote key | `phase5/test` |
| Remote property | `value` |
| Target Secret | `phase5-test` |
| Target key | `message` |
| Refresh policy | `Periodic` |
| Refresh interval | `1m` |
| Creation policy | `Owner` |
| Deletion policy | `Retain` |

The test reached `SecretSynced`.

The generated Kubernetes Secret value was not printed. The expected and actual
values were compared through SHA-256 hashes, and the completion result was:

```text
MATCH
```

This verified more than the existence of a Secret: it verified that ESO
retrieved the intended property without exposing its value in the report.

## Argo CD Repository Credential Handover

The second ExternalSecret manages the Argo CD repository credential:

```text
ExternalSecret/argocd/argocd-repository
```

Remote reference:

| Item | Value |
| --- | --- |
| OpenBao key | `gitops/argocd/repository` |
| OpenBao property | `sshPrivateKey` |

Target:

| Item | Value |
| --- | --- |
| Kubernetes Secret | `repo-rke2-gitops` |
| Namespace | `argocd` |
| Argo CD secret type | `repository` |
| Repository type | `git` |
| Repository URL | `git@github.com:bertughas123/rke2-gitops.git` |
| Creation policy | `Orphan` |
| Deletion policy | `Retain` |
| Refresh interval | `1m` |

The target carries the Argo CD repository label:

```text
argocd.argoproj.io/secret-type: repository
```

The handover preserved the stable Secret name used by Argo CD. Before the
manual bootstrap Secret was removed:

- a local recovery private key was identified without printing its contents;
- its derived public-key fingerprint was checked against the read-only GitHub
  Deploy Key;
- read-only repository access was verified;
- the private key existed in OpenBao without being printed;
- the repository ExternalSecret reached `SecretSynced`;
- the Argo CD repository connection remained successful.

After those checks, the bootstrap Secret was removed in the controlled handover
step. ESO recreated the same target Secret contract, and Argo CD continued to
read the GitOps repository.

`creationPolicy: Orphan` prevents deletion of the ExternalSecret resource from
automatically garbage-collecting the repository credential Secret. It does not
replace recovery-key verification or credential rotation procedures.

## Read-Only Deploy Key Rotation

The repository-specific read-only Deploy Key was rotated in a controlled order:

1. A new key pair was prepared.
2. Only the new public key was added to the GitHub repository.
3. Write access remained disabled.
4. The new private key value was updated in OpenBao without being printed.
5. ESO refreshed the target Argo CD repository Secret.
6. Argo CD repository access and root synchronization were verified.
7. The old key remained available during the rollback window.
8. The old Deploy Key was revoked only after the new credential worked.

No private key content or fingerprint is included in this report.

## Static Validation At Report Time

The repository configuration was revalidated while this report was written.

OpenBao chart validation:

- official chart `0.28.5` rendered successfully with the committed values;
- image `quay.io/openbao/openbao:2.6.0` was present;
- `openbao-local` storage settings were present;
- non-root security settings were present;
- no OpenBao Ingress, injector or CSI workload was rendered.

ESO chart validation:

- official chart `2.8.0` rendered successfully with the committed values;
- image `ghcr.io/external-secrets/external-secrets:v2.8.0` was present;
- the required ExternalSecret, SecretStore and ClusterSecretStore CRDs were
  present;
- `serviceaccounts/token` RBAC was present.

All committed Phase 5 Application, values and raw manifest YAML files parsed
successfully.

## Git History

Phase 5 was introduced through small dependency-oriented commits:

| Commit | Purpose |
| --- | --- |
| `79a5c72` | Prepare Phase 5 namespaces |
| `142ac25` | Remove the platform ApplicationSets |
| `ef2ea5e` | Add OpenBao local storage resources |
| `b47a584` | Add OpenBao Helm deployment configuration |
| `db434c8` | Add External Secrets Operator |
| `baa0c2c` | Add the OpenBao ClusterSecretStore |
| `7c91536` | Add the harmless test ExternalSecret |
| `3b0af6d` | Reduce secret refresh intervals to one minute |
| `8d6e40f` | Add the Argo CD repository credential ExternalSecret |

Each Git step followed the dependency order so that storage, OpenBao, ESO,
CRDs, the store and ExternalSecrets could be reviewed independently.

## Security Review

The Phase 5 repository files contain:

- chart and image versions;
- non-sensitive Helm values;
- storage configuration;
- OpenBao connection metadata;
- authentication role and ServiceAccount references;
- remote secret paths and property names;
- ExternalSecret target metadata.

They do not contain:

- OpenBao root tokens;
- unseal or recovery keys;
- SSH private key values;
- GitHub tokens;
- kubeconfig content;
- generated Kubernetes Secret values;
- base64-encoded secret payloads.

The repository credential is read-only. Jenkins will use a separate write
credential in a later phase and will not reuse the Argo CD Deploy Key.

## Current Revalidation Note

At report-writing time, Multipass reported the VM as running:

```text
VM: rke2-server
State: Running
IPv4: 192.168.252.2
```

A fresh read-only Kubernetes API check from the macOS host could not be
completed because the host route to the VM returned:

```text
Unable to connect to the server: dial tcp 192.168.252.2:6443: connect: no route to host
```

The same network path also prevented `multipass exec` from opening an SSH
session. This is a host-to-Multipass networking condition, not evidence of an
OpenBao, ESO or manifest failure.

The completion results in this report record the Phase 5 execution state. Chart
rendering and repository YAML were freshly revalidated, but live cluster state
should be refreshed after the Multipass route is restored.

Required read-only refresh checks are:

- RKE2 node `Ready`;
- `bootstrap-root` `Synced/Healthy`;
- all five platform Applications synchronized;
- OpenBao Pod ready and PVC bound;
- ESO controller, webhook and certificate controller available;
- required CRDs `Established`;
- `ClusterSecretStore/openbao` ready;
- both ExternalSecrets `SecretSynced`;
- Argo CD repository connection successful.

No Phase 5 resource should be recreated solely because this route check failed.

## Known Limitations

- OpenBao is standalone and is not highly available.
- Auto-unseal is not configured.
- TLS is disabled only for the internal local-lab OpenBao listener.
- OpenBao has no public Ingress.
- Local PV data is tied to `rke2-server`.
- The local PV is not a production backup.
- RKE2 etcd snapshots do not back up OpenBao application data.
- A second RKE2 agent would not make this OpenBao storage highly available.
- Secret retention policies do not restore deleted provider data.
- Multipass IP or host routing may change after host or VM lifecycle events.

## Phase 5 Boundary

Phase 5 did not introduce:

- Redis;
- CloudNativePG;
- PostgreSQL;
- Jenkins;
- Oracle Cloud infrastructure;
- a second RKE2 agent;
- stage or production environments;
- Keycloak;
- Prometheus or Grafana;
- production backup or auto-unseal.

Those concerns belong to later phases.

## Phase 5 Completion Summary

Phase 5 completed:

- explicit Argo CD Application ownership for platform components;
- local persistent storage for OpenBao;
- a pinned and resource-constrained standalone OpenBao deployment;
- secure initialize/unseal handling outside Git;
- KV v2 and minimum-scope Kubernetes authentication;
- a pinned ESO deployment and its CRDs;
- namespace-restricted access through `ClusterSecretStore/openbao`;
- a harmless end-to-end ExternalSecret test;
- secret-value verification without plaintext output;
- Argo CD repository credential handover to OpenBao and ESO;
- read-only Deploy Key rotation;
- retention and recovery boundaries for the local lab.

The next platform phase is Phase 6: deploy a maintained, ARM64-compatible Redis
Open Source chart from a public Helm source, consume authentication through
OpenBao and ESO, and record resource usage before starting CloudNativePG.
