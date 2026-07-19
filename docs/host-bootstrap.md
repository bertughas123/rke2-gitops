# Host Bootstrap

This document describes the repeatable host setup for a new macOS machine.

The project standard is:

- Homebrew for CLI tooling;
- Docker Desktop for the Docker Engine;
- Multipass for local Ubuntu VMs;
- no project dependency on Colima.

## 1. Install Homebrew

Check whether Homebrew exists:

```bash
brew --version
```

If it is missing, install it from the official Homebrew install page:

```text
https://brew.sh/
```

Do not paste tokens, private keys or credentials into the terminal during this
step.

## 2. Install Project Tools

From the GitOps repository root:

```bash
cd /Users/bertughas/Documents/rke2-gitops/rke2-gitops
brew bundle install --file Brewfile
```

This installs:

```text
gh
kubectl
helm
argocd
trivy
ripgrep
Multipass
Docker Desktop
```

If you prefer installing Docker Desktop manually from Docker's website, install
it first and then run the same `brew bundle install --file Brewfile` command.
Homebrew will skip already satisfied items.

To check what is still missing without installing anything:

```bash
brew bundle check --file Brewfile
```

If Docker Desktop is not installed yet, this check is expected to report:

```text
Cask docker needs to be installed or updated.
```

## 3. Start Docker Desktop

Docker Desktop is the project standard for the Docker Engine on macOS.

After installation, start Docker Desktop from Applications or Spotlight and wait
until it reports that the engine is running.

Then verify:

```bash
docker version
docker info
```

Expected result:

- `docker version` shows both Client and Server sections;
- `docker info` returns engine information without a daemon connection error.

## 4. Verify CLI Tools

```bash
gh --version
kubectl version --client=true
helm version
argocd version --client
trivy --version
rg --version
multipass version
docker version
```

## 5. Notes About Colima

Colima is a valid local Docker Engine alternative, but it is not the chosen
standard for this project.

Use Docker Desktop unless a later troubleshooting note explicitly says
otherwise.

## 6. Updating The Tool List

When a new host tool becomes part of the project, update both:

```text
Brewfile
docs/host-tools.md
```

Then document why the tool is needed.
