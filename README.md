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

Current phase: Phase 4 completed. The demo application is now managed through
an Argo CD ApplicationSet.

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
- `AppProject/platform`;
- `ApplicationSet/apps`;
- `ApplicationSet/platform-helm`;
- `ApplicationSet/platform-manifests`.

The `apps` ApplicationSet scans:

```text
apps-values/dev/*
```

It currently finds:

```text
apps-values/dev/demo-app
```

and generates the `dev-demo-app` Argo CD Application.

The running demo application uses:

- image `ghcr.io/bertughas123/app-for-gitops:git-e19bd71`;
- reusable chart repository `https://github.com/bertughas123/reusable-helm-charts`;
- chart tag `generic-app-chart-v0.1.0`;
- values file `apps-values/dev/demo-app/values.yaml`;
- namespace `dev`.

The `platform-helm` and `platform-manifests` ApplicationSets are prepared for
future platform services. They currently generate zero Applications because no
matching `source.yaml` files exist yet.

Generic chart source is not owned by this GitOps repository. Reusable chart
code lives in `reusable-helm-charts`; this repository keeps environment-specific
values files, Argo CD bootstrap manifests and documentation.

See [docs/repository-structure.md](docs/repository-structure.md) for the current
three-repository layout.

For repeatable macOS host setup, see [docs/host-bootstrap.md](docs/host-bootstrap.md).

For the Phase 1 completion record, see [docs/phase-1-report.md](docs/phase-1-report.md).

For the Phase 2 completion record, see [docs/phase-2-report.md](docs/phase-2-report.md).

For the Phase 3 completion record, see [docs/phase-3-report.md](docs/phase-3-report.md).

For the Phase 4 completion record, see [docs/phase-4-report.md](docs/phase-4-report.md).
