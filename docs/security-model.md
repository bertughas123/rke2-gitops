# Security Model

## Main Rule

Never commit real secrets.

This includes:

- passwords;
- tokens;
- SSH private keys;
- kubeconfig files;
- OpenBao root or recovery material;
- GitHub personal access tokens;
- Jenkins credentials;
- TLS private keys.

## Allowed In Git

The repository may contain:

- placeholders;
- documentation;
- Kubernetes manifests without real secret values;
- ExternalSecret resources that reference external secret keys;
- sealed or encrypted examples only if their private decrypt material is not in Git.

## Not Allowed In Git

The repository must not contain:

- literal secret values;
- base64-encoded Kubernetes Secret values that represent real credentials;
- private keys;
- local kubeconfig files;
- `.env` files with real values.

## Argo CD Credential Model

Argo CD uses a read-only GitHub Deploy Key for the GitOps repository.

The private key is not committed. During bootstrap it is stored manually as a
Kubernetes Secret in the `argocd` namespace. Later it will be handed over to
OpenBao and External Secrets Operator.

## Jenkins Credential Model

Jenkins uses a separate write credential to update the GitOps repository.

Jenkins must not use the Argo CD read-only Deploy Key. Jenkins must not receive
the local cluster kubeconfig.

## Secret Review Checklist

Before committing, search for suspicious values:

```bash
rg -n "BEGIN (RSA|OPENSSH|EC|PRIVATE)|password|passwd|token|secret|kubeconfig|ghp_|github_pat_|AKIA|xoxb-" .
```

Every match must be either a placeholder, documentation warning or non-secret
example.
