# Repository Structure

This project uses three GitHub repositories.

## Repositories

| Repository         | Responsibility                                                                      | Local path                                                |
| ------------------ | ----------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `rke2-gitops`    | GitOps desired state, Argo CD resources, values files and documentation             | `/Users/bertughas/Documents/rke2-gitops/rke2-gitops`    |
| `app-for-gitops` | Application source code, tests, Dockerfile, Jenkinsfile and CI scripts              | `/Users/bertughas/Documents/rke2-gitops/app-for-gitops` |
| `reusable-helm-charts` | Reusable Helm chart source shared by GitOps projects                          | `/Users/bertughas/Documents/rke2-gitops/reusable-helm-charts` |

These are sibling repositories under the same parent directory:

```text
/Users/bertughas/Documents/rke2-gitops/
├── rke2-gitops/
├── app-for-gitops/
└── reusable-helm-charts/
```

They are pushed to GitHub separately. `rke2-gitops` is not the parent repository
of `app-for-gitops` or `reusable-helm-charts`, and sibling repositories must not
be committed inside each other.

## Current GitOps Tree

### `rke2-gitops`

```text
rke2-gitops/
├── argocd/
│   ├── namespaces/
│   │   ├── argocd.yaml
│   │   └── dev.yaml
│   └── projects/
│       ├── dev.yaml
│       └── platform.yaml
├── apps-values/
├── bootstrap/
│   └── root-app.yaml
├── Brewfile
├── README.md
├── .gitignore
├── docs/
│   ├── concepts.md
│   ├── decisions.md
│   ├── examples/
│   │   └── repository-secret.example.yaml
│   ├── git-ssh-identity.md
│   ├── host-bootstrap.md
│   ├── host-tools.md
│   ├── phase-1-report.md
│   ├── phase-2-report.md
│   ├── placeholders.md
│   ├── repository-structure.md
│   ├── security-model.md
│   ├── troubleshooting.md
│   └── phases/
│       ├── faz-0.md
│       ├── faz-0-manuel-git-push.md
│       ├── faz1.md
│       └── faz2.md
└── rke2-gitops-portfoy-aksiyon-plani-final.md
```

Generic Helm chart source is not part of the desired-state tree. Reusable chart
source lives in the sibling `reusable-helm-charts` repository under:

```text
reusable-helm-charts/
└── charts/
    └── generic-app-chart/
```

## Argo CD Recursive Directory Boundary

`bootstrap/root-app.yaml` points Argo CD at the `argocd` path.

The current bootstrap model uses Argo CD directory recursion:

```yaml
directory:
  recurse: true
```

There is no `argocd/kustomization.yaml` in the current model. Argo CD reads real
manifest YAML files under `argocd/` recursively.

Current bootstrap-managed manifests:

```text
argocd/namespaces/argocd.yaml
argocd/namespaces/dev.yaml
argocd/projects/dev.yaml
argocd/projects/platform.yaml
```

Empty folders are not tracked with placeholder files.

The repository Secret example is deliberately outside the `argocd` tree:

```text
docs/examples/repository-secret.example.yaml
```

It is documentation only and must not contain a real private key.

### `app-for-gitops`

```text
app-for-gitops/
├── README.md
├── .gitignore
└── docs/
    └── repository-scope.md
```

## Future GitOps Target Tree

The following tree is the broader target direction. Paths that are not needed
yet will be created only when their phase starts.

```text
rke2-gitops/
├── bootstrap/
├── argocd/
├── apps-values/
├── platform-values/
└── platform-manifests/
```

The GitOps repository stores values and Argo CD Application resources that
reference reusable charts. It does not own the source of
`generic-app-chart`.

## Future Application Target Tree

The following tree is the target direction for the application repository. It
will be created only when the application phase starts.

```text
app-for-gitops/
├── src/
├── tests/
├── scripts/
├── package.json
├── Dockerfile
├── Jenkinsfile
├── .dockerignore
├── .gitignore
└── README.md
```
