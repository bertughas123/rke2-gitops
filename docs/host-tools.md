# Host Tools Inventory

This file records which tools are expected on the local macOS host before later
phases create VMs or clusters.

Phase 0 started as an inventory step. The missing host tools were then installed
with Homebrew on request.

For a repeatable setup on another macOS machine, use the repository `Brewfile`:

```bash
brew bundle install --file Brewfile
```

## Current Host

| Item | Value |
| --- | --- |
| Architecture | `arm64` |
| OS | `macOS 26.5.2` |
| Build | `25F84` |

## Inventory

| Tool | Required later? | Status | Detected version or note |
| --- | --- | --- | --- |
| macOS ARM64 | Yes | Installed | `arm64` |
| Git | Yes | Installed | `git version 2.50.1 (Apple Git-155)` |
| SSH | Yes | Installed | `OpenSSH_10.2p1, LibreSSL 3.3.6` |
| GitHub CLI | Helpful | Installed | `gh version 2.96.0` |
| Multipass | Yes | Installed | `multipass 1.16.3+mac` |
| kubectl | Yes | Installed | `Client Version: v1.36.2`, `Kustomize Version: v5.8.1` |
| Helm | Yes | Installed | `v4.2.3` |
| Argo CD CLI | Helpful | Installed | `v3.4.5` |
| Docker Desktop | Yes | To be installed manually or by `Brewfile` | Project standard Docker Engine for macOS |
| Docker CLI | Yes | Installed | `Docker version 29.6.2`; server requires Docker Desktop to be running |
| Colima | No | Installed locally, not project standard | Do not use for the normal project path |
| Trivy | Yes | Installed | `Version: 0.72.0` |
| ripgrep | Helpful | Installed | `ripgrep 15.2.0` |

## Commands Used

```bash
uname -m
sw_vers
git --version
ssh -V
gh --version
multipass version
kubectl version --client=true
helm version
argocd version --client
docker version
trivy --version
rg --version
```

## Notes

Do not paste real tokens or private key material into this file.

Docker CLI is installed, but Docker workloads require a running daemon. The
project standard is Docker Desktop, not Colima.

Start Docker Desktop from Applications or Spotlight, then verify:

```bash
docker version
docker info
```

`docker version` should show both Client and Server sections. If only the Client
section appears, Docker Desktop is not running yet.
