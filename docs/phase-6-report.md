# Phase 6 Implementation Report

## 1. Summary

Phase 6 deployed Redis Open Source through Argo CD by using a public
third-party Helm chart and a GitOps workflow.

The implementation:

- pinned the CloudPirates `redis` chart to version `0.32.4`;
- pinned the Redis Docker Official Image `8.8.0` by digest;
- ran Redis in standalone mode as a single-replica StatefulSet;
- obtained the Redis password through OpenBao and External Secrets Operator
  without storing the real value in Git;
- used a static local PV and the shared `StorageClass/local-storage`;
- verified Service DNS, required authentication, and persistence after a pod
  recreation;
- measured resource and disk usage before and after the installation.

The final live result was successful:

- `Application/platform-redis` became `Synced` and `Healthy`;
- `Pod/redis-0` became `Running` and `Ready`;
- the Redis PVC and `PersistentVolume/redis-data` became `Bound`.

The secret flow is:

```text
OpenBao
  → ClusterSecretStore/openbao
  → External Secrets Operator
  → ExternalSecret/redis-auth
  → Secret/redis-auth
  → Pod/redis-0
```

The storage flow is:

```text
Redis /data
  → Redis PVC
  → PersistentVolume/redis-data
  → /var/lib/rancher/rke2/storage/redis
  → rke2-server VM disk
```

## 2. Phase 5 Health Gate

Redis depends on the OpenBao and ESO secret infrastructure created in Phase 5.
The live health gate was therefore checked before installing Redis.

Verified state:

| Check | Result |
| --- | --- |
| `Namespace/redis` | `Active` |
| `ClusterSecretStore/openbao` | `Ready=True` |
| OpenBao sealed state | `sealed=false` |
| ESO controllers and existing ExternalSecret flow | Healthy |

The allowed namespaces in `ClusterSecretStore/openbao` were:

- `external-secrets`;
- `argocd`;
- `redis`.

This confirmed that an `ExternalSecret` in the `redis` namespace was allowed
to use the existing cluster-scoped Store.

Redis installation did not proceed until OpenBao was unsealed, the Store was
ready, and the existing ESO chain was healthy. This avoided starting Redis
without its required authentication Secret.

## 3. Chart, Image, License, and ARM64 Verification

The selected chart and image:

| Field | Value |
| --- | --- |
| Chart provider | CloudPirates |
| Chart | `redis` |
| OCI address | `oci://registry-1.docker.io/cloudpirates/redis` |
| Chart version | `0.32.4` |
| Chart license | Apache-2.0 |
| Container image | `docker.io/redis:8.8.0` |
| Image digest | `sha256:234c902a2db49461a129e2d4aeff85b28cf20187ed274a67f6e50995fa713c7b` |
| Target architecture | ARM64 |

The CloudPirates chart is a third-party Helm chart; it is not an official Redis
chart. The `docker.io/redis` image used by the chart is a Docker Official
Image. The chart and runtime image therefore have different maintainers.

The chart version and container image version also represent different
artifacts:

- chart version `0.32.4` identifies the Helm package that renders Kubernetes
  resources;
- image version `8.8.0` identifies the Redis software running in the
  container;
- the image digest prevents a different runtime artifact from being pulled
  under the same tag.

The chart source is licensed under Apache-2.0. Redis 8 Open Source is published
under the AGPLv3, RSALv2, and SSPLv1 license options. This local learning
environment used Redis Open Source.

ARM64 support was verified in the image manifest. This check confirmed that
the runtime image, not only the Helm templates, was compatible with the
`rke2-server` architecture.

## 4. Namespace and ClusterSecretStore Changes

`Namespace/redis` was created for the Redis workload. Its GitOps contract
contains:

- sync wave `0`;
- `Prune=confirm`;
- no `CreateNamespace=true` option in the Redis Helm Application.

The existing `ClusterSecretStore/openbao` was not deleted or recreated.
The `redis` namespace was added to its allowed namespace list while preserving
the existing `external-secrets` and `argocd` entries.

The Store connection contract remained unchanged:

- OpenBao service:
  `http://openbao.openbao.svc.cluster.local:8200`;
- KV mount: `secret`;
- KV version: `v2`;
- Kubernetes auth mount: `kubernetes`;
- auth role: `external-secrets`;
- ServiceAccount: `external-secrets/external-secrets`;
- audience: `vault`.

`ExternalSecret/redis-auth` does not repeat these connection details. It only
references the existing Store through `secretStoreRef.name: openbao`.

## 5. Storage Architecture

The shared static `StorageClass/local-storage` was created for Redis and future
PostgreSQL workloads.

StorageClass contract:

| Field | Value |
| --- | --- |
| Provisioner | `kubernetes.io/no-provisioner` |
| Binding mode | `WaitForFirstConsumer` |
| Reclaim policy | `Retain` |

`kubernetes.io/no-provisioner` does not automatically create a disk,
directory, PV, or PVC. The VM directory was prepared manually, while the
static PV was declared in the GitOps repository.

Redis PV contract:

| Field | Value |
| --- | --- |
| PersistentVolume | `redis-data` |
| Capacity declaration | `2Gi` |
| Access mode | `ReadWriteOnce` |
| Reclaim policy | `Retain` |
| StorageClass | `local-storage` |
| Local path | `/var/lib/rancher/rke2/storage/redis` |
| Node affinity | `kubernetes.io/hostname=rke2-server` |

The VM directory was prepared with:

| Field | Value |
| --- | --- |
| UID | `999` |
| GID | `999` |
| Mode | `700` |

The Redis PV and PVC became `Bound`. `WaitForFirstConsumer` allowed the binding
decision to be made together with the Redis pod scheduling constraints.

`capacity: 2Gi` does not create a new physical disk or partition. Redis data is
written to the existing Multipass VM filesystem under
`/var/lib/rancher/rke2/storage/redis`. Because no filesystem quota was
configured for this directory, `2Gi` may not be a strict physical disk usage
limit. It is primarily a Kubernetes capacity and matching declaration.

The planned storage separation is:

```text
rke2-server VM disk
└── /var/lib/rancher/rke2/storage/
    ├── openbao/
    ├── redis/
    └── postgresql/

StorageClass/openbao-local
└── Existing OpenBao PV/PVC

StorageClass/local-storage
├── PV/redis-data
│   └── /var/lib/rancher/rke2/storage/redis
└── Separate PostgreSQL PV/PVC
    └── /var/lib/rancher/rke2/storage/postgresql
```

Redis and PostgreSQL will share the StorageClass policy but will not share a
PV, PVC, or VM directory. OpenBao remained on its existing `openbao-local`
StorageClass, and no OpenBao storage migration was performed.

## 6. OpenBao Policy and Redis Password

A new least-privilege policy was created for the Redis secret:

```text
external-secrets-redis
```

The policy granted `read` access only to these KV v2 paths:

```text
secret/data/platform/redis/auth
secret/metadata/platform/redis/auth
```

The existing Kubernetes auth role continued to be used:

```text
auth/kubernetes/role/external-secrets
```

Preserved role settings:

| Field | Value |
| --- | --- |
| `bound_service_account_names` | `external-secrets` |
| `bound_service_account_namespaces` | `external-secrets` |
| `audience` | `vault` |
| `ttl` | `1h` |

The final role policy list was:

- `eso-read`;
- `external-secrets-redis`.

The Redis password was stored in OpenBao KV v2 with this contract:

| Field | Value |
| --- | --- |
| Mount | `secret` |
| Logical path | `platform/redis/auth` |
| Property | `password` |

The real password was not written to the report, Git, YAML, or an example
terminal output.

The first two write attempts produced empty secret versions because `openssl`
was not available inside the OpenBao container. Versions `1` and `2` were not
accepted as valid secrets. A strong password was then generated with `openssl`
on the Mac and passed to OpenBao through stdin without printing it. This
created valid version `3`. The incorrect versions `1` and `2` were destroyed.

This left only the valid version available and prevented either empty version
from being restored accidentally.

## 7. ExternalSecret and Kubernetes Secret Flow

The `ExternalSecret` contract:

| Field | Value |
| --- | --- |
| ExternalSecret | `redis-auth` |
| Namespace | `redis` |
| ClusterSecretStore | `openbao` |
| Remote key | `platform/redis/auth` |
| Remote property | `password` |
| Target Secret | `redis-auth` |
| Target key | `redis-password` |
| `creationPolicy` | `Owner` |
| `deletionPolicy` | `Retain` |
| Refresh policy | `Periodic`, `1m` |

Field responsibilities:

- `secretStoreRef.name: openbao` selects the existing
  `ClusterSecretStore/openbao`;
- `remoteRef.key` identifies the logical OpenBao secret path;
- `remoteRef.property` identifies the field read from that record;
- `secretKey` defines the key created in the Kubernetes Secret;
- `creationPolicy: Owner` allows ESO to create and manage the target Secret;
- `deletionPolicy: Retain` prevents automatic deletion of the existing
  Kubernetes Secret if the provider-side value disappears.

Live result:

- `ExternalSecret/redis-auth` became `Ready=True`;
- `Secret/redis-auth` was created successfully;
- the Redis pod consumed the auth value from the `redis-password` key.

The complete flow was:

```text
OpenBao platform/redis/auth:password
  → ClusterSecretStore/openbao
  → ESO controller
  → ExternalSecret/redis-auth
  → Secret/redis-auth:redis-password
  → Pod/redis-0
```

`deletionPolicy: Retain` is not a backup mechanism and cannot restore a value
lost from OpenBao.

## 8. Redis Helm Values and Render Test

The Redis values contract configured:

- `architecture: standalone`;
- a single replica and StatefulSet model;
- `fullnameOverride: redis`;
- authentication enabled;
- existing `Secret/redis-auth` and key `redis-password`;
- persistence enabled;
- `StorageClass/local-storage`;
- a `2Gi` PVC request;
- `/data` as the Redis data path;
- AOF persistence;
- `appendfsync everysec`;
- small local-lab resource requests and limits;
- enabled liveness and readiness probes;
- disabled TLS, metrics, and NetworkPolicy.

Resource profile:

| Type | CPU | Memory |
| --- | ---: | ---: |
| Request | `50m` | `128Mi` |
| Limit | `250m` | `256Mi` |

Helm lint result:

```text
1 chart(s) linted, 0 chart(s) failed
```

The render produced and verified:

- a ServiceAccount;
- a client Service;
- a headless Service for StatefulSet discovery;
- a StatefulSet;
- a PersistentVolumeClaim.

Critical rendered values:

- `docker.io/redis:8.8.0` with the pinned image digest;
- Secret name `redis-auth`;
- Secret key `redis-password`;
- StorageClass `local-storage`;
- PVC request `2Gi`;
- `runAsUser: 999`;
- `runAsGroup: 999`;
- `runAsNonRoot: true`;
- `allowPrivilegeEscalation: false`;
- `readOnlyRootFilesystem: true`;
- `RuntimeDefault` seccomp;
- all Linux capabilities dropped.

AOF settings ensured that Redis wrote persistent data under `/data`:

```text
dir /data
appendonly yes
appendfsync everysec
```

No literal password appeared in the Helm render or Git.

## 9. Argo CD Redis Installation

Two explicit Argo CD Applications were used:

| Application | Wave | Responsibility |
| --- | ---: | --- |
| `platform-local-storage` | `60` | Shared StorageClass and Redis static PV |
| `platform-redis` | `70` | Redis Helm release |

`platform-local-storage` recursively read
`platform-manifests/local-storage`. Because its StorageClass and PV are
cluster-scoped resources, the Application destination namespace was set to
`default`.

`platform-redis` used the Argo CD multi-source model:

1. CloudPirates `redis` chart `0.32.4` came from the OCI registry.
2. `platform-values/redis/values.yaml` came from the GitOps repository's
   `main` branch through the `$values` reference.
3. Argo CD combined both sources and deployed the result to the `redis`
   namespace.

Both Applications preserved:

- `project: platform`;
- automated prune;
- self-heal;
- `Prune=confirm`;
- no Application finalizer.

Live result:

- `Application/platform-local-storage` reconciled successfully;
- `Application/platform-redis` became `Synced` and `Healthy`;
- `StatefulSet/redis` ran one replica;
- `Pod/redis-0` became `Running` and `Ready`;
- the Redis PVC and `PV/redis-data` became `Bound`.

## 10. DNS and Authentication Test

A temporary `redis-cli-test` Pod was created in the `redis` namespace.
The `redis-password` field from `Secret/redis-auth` was injected into the Pod
as the `REDIS_PASSWORD` environment variable. The real password was not placed
in the Pod YAML or this report.

Authenticated PING result:

```text
PONG
```

This result verified:

- Kubernetes DNS resolution;
- the short `redis` name resolving to `redis.redis.svc.cluster.local`;
- `Service/redis` routing to the correct endpoint;
- connectivity to TCP port `6379`;
- correct Secret injection;
- successful Redis authentication.

Unauthenticated PING result:

```text
NOAUTH Authentication required.
```

The rejected unauthenticated request proved that authentication was required
by the live Redis process, not merely enabled in the values file.

## 11. Persistence Test

A non-sensitive test record was written:

| Field | Value |
| --- | --- |
| Key | `phase6:persistence` |
| Value | `survives-restart` |

Initial read result:

```text
survives-restart
```

Only `Pod/redis-0` was then deleted. Because the StatefulSet desired one
replica, it automatically created a replacement `redis-0` Pod.

The replacement Pod:

- used the same PVC;
- mounted the same `PV/redis-data`;
- became `Running` and `Ready`.

Read result after the Pod recreation:

```text
survives-restart
```

This confirmed that the value was stored through:

```text
Redis /data
  → Redis PVC
  → PV/redis-data
  → /var/lib/rancher/rke2/storage/redis
```

After the test, the `phase6:persistence` key was removed with `DEL`, and the
temporary `redis-cli-test` Pod was deleted.

This test proves persistence across a Pod recreation only. It does not prove
backup, disaster recovery, or high availability against VM, node, or disk
loss.

## 12. Resource and Disk Measurements

Node measurements:

| Measurement | Before Redis | After Redis |
| --- | ---: | ---: |
| CPU | `205m` | `234m` |
| CPU percentage | `5%` | `5%` |
| Memory | `3956Mi` | `3942Mi` |
| Memory percentage | `49%` | `49%` |

Redis Pod measurement:

| Resource | Usage |
| --- | ---: |
| CPU | approximately `5m` |
| Memory | approximately `4Mi` |

VM disk measurement:

| Field | Value |
| --- | ---: |
| Filesystem | `/dev/sda1` |
| Total | `48G` |
| Used | `8.7G` |
| Available | `39G` |
| Usage | `19%` |

The Redis directory used approximately `20K` of real disk space.

These values are point-in-time measurements. The small CPU increase and slight
memory decrease after Redis do not independently describe Redis's long-term
resource effect. Metrics Server sampling, background controller activity,
caches, and normal runtime variation can affect the numbers.

The Phase 7 capacity decision therefore considered node percentages, pod
stability, available disk space, and Redis's own usage together.

## 13. Issues Encountered and Resolutions

### `openssl` Was Missing from the OpenBao Container

The initial plan attempted to generate the Redis password inside the OpenBao
container. The image did not contain `openssl`, so the first two operations
created empty KV v2 versions `1` and `2`.

Resolution:

1. The empty versions were not accepted as valid secrets.
2. A password was generated with `openssl` on the Mac.
3. It was passed to OpenBao through stdin without printing the value.
4. The valid value was stored as version `3`.
5. Incorrect versions `1` and `2` were destroyed.
6. ESO successfully read the valid version, and
   `ExternalSecret/redis-auth` became `Ready=True`.

This incident demonstrated that a failure in the first command of a secret
generation pipeline can pass empty stdin to the next command. Secret writes
must therefore be verified through version metadata and consumer health, not
only through a final command exit status.

### Risk of Misinterpreting Static PV Capacity

The `2Gi` PV declaration did not create a new physical disk. Data was written
to the existing VM filesystem. Because no filesystem quota was configured,
the capacity field alone was not a strict physical usage limit.

Disk usage was therefore measured both at filesystem level and for the Redis
directory.

## 14. Security Checks

The following security boundaries were preserved:

- the real Redis password was not written to Git or this report;
- the Secret's base64 value was not printed;
- no OpenBao root token, recovery/unseal key, kubeconfig content, or private
  SSH key was recorded;
- Redis values referenced `Secret/redis-auth` instead of containing a literal
  password;
- the OpenBao policy granted `read` only on the exact Redis KV v2 paths;
- the Kubernetes auth role remained restricted by ServiceAccount, namespace,
  and audience;
- Redis ran as non-root UID/GID `999:999`;
- privilege escalation was disabled;
- the root filesystem was read-only;
- all Linux capabilities were dropped;
- seccomp used `RuntimeDefault`;
- the ServiceAccount token was not mounted automatically;
- the Redis Service remained `ClusterIP`.

`creationPolicy: Owner` made ESO responsible for the target Kubernetes Secret.
`deletionPolicy: Retain` protected the existing target Secret from automatic
deletion if the provider value disappeared, but it did not provide backup.

TLS, metrics, and NetworkPolicy remained disabled in this phase. Redis was not
exposed externally and was reachable only through the cluster-internal
Service. This is a local learning-lab boundary, not a production security
model.

## 15. Phase 7 Capacity Decision

The measured environment has enough capacity to proceed with Phase 7
CloudNativePG work:

- node CPU usage remained around `5%`;
- memory usage remained around `49%`;
- the Redis Pod used approximately `5m` CPU and `4Mi` memory;
- approximately `39G` remained available on the VM disk;
- the Redis directory used approximately `20K`;
- Redis did not destabilize the existing platform workloads.

Decision: Phase 7 planning and controlled implementation may proceed.

Before implementing CloudNativePG:

1. Repeat node and all-pod resource measurements.
2. Prepare a separate VM directory under
   `/var/lib/rancher/rke2/storage/postgresql`.
3. Determine how many PVCs CloudNativePG will create.
4. Prepare a separate suitable static PV for every PostgreSQL PVC.
5. Never reuse `PV/redis-data` or the Redis VM directory for PostgreSQL.
6. Preserve the existing Redis PVC/PV binding.

`local-storage` will be the shared StorageClass, but Redis and PostgreSQL will
not share data volumes. This static local model will not provide PostgreSQL HA
or backup.

## 16. Conclusion and Lessons Learned

Phase 6 completed successfully. A public third-party OCI Helm chart was
combined with a Git-hosted values file through Argo CD multi-source and
deployed as a standalone Redis StatefulSet.

The implementation proved:

- separate pinning of the chart package and runtime image;
- use of an ARM64-compatible Docker Official Redis image;
- Kubernetes Secret generation from OpenBao through ESO;
- keeping the real password outside Git;
- separation between a shared static StorageClass and a Redis-specific PV/PVC;
- working Service DNS and required authentication;
- persistence after Pod recreation;
- non-root and minimum-privilege container settings;
- sufficient CPU, memory, and disk capacity for Phase 7.

Operational lessons:

- a Helm chart and a container image are different release artifacts;
- an `ExternalSecret` references a Store instead of carrying OpenBao
  connection details;
- a static local PV can survive Pod recreation but not VM or node loss;
- `Retain` is not backup;
- a PV capacity declaration does not create a physical disk or filesystem
  quota;
- secret generation pipelines and resulting KV versions must be verified;
- point-in-time resource measurements must be interpreted together with normal
  runtime variation.

Phase 6 met its chart, GitOps, secret, storage, authentication, persistence,
security, and capacity goals through live implementation tests.
