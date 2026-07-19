# Repository Structure

This project uses two GitHub repositories.

## Repositories

| Repository | Responsibility | Phase 0 local path |
| --- | --- | --- |
| `rke2-gitops` | GitOps desired state, Argo CD resources, Helm chart, values files and documentation | `/Users/bertughas/Documents/rke2-gitops/rke2-gitops` |
| `app-for-gitops` | Application source code, tests, Dockerfile, Jenkinsfile and CI scripts | `/Users/bertughas/Documents/rke2-gitops/app-for-gitops` |

These are sibling repositories under the same parent directory:

```text
/Users/bertughas/Documents/rke2-gitops/
├── rke2-gitops/
└── app-for-gitops/
```

They are pushed to GitHub separately. `rke2-gitops` is not the parent repository
of `app-for-gitops`, and `app-for-gitops` must not be committed inside
`rke2-gitops`.

## Current Phase 0 Tree

### `rke2-gitops`

```text
rke2-gitops/
├── Brewfile
├── README.md
├── .gitignore
├── docs/
│   ├── concepts.md
│   ├── decisions.md
│   ├── git-ssh-identity.md
│   ├── host-bootstrap.md
│   ├── host-tools.md
│   ├── placeholders.md
│   ├── repository-structure.md
│   ├── security-model.md
│   └── phases/
│       ├── faz-0.md
│       └── faz-0-manuel-git-push.md
└── rke2-gitops-portfoy-aksiyon-plani-final.md
```

### `app-for-gitops`

```text
app-for-gitops/
├── README.md
├── .gitignore
└── docs/
    └── repository-scope.md
```

## Future GitOps Target Tree

The following tree is the target direction, not the Phase 0 output. These paths
will be created only when their phase starts.

```text
rke2-gitops/
├── bootstrap/
├── argocd/
├── helm-charts/
├── apps-values/
├── platform-values/
└── platform-manifests/
```

## Future Application Target Tree

The following tree is the target direction for the application repository. It is
not created in Phase 0.

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
