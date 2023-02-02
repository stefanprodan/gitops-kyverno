# gitops-kyverno

Kubernetes' policy managed with Flux and Kyverno

## Prerequisites

You will need a Kubernetes cluster version 1.21 or newer.
For a quick local test, you can use [Kubernetes kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
Any other Kubernetes setup will work as well though.

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on MacOS or Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains a demo app (podinfo) and its configuration for each environment
- **infrastructure** dir contains common infra tools such as Kyverno and its cluster policies
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── production 
│   └── staging
├── infrastructure
│   ├── configs
│   └── controllers
└── clusters
    ├── production
    └── staging
```

## Bootstrap the staging cluster

Create a cluster named staging with kind:

```shell
kind create cluster --name staging
```

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Set the kubectl context to your staging cluster and bootstrap Flux:

```sh
flux bootstrap github \
    --context=kind-staging \
    --owner=${GITHUB_USER} \
    --repository=gitops-kyverno \
    --branch=main \
    --personal \
    --path=clusters/staging
```

The bootstrap command commits the manifests for the Flux components in `clusters/staging/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Wait for Flux to install the controllers and the demo app with:

```shell
watch flux get kustomizations
```

## Access the Flux UI

To access the Flux UI on a cluster, first start port forwarding with:

```sh
kubectl -n flux-system port-forward svc/weave-gitops 9001:9001
```

Navigate to `http://localhost:9001` and login using the username `admin` and the password `flux`.

[Weave GitOps](https://docs.gitops.weave.works/) provides insights into your application deployments,
and makes continuous delivery with Flux easier to adopt and scale across your teams.
The GUI provides a guided experience to build understanding and simplify getting started for new users;
they can easily discover the relationship between Flux objects and navigate to deeper levels of information as required.

## Mutating deployments with Kyverno

In the `infrastructure/configs` dir there are two Kyverno policies that mutate Kubernetes Deployments to set
a restricted security context and to replace the app container image tag with its digest.

Even if in Git, the podinfo image is set to `ghcr.io/stefanprodan/podinfo:6.2.3`, the actual
deployment image in-cluster is mutated by Kyverno and the tag is replaced with the image digest.
Same thing with the security context, in Git, podinfo has no such fields, but in-cluster the deployment ends up with:

```console
$ kubectl -n podinfo get deployments.apps podinfo -oyaml | yq '.spec.template.spec.containers[0]'
image: ghcr.io/stefanprodan/podinfo@sha256:4a72d3ce7eda670b78baadd8995384db29483dfc76e12f81a24e1fc1256c0a8e
imagePullPolicy: IfNotPresent
name: podinfo
ports:
  - containerPort: 9898
    name: http
    protocol: TCP
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

## Flux vs Argo drift detection

If you've been using Argo, and you're switching to Flux, you may expect for Flux to report
the above changes as a drift from Git and reapply the deployment continuously. With Argo, 
you must define policies for each field that Kyverno mutates to ignore differences, while
with Flux this is not need.
Flux works with any admission controller as long the mutation also happens at dry-run.

To test that Flux has included the Kyverno mutations in its "desired state" we can run `flux diff`
and check that no differences are reported:

```shell
 flux diff kustomization apps --path ./apps/staging/
```

Now if we change the podinfo image tag in `apps/staging/kustomization.yaml` to `6.3.0` and we run diff:

```console
$ flux diff kustomization apps --path ./apps/staging/
✓  Kustomization diffing...
► Deployment/podinfo/podinfo drifted

metadata.generation
  ± value change
    - 2
    + 3

spec.template.spec.containers.podinfo.image
  ± value change
    - ghcr.io/stefanprodan/podinfo@sha256:4a72d3ce7eda670b78baadd8995384db29483dfc76e12f81a24e1fc1256c0a8e
    + ghcr.io/stefanprodan/podinfo@sha256:c0d72aa8829a310b998308b8d3f05bde6840c66eaf2862b218f87e21ea5fb275

⚠️ identified at least one change, exiting with non-zero exit code
```

We see that Flux detects a drift in the digest, instead of the image tag. Unlike Argo and other tools,
Flux uses Kubernetes server-side apply dry-run to trigger the mutation webhooks, before it runs the
drift detection algorithm.

With Flux, you don't have to create special rules for each mutation made by admission controllers.

