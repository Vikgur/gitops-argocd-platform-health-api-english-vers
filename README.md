# Table of Contents

- [About the Project](#about-the-project)  
  - [Relation to Other Repositories and Bootstrap](#relation-to-other-repositories-and-bootstrap)  
- [Architecture](#architecture)  
  - [apps/](#apps)  
    - [ingress-nginx.yaml — entry point](#ingress-nginxyaml--entry-point)  
    - [cert-manager.yaml — automatic TLS](#cert-manageryaml--automatic-tls)  
    - [argo-rollouts.yaml — bluegreen and canary deployment](#argo-rolloutsyaml--bluegreen-and-canary-deployment)  
    - [external-secrets.yaml — secrets from the cloud](#external-secretsyaml--secrets-from-the-cloud)  
    - [monitoring.yaml — metrics and alerts](#monitoringyaml--metrics-and-alerts)  
    - [policy-engine.yaml — security policies](#policy-engineyaml--security-policies)  
- [Implemented DevSecOps Practices](#implemented-devsecops-practices)  
  - [Security linting & validation](#security-linting--validation)  
    - [.yamllint.yml](#yamllintyml)  
    - [.checkov.yaml](#checkovyaml)  
    - [policy/argo/secure-apps.rego](#policyargosecure-appsrego)  
    - [.gitleaks.toml](#gitleakstoml)  
    - [.pre-commit-config.yaml](#pre-commit-configyaml)  

---

# About the Project

This repository is the **App of Apps** for the production GitOps platform of [`health-api`](https://github.com/vikgur/health-api-for-microservice-stack-english-vers). It defines system Argo CD applications (`kind: Application`) for the core components required **before microservices are deployed**:

- `ingress-nginx` — entry point  
- `cert-manager` — automatic TLS  
- `argo-rollouts` — deployment strategies  
- `external-secrets` — secrets from the cloud  
- `kube-prometheus-stack` — monitoring  
- `kyverno` — security policies  

The platform outlives **any individual service** and is independent of business logic. It can be updated separately.

> Although the cluster currently serves only `health-api`, its architecture has been designed from the start as a **multi-layered and scalable platform** — with a clear separation between the system and application layers, ready to onboard new services without modifying the core.

## Relation to Other Repositories and Bootstrap

- This repository is **not cloned directly** during cluster initialization with [ansible-gitops-bootstrap-health-api](https://github.com/vikgur/ansible-gitops-bootstrap-health-api-english-vers).  
- Instead, it is connected **as a child application** from [argocd-config-health-api](https://github.com/Vikgur/argocd-config-health-api-english-vers) — via the `Application` object defined in [apps/platform-apps.yaml](https://github.com/Vikgur/argocd-config-health-api-english-vers/-/blob/main/apps/platform-apps.yaml).  
- This approach allows:
  - centralized management of all sources in [argocd-config-health-api](https://github.com/Vikgur/argocd-config-health-api-english-vers),  
  - separation of platform components from user applications,  
  - initialization of Argo CD with **a single root application** that recursively provisions the entire stack.  

Thus, Argo CD itself clones and synchronizes this repository **by reference**, without direct Ansible involvement.

---

# Architecture

Each file in the `apps/` directory is a separate Argo CD application (`Application`) for a platform component. Below is a brief description of the purpose of each.

## `apps/`

### `ingress-nginx.yaml` — entry point

**Purpose:**  
Deploys `ingress-nginx` — the main Ingress controller.  
Handles external HTTP/HTTPS traffic and routes it to services.

---

### `cert-manager.yaml` — automatic TLS

**Purpose:**  
Deploys `cert-manager`, which automatically obtains and renews TLS certificates (e.g., from Let's Encrypt).  
Required for HTTPS and secure interaction with external services.

---

### `argo-rollouts.yaml` — blue/green and canary deployment

**Purpose:**  
Deploys `argo-rollouts` for advanced deployment strategies.  
Supports `blueGreen`, `canary`, `step analysis`, manual promotion, and rollbacks.

---

### `external-secrets.yaml` — secrets from the cloud

**Purpose:**  
Deploys `external-secrets` — a component that synchronizes Kubernetes `Secrets` with external Secret Managers (e.g., Yandex Cloud, HashiCorp Vault).  
Eliminates the need to store secrets in Git.

---

### `monitoring.yaml` — metrics and alerts

**Purpose:**  
Deploys `kube-prometheus-stack`: Prometheus, Grafana, node-exporter, alerts, and dashboards.  
Provides cluster and service observability, supports metrics for rollout strategies.

---

### `policy-engine.yaml` — security policies

**Purpose:**  
Deploys `kyverno` — a Policy-as-Code engine.  
Applies policies to Kubernetes objects: annotation checks, blocking privileged containers, enforce logic, and more.

# Implemented DevSecOps Practices

This repository includes a set of DevSecOps tools to ensure quality and security control of GitOps manifests.

## Security linting & validation

### `.yamllint.yml`

**Purpose:**  
Checks YAML syntax and style.  
Prevents formatting errors and enforces a consistent standard.

---

### `.checkov.yaml`

**Purpose:**  
Analyzes Kubernetes manifests with Checkov.  
Detects insecure practices: using `latest` images, missing resource limits, misconfigured Ingress, etc.

---

### `policy/argo/secure-apps.rego`

**Purpose:**  
OPA policies for Argo CD `Application`.  
They prohibit:  
- using `targetRevision: HEAD`,  
- disabling `prune` and `selfHeal` in `syncPolicy`.  
Ensures correct and secure configuration of Argo CD applications.

---

### `.gitleaks.toml`

**Purpose:**  
Scans commits and files for secrets.  
Prevents leakage of tokens, passwords, and other sensitive data into Git.

---

### `.pre-commit-config.yaml`

**Purpose:**  
Runs automated checks before commit (`yamllint`, `checkov`, `gitleaks`).  
Helps detect issues before changes are merged into the repository.

---

> As a result, the repository gains baseline DevSecOps coverage: from YAML syntax and style validation to secret scanning and Argo CD security policy enforcement.
