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

---

 # ClusterSecretStore and ExternalSecret

* External Secrets Controller was deployed to give us the CRDs e.g. (ClusterSecretStore and ExternalSecret)
* ClusterSecretStore defines the external secret backend e.g. AWS Secrets Manager in our case.
  * Responsible for authentication and the access (we are using a Service Account to access that secret backend)
* ExternalSecret references secrets from the backend and syncs them to the cluster.
  * References the SecretStore as well which knows how to saccess the data.
  * It creates s Kubernetes native secret with the value of the Remote secret from the secret backend (e.g. AWS Secrets Manager)

* We created the ClusterSecretStore under the platform directory - which has the ServiceAccount reference for accessing AWS Secrets Manager.
* The ExternalSecret is part of the application overlays hence this lives under the components and deployed with the app overlays.
  * There we defined the SecretStore (ClusterSecretStore reference) and the Remote secret name/key to fetch.
  * Target is the name of the K8s native secret that ExternalSecret will create.