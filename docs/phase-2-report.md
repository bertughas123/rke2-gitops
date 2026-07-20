# Phase 2 Report - Argo CD Bootstrap

Status: completed.

Date recorded: 2026-07-20

## Result

Phase 2 successfully bootstrapped Argo CD on the local RKE2 cluster.

Argo CD is installed, reachable through local `kubectl port-forward`, connected
to the GitOps repository with a read-only GitHub Deploy Key, and managing the
bootstrap resources through Kustomize.

## Versions

| Item | Value |
| --- | --- |
| Kubernetes distribution | `RKE2 v1.36.2+rke2r1` |
| Argo CD version | `v3.4.5` |
| Argo CD install manifest | pinned `v3.4.5` manifest |
| Tracked Git branch | `main` |
| Argo CD source path | `argocd` |

## Cluster Baseline

```text
NAME          STATUS   ROLES                AGE   VERSION          INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION              CONTAINER-RUNTIME
rke2-server   Ready    control-plane,etcd   20h   v1.36.2+rke2r1   192.168.252.2   <none>        Ubuntu 24.04.4 LTS   6.8.0-136-generic (arm64)   containerd://2.3.2-k3s2
```

## Namespace

```text
NAME     STATUS   AGE
argocd   Active   179m
```

The `argocd` namespace was created manually at the beginning of bootstrap.

This is intentional because Argo CD cannot manage its own namespace before the
initial Argo CD installation and root Application exist.

## Argo CD Pods

All Argo CD pods are `Running` and `Ready`.

```text
NAME                                                READY   STATUS    RESTARTS   AGE    IP           NODE          NOMINATED NODE   READINESS GATES
argocd-application-controller-0                     1/1     Running   0          177m   10.42.0.29   rke2-server   <none>           <none>
argocd-applicationset-controller-7d4d7c7b89-vwrnf   1/1     Running   0          177m   10.42.0.24   rke2-server   <none>           <none>
argocd-dex-server-cbddb9676-dxrlg                   1/1     Running   0          177m   10.42.0.23   rke2-server   <none>           <none>
argocd-notifications-controller-7b55c64b69-wrh8p    1/1     Running   0          177m   10.42.0.25   rke2-server   <none>           <none>
argocd-redis-68bc658cfb-9ltrt                       1/1     Running   0          177m   10.42.0.26   rke2-server   <none>           <none>
argocd-repo-server-56c67cf674-6f78h                 1/1     Running   0          177m   10.42.0.27   rke2-server   <none>           <none>
argocd-server-66b7d96445-mcj2l                      1/1     Running   0          177m   10.42.0.28   rke2-server   <none>           <none>
```

Container image verification:

```text
NAME                                                READY   IMAGES
argocd-application-controller-0                     true    quay.io/argoproj/argocd:v3.4.5
argocd-applicationset-controller-7d4d7c7b89-vwrnf   true    quay.io/argoproj/argocd:v3.4.5
argocd-dex-server-cbddb9676-dxrlg                   true    ghcr.io/dexidp/dex:v2.45.0
argocd-notifications-controller-7b55c64b69-wrh8p    true    quay.io/argoproj/argocd:v3.4.5
argocd-redis-68bc658cfb-9ltrt                       true    public.ecr.aws/docker/library/redis:8.2.3-alpine
argocd-repo-server-56c67cf674-6f78h                 true    quay.io/argoproj/argocd:v3.4.5
argocd-server-66b7d96445-mcj2l                      true    quay.io/argoproj/argocd:v3.4.5
```

## Argo CD Services

```text
NAME                                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.43.81.65     <none>        7000/TCP,8080/TCP            177m
argocd-dex-server                         ClusterIP   10.43.25.198    <none>        5556/TCP,5557/TCP,5558/TCP   177m
argocd-metrics                            ClusterIP   10.43.131.175   <none>        8082/TCP                     177m
argocd-notifications-controller-metrics   ClusterIP   10.43.153.21    <none>        9001/TCP                     177m
argocd-redis                              ClusterIP   10.43.218.60    <none>        6379/TCP                     177m
argocd-repo-server                        ClusterIP   10.43.74.146    <none>        8081/TCP,8084/TCP            177m
argocd-server                             ClusterIP   10.43.11.63     <none>        80/TCP,443/TCP               177m
argocd-server-metrics                     ClusterIP   10.43.194.46    <none>        8083/TCP                     177m
```

Argo CD UI access was verified through local `kubectl port-forward`.

No public ingress was created in this phase.

## Repository Access

Git repository:

```text
git@github.com:bertughas123/rke2-gitops.git
```

Manual verification:

- GitHub repository `Settings` -> `Deploy keys` shows
  `argocd-rke2-gitops-readonly` as active.
- `Allow write access` is not selected.
- The Deploy Key is attached only to the `rke2-gitops` repository.
- Argo CD `Settings` -> `Repositories` shows the repository connection as
  `Successful`.

The private key was not written to Git. It was created as a Kubernetes
repository Secret for Argo CD.

## Root Application

`bootstrap-root` was initially applied manually with `kubectl`.

After that, Argo CD started tracking:

```text
branch: main
path: argocd
```

Application status:

```text
NAME             SYNC STATUS   HEALTH STATUS
bootstrap-root   Synced        Healthy
```

Manual UI verification:

```text
Sync Status: Synced
Health Status: Healthy
Auto Sync: Enabled
```

Automated sync is enabled with `prune: true` and `selfHeal: true`.

## AppProjects

```text
NAME       AGE
default    172m
dev        10m
platform   10m
```

Kustomize manages:

```text
Namespace/argocd
AppProject/dev
AppProject/platform
```

The root Application points to the `argocd` path. Because
`argocd/kustomization.yaml` exists, Argo CD processes that path with Kustomize.
Only resources listed in `argocd/kustomization.yaml` are managed.

## Security Notes

No real SSH private key, password, token or kubeconfig content belongs in Git.

The repository Secret example remains documentation-only:

```text
docs/examples/repository-secret.example.yaml
```

The real private key was used only to create the Kubernetes repository Secret.

## Phase 2 Boundary

Phase 2 did not create:

- demo application deployment;
- Jenkins;
- OpenBao;
- External Secrets Operator;
- GHCR image flow;
- public Argo CD ingress;
- production environment.

Those belong to later phases.
