# Phase 4 Report - ApplicationSet Migration

Status: completed.

Date recorded: 2026-07-22

## Result

Phase 4 completed the migration from the single Argo CD `Application` model
introduced in Phase 3 to an `ApplicationSet` based generation model.

The running demo application's name, image, Helm chart, values file and target
namespace were intentionally kept the same. The goal of this phase was not to
change the workload. The goal was to automate how the Argo CD `Application`
object is produced.

The migration was completed without workload interruption.

## System Versions

| Component | Version |
| --- | --- |
| RKE2 | `v1.36.2+rke2r1` |
| Argo CD | `v3.4.5` |

## ApplicationSet Resources

The following ApplicationSet manifests were added to the GitOps repository:

- `argocd/appsets/apps-appset.yaml`;
- `argocd/appsets/platform-helm-appset.yaml`;
- `argocd/appsets/platform-manifests-appset.yaml`.

The previous single demo application manifest was removed:

```text
argocd/applications/dev-demo-app.yaml
```

This removal was done in the controlled migration commit after the replacement
ApplicationSet manifests were prepared.

## Generator Model

The `apps` ApplicationSet uses a Git directory generator.

Generator input:

```text
apps-values/dev/*
```

Matched directory:

```text
apps-values/dev/demo-app
```

Generated Application:

```text
dev-demo-app
```

The `platform-helm` ApplicationSet uses a Git file generator.

Generator input:

```text
platform-values/*/source.yaml
```

The `platform-manifests` ApplicationSet also uses a Git file generator.

Generator input:

```text
platform-manifests/*/source.yaml
```

No `source.yaml` files exist for the two platform generators yet. Because of
that, both platform ApplicationSets currently generate zero Applications. This
is expected in Phase 4. The platform generator contracts are prepared, but no
platform services are onboarded yet.

## Template And Safety Decisions

All three ApplicationSets use:

- `goTemplate: true`;
- `goTemplateOptions: ["missingkey=error"]`;
- `preserveResourcesOnDeletion: true`.

The `missingkey=error` option makes missing template fields fail early instead
of producing incomplete Application manifests silently.

Project ownership is intentionally fixed in the templates:

- applications from `apps` use `project: dev`;
- applications from the platform generators use `project: platform`.

`CreateNamespace=true` is not used.

The `dev` namespace ownership stays with `bootstrap-root`. The generated
`dev-demo-app` Application manages only application resources inside the `dev`
namespace.

## Multi-Source Application

The generated `dev-demo-app` Application keeps the same multi-source model from
Phase 3.

| Item | Value |
| --- | --- |
| Chart repository | `https://github.com/bertughas123/reusable-helm-charts.git` |
| Chart path | `charts/generic-app-chart` |
| Chart revision | `generic-app-chart-v0.1.0` |
| Values repository | `git@github.com:bertughas123/rke2-gitops.git` |
| Values revision | `main` |
| Values ref | `values` |
| Values file | `$values/apps-values/dev/demo-app/values.yaml` |
| Helm release name | `demo-app` |
| Destination namespace | `dev` |

The `$values` prefix tells Argo CD to load the Helm values file from the second
source, the GitOps repository, while the chart itself comes from the reusable
Helm chart repository.

## Preview And Equivalence Checks

Before the ownership migration, the new ApplicationSet manifests were checked
without changing the cluster state.

Validation results:

- `kubectl` server dry-run completed successfully;
- `argocd appset generate` produced a preview for `dev-demo-app`;
- the existing `dev-demo-app` Application spec was compared with the generated
  Application spec;
- the diff output was empty;
- the generated spec and existing spec matched completely;
- each platform ApplicationSet preview generated zero Applications.

This confirmed that the migration changed the Application creation method, not
the desired workload configuration.

## CLI Situation During Validation

The first `argocd appset generate` attempt returned:

```text
Argo CD server address unspecified
```

This was not a YAML or ApplicationSet template problem. It meant the local
`argocd` CLI did not yet know which Argo CD API server to contact.

A local port-forward to the Argo CD server was opened, and the CLI was then
used with the `admin` user through an interactive login flow.

After login, the Git generator worked successfully. It used the read-only
repository credential already registered in Argo CD.

No password, personal access token or private key value is documented in this
report.

## Safe Ownership Transfer

The live `bootstrap-root` Application auto-sync was temporarily disabled during
the migration.

The Git file `bootstrap/root-app.yaml` was not changed. The pause was only a
live operational step so the old single Application could be removed safely
before the generated Application appeared.

Migration sequence:

- the migration commit was pushed to GitHub;
- the old `dev-demo-app` Application was deleted with a non-cascading delete;
- the command used was `argocd app delete dev-demo-app --cascade=false`;
- the Deployment, Pod, Service and Ingress were not deleted;
- `bootstrap-root` was manually synced once;
- the three ApplicationSets were created;
- the `apps` ApplicationSet generated the `dev-demo-app` Application again.

Because the old Application was deleted without cascading, the existing
Kubernetes resources stayed alive while Argo CD ownership moved to the generated
Application.

No workload interruption was observed.

## Ownership Evidence

The new `dev-demo-app` Application has this owner reference:

```text
ApplicationSet/apps
```

The generated Argo CD `Application` object exists in the `argocd` namespace.

The resources managed by that Application are in the `dev` namespace:

- `Deployment`;
- `Pod`;
- `Service`;
- `ServiceAccount`;
- `Ingress`.

## Final Runtime State

Argo CD state:

| Application | Sync | Health |
| --- | --- | --- |
| `bootstrap-root` | `Synced` | `Healthy` |
| `dev-demo-app` | `Synced` | `Progressing` |

The `Progressing` health state is caused by local Ingress status reporting.
Traefik is running in a hostPort and ClusterIP model, so the Ingress `ADDRESS`
field remains empty. This is a known local environment state and it does not
block application traffic.

Runtime state:

- Deployment is `Ready 1/1`;
- pod is `Running` and `Ready 1/1`;
- restart count is `0`;
- image is `ghcr.io/bertughas123/app-for-gitops:git-e19bd71`;
- Service exists;
- Ingress exists.

Traefik endpoint checks succeeded:

```text
/health/live  -> {"status":"ok"}
/health/ready -> {"status":"ready"}
/api/version  -> {"version":"git-e19bd71"}
```

## Auto-Sync Restored

After the migration, automated sync was enabled again for `bootstrap-root`.

The live Application was verified with:

- `prune: true`;
- `selfHeal: true`.

The desired Git file `bootstrap/root-app.yaml` already contained these settings,
so no persistent bootstrap manifest change was required.

## Git Information

Migration commit:

```text
d06bc6a77f191915c5f4fa49e6d19cc237225801
```

Commit message:

```text
feat: migrate demo app to applicationsets
```

## Security And Scope

No secret, password, personal access token or private key value is included in
the ApplicationSet manifests or in this report.

Argo CD used the existing read-only Kubernetes Secret repository credential for
the GitOps repository.

The local `docs/git-ssh-identity.md` file was not touched and was not committed.

`git add .` was not used.

The `app-for-gitops` and `reusable-helm-charts` repositories were not changed in
Phase 4.

The application image tag stayed fixed:

```text
git-e19bd71
```

The chart tag stayed fixed:

```text
generic-app-chart-v0.1.0
```

## Phase 4 Boundary

Phase 4 did not introduce:

- Jenkins;
- OpenBao;
- External Secrets Operator;
- Redis;
- PostgreSQL;
- production environment.

ApplicationSet was introduced only for controlled Argo CD Application
generation.

## Phase 4 Completion Summary

Phase 4 successfully completed the migration from a single committed Argo CD
Application manifest to an ApplicationSet-generated Application model.

The `dev-demo-app` workload kept the same image, chart, values file, Helm
release name and destination namespace. The migration changed ownership and
generation flow while keeping the deployed application stable.

The result is a safer base for adding more dev applications later: new
applications can be introduced by adding values directories under
`apps-values/dev/`, while the ApplicationSet template keeps the Argo CD
Application structure consistent.
