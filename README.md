# RKE2 GitOps

This repository contains the GitOps side of a learning-focused RKE2 portfolio
project.

The project demonstrates:

- RKE2 on a local ARM64 macOS host;
- Argo CD driven GitOps;
- a generic Helm chart for demo applications;
- External Secrets Operator with OpenBao;
- public third-party Helm charts;
- an operator-based PostgreSQL example with CloudNativePG;
- Jenkins-driven image build and GitOps tag update flow.

## Repository Split

| Repository      | Responsibility                                                                         |
| --------------- | -------------------------------------------------------------------------------------- |
| `app-for-gitops` | Application source code, tests, Dockerfile, Jenkinsfile and CI scripts              |
| `rke2-gitops` | Kubernetes manifests, Argo CD applications, Helm chart, values files and documentation |

## Security Rule

No real secret belongs in this repository.

Use placeholders such as `<GITHUB_USERNAME>`, `<LOCAL_NODE_IP>` and
`<PLACEHOLDER_SECRET_NAME>` in documentation and manifests. Real values must be
kept in the correct external system, such as GitHub credentials, Jenkins
credentials, Kubernetes Secrets during bootstrap, or OpenBao.

## Phase Status

Current phase: Phase 0 - preparation and decisions.

No VM or Kubernetes cluster is expected to exist at this point.

See [docs/repository-structure.md](docs/repository-structure.md) for the current
two-repository layout.

For repeatable macOS host setup, see [docs/host-bootstrap.md](docs/host-bootstrap.md).
