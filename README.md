# 🚀 Argo CD ApplicationSet — Complete End-to-End Guide

[![Argo CD](https://img.shields.io/badge/Argo%20CD-ApplicationSet-orange)](https://argo-cd.readthedocs.io/)
![Kubernetes](https://img.shields.io/badge/Kubernetes-GitOps-blue)
![GitOps](https://img.shields.io/badge/GitOps-Automation-success)

> A practical and detailed end-to-end guide to **Argo CD ApplicationSet**, covering core concepts, generators, multi-application and multi-environment deployment patterns, cluster targeting, Matrix and Merge generators, troubleshooting, and production best practices.

---

## 📚 Table of Contents

* [Overview](#overview)
* [Why ApplicationSet?](#why-applicationset)
* [Application vs App of Apps vs ApplicationSet](#application-vs-app-of-apps-vs-applicationset)
* [ApplicationSet Architecture](#applicationset-architecture)
* [ApplicationSet Basic Structure](#applicationset-basic-structure)
* [Namespace Concepts](#namespace-concepts)
* [Go Templates](#go-templates)
* [Generators](#generators)

  * [List Generator](#1-list-generator)
  * [Git Directory Generator](#2-git-directory-generator)
  * [Git File Generator](#3-git-file-generator)
  * [Cluster Generator](#4-cluster-generator)
  * [Matrix Generator](#5-matrix-generator)
  * [Merge Generator](#6-merge-generator)
* [Generator Comparison](#generator-comparison)
* [Multi-App Multi-Environment Design](#multi-app--multi-environment-design)
* [Cluster Selection and Filtering](#cluster-selection-and-filtering)
* [Revision vs TargetRevision](#revision-vs-targetrevision)
* [ApplicationSet and Helm](#applicationset-and-helm)
* [Production Repository Structure](#production-repository-structure)
* [Production Best Practices](#production-best-practices)
* [Useful Commands](#useful-commands)
* [Troubleshooting](#troubleshooting)
* [Frequently Asked Questions](#frequently-asked-questions)
* [Quick Reference](#quick-reference)
* [Learning Status](#learning-status)

---

# Overview

An **ApplicationSet** is an Argo CD resource used to automatically generate and manage multiple Argo CD `Application` resources from a common template.

Without ApplicationSet, we may manually maintain application manifests such as:

```text
frontend-dev.yaml
frontend-qa.yaml
frontend-prod.yaml

backend-dev.yaml
backend-qa.yaml
backend-prod.yaml
```

As the number of applications, environments, and Kubernetes clusters grows, managing one Argo CD `Application` manifest per deployment becomes repetitive and difficult to maintain.

ApplicationSet solves this problem by introducing:

```text
Generator
    +
Application Template
    ↓
Generated Argo CD Applications
```

For example:

```text
10 applications
      ×
3 environments
      =
30 Argo CD Applications
```

Instead of manually maintaining 30 Application YAML files, ApplicationSet can generate them dynamically.

> [!TIP]
> Think of ApplicationSet as an **Application factory**.
>
> Generators provide the data, and the ApplicationSet template defines how every generated Argo CD Application should look.

---

# Why ApplicationSet?

Consider the following platform:

```text
Applications
------------
frontend
backend
payment
orders


Environments
------------
DEV
QA
PROD
```

The required Argo CD Applications would be:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod

payment-dev
payment-qa
payment-prod

orders-dev
orders-qa
orders-prod
```

This results in:

```text
4 Applications × 3 Environments
              =
12 Argo CD Applications
```

Without ApplicationSet, we would maintain 12 separate Application manifests.

With ApplicationSet, we define the rules once and let Argo CD generate the required Applications.

### Main benefits

* Reduces Application YAML duplication
* Automates application onboarding
* Automates environment onboarding
* Simplifies multi-cluster deployments
* Enables Git-driven application discovery
* Standardizes generated Applications
* Improves scalability
* Simplifies large GitOps repositories
* Allows application/environment combinations
* Supports default and override patterns
* Makes cluster-based targeting easier

---

# Application vs App of Apps vs ApplicationSet

Understanding the difference between these three concepts is important.

## Argo CD Application

An `Application` represents one Argo CD-managed deployment.

```text
Application
     |
     v
Git Repository
     |
     v
Kubernetes Cluster
```

Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: frontend
  namespace: argocd

spec:
  project: applications

  source:
    repoURL: https://github.com/example/gitops.git
    targetRevision: main
    path: apps/frontend

  destination:
    server: https://kubernetes.default.svc
    namespace: frontend
```

---

## App of Apps

App of Apps uses one **root Application** to deploy other Application manifests.

```text
Root Application
       |
       +------> frontend Application
       |
       +------> backend Application
       |
       +------> payment Application
```

Example repository:

```text
app-of-apps/
├── bootstrap/
│   └── root-application.yaml
│
├── applications/
│   ├── frontend.yaml
│   ├── backend.yaml
│   └── payment.yaml
│
└── manifests/
    ├── frontend/
    ├── backend/
    └── payment/
```

The child Application YAML files still exist and are maintained manually.

---

## ApplicationSet

ApplicationSet generates child Applications automatically.

```text
ApplicationSet
      |
      | Generator
      |
      +------> Application
      |
      +------> Application
      |
      +------> Application
```

The individual generated Application YAML files do not necessarily need to exist in Git.

---

## Comparison

| Feature                              | Application | App of Apps                    | ApplicationSet         |
| ------------------------------------ | ----------- | ------------------------------ | ---------------------- |
| Manages workloads                    | Yes         | Through child apps             | Through generated apps |
| Generates Applications               | No          | No, deploys existing manifests | Yes                    |
| Individual Application YAML required | Yes         | Yes                            | Not always             |
| Good for many applications           | Limited     | Good                           | Excellent              |
| Dynamic application discovery        | No          | No                             | Yes                    |
| Multi-cluster generation             | Manual      | Manual                         | Automated              |
| App × Environment combinations       | Manual      | Manual                         | Matrix Generator       |

---

# ApplicationSet Architecture

The high-level architecture is:

```text
                    ApplicationSet
                          |
             +------------+-------------+
             |                          |
         Generator                   Template
             |                          |
             |                          |
     Generate parameters       Define Application
             |                          |
             +------------+-------------+
                          |
                          v
                Generated Applications
                          |
              +-----------+-----------+
              |                       |
              v                       v
          Application             Application
              |                       |
              v                       v
         Kubernetes              Kubernetes
          Cluster                  Cluster
```

The key idea is:

```text
ApplicationSet
      ↓
generates
      ↓
Application
      ↓
deploys
      ↓
Kubernetes Resources
```

> [!IMPORTANT]
> ApplicationSet itself does **not** directly deploy Pods, Deployments, or Services.
>
> It generates Argo CD `Application` resources.
>
> The normal Argo CD Application Controller deploys the actual Kubernetes workloads.

---

# ApplicationSet Basic Structure

A basic ApplicationSet looks like:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:
  name: example-appset
  namespace: argocd

spec:

  goTemplate: true

  goTemplateOptions:
    - missingkey=error

  generators:
    # Generator configuration

  template:

    metadata:
      name: '{{.appName}}'

    spec:

      project: applicationset

      source:
        repoURL: https://github.com/example/gitops.git
        targetRevision: main
        path: '{{.path}}'

      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.namespace}}'

      syncPolicy:

        automated:
          prune: true
          selfHeal: true

        syncOptions:
          - CreateNamespace=true
```

There are two major sections:

```text
generators
    ↓
Where does the data come from?


template
    ↓
What should the generated Application look like?
```

---

# Namespace Concepts

This is one of the most important ApplicationSet concepts.

Consider:

```yaml
metadata:
  namespace: argocd
```

and:

```yaml
template:
  spec:
    destination:
      namespace: apps-dev
```

These mean completely different things.

## `metadata.namespace`

```yaml
metadata:
  namespace: argocd
```

means:

> The ApplicationSet resource belongs to the `argocd` namespace.

The Argo CD management objects normally live in the control plane:

```text
Argo CD Cluster

namespace: argocd

├── ApplicationSet
├── Application/frontend-dev
├── Application/backend-dev
├── Application/frontend-prod
└── Application/backend-prod
```

---

## `destination.namespace`

```yaml
destination:
  namespace: apps-dev
```

means:

> Deploy the actual application workloads into the `apps-dev` namespace.

Example:

```text
Application/frontend-dev
        |
        | destination
        v

DEV Kubernetes Cluster
        |
        └── namespace: apps-dev
              |
              ├── Deployment
              ├── Service
              ├── ConfigMap
              └── Pods
```

Therefore:

```text
metadata.namespace
        ↓
Where Argo CD management resource lives


destination.namespace
        ↓
Where actual workload runs
```

> [!WARNING]
> Do not set `template.spec.destination.namespace: argocd` unless the actual Kubernetes workload genuinely needs to run in the Argo CD namespace.

---

# Go Templates

A recommended ApplicationSet configuration is:

```yaml
spec:

  goTemplate: true

  goTemplateOptions:
    - missingkey=error
```

## `goTemplate: true`

This enables Go templating.

For example:

```yaml
name: '{{.appName}}'
```

or:

```yaml
namespace: '{{.environment}}'
```

or:

```yaml
server: '{{.server}}'
```

Generator values are substituted into the template.

Example generator data:

```yaml
appName: frontend
namespace: frontend-dev
```

Template:

```yaml
metadata:
  name: '{{.appName}}'

spec:
  destination:
    namespace: '{{.namespace}}'
```

Generated result:

```yaml
metadata:
  name: frontend

spec:
  destination:
    namespace: frontend-dev
```

---

## `missingkey=error`

Consider:

```yaml
metadata:
  name: '{{.appName}}'
```

but the generator does not produce `appName`.

With:

```yaml
goTemplateOptions:
  - missingkey=error
```

ApplicationSet reports an error instead of silently rendering an invalid value.

> [!TIP]
> `missingkey=error` is useful in production because template mistakes fail early.

---

# Generators

ApplicationSet generators determine **what data is available to generate Applications**.

The main generators covered in this guide are:

```text
List
Git Directory
Git File
Cluster
Matrix
Merge
```

---

# 1. List Generator

The List Generator provides a manually defined list of parameters.

Example:

```yaml
generators:

  - list:

      elements:

        - appName: frontend
          path: apps/frontend
          namespace: frontend

        - appName: backend
          path: apps/backend
          namespace: backend
```

Template:

```yaml
template:

  metadata:
    name: '{{.appName}}'

  spec:

    source:
      repoURL: https://github.com/example/gitops.git
      targetRevision: main
      path: '{{.path}}'

    destination:
      server: https://kubernetes.default.svc
      namespace: '{{.namespace}}'
```

Generated Applications:

```text
frontend
backend
```

### Commented example

```yaml
generators:

  # List Generator creates one generated Application
  # for each item defined under elements.
  - list:

      elements:

        # Values that can be referenced using
        # {{.appName}}, {{.path}}, and {{.namespace}}.
        - appName: frontend
          path: apps/frontend
          namespace: frontend

        - appName: backend
          path: apps/backend
          namespace: backend
```

### When to use List Generator

Use it when:

* The number of items is small
* Data is static
* Explicit control is required
* Dynamic discovery isn't required
* You're learning ApplicationSet

---

# 2. Git Directory Generator

Git Directory Generator discovers application directories automatically.

Suppose the repository contains:

```text
apps/
├── frontend/
│   ├── deployment.yaml
│   └── service.yaml
│
├── backend/
│   ├── deployment.yaml
│   └── service.yaml
│
├── payment/
│   └── deployment.yaml
│
└── orders/
    └── deployment.yaml
```

Generator:

```yaml
generators:

  - git:
      repoURL: https://github.com/example/gitops.git
      revision: main

      directories:
        - path: apps/*
```

The generator discovers:

```text
frontend
backend
payment
orders
```

Template example:

```yaml
template:

  metadata:
    name: '{{.path.basename}}'

  spec:

    source:
      repoURL: https://github.com/example/gitops.git
      targetRevision: main
      path: '{{.path.path}}'
```

Conceptually:

```text
apps/frontend
      ↓
frontend Application


apps/backend
      ↓
backend Application
```

### Useful path values

Typical path-based values include:

```text
.path.path
.path.basename
```

Example:

```text
Matched directory:
apps/frontend

.path.path
=
apps/frontend

.path.basename
=
frontend
```

### When to use Git Directory Generator

Use it when:

```text
one directory = one application
```

and you want applications discovered automatically.

---

# 3. Git File Generator

Git File Generator reads structured configuration files from Git.

Example repository:

```text
configs/
├── dev.yaml
├── qa.yaml
└── prod.yaml
```

`dev.yaml`:

```yaml
environment: dev
namespace: apps-dev
revision: develop
replicas: "1"
```

`qa.yaml`:

```yaml
environment: qa
namespace: apps-qa
revision: release
replicas: "2"
```

`prod.yaml`:

```yaml
environment: prod
namespace: apps-prod
revision: main
replicas: "5"
```

Generator:

```yaml
generators:

  - git:
      repoURL: https://github.com/example/gitops.git
      revision: main

      files:
        - path: configs/*.yaml
```

The values from the file can be consumed in the template:

```yaml
metadata:
  name: 'nginx-{{.environment}}'
```

```yaml
source:
  targetRevision: '{{.revision}}'
```

```yaml
destination:
  namespace: '{{.namespace}}'
```

### Git Directory vs Git File

```text
Git Directory Generator
        ↓
"What application directories exist?"


Git File Generator
        ↓
"What configuration values are defined in Git?"
```

### When Git File Generator is useful

Use it for:

* Environment configuration
* Namespace selection
* Revision/branch selection
* Helm values file selection
* Deployment metadata
* Feature/configuration values
* Release channels
* Per-environment settings

---

# 4. Cluster Generator

The Cluster Generator discovers clusters registered with Argo CD.

Basic configuration:

```yaml
generators:

  - clusters: {}
```

Suppose Argo CD knows:

```text
dev-cluster
qa-cluster
prod-cluster
```

Then the Cluster Generator can produce one generated Application per cluster.

Template:

```yaml
metadata:
  name: '{{.nameNormalized}}-nginx'
```

Destination:

```yaml
destination:
  server: '{{.server}}'
  namespace: applications
```

Generated Applications:

```text
dev-cluster-nginx
qa-cluster-nginx
prod-cluster-nginx
```

### Important cluster parameters

Common parameters include:

```text
.name
.nameNormalized
.server
.metadata.labels
.metadata.annotations
```

Example:

```yaml
metadata:
  name: '{{.nameNormalized}}-nginx'
```

### Why `nameNormalized`?

If a cluster is named:

```text
AKS_DEV_CLUSTER
```

that value may not be valid in all Kubernetes resource-name contexts.

`nameNormalized` provides a safer DNS-style form.

Example:

```text
aks-dev-cluster
```

### When to use Cluster Generator

Use it when:

* Argo CD manages multiple clusters
* Cluster inventory should come from Argo CD
* Applications should be deployed dynamically to registered clusters
* Cluster labels determine workload placement

---

# Cluster Selection and Filtering

Suppose Argo CD manages:

```text
dev-cluster
prod-cluster
```

but you only want to deploy to DEV.

Label the Argo CD cluster Secret:

```bash
kubectl label secret <dev-cluster-secret> \
  -n argocd \
  environment=dev
```

PROD:

```bash
kubectl label secret <prod-cluster-secret> \
  -n argocd \
  environment=prod
```

Then:

```yaml
generators:

  - clusters:

      selector:

        matchLabels:
          environment: dev
```

Only DEV is selected.

For PROD:

```yaml
selector:

  matchLabels:
    environment: prod
```

You can also use:

```yaml
selector:

  matchExpressions:

    - key: environment
      operator: In
      values:
        - dev
        - qa
```

---

# 5. Matrix Generator

Matrix combines the output of two child generators.

Think:

```text
A × B
```

Example:

```text
Applications
------------
frontend
backend

×

Environments
------------
dev
qa
prod
```

Result:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod
```

Therefore:

```text
2 × 3 = 6 generated Applications
```

---

## Git Directory + Git File Matrix

This is a common and clean design.

Repository:

```text
applicationset/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   │
│   └── backend/
│       ├── deployment.yaml
│       └── service.yaml
│
├── environments/
│   ├── dev.yaml
│   ├── qa.yaml
│   └── prod.yaml
│
└── applicationset.yaml
```

Application discovery:

```yaml
- git:

    repoURL: https://github.com/example/gitops.git
    revision: main

    directories:

      - path: applicationset/apps/*
```

Environment discovery:

```yaml
- git:

    repoURL: https://github.com/example/gitops.git
    revision: main

    files:

      - path: applicationset/environments/*.yaml
```

Matrix:

```yaml
generators:

  - matrix:

      generators:

        # Discover applications
        - git:

            repoURL: https://github.com/example/gitops.git
            revision: main

            directories:

              - path: applicationset/apps/*

        # Read environments/configuration
        - git:

            repoURL: https://github.com/example/gitops.git
            revision: main

            files:

              - path: applicationset/environments/*.yaml
```

Template:

```yaml
template:

  metadata:
    name: '{{.path.basename}}-{{.environment}}'

  spec:

    source:

      repoURL: https://github.com/example/gitops.git

      targetRevision: '{{.revision}}'

      path: '{{.path.path}}'

    destination:

      server: https://kubernetes.default.svc

      namespace: '{{.namespace}}'
```

Generated Applications:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod
```

---

## 4 apps × 3 environments

Suppose Git Directory Generator discovers:

```text
frontend
backend
payment
orders
```

and Git File Generator discovers:

```text
dev
qa
prod
```

Matrix generates:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod

payment-dev
payment-qa
payment-prod

orders-dev
orders-qa
orders-prod
```

Total:

```text
4 × 3 = 12 generated Applications
```

### Matrix rule

Ask:

> Do I want every output from generator A combined with every output from generator B?

If yes, Matrix is appropriate.

---

# 6. Merge Generator

Merge Generator solves a completely different problem.

The easiest way to remember it:

```text
Matrix
=
COMBINE


Merge
=
MATCH + OVERLAY
```

Suppose the base configuration is:

```text
DEV
revision=main
replicas=2

QA
revision=main
replicas=2

PROD
revision=main
replicas=2
```

But you need:

```text
DEV
revision=develop

PROD
replicas=10
```

Merge can match items by:

```yaml
mergeKeys:

  - environment
```

Example:

```yaml
generators:

  - merge:

      mergeKeys:

        - environment

      generators:

        # Base configuration
        - list:

            elements:

              - environment: dev
                revision: main
                replicas: "2"

              - environment: qa
                revision: main
                replicas: "2"

              - environment: prod
                revision: main
                replicas: "2"

        # Overrides
        - list:

            elements:

              - environment: dev
                revision: develop

              - environment: prod
                replicas: "10"
```

Final result:

```text
DEV
revision=develop
replicas=2


QA
revision=main
replicas=2


PROD
revision=main
replicas=10
```

The output count remains:

```text
3
```

It does not become:

```text
3 × 2
```

because Merge does not create a Cartesian product.

### When to use Merge Generator

Use Merge when you have:

```text
base/default configuration
        +
specific exceptions/overrides
```

Typical examples:

* PROD-specific values
* Region-specific overrides
* Cluster-specific values
* Revision overrides
* Helm values overrides
* Feature overrides
* Deployment policy differences

---

# Matrix vs Merge

| Matrix                         | Merge                           |
| ------------------------------ | ------------------------------- |
| Combines generator outputs     | Matches generator outputs       |
| Creates Cartesian product      | Uses `mergeKeys`                |
| Usually increases result count | Usually keeps base result count |
| Apps × Environments            | Defaults + Overrides            |
| Apps × Clusters                | Cluster-specific exceptions     |
| No matching key required       | Matching key required           |

### Memory shortcut

```text
MATRIX
A × B


MERGE
A + matching B
```

---

# Generator Comparison

| Generator     | Main Question                        | Typical Use            |
| ------------- | ------------------------------------ | ---------------------- |
| List          | What static items did I define?      | Small/static use cases |
| Git Directory | What application directories exist?  | App discovery          |
| Git File      | What configuration files define?     | Env/config management  |
| Cluster       | What clusters does Argo CD know?     | Multi-cluster          |
| Matrix        | How do I combine A × B?              | Apps × envs/clusters   |
| Merge         | How do I apply defaults + overrides? | Exceptions             |

---

# Multi-App Multi-Environment Design

A recommended pattern:

```text
                    Git Repository
                          |
            +-------------+-------------+
            |                           |
          apps/*                  environments/*
            |                           |
            v                           v
     Git Directory                 Git File
       Generator                   Generator
            |                           |
            +-------------+-------------+
                          |
                          v
                       MATRIX
                          |
                          v
                Generated Applications
                          |
              +-----------+-----------+
              |           |           |
             DEV          QA         PROD
```

Example:

```text
4 Applications × 3 Environments
              =
12 Applications
```

---

# Complete Multi-App Multi-Environment Example

Repository:

```text
applicationset/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   │
│   └── backend/
│       ├── deployment.yaml
│       └── service.yaml
│
├── environments/
│   ├── dev.yaml
│   ├── qa.yaml
│   └── prod.yaml
│
└── applicationset.yaml
```

DEV configuration:

```yaml
environment: dev
namespace: apps-dev
revision: develop
```

QA:

```yaml
environment: qa
namespace: apps-qa
revision: release
```

PROD:

```yaml
environment: prod
namespace: apps-prod
revision: main
```

ApplicationSet:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet

metadata:

  name: platform-applications
  namespace: argocd

spec:

  goTemplate: true

  goTemplateOptions:

    - missingkey=error

  generators:

    - matrix:

        generators:

          # -------------------------------------
          # Discover applications
          # -------------------------------------
          - git:

              repoURL: https://github.com/example/gitops.git

              revision: main

              directories:

                - path: applicationset/apps/*


          # -------------------------------------
          # Read environment configuration
          # -------------------------------------
          - git:

              repoURL: https://github.com/example/gitops.git

              revision: main

              files:

                - path: applicationset/environments/*.yaml

  template:

    metadata:

      name: '{{.path.basename}}-{{.environment}}'

      labels:

        environment: '{{.environment}}'

        managed-by: applicationset

    spec:

      project: applicationset

      source:

        repoURL: https://github.com/example/gitops.git

        targetRevision: '{{.revision}}'

        path: '{{.path.path}}'

      destination:

        server: https://kubernetes.default.svc

        namespace: '{{.namespace}}'

      syncPolicy:

        automated:

          prune: true

          selfHeal: true

        syncOptions:

          - CreateNamespace=true

          - ApplyOutOfSyncOnly=true

          - PruneLast=true

      revisionHistoryLimit: 10
```

Expected generated Applications:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod
```

---

# Revision vs TargetRevision

This is a common source of confusion.

You may see:

```yaml
git:

  repoURL: ...

  revision: main
```

and:

```yaml
source:

  repoURL: ...

  targetRevision: main
```

They have different responsibilities.

## Generator `revision`

```yaml
git:
  revision: main
```

means:

> ApplicationSet should inspect the `main` revision while discovering directories/files.

Think:

```text
DISCOVERY
```

---

## Application `targetRevision`

```yaml
source:
  targetRevision: main
```

means:

> The generated Application should deploy manifests from `main`.

Think:

```text
DEPLOYMENT
```

Therefore:

```text
Generator
revision
    ↓
Where ApplicationSet discovers data


Generated Application
targetRevision
    ↓
Which revision Argo CD deploys
```

> [!IMPORTANT]
> Do not assume that writing `revision: main` automatically exposes `{{.revision}}` as a template variable.
>
> Generator configuration and generator-generated parameters are separate concepts.

---

# ApplicationSet and Helm

Suppose an environment configuration contains:

```yaml
environment: prod
namespace: apps-prod
replicas: "5"
```

ApplicationSet can use `replicas` while templating the generated `Application`.

However, this does **not automatically change** a plain Kubernetes manifest such as:

```yaml
spec:
  replicas: 2
```

ApplicationSet templates the Argo CD `Application`, not arbitrary Kubernetes fields inside plain YAML.

Helm can bridge this gap.

ApplicationSet:

```yaml
source:

  repoURL: https://github.com/example/gitops.git

  targetRevision: '{{.revision}}'

  path: '{{.path.path}}'

  helm:

    parameters:

      - name: replicaCount
        value: '{{.replicas}}'
```

Helm Deployment template:

```yaml
spec:

  replicas: {{ .Values.replicaCount }}
```

Then:

```text
prod.yaml

replicas=5

      ↓

Git File Generator

      ↓

ApplicationSet

      ↓

Helm parameter

replicaCount=5

      ↓

Deployment

replicas=5
```

---

# Production Repository Structure

A scalable repository might look like:

```text
gitops/
│
├── applicationset/
│   └── applicationset.yaml
│
├── apps/
│   │
│   ├── frontend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   │
│   ├── backend/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │
│   ├── payment/
│   │
│   └── orders/
│
└── environments/
    ├── dev.yaml
    ├── qa.yaml
    └── prod.yaml
```

Responsibilities:

```text
apps/
   ↓
What applications exist?


environments/
   ↓
What environment configuration should be used?


ApplicationSet
   ↓
How should Applications be generated?
```

---

# Production Best Practices

> [!IMPORTANT]
> ApplicationSet can create and delete large numbers of Applications automatically. Treat changes to ApplicationSet manifests as production infrastructure changes.

Recommended practices:

* Enable `goTemplate: true`
* Use `missingkey=error`
* Use AppProjects to restrict repositories and destinations
* Avoid unrestricted `*` permissions unless necessary
* Use cluster labels for workload targeting
* Separate application definitions from environment configuration
* Never store credentials directly in Git
* Protect production branches
* Require pull requests and code review
* Carefully review `prune`
* Understand deletion behavior before removing generator entries
* Use dedicated namespaces for workloads
* Avoid deploying ordinary workloads to `argocd`
* Use Helm or Kustomize for environment-specific workload configuration
* Prefer immutable production release references where appropriate
* Use AppProject restrictions for production clusters
* Consider sync windows for critical production environments
* Test ApplicationSet changes in non-production first
* Review generated Applications before applying large changes
* Use clear labels such as:

  * environment
  * region
  * team
  * platform
  * workload type
* Define ownership between platform teams and application teams

---

# Useful Commands

## Verify ApplicationSet CRD

```bash
kubectl get crd applicationsets.argoproj.io
```

---

## List ApplicationSets

```bash
kubectl get applicationsets -n argocd
```

or:

```bash
argocd appset list
```

---

## Describe ApplicationSet

```bash
kubectl describe applicationset <name> -n argocd
```

---

## Inspect ApplicationSet

```bash
kubectl get applicationset <name> \
  -n argocd \
  -o yaml
```

---

## View generated Applications

```bash
kubectl get applications -n argocd
```

---

## Inspect generated Application

```bash
kubectl get application frontend-dev \
  -n argocd \
  -o yaml
```

---

## Watch Applications

```bash
kubectl get applications \
  -n argocd \
  -w
```

---

## ApplicationSet controller logs

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-applicationset-controller
```

---

## List Argo CD clusters

```bash
argocd cluster list
```

---

## List registered cluster Secrets

```bash
kubectl get secrets \
  -n argocd \
  -l argocd.argoproj.io/secret-type=cluster
```

---

# Troubleshooting

## Generated Applications are not appearing

Check:

```bash
kubectl describe applicationset <name> -n argocd
```

Then inspect controller logs:

```bash
kubectl logs \
  -n argocd \
  deployment/argocd-applicationset-controller
```

Common causes:

```text
Incorrect Git path

Incorrect repository URL

Repository authentication problem

Missing template variable

Incorrect Go template syntax

Invalid generator configuration

Invalid AppProject

Destination not allowed

Cluster selector does not match

Invalid configuration file
```

---

## Template variable missing

Example:

```yaml
namespace: '{{.namespace}}'
```

requires the generator to produce:

```text
namespace
```

When:

```yaml
goTemplateOptions:
  - missingkey=error
```

is enabled, missing variables generate an explicit error.

---

## Destination not allowed

Example error:

```text
application destination server and namespace
do not match any allowed destinations
```

Check the AppProject:

```yaml
spec:

  destinations:

    - server: https://kubernetes.default.svc
      namespace: apps-dev

    - server: https://kubernetes.default.svc
      namespace: apps-qa

    - server: https://kubernetes.default.svc
      namespace: apps-prod
```

> [!IMPORTANT]
> `CreateNamespace=true` does **not** override AppProject permissions.
>
> AppProject authorization is checked first.

---

## Application exists but workload is missing

Inspect:

```bash
argocd app get <application-name>
```

Check:

```text
Source
Destination
Conditions
Resources
Sync Status
Health Status
```

Verify:

```bash
argocd app manifests <application-name>
```

Check whether the expected manifests are rendered.

---

# Frequently Asked Questions

<details>
<summary><b>1. Do we need two namespaces when using ApplicationSet?</b></summary>

There are two namespace concepts.

```text
argocd
    ↓
ApplicationSet / Argo CD management resources


application namespace
    ↓
Deployment / Service / Pods
```

Example:

```yaml
metadata:
  namespace: argocd
```

versus:

```yaml
destination:
  namespace: apps-dev
```

The first controls where the ApplicationSet resource lives.

The second controls where the actual workload is deployed.

</details>

---

<details>
<summary><b>2. Do we need destination.namespace: argocd in ApplicationSet?</b></summary>

No.

This:

```yaml
template:

  spec:

    destination:

      namespace: apps-dev
```

defines where the generated Application deploys its workload.

You do not set it to `argocd` just because the ApplicationSet itself lives there.

</details>

---

<details>
<summary><b>3. How does Argo CD know whether to deploy frontend or backend?</b></summary>

Through:

```yaml
source:
  path: ...
```

For frontend:

```yaml
path: apps/frontend
```

For backend:

```yaml
path: apps/backend
```

The Deployment itself does not know anything about Git folders.

The generated Argo CD Application points to the correct source directory.

</details>

---

<details>
<summary><b>4. If Argo CD knows DEV and PROD but I want to deploy only to DEV, what should I do?</b></summary>

Use a Cluster Generator selector:

```yaml
clusters:

  selector:

    matchLabels:

      environment: dev
```

Only clusters with:

```text
environment=dev
```

are selected.

</details>

---

<details>
<summary><b>5. Why do we define revision under the Git Generator and targetRevision under the Application source?</b></summary>

They have different purposes.

```yaml
git:
  revision: main
```

means:

> Discover directories/files from `main`.

Whereas:

```yaml
source:
  targetRevision: main
```

means:

> Deploy the generated Application from `main`.

Think:

```text
revision
   ↓
Discovery


targetRevision
   ↓
Deployment
```

</details>

---

<details>
<summary><b>6. Why can't we directly use {{.revision}}?</b></summary>

A field used to configure a generator does not automatically become a generated template parameter.

For example:

```yaml
git:
  revision: main
```

configures the Git Generator.

It should not automatically be assumed to create:

```text
{{.revision}}
```

Template variables must come from generator output or explicitly supplied data.

</details>

---

<details>
<summary><b>7. If we have 4 apps and 3 environments, how many generated Applications will we get?</b></summary>

Using Matrix:

```text
4 × 3 = 12
```

Example:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod

payment-dev
payment-qa
payment-prod

orders-dev
orders-qa
orders-prod
```

</details>

---

<details>
<summary><b>8. Do we always need Git Directory + Cluster Generator?</b></summary>

No.

Choose generators based on where the information comes from.

If Git files contain environment and destination information:

```text
Git Directory
+
Git File
↓
Matrix
```

may be sufficient.

If Argo CD registered clusters are the source of truth:

```text
Git Directory
+
Cluster Generator
↓
Matrix
```

may be better.

</details>

---

<details>
<summary><b>9. Can Git File and Cluster Generator be used together?</b></summary>

Yes.

For example:

```text
Git File
    ↓
Deployment/environment configuration


Cluster Generator
    ↓
Actual registered cluster information
```

Use both when they contribute different information.

Avoid duplicating the same responsibility.

</details>

---

<details>
<summary><b>10. Can any generator combination be used inside Matrix?</b></summary>

Conceptually, Matrix combines two child generator outputs.

The important question is:

> Do I want every result from generator A combined with every result from generator B?

If yes, Matrix is appropriate.

Common combinations include:

```text
Git Directory + Git File

Git Directory + Cluster

List + Cluster

List + Git File
```

</details>

---

<details>
<summary><b>11. Can Git Directory + Git File deploy frontend and backend to DEV, QA, and PROD?</b></summary>

Yes.

Example:

```text
Git Directory
---------------
frontend
backend

       ×

Git File
---------------
dev
qa
prod
```

Matrix generates:

```text
frontend-dev
frontend-qa
frontend-prod

backend-dev
backend-qa
backend-prod
```

</details>

---

<details>
<summary><b>12. Can common environment values be stored in Git File configuration?</b></summary>

Yes.

Example:

```yaml
environment: prod
namespace: apps-prod
revision: main
valuesFile: values-prod.yaml
replicas: "5"
```

These values can be used while generating the Application.

For workload fields such as replicas, Helm or Kustomize is normally required to inject them into the actual Kubernetes manifests.

</details>

---

<details>
<summary><b>13. What is the main use of Merge Generator?</b></summary>

Merge is mainly useful for:

```text
Base/default values
        +
Specific overrides
```

Example:

```text
Default:
replicas=2

PROD:
replicas=10
```

Final PROD configuration:

```text
replicas=10
```

</details>

---

<details>
<summary><b>14. Is Merge Generator basically Matrix Generator with override support?</b></summary>

No.

They have different behaviors.

Matrix:

```text
A × B
```

creates combinations.

Merge:

```text
A + matching B
```

matches records and overlays values.

Example:

```text
Matrix:

2 apps × 3 environments
=
6 results


Merge:

3 base environments
+
2 matching overrides
=
still 3 results
```

</details>

---

<details>
<summary><b>15. Do we use Merge Generator extensively in real-world environments?</b></summary>

Not always.

Matrix is often used when organizations have:

```text
many applications
×
many environments/clusters
```

Merge is more specialized and is useful when there are:

```text
standard defaults
+
specific exceptions
```

Large platforms may still use Merge significantly for cluster-specific or region-specific overrides.

</details>

---

<details>
<summary><b>16. If Git File already contains cluster server information, do we still need Cluster Generator?</b></summary>

Not necessarily.

Example:

```yaml
environment: dev
server: https://dev-cluster
namespace: apps-dev
```

can be consumed as:

```yaml
destination:

  server: '{{.server}}'

  namespace: '{{.namespace}}'
```

In this architecture:

```text
Git
=
source of truth for cluster mapping
```

Cluster Generator becomes useful when Argo CD's registered cluster inventory should be the source of truth.

</details>

---

<details>
<summary><b>17. What is the easiest way to remember all the generators?</b></summary>

```text
LIST
"What static items did I define?"


GIT DIRECTORY
"What application directories exist?"


GIT FILE
"What does my configuration say?"


CLUSTER
"What clusters does Argo CD know?"


MATRIX
"Combine A × B."


MERGE
"Match + overlay."
```

</details>

---

# Quick Reference

| Generator         | Meaning                        | Typical Use              |
| ----------------- | ------------------------------ | ------------------------ |
| **List**          | Static parameter list          | Small/simple deployments |
| **Git Directory** | Discover app directories       | App onboarding           |
| **Git File**      | Read structured Git config     | Environment config       |
| **Cluster**       | Discover Argo-managed clusters | Multi-cluster            |
| **Matrix**        | Combine two dimensions         | Apps × Env/Cluster       |
| **Merge**         | Match + override               | Defaults + exceptions    |

---

## Golden Rules

```text
List
   ↓
Static data


Git Directory
   ↓
Discover applications


Git File
   ↓
Read configuration


Cluster
   ↓
Discover deployment targets


Matrix
   ↓
COMBINE


Merge
   ↓
MATCH + OVERRIDE
```

---

# Recommended Production Pattern

For multiple applications across multiple environments:

```text
                    Git Repository
                          |
            +-------------+-------------+
            |                           |
          apps/*                  environments/*
            |                           |
            v                           v
     Git Directory                 Git File
       Generator                   Generator
            |                           |
            +-------------+-------------+
                          |
                          v
                       MATRIX
                          |
                          v
                Generated Applications
                          |
              +-----------+-----------+
              |           |           |
             DEV          QA         PROD
```

Example:

```text
4 Applications × 3 Environments
              =
12 generated Applications
```

---

# Learning Status

```text
ApplicationSet Fundamentals       ✅
Go Templates                      ✅
List Generator                    ✅
Git Directory Generator           ✅
Git File Generator                ✅
Cluster Generator                 ✅
Matrix Generator                  ✅
Merge Generator                   ✅

Selectors / Filtering             ⏭️
Sync & Deletion Safety            ⏭️
Progressive Sync / RollingSync    ⏭️
Production Multi-Cluster Design   ⏭️
```

---

# Final Mental Model

```text
ApplicationSet
     |
     | Generates
     v
Application
     |
     | Reads
     v
Git
     |
     | Deploys
     v
Kubernetes Cluster
```

For a larger GitOps platform:

```text
                    Git Repository

              +----------+----------+
              |                     |
          Applications          Environments
              |                     |
              v                     v
       Git Directory           Git File
         Generator             Generator
              |                     |
              +----------+----------+
                         |
                       Matrix
                         |
                         v
               Generated Applications
                         |
           +-------------+-------------+
           |             |             |
          DEV            QA           PROD
           |             |             |
           v             v             v
       Kubernetes    Kubernetes    Kubernetes
```

ApplicationSet allows us to move from:

```text
"Create one Argo CD Application YAML
for every deployment manually"
```

to:

```text
"Define discovery rules, deployment configuration,
cluster targeting, and templates once,
then let Argo CD generate the required Applications."
```

That is the core value of **Argo CD ApplicationSet for scalable GitOps**.

---

## References

* [Argo CD Documentation](https://argo-cd.readthedocs.io/)
* [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
