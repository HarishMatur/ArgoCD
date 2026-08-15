GitHub Organization
│
├── frontend-service/                 ← Application Repository
│   ├── src/
│   ├── tests/
│   ├── Dockerfile
│   │
│   └── .github/
│       └── workflows/
│           ├── ci.yaml
│           └── security.yaml
│
├── backend-service/                  ← Application Repository
│   ├── src/
│   ├── tests/
│   ├── Dockerfile
│   │
│   └── .github/
│       └── workflows/
│           ├── ci.yaml
│           └── security.yaml
│
└── company-gitops/                   ← GitOps Repository
    │
    ├── apps/
    │   ├── frontend/
    │   │   ├── Chart.yaml
    │   │   ├── values.yaml
    │   │   └── templates/
    │   │       ├── deployment.yaml
    │   │       ├── service.yaml
    │   │       └── ingress.yaml
    │   │
    │   └── backend/
    │       ├── Chart.yaml
    │       ├── values.yaml
    │       └── templates/
    │           ├── deployment.yaml
    │           ├── service.yaml
    │           └── ingress.yaml
    │
    ├── environments/
    │   ├── dev/
    │   │   ├── frontend-values.yaml
    │   │   └── backend-values.yaml
    │   │
    │   └── prod/
    │       ├── frontend-values.yaml
    │       └── backend-values.yaml
    │
    ├── applicationsets/
    │   └── workloads.yaml
    │
    ├── projects/
    │   ├── dev-project.yaml
    │   └── prod-project.yaml
    │
    └── bootstrap/
        └── root-application.yaml