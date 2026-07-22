# Placeholder Values

Use placeholders until the relevant phase asks for a real value.

| Placeholder | Example | Used for |
| --- | --- | --- |
| `<GITHUB_USERNAME>` | `kullanici-adi` | GitHub and GHCR namespace |
| `<APP_REPO>` | `app-for-gitops` | Application repository name |
| `<GITOPS_REPO>` | `rke2-gitops` | GitOps repository name |
| `<REUSABLE_HELM_CHARTS_REPO>` | `reusable-helm-charts` | Reusable Helm chart repository name |
| `<LOCAL_NODE_IP>` | `192.168.64.x` | Local RKE2 node IP |
| `<LOCAL_DOMAIN>` | `demo.<LOCAL_NODE_IP>.sslip.io` | Local ingress host |
| `<OCI_REGION>` | `eu-frankfurt-1` | Jenkins VM region placeholder |
| `<GHCR_IMAGE>` | `ghcr.io/<GITHUB_USERNAME>/app-for-gitops` | Demo app image repository |
| `<GENERIC_APP_CHART_PATH>` | `charts/generic-app-chart` | Reusable generic app chart path |
| `<IMAGE_TAG>` | `git-<SHORT_SHA>` | Immutable image tag |
| `<ARGOCD_REPO_SECRET_NAME>` | `gitops-repo-ssh` | Argo CD repository Secret name |
| `<OPENBAO_ADDR>` | `https://openbao.<LOCAL_NODE_IP>.sslip.io` | OpenBao address |

Never replace these placeholders with real secret values in Git.
