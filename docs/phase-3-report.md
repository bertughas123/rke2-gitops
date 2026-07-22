# Phase 3 Report - Demo Application Delivery

Status: completed.

Date recorded: 2026-07-22

## Result

Phase 3 completed the first end-to-end demo application delivery flow.

The phase connected application code, tests, Docker image build, Trivy scanning,
GHCR publishing, GitOps values, Argo CD reconciliation, RKE2 deployment and
Traefik ingress access.

Jenkins was intentionally not introduced in this phase. The build, scan and
publish steps were performed manually so the flow could be understood before it
is automated in Phase 8.

## Final Architecture

The project now uses three repositories.

| Repository | Responsibility |
| --- | --- |
| `app-for-gitops` | Application source code, tests and Dockerfile |
| `reusable-helm-charts` | Shared generic Helm chart source |
| `rke2-gitops` | Environment values files and Argo CD Application manifests |

The Phase 3 delivery flow is:

```text
App code -> test -> Docker build -> Trivy -> GHCR -> GitOps values -> Argo CD -> RKE2 -> Traefik
```

This means the image is built from the application repository, published to
GHCR, referenced from the GitOps repository, and then reconciled into the RKE2
cluster by Argo CD.

## GitOps Bootstrap Model

The root Argo CD Application, `bootstrap-root`, reads the `argocd` directory
with recursive directory mode:

```yaml
directory:
  recurse: true
```

Kustomize is not used in the current `argocd` tree. Argo CD reads plain
Kubernetes manifest files under `argocd/` recursively.

Ownership model:

- `dev` namespace is managed by `bootstrap-root`;
- `dev-demo-app` manages only application resources inside the `dev` namespace;
- `CreateNamespace=true` is not used in the child Application;
- `dev` namespace has sync wave `"0"`;
- `dev-demo-app` has sync wave `"10"`;
- `argocd` namespace uses `Prune=false`;
- `dev` namespace uses `Prune=confirm`.

Sync waves tell Argo CD which resources should be applied first. In this phase,
the namespace is created before the application that deploys into it.

## Node.js Application

The demo application is a small Node.js HTTP service.

Runtime behavior:

- listens on `0.0.0.0:3000`;
- `GET /health/live` returns `{"status":"ok"}`;
- `GET /health/ready` returns `{"status":"ready"}`;
- `GET /api/version` returns the `APP_VERSION` environment variable;
- unknown paths return HTTP `404` with `{"error":"not_found"}`.

The application uses Node.js built-in modules only. It does not use Express or
third-party runtime dependencies.

Tests use the built-in `node:test` and `assert` modules. The final test result
was:

```text
5 pass
0 fail
```

## Docker Image

The production image was built for ARM64.

| Item | Value |
| --- | --- |
| Application commit SHA | `e19bd71` |
| Image | `ghcr.io/bertughas123/app-for-gitops:git-e19bd71` |
| Image digest | `sha256:8e1303fade2444e0d64da19a925210882f81555118b3a517c7816207cc85a617` |

The image passed local container tests before it was pushed.

The GHCR package is public and linked to the `app-for-gitops` repository.
Personal access token values and token creation details are intentionally not
documented here.

## Trivy Security Scan

The first image scan found HIGH and CRITICAL findings under unused npm tooling
from the base image, not from application dependencies.

The runtime command is:

```text
node src/server.js
```

Because the production container does not need `npm`, `npx`, `yarn` or
`corepack`, those tools were removed from the final image while keeping the
`node` executable.

After rebuilding the image, the final Trivy scan was clean. Secret scanning was
also clean.

## Reusable Helm Chart

Repository:

```text
https://github.com/bertughas123/reusable-helm-charts
```

Chart path:

```text
charts/generic-app-chart
```

Chart version source:

| Item | Value |
| --- | --- |
| Chart Git commit | `2bf2e243f4194978861d1ac3304ccd0fee48fd22` |
| Chart Git tag | `generic-app-chart-v0.1.0` |

The chart supports these Kubernetes resources:

- `Deployment`;
- `Service`;
- `ServiceAccount`;
- `Ingress`;
- `HorizontalPodAutoscaler`;
- `NetworkPolicy`.

`HorizontalPodAutoscaler` and `NetworkPolicy` are disabled by default.

Default security and operations settings include:

- `runAsNonRoot`;
- `seccompProfile: RuntimeDefault`;
- `allowPrivilegeEscalation: false`;
- drop all Linux capabilities;
- `readOnlyRootFilesystem: true`;
- `automountServiceAccountToken: false`;
- CPU and memory requests and limits;
- liveness and readiness probes.

Both `helm lint` and `helm template` checks completed successfully.

## Dev Values

Values file:

```text
apps-values/dev/demo-app/values.yaml
```

Important settings:

- image repository is `ghcr.io/bertughas123/app-for-gitops`;
- image tag is `git-e19bd71`;
- `APP_VERSION` is `git-e19bd71`;
- container port is `3000`;
- Service type is `ClusterIP`;
- Service port is `80`;
- Service `targetPort` points to the named container port `http`;
- Ingress class is `traefik`;
- Ingress host is `demo.192.168.252.2.sslip.io`;
- HPA is disabled;
- NetworkPolicy is disabled.

The VM IP can change after VM or host restarts. The current IP must be checked
again before relying on the `sslip.io` hostname.

## Argo CD Multi-Source Application

Application file:

```text
argocd/applications/dev-demo-app.yaml
```

The application is a normal Argo CD `Application`, not an `ApplicationSet`.

Configuration:

- `project: dev`;
- chart repository is `https://github.com/bertughas123/reusable-helm-charts.git`;
- chart `targetRevision` is `generic-app-chart-v0.1.0`;
- chart path is `charts/generic-app-chart`;
- values repository is `rke2-gitops` on branch `main`;
- values source uses `ref: values`;
- Helm values path is `$values/apps-values/dev/demo-app/values.yaml`;
- destination namespace is `dev`;
- automated sync has `prune: true`;
- automated sync has `selfHeal: true`;
- `CreateNamespace=true` is not used.

The `$values` prefix tells Argo CD to load the Helm values file from the second
source instead of from the chart repository.

## Deployment Issue And Fix

During deployment, the application pod hit:

```text
CreateContainerConfigError
```

The reason was the interaction between Kubernetes security settings and the
Docker image user.

The generic chart sets:

```yaml
runAsNonRoot: true
```

The Docker image runs as the named user `node`. Kubernetes could not prove from
the user name alone that the container was non-root.

The image was checked and the `node` user maps to:

```text
UID=1000
GID=1000
```

The GitOps fix was added to the demo values file:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
```

No manual `kubectl patch` was used. After the values change was committed and
pushed, Argo CD reconciled the desired state and created a new pod.

## Final Verification

Final runtime verification:

- Deployment is `Ready 1/1`;
- pod is `Running 1/1`;
- restart count is `0`;
- the running image tag is `git-e19bd71`;
- container runs with UID/GID `1000`;
- Service and EndpointSlice are connected;
- Ingress exists;
- curl through Traefik succeeds.

Endpoint results:

```text
/health/live  -> {"status":"ok"}
/health/ready -> {"status":"ready"}
/api/version  -> {"version":"git-e19bd71"}
unknown path  -> HTTP 404 and {"error":"not_found"}
```

## Known State

Argo CD sync status is `Synced`.

The application is functional and external access through Traefik was verified
with curl.

Argo CD health is `Progressing`, not `Healthy`.

The reason is ingress status reporting. Traefik is running with a hostPort
model, while the `rke2-traefik` Service is `ClusterIP` and does not publish a
LoadBalancer address. Because there is no published ingress address, the Ingress
`ADDRESS` or status field stays empty.

This does not block application traffic in the local setup.

Publishing Traefik ingress status is left as a later platform improvement.

TLS was not configured in this phase.

## Security And Git Controls

No secret, PAT or private key value is included in this report.

No real private key or GitHub token pattern was found in the Phase 3 application
and GitOps files that were reviewed. The only pattern-like match found during a
repository scan was an example `rg` command in security documentation.

The Docker image is pulled from public GHCR.

Current local working trees are not all clean because there are separate local
documentation changes outside the runtime Phase 3 delivery scope. Before the
next final handoff commit, all three repositories should be checked again with
`git status`.

## Phase 3 Boundary

Phase 3 did not introduce:

- Jenkins;
- ApplicationSet;
- OpenBao;
- External Secrets Operator;
- Redis;
- PostgreSQL;
- production environment;
- TLS.

## Phase 3 Completion Summary

Phase 3 successfully completed:

- a tested Node.js demo application;
- an ARM64 production Docker image;
- Trivy cleanup and final clean scan;
- GHCR image publishing;
- a reusable generic Helm chart in a separate repository;
- GitOps values for the dev demo application;
- a multi-source Argo CD Application;
- RKE2 deployment through Argo CD;
- Traefik ingress access;
- runtime endpoint verification.

In a later phase, Jenkins will automate the manual test, image build, scan,
publish and GitOps update steps that were performed by hand in this phase.
