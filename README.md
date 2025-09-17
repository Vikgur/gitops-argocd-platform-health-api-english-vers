# Table of Contents

- [About the Project](#about-the-project)  
  - [Relation to Other Repositories and Bootstrap](#relation-to-other-repositories-and-bootstrap)  
- [Architecture](#architecture)  
  - [apps/](#apps)  
    - [argo-rollouts.yaml — blue/green and canary deployment](#argo-rolloutsyaml--bluegreen-and-canary-deployment)  
    - [argocd-image-updater.yaml — automatic image updates](#argocd-image-updateryaml--automatic-image-updates)  
    - [cert-manager.yaml — automatic TLS](#cert-manageryaml--automatic-tls)  
    - [external-secrets.yaml — cloud secrets](#external-secretsyaml--cloud-secrets)  
    - [ingress-nginx.yaml — entry point](#ingress-nginxyaml--entry-point)  
    - [monitoring.yaml — metrics and alerts](#monitoringyaml--metrics-and-alerts)  
    - [policy-engine.yaml — security policies](#policy-engineyaml--security-policies)  
  - [argocd-image-updater](#argocd-image-updater)  
    - [argocd-image-updater-config.yaml](#argocd-image-updater-configyaml)  
    - [secret.yaml](#secretyaml)  
    - [Integration steps into health-api ](#integration-steps-into-health-api)
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

- `argo-rollouts` — deployment strategies  
- `argocd-image-updater` — automatic image tag updates with strategy and signature verification  
- `cert-manager` — automatic TLS  
- `external-secrets` — cloud secrets  
- `ingress-nginx` — entry point  
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

### `argo-rollouts.yaml` — blue/green and canary deployment

**Purpose:**  
Deploys `argo-rollouts` for advanced deployment strategies.  
Supports `blueGreen`, `canary`, `step analysis`, manual promotion, and rollbacks.

### `argocd-image-updater.yaml` — automatic image updates

**Purpose:**  
Deploys the Argo Image Updater component via a Helm chart.  
This service tracks new image tags in the container registry, verifies signatures (`cosign`), and updates Argo CD Applications with the latest versions.  
Works together with the manifests in the `argocd-image-updater/` directory (ConfigMap and Secret).

### `cert-manager.yaml` — automatic TLS

**Purpose:**  
Deploys `cert-manager`, which automatically obtains and renews TLS certificates (e.g., from Let's Encrypt).  
Required for HTTPS and secure interaction with external services.

### `external-secrets.yaml` — secrets from the cloud

**Purpose:**  
Deploys `external-secrets` — a component that synchronizes Kubernetes `Secrets` with external Secret Managers (e.g., Yandex Cloud, HashiCorp Vault).  
Eliminates the need to store secrets in Git.

### `ingress-nginx.yaml` — entry point

**Purpose:**  
Deploys `ingress-nginx` — the main Ingress controller.  
Handles external HTTP/HTTPS traffic and routes it to services.

### `monitoring.yaml` — metrics and alerts

**Purpose:**  
Deploys `kube-prometheus-stack`: Prometheus, Grafana, node-exporter, alerts, and dashboards.  
Provides cluster and service observability, supports metrics for rollout strategies.

### `policy-engine.yaml` — security policies

**Purpose:**  
Deploys `kyverno` — a Policy-as-Code engine.  
Applies policies to Kubernetes objects: annotation checks, blocking privileged containers, enforce logic, and more.

---

## `argocd-image-updater`

Configuration and secrets for running Argo Image Updater.

### `argocd-image-updater-config.yaml`

**Purpose:**  
Defines the global settings for Argo Image Updater.  
Specifies the container registries in use, authentication parameters, and image signature verification (`cosign`).  
Applied in the `argocd` namespace.

### `secret.yaml`

**Purpose:**  
Contains credentials for accessing GitHub Container Registry (GHCR).  
Used by Argo Image Updater to read tag information and related signatures.  
Created as a Kubernetes Secret in the `argocd` namespace.

### Integration steps into health-api 

1. **Dockerfile**  
   Add standard OCI labels (`source`, `revision`, `created`) to each service (backend, frontend).  
   These labels allow Argo Image Updater to track the origin and freshness of images.

2. **Helm values**  
   In the `helm-blue-green-canary-gitops-health-api` repository, configure `image.repository` and Argo Image Updater annotations for each service (e.g., `helm/values/backend.yaml`).  
   This tells AIU which images to update and according to which rules.

3. **ConfigMap**  
   In the `gitops-argocd-platform-health-api` repository, create `argocd-image-updater-config.yaml` with global configuration.  
   It defines which registries AIU works with and where the cosign key for signature verification is stored.

4. **Secret**  
   In the same repository, add a `Secret` with credentials for accessing GHCR.  
   This secret is used by AIU to fetch tag and signature information.

---

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
