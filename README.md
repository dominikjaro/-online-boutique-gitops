# Online Boutique — GitOps Repository

GitOps repository managing the declarative Kubernetes manifests and platform policies for the my Online Boutique microservices application running on AWS EKS.

This repository acts as the single source of truth for cluster state, managed via Argo CD and structured using Kustomize. It manages two primary scopes:

* **Application Workloads:** Declarative Kubernetes deployments, services, and configurations for all Online Boutique microservices.

* **Platform & Policy Enforcement:** Open Policy Agent (OPA) Gatekeeper policies enforcing security baselines across the cluster.

```plaintext
.
├── base/                   # Baseline manifests shared across all environments
│   ├── adservice.yaml
│   ├── frontend.yaml
│   └── kustomization.yaml
├── overlays/               # Environment-specific overrides and configuration patches
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches/
│       
└── platform/               # Cluster-wide infrastructure & governance manifests
    └── open-policy-agent/
        ├── templates/      # OPA ConstraintTemplates (Rego logic definitions)
        └── constraints/    # OPA Constraints enforcing active cluster policies
```

## Policy-as-Code (OPA Gatekeeper)

Cluster security policies are declared under platform/ using OPA Gatekeeper ConstraintTemplates and Constraints:

* **Disallow Privileged Containers:** Blocks any container attempting to run with securityContext.privileged: true to prevent container breakout risks.

* **Block NodePort Services:** Prevents services from using NodePort to avoid exposing them outside the cluster.

## GitOps Deployment Workflow

```plaintext
Microservice CI Pipeline
       │
       ▼ (Downstream trigger on image build/tag)
GitOps Repository (.gitlab-ci.yml)
       │
       ▼ (Updates image tags in overlays/ via Kustomize)
Git Commit to main
       │
       ▼ (Detects drift / new commit)
Argo CD
       │
       ▼ (Reconciles state)
AWS EKS Cluster
```

1. **Upstream Trigger:** When an application microservice CI pipeline builds and pushes a new container image to ECR, it triggers this repository's downstream pipeline.

2. **Overlay Patching:** The pipeline updates the target image tag in the overlay files.

3. **Automated Sync:** Argo CD applies the changes to the cluster.