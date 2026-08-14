# 🚀 Argo CD ApplicationSet — Complete Guide


[![Argo CD](https://img.shields.io/badge/Argo%20CD-ApplicationSet-orange)](https://argo-cd.readthedocs.io/)
![Kubernetes](https://img.shields.io/badge/Kubernetes-GitOps-blue)
![GitOps](https://img.shields.io/badge/GitOps-Automation-success)


> A practical end-to-end guide to **Argo CD ApplicationSet**, covering generators,
> multi-application deployments, multi-environment deployments, Matrix and Merge
> generators, cluster selection, Git repository design, troubleshooting, and
> production best practices.


---


## 📚 Table of Contents


- [Overview](#overview)
- [Why ApplicationSet?](#why-applicationset)
- [Architecture](#architecture)
- [Application vs App of Apps vs ApplicationSet](#application-vs-app-of-apps-vs-applicationset)
- [ApplicationSet Structure](#applicationset-structure)
- [Go Templates](#go-templates)
- [Generators](#generators)
  - [List Generator](#1-list-generator)
  - [Git Directory Generator](#2-git-directory-generator)
  - [Git File Generator](#3-git-file-generator)
  - [Cluster Generator](#4-cluster-generator)
  - [Matrix Generator](#5-matrix-generator)
  - [Merge Generator](#6-merge-generator)
- [Generator Comparison](#generator-comparison)
- [Multi-App Multi-Environment Design](#multi-app--multi-environment-design)
- [Repository Structure](#repository-structure)
- [Production Best Practices](#production-best-practices)
- [Troubleshooting](#troubleshooting)
- [Frequently Asked Questions](#frequently-asked-questions)
- [Quick Reference](#quick-reference)


---


# Overview


An **ApplicationSet** is an Argo CD resource used to automatically generate
and manage multiple Argo CD `Application` resources from a common template.


Instead of manually maintaining:


```text
frontend-dev.yaml
frontend-qa.yaml
frontend-prod.yaml
backend-dev.yaml
backend-qa.yaml
backend-prod.yaml

ApplicationSet allows us to define:

                 ApplicationSet
                       │
              ┌────────┴────────┐
              │                 │
          Generator          Template
              │                 │
              └────────┬────────┘
                       │
                       ▼
              Generated Applications
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
         DEV           QA          PROD

[!TIP]
Think of ApplicationSet as an Application factory.
Generators provide the data, and the template defines how each generated
Argo CD Application should look.



Then for generator explanations, I would use a consistent pattern like:


```markdown
## 5. Matrix Generator


### 🎯 Purpose


The Matrix Generator combines the output of **two child generators**.


```text
Generator A × Generator B

For example:

Applications                Environments


frontend                         dev
backend             ×            qa
payment                          prod
orders

Result:

4 Applications × 3 Environments
              =
12 Argo CD Applications
Example
generators:
  - matrix:
      generators:


        # Discover applications
        - git:
            repoURL: https://github.com/example/gitops.git
            revision: main
            directories:
              - path: apps/*


        # Discover environment configuration
        - git:
            repoURL: https://github.com/example/gitops.git
            revision: main
            files:
              - path: environments/*.yaml
Generated Applications
Application	Environment
frontend-dev	DEV
frontend-qa	QA
frontend-prod	PROD
backend-dev	DEV
backend-qa	QA
backend-prod	PROD

[!IMPORTANT]
Use Matrix when you want combinations.

Matrix = A × B



And Merge would deliberately look different:


```markdown
## 6. Merge Generator


### 🎯 Purpose


Merge Generator combines parameter sets representing the **same logical
target** and allows later generators to override values from earlier ones.


```text
Base Configuration
        +
Environment/Cluster Override
        │
        ▼
      MERGE
        │
        ▼
Final Configuration
Example

Base:

DEV   replicas=2
QA    replicas=2
PROD  replicas=2

Override:

PROD  replicas=10

Result:

DEV   replicas=2
QA    replicas=2
PROD  replicas=10

The matching field is configured using:

mergeKeys:
  - environment

[!IMPORTANT]
Matrix = create combinations

Merge = match + overlay/override



For your Q&A, GitHub's collapsible sections would look much better than 15+ large headings:


```markdown
# ❓ Frequently Asked Questions


<details>
<summary><b>1. Do we need destination.namespace: argocd in ApplicationSet?</b></summary>


No.


The ApplicationSet itself normally lives in:


```yaml
metadata:
  namespace: argocd

Whereas:

template:
  spec:
    destination:
      namespace: apps-dev

specifies where the actual Kubernetes workload should be deployed.

argocd namespace
      │
      ├── ApplicationSet
      └── Generated Applications
                  │
                  ▼
           destination.namespace
                  │
                  ▼
             apps-dev
                  │
                  ├── Deployment
                  ├── Service
                  └── Pods
</details> <details> <summary><b>2. Why do we have revision and targetRevision?</b></summary>

They serve different purposes.

git:
  revision: main

means:

Discover Git directories/files from main.

Whereas:

source:
  targetRevision: main

means:

The generated Application should deploy from main.

Therefore:

revision
   ↓
ApplicationSet discovery


targetRevision
   ↓
Application deployment
</details> <details> <summary><b>3. If I have 4 apps and 3 environments, how many Applications are generated?</b></summary>

With Matrix:

4 applications × 3 environments
              =
12 Applications
</details> <details> <summary><b>4. Is Merge Generator basically Matrix with override support?</b></summary>

No.

Matrix
=======
A × B
Creates combinations




Merge
=====
A + matching B
Merges/overrides values

For example:

Matrix:


2 apps × 3 environments
= 6 results




Merge:


3 base environments
+ 2 matching overrides
= still 3 results
</details> ```

I'd also finish it with a compact visual cheat sheet:

# 🧠 ApplicationSet Quick Reference


| Generator | Question it answers | Typical use |
|---|---|---|
| **List** | What static items did I define? | Small/static configuration |
| **Git Directory** | What application directories exist? | Application discovery |
| **Git File** | What does my configuration say? | Environment/configuration data |
| **Cluster** | What clusters does Argo CD know? | Multi-cluster deployment |
| **Matrix** | How do I combine A × B? | Apps × environments/clusters |
| **Merge** | How do I apply defaults + overrides? | Exceptions and overrides |


## Golden Rules


```text
List
   → Static data


Git Directory
   → Discover applications


Git File
   → Read configuration


Cluster
   → Discover deployment targets


Matrix
   → COMBINE


Merge
   → MATCH + OVERRIDE
🏗️ Recommended Production Pattern

For multiple applications across multiple environments:

                    Git Repository
                          │
            ┌─────────────┴─────────────┐
            │                           │
          apps/*                  environments/*
            │                           │
            ▼                           ▼
     Git Directory                 Git File
       Generator                   Generator
            │                           │
            └─────────────┬─────────────┘
                          │
                          ▼
                     MATRIX
                          │
                          ▼
                Generated Applications
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
             DEV          QA         PROD

Example:

4 applications × 3 environments
              =
12 generated Applications
🎓 Learning Status
 ApplicationSet Fundamentals
 Go Templates
 List Generator
 Git Directory Generator
 Git File Generator
 Cluster Generator
 Matrix Generator
 Merge Generator
 Selectors & Filtering
 Sync and Deletion Safety
 Progressive Sync / RollingSync
 Production Multi-Cluster Design
📖 References
Argo CD Documentation
ApplicationSet Documentation


### What I recommend


The previous document has the **right technical content**, but I would restructure the whole thing into this style before publishing it.


It will make your repository look much more like **engineering documentation** rather than copied learning notes:


```text
README.md
│
├── 🚀 Introduction
├── 🎯 Why ApplicationSet
├── 🏗️ Architecture
├── 🧩 Core Concepts
├── ⚙️ Generators
│   ├── List
│   ├── Git Directory
│   ├── Git File
│   ├── Cluster
│   ├── Matrix
│   └── Merge
├── 🏭 Production Example
├── 🔐 Best Practices
├── 🔧 Troubleshooting
├── ❓ Collapsible Q&A
├── 🧠 Quick Reference
└── 🎓 Learning Status

One more improvement: I would use the real Argo CD documentation links and GitHub badges, but avoid excessive emojis throughout the body. A few section icons make the README attractive; too many make technical documentation harder to scan.

If you're going to publish this, I can also produce the entire polished README.md as one continuous Markdown block, incorporating all of the ApplicationSet documentation and all the Q&A from our discussion, so you can copy it once and commit it directly.