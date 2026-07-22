# RKE2 GitOps

This repository contains the GitOps side of a learning-focused RKE2 portfolio
project.

The project demonstrates:

- RKE2 on a local ARM64 macOS host;
- Argo CD driven GitOps;
- a reusable generic Helm chart consumed from a separate chart repository;
- External Secrets Operator with OpenBao;
- public third-party Helm charts;
- an operator-based PostgreSQL example with CloudNativePG;
- Jenkins-driven image build and GitOps tag update flow.

## Repository Split

| Repository         | Responsibility                                                                         |
| ------------------ | -------------------------------------------------------------------------------------- |
| `app-for-gitops` | Application source code, tests, Dockerfile, Jenkinsfile and CI scripts                 |
| `rke2-gitops`    | Kubernetes manifests, Argo CD applications, values files and documentation             |
| `reusable-helm-charts` | Reusable Helm chart source shared by GitOps projects                              |

## Security Rule

No real secret belongs in this repository.

Use placeholders such as `<GITHUB_USERNAME>`, `<LOCAL_NODE_IP>` and
`<PLACEHOLDER_SECRET_NAME>` in documentation and manifests. Real values must be
kept in the correct external system, such as GitHub credentials, Jenkins
credentials, Kubernetes Secrets during bootstrap, or OpenBao.

## Phase Status

Current phase: Phase 2 completed. Preparing the next application/bootstrap
phase.

Local RKE2 single-node cluster exists on Multipass VM `rke2-server`.

Argo CD `v3.4.5` is installed on the cluster and bootstrapped through
`bootstrap-root`.

Current GitOps bootstrap path:

```text
bootstrap/root-app.yaml -> argocd/ recursive directory
```

`bootstrap-root` follows the `argocd` path with `directory.recurse: true`.
Therefore every real manifest YAML under `argocd/` is in bootstrap scope.

The `argocd` path currently contains:

- `Namespace/argocd`;
- `Namespace/dev`;
- `AppProject/dev`;
- `AppProject/platform`.

Generic chart source is not owned by this GitOps repository anymore. Reusable
chart code lives in `reusable-helm-charts`; this repository will keep
application-specific Argo CD Applications and values files.

See [docs/repository-structure.md](docs/repository-structure.md) for the current
three-repository layout.

For repeatable macOS host setup, see [docs/host-bootstrap.md](docs/host-bootstrap.md).

For the Phase 1 completion record, see [docs/phase-1-report.md](docs/phase-1-report.md).

For the Phase 2 completion record, see [docs/phase-2-report.md](docs/phase-2-report.md).
