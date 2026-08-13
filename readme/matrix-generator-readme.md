# Matrix generator for Git + cluster generators architecture

Git Repository
│
└── manifest/
    │
    ├── frontend/
    │      └── deployment.yaml
    │
    └── backend/
           └── deployment.yaml

          Git Generator
               |
               | discovers folders
               |
        +------+------+
        |             |
    frontend       backend
        |             |
        +------+------+
               |
               X
         Matrix Generator
               X
               |
        +------+------+
        |             |
       DEV           PROD
        |             |
        +------+------+
               |
               v

    Four Applications generated

dev-frontend
dev-backend
prod-frontend
prod-backend



Git Generator
      =
Which application/folder?


Cluster Generator
      =
Which cluster?


Matrix Generator
      =
Which application goes to which cluster?


source.path
      =
Which exact manifests should that Application deploy?





selector:
  matchExpressions:
    - key: environment
      operator: In
      values:
        - dev


This is equivalent in this simple case, but matchExpressions is more useful when you want things like:

values:
  - dev
  - qa

or exclusions using NotIn. Argo CD supports the Kubernetes label-selector operators such as In, NotIn, Exists, and DoesNotExist.

The simplest rule to remember is:

clusters: {}
    =
Deploy to all registered clusters

clusters + matchLabels environment=dev
    =
Deploy only to DEV

clusters + matchLabels environment=prod
    =
Deploy only to PROD