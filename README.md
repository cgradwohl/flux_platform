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
├── clusters/                  # Cluster-specific Flux configuration
│   ├── dev/
│   │   ├── flux-system/       # Flux bootstrap for dev cluster
│   │   │   ├── gotk-components.yaml
│   │   │   └── gotk-sync.yaml
│   │   ├── beetle/
│   │   │   ├── gitrepository.yaml
│   │   │   └── kustomization.yaml
│   │   ├── sonar/
│   │   │   ├── gitrepository.yaml
│   │   │   └── kustomization.yaml
│   │   └── tiger/
│   │       ├── gitrepository.yaml
│   │       └── kustomization.yaml
│   └── prd/
│       ├── flux-system/       # Flux bootstrap for prod cluster
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       ├── beetle/
│       │   ├── gitrepository.yaml
│       │   └── kustomization.yaml
│       ├── sonar/
│       │   ├── gitrepository.yaml
│       │   └── kustomization.yaml
│       └── tiger/
│           ├── gitrepository.yaml
│           └── kustomization.yaml
│
├── apps/                      # Microservice manifests (managed by platform)
│   ├── beetle/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── kustomization.yaml # references deployment/service, generates ConfigMap
│   │   └── configmap-generator.yaml # optional base generator
│   ├── sonar/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── tiger/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
│
├── infrastructure/            # Cluster-wide resources
│   ├── kyverno/               # admission controllers
│   │   └── ...
│   ├── kyverno-policies/      # example policies
│   │   └── ...
│   ├── crds/                  # custom resource definitions
│   │   └── ...
│   └── namespaces/            # if needed, default namespace can be defined
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

`infrastructure/`

- infrastructure dir contains common infra tools
