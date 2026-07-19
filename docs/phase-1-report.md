# Phase 1 Report - Single RKE2 Server Node

Date recorded: 2026-07-20

## Result

Phase 1 is complete.

The local Kubernetes base now exists as a single-node RKE2 cluster running inside
a Multipass VM.

## Final State

| Item | Value |
| --- | --- |
| VM name | `rke2-server` |
| Ubuntu version | `24.04.4 LTS` |
| Architecture | `ARM64` |
| VM resources | `4 CPU / 8 GB RAM / 50 GB disk` |
| RKE2 version | `v1.36.2+rke2r1` |
| Node status | `Ready` |
| CNI / network plugin | `Canal` |
| Ingress controller | `Traefik` |
| Metrics Server | Running |
| StorageClass | None |
| Snapshot | `phase1-manuel-...` |

## Meaning

This confirms that the project now has a working local Kubernetes foundation.

At this point:

- RKE2 server is installed and running;
- the Kubernetes node is accepted by the control plane;
- cluster networking is available through Canal;
- Traefik is present as the ingress controller;
- Metrics Server is running;
- no default StorageClass exists yet;
- an etcd snapshot was taken manually.

## Security Notes

No real secrets were added to Git as part of Phase 1.

The Mac-side kubeconfig is local machine state and must stay out of the
repository. The repository `.gitignore` blocks common kubeconfig names, private
keys and secret files.

## Phase 1 Boundary

Phase 1 did not create:

- Argo CD;
- GitHub Deploy Keys;
- Jenkins credentials;
- OpenBao;
- External Secrets Operator;
- application manifests;
- GHCR images;
- worker nodes.

Those belong to later phases.

## Follow-Up For Phase 2

Phase 2 should start from the existing `rke2-server` cluster and bootstrap
Argo CD in a controlled, reviewable way.

Before Phase 2 starts, confirm:

```bash
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl get nodes -o wide
KUBECONFIG=~/.kube/rke2-gitops.yaml kubectl get pods -A
```

Expected baseline:

- node is `Ready`;
- core system pods are running or completed;
- no manual application workloads are required yet.
