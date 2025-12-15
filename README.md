# Flux Platform

This repository demonstrates how Platform and DevOps teams can manage a
multi-tenant K8 cluster using git and Flux.

In this example each tenant is an application microservice deployment, which is
managed and configure here. Everything required for the microservice to run on
Kubernetes is defined here. The values that the configmaps are generated from
are the responsibility of the developer team, who has no knowledge of this
repository.

The developer team owns and manages their application config values and schemas
here: https://github.com/cgradwohl/flux_developer

## A Note about manifest templates

Its tempting to create a single template set for all microservice manifests.

In this paradigm or example microservices cannot be deployed using the same
kubernetes manifest templates. If it did then there would be a requirement for
additional logic to apply to both the environment and service data.

Instead we will split this these service <> environment concerns across
repository functionality.

```bash
flux_platform/
├── clusters/                         # 🔑 Cluster wiring & specialization (ONLY place env/service varies)
│   ├── dev/
│   │   ├── flux-system/              # Flux bootstrap (managed by flux)
│   │   │   ├── gotk-components.yaml
│   │   │   └── gotk-sync.yaml
│   │   │
│   │   ├── beetle/                   # Service specialization (dev)
│   │   │   ├── gitrepository.yaml    # points to flux_platform
│   │   │   └── kustomization.yaml    # renders apps/base + dev values
│   │   │
│   │   ├── sonar/
│   │   │   ├── gitrepository.yaml
│   │   │   └── kustomization.yaml
│   │   │
│   │   └── tiger/
│   │       ├── gitrepository.yaml
│   │       └── kustomization.yaml
│   │
│   └── prd/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       │
│       ├── beetle/                   # Service specialization (prd)
│       │   ├── gitrepository.yaml
│       │   └── kustomization.yaml
│       │
│       ├── sonar/
│       │   ├── gitrepository.yaml
│       │   └── kustomization.yaml
│       │
│       └── tiger/
│           ├── gitrepository.yaml
│           └── kustomization.yaml
│
├── apps/                             # ♻️ Single reusable manifest base (NO env, NO service)
│   └── base/
│       ├── deployment.yaml           # generic Deployment template
│       ├── service.yaml              # generic Service template
│       ├── kustomization.yaml        # composes manifests
│       └── configmap-generator.yaml  # consumes flux_developer values
│
├── infrastructure/                   # 🧱 Cluster-wide platform components
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
