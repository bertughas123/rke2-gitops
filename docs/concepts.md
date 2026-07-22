# Concepts

## Application Repository vs GitOps Repository vs Chart Repository

The application repository answers what should be built.

The GitOps repository answers what should run in the cluster and with which
configuration.

The reusable chart repository answers which common Helm templates can be shared
by multiple GitOps repositories.

## CI vs CD

Jenkins performs CI tasks:

- checkout;
- lint;
- tests;
- image build;
- security scan;
- image push;
- GitOps image tag update.

Argo CD performs CD tasks:

- watch the GitOps repository;
- render manifests and Helm charts;
- apply the declared state to Kubernetes;
- report sync and health status.

Jenkins does not deploy directly to the Kubernetes cluster.

## Helm Chart vs Values

A Helm chart contains reusable templates. A values file contains the environment
or application-specific input for those templates.

In this project:

- reusable chart templates live in `reusable-helm-charts`;
- application and environment values live in `rke2-gitops`;
- application source code and Docker image build files live in `app-for-gitops`.

## Application vs ApplicationSet

An Argo CD Application describes one deployment target.

An ApplicationSet generates multiple Applications from a template.

The first demo application will use a plain Application before moving to
ApplicationSet.

## AppProject

An AppProject defines where Applications may deploy, which repositories they may
use and which Kubernetes resources they may create.

In this project:

- `default` is used only for the bootstrap root Application;
- `dev` is used for demo applications;
- `platform` is used for platform components.

## Secret Reference vs Secret Value

Git may contain references to secrets, such as ExternalSecret manifests.

Git must not contain the real secret value.
