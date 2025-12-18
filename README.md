# Flux Platform

This repository demonstrates how Platform and DevOps teams can manage a
multi-environment K8 cluster using git and Flux.

In this example each environment has exactly one cluster, which hosts three
application microservice deployment.

Everything required for the microservice to run on Kubernetes is defined here,
except configmaps, which are managed and generated in the developer owned
repository: https://github.com/cgradwohl/flux_developer

## A Note about manifest templates

Its tempting to create a single template set for all microservice manifests.

In this paradigm or example microservices cannot be deployed using the same
kubernetes manifest templates. If it did then there would be a requirement for
additional logic to apply to both the environment and service data.

Instead we will split this these service <> environment concerns across
repository functionality.

```bash
flux_platform/
├── clusters/
│   ├── dev/
│   │   ├── flux-system/                     # flux bootstrap (managed by flux)
│   │   │   ├── gotk-components.yaml
│   │   │   └── gotk-sync.yaml
│   │   │
│   │   ├── gitrepository-platform.yaml      # points to flux_platform repo
│   │   ├── gitrepository-developer.yaml     # points to flux_developer repo
│   │   │
│   │   ├── beetle/
│   │   │   └── kustomization-developer.yaml # renders and deploys configmaps
│   │   │   └── kustomization-platform.yaml  # renders and deploys manifests
│   │   │
│   │   ├── sonar/
│   │   │   ├── kustomization-developer.yaml
│   │   │   └── kustomization-platform.yaml
│   │   │
│   │   └── tiger/
│   │       ├── kustomization-developer.yaml
│   │       └── kustomization-platform.yaml
│   │
│   └── prd/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       ├── beetle/
│       ├── sonar/
│       └── tiger/
│
├── apps/
│   └── base/
│       ├── deployment.yaml           # generic Deployment template
│       ├── service.yaml              # generic Service template
│       ├── kustomization.yaml        # composes manifests
│       └── configmap-generator.yaml  # consumes flux_developer values
│
├── infrastructure/                   # shared, cluster wide platform components
│   ├── namespaces/
│   │   └── apps.yaml
│   ├── crds/
│   │   └── ...
│   ├── kyverno/
│   │   └── ...
│   ├── kyverno-policies/
│   │   └── ...
│   └── ingress/
│       └── ...
│
└── README.md

```

`apps/`

- Kubernetes manifests, Kustomizations, ConfigMap generators
- Organize by microservice, not by environment.
- Keep base manifests for deployments, services, configmaps per microservice
  here.
- Environment differences come from ConfigMap generators or patches that point
  to the tenant repo.

`clusters/`

- Flux CRDs for each environment
- Organize by environment, not by microservice.
- Each cluster folder contains Flux bootstrap, GitRepository, and Kustomization
  CRDs pointing to the apps repo.
- Where service identity comes from, i.e.
  `clusters/env/service/kustomization.yaml`

`infrastructure/`

- infrastructure dir contains common infra tools

## Rendering ConfigMaps

In this demo, we require that the developer owned repository, `flux_developer`,
has no knowledge of Kubernetes or Flux CD resources. Developer teams only
provide structured YAML configuration files and kustomize files, organized by
service and environment:

```bash
flux_developer/
├── beetle/
│   ├── base/
│   │   └── config.yaml
│   ├── dev/
│   │   └── config.yaml
│   │   └── kustomization.yaml
│   └── prd/
│       └── config.yaml
│   │   └── kustomization.yaml
├── sonar/
│   ├── base/
│   ├── dev/
│   └── prd/
└── tiger/
    ├── base/
    ├── dev/
    └── prd/
```

The GitOps pipeline is defined:
`clusters/dev/beetle/kustomization-developer.yaml`

```yaml
# flux_platform/clusters/dev/beetle/kustomization-developer.yaml
# --------------------------------------------------------------

apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: beetle-dev-config
  namespace: flux-system
spec:
  interval: 1m
  prune: true
  wait: true

  sourceRef:
    kind: GitRepository
    name: flux-developer

  # points to the dev folder in the developer repo aka the folder containing
  # kustomization.yaml
  path: ./beetle/dev

  postBuild:
    substitute:
      SERVICE_NAME: beetle
      ENV: dev
```

Note that this Kustomization references the flux-developer GitRepository, which
is defined at `clusters/dev/gitrepository-developer.yaml`

```yaml
# flux_platform/clusters/dev/gitrepository-developer.yaml
# --------------------------------------------------------

apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: flux-developer
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/cgradwohl/flux_developer
  ref:
    branch: main
```

The actual ConfigMap is generated inside `flux_developer` by a Kustomize
configMapGenerator:

```yaml
# flux_developer/beetle/dev/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
  - name: beetle-config
    files:
      - ../../base/config.yaml # base defaults
      - config.yaml # dev overrides
```

Flux monitors the `flux_developer` repo via the GitRepository CRD.

When either `base/config.yaml` or `dev/config.yaml` changes, Flux sees a new
GitRepository artifact.

The Kustomization CRD in `flux_platform` builds the ConfigMap by executing the
kustomization.yaml in `flux_developer/dev`.

The resulting ConfigMap is applied to the cluster.

This preserves the developer-only control over configuration while allowing Flux
to handle deployment automation.

## Rendering Kubernetes Manifests

## Example Files

In this example repoisotry setup, a differnet, corresponding repository defines
all the kustomizations and kubernetes manifests required for the microservice
deployment.

`dev/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: apps
resources:
  - github.com/your-org/flux_developer//base?ref=main
configMapGenerator:
  - name: dev-beetle-config
    files:
      - github.com/your-org/tenant-repo//base/beetle/config.yaml
      - github.com/your-org/tenant-repo//dev/beetle/config.yaml
```

`prd/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: apps
resources:
  - github.com/your-org/flux_developer//base?ref=main
configMapGenerator:
  - name: prd-beetle-config
    files:
      - github.com/your-org/tenant-repo//base/beetle/config.yaml
      - github.com/your-org/tenant-repo//prd/beetle/config.yaml
```

---

## Flux Bootstrapping

1. start two minikube clusters

```bash
minikube start -p minikube-dev

minikube start -p minikube-prd

```

2. expose PAT

```bash
export GITHUB_TOKEN="your pat token"
```

3. bootstrap dev cluster

```bash
k config use-context minikube-dev

flux bootstrap github \
  --owner=cgradwohl \
  --repository=flux_platform \
  --branch=main \
  --path=clusters/dev \
  --personal=false
```

4. bootstrap prd cluster

```bash
k config use-context minikube-prd

flux bootstrap github \
  --owner=cgradwohl \
  --repository=flux_platform \
  --branch=main \
  --path=clusters/prd \
  --personal=false
```

5. verify

```bash
cd flux_platform && git pull

kubectl config use-context dev
kubectl get pods -n flux-system

kubectl config use-context prd
kubectl get pods -n flux-system
```
