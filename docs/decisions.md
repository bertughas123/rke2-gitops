# Decisions

## Repository Split

Decision: use two repositories.

| Repository | Responsibility |
| --- | --- |
| `app-for-gitops` | Application source code, tests, Dockerfile, Jenkinsfile and CI scripts |
| `rke2-gitops` | Kubernetes manifests, Argo CD applications, Helm chart, values files and documentation |

Reason: application build concerns and cluster desired state should remain
separate.

## Public Repositories

Decision: repositories may be public for portfolio purposes.

Reason: the project is meant to be reviewable.

Constraint: no real secret may be committed.

## Argo CD Repository Access

Decision: Argo CD will access the GitOps repository through SSH with a
repository-specific read-only GitHub Deploy Key.

Reason: this teaches repository credentials, least privilege, bootstrap Secret
handling and later OpenBao handover.

## Jenkins Repository Access

Decision: Jenkins will use a separate write credential for GitOps updates.

Reason: Jenkins must update `image.tag` after a successful build, while Argo CD
only needs read access.

## Cluster Access From Jenkins

Decision: Jenkins will not receive the local RKE2 kubeconfig.

Reason: Jenkins should not deploy directly to the cluster. Argo CD owns the
deployment path.

## First Environment

Decision: only `dev` exists at the beginning.

Reason: empty `stage` and `prod` folders do not teach promotion until real
environment differences exist.

## AppProject Ownership

Decision:

| AppProject | Owner scope |
| --- | --- |
| `default` | Bootstrap root Application only |
| `dev` | Demo and business applications |
| `platform` | Platform components such as OpenBao, ESO, Redis, CloudNativePG, Keycloak and monitoring |

Reason: bootstrap must not depend on AppProjects that are created by bootstrap
itself.
