# Azure AKS GitOps Argo CD Delivery

![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-844FBA?style=flat&logo=terraform&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Argo CD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=flat&logo=argo&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat&logo=githubactions&logoColor=white)

An Azure AKS delivery platform that provisions the cloud foundation with Terraform, promotes a canary image through GitHub Actions, and reconciles workload desired state with Argo CD.

**Azure • AKS • Terraform • Argo CD • Helm • GitHub Actions**

**Navigate:**

- [Overview](#platform-at-a-glance)
- [Architecture](#architecture)
- [Infrastructure](#infrastructure-as-code)
- [GitOps](#gitops-with-argo-cd)
- [Workload](#demo-workload)
- [Quick start](#quick-start)
- [Security](#security-posture)

## Platform at a Glance

This project separates cloud foundation, Kubernetes desired state, and application delivery concerns deliberately:

| Concern | What is implemented |
| --- | --- |
| Cloud platform | Terraform composes an Azure resource group, VNet and AKS subnet, AKS, Basic ACR, and the required identity/RBAC handoffs. |
| Kubernetes delivery | Argo CD reconciles two Helm sources from Git: the demo workload in the <code>otel-demo</code> namespace and a small internal canary in <code>default</code>. |
| Workload | The pinned upstream demo workload is integrated through a Helm wrapper; this repository does not claim authorship of the application. |
| Application delivery | GitHub Actions validates infrastructure/chart changes. A separate OIDC-based workflow builds the small <code>azure-webapp</code> canary, pushes a Git-SHA tag to ACR, and commits the desired image tag back to Git. |

## What This Project Demonstrates

- A reusable Azure Kubernetes foundation expressed as small Terraform modules rather than a one-off portal build.
- A clear control-plane boundary: Terraform provisions Azure and explicit Kubernetes prerequisites; Argo CD owns Helm-rendered workload state.
- GitOps reconciliation with automated sync, prune, self-heal, and namespace creation on both primary Argo CD Applications.
- GitHub Actions image promotion through Git rather than direct cluster deployment from CI.
- Scoped workload and delivery identities, including short-lived GitHub-to-Azure authentication for the canary delivery workflow.

## Architecture

The platform is designed around a simple operating model: Git declares intent, Terraform creates the Azure foundation, GitHub Actions promotes the canary image through Git, and Argo CD reconciles workloads.

### 1. High-Level Platform Architecture

~~~mermaid
flowchart TB
  Developer[Developer] --> Repo[Git repository]

  subgraph Delivery["Delivery and reconciliation"]
    Repo --> Validation[Validation workflow]
    Repo --> CanaryCI[Canary delivery workflow]
    CanaryCI --> ACR[Azure Container Registry]
    CanaryCI --> DesiredState[Committed image tag]
    Repo --> DesiredState
    DesiredState --> Argo[Argo CD]
  end

  subgraph Provisioning["Platform provisioning"]
    Repo --> TerraformCode[Terraform configuration]
    TerraformCode --> Operator[Terraform operator]
    Operator --> AzureRG[Azure project resource group]
  end

  subgraph AzurePlatform["Azure platform"]
    AzureRG --> AKS[AKS cluster]
    AzureRG --> Registry[ACR]

    subgraph Cluster["AKS workloads"]
      AKS --> Canary[Internal canary]
      AKS --> Demo[Demo workload]
      Canary --> Registry
      Demo --> Registry
    end
  end

  Argo --> Canary
  Argo --> Demo
  Browser[Browser] --> PublicLB[frontend-proxy LoadBalancer]
  PublicLB --> Demo
~~~

### 2. GitOps Delivery Flow

~~~mermaid
flowchart TB
  Change[Developer change] --> GitHub[GitHub repository]

  subgraph ValidationPath["Pull request and branch validation"]
    GitHub --> Validate[validate.yml]
    Validate --> TFChecks[Terraform format and validate]
    Validate --> HelmChecks[Helm dependency build lint and render]
  end

  subgraph CanaryPath["Canary image promotion on main"]
    GitHub --> Deploy[deploy-aks.yml]
    Deploy --> GitHubOIDC[GitHub OIDC login]
    GitHubOIDC --> PushImage[Build and push azure-webapp]
    PushImage --> ACRFlow[ACR Git-SHA tag]
    PushImage --> ValuesCommit[Commit only values image tag]
    ValuesCommit --> GitHub
  end

  subgraph Reconciliation["GitOps reconciliation"]
    GitHub --> ArgoFlow[Argo CD watches desired state]
    ArgoFlow --> DemoApp[Demo Application]
    ArgoFlow --> CanaryApp[azure-webapp Application]
    DemoApp --> DemoNS[otel-demo namespace]
    CanaryApp --> DefaultNS[default namespace]
  end
~~~

The workflow validates Terraform and Helm sources; it does not run Terraform plan/apply, retrieve AKS credentials, or call Helm or kubectl against the cluster. Argo CD is deliberately the workload reconciler, while the CI workflow only promotes the canary image through Git.

### 3. Identity and Access Flow

~~~mermaid
flowchart TB
  subgraph DeliveryIdentity["Canary delivery identity"]
    Actions[GitHub Actions] --> OIDC[Short-lived OIDC assertion]
    OIDC --> Entra[Microsoft Entra ID]
    Entra --> AcrPush[AcrPush at ACR scope]
    AcrPush --> RegistryIdentity[Azure Container Registry]
  end

  subgraph ClusterIdentity["AKS identities"]
    Kubelet[AKS kubelet identity] --> AcrPull[AcrPull at ACR scope]
    AcrPull --> RegistryIdentity
    AKSControl[AKS control-plane identity] --> NetworkRole[Network Contributor at AKS subnet scope]
  end
~~~

GitHub-to-Azure OIDC is configured outside Terraform and used by the canary workflow. ACR admin access is disabled.

## Technology Stack

| Layer | Technology | Purpose in this repository |
| --- | --- | --- |
| Cloud | Microsoft Azure | Resource group, network, registry, AKS, and identity services |
| Infrastructure as Code | Terraform 1.6 or newer | Modular provisioning and environment configuration |
| Kubernetes | Azure Kubernetes Service | Runtime for the demo and internal canary |
| Networking | Azure CNI and Standard Load Balancer | Pod networking and public demo exposure |
| Registry | Azure Container Registry Basic | Stores the Git-SHA-tagged canary image |
| Packaging | Helm | Packages the canary and demo wrapper |
| GitOps | Argo CD | Reconciles Helm desired state to AKS |
| CI | GitHub Actions | Validates Terraform/Helm and promotes the canary image through Git |

## Repository Structure

~~~text
.
├── .github/workflows/
│   ├── validate.yml                         # Terraform and Helm validation
│   └── deploy-aks.yml                       # OIDC canary image promotion
├── argocd/
│   ├── azure-webapp-application.yaml        # Internal canary Application
│   └── opentelemetry-demo-application.yaml  # Demo Application
├── app/                                     # Small nginx-based canary source
├── helm/
│   ├── azure-webapp/                        # Canary chart
│   └── opentelemetry-demo/                  # Pinned upstream-demo wrapper
├── modules/
│   ├── aks/ network/ acr/                   # Azure platform components
│   └── otel_cluster_config/                 # Explicit Kubernetes handoff objects
├── backend.hcl.example                      # Ignored remote-state configuration template
├── main.tf                                  # Module composition
├── terraform.tfvars.example                 # Safe local-input template
├── variables.tf                             # Root inputs and validation
└── README.md
~~~

## Infrastructure as Code

### Terraform Design

Terraform is the cloud-foundation control plane. The root module composes focused modules for the resource group, network, AKS, ACR, identity/RBAC, and in-cluster prerequisites needed by the delivery platform.

| Area | Terraform-managed implementation | Boundary |
| --- | --- | --- |
| Network | VNet <code>10.20.0.0/16</code> and AKS subnet <code>10.20.1.0/24</code> | No NSGs, UDRs, NAT gateway, private endpoints, or network policy are declared. |
| AKS | Free tier, one static system pool, Standard Load Balancer, Azure CNI, OIDC issuer, and workload identity | No private cluster, autoscaling, zones, backup/DR, Azure RBAC integration, or authorized-IP configuration is declared. |
| Registry | Basic ACR with admin access disabled | AKS kubelet receives <code>AcrPull</code> at ACR scope. GitHub CI federation and push role are external setup, not Terraform resources. |
| In-cluster prerequisites | Namespace and required handoff objects | Terraform does not install Argo CD or own Helm-rendered application resources. |

### AKS Design

The main profile uses two static <code>Standard_D2s_v5</code> nodes in Central India with the Azure CNI default profile.

The workload boundary is equally intentional:

- The primary demo runs in <code>otel-demo</code>.
- The only public workload service in the primary demo is <code>frontend-proxy</code>, exposed as a LoadBalancer on port 8080.
- The small <code>azure-webapp</code> canary is currently one replica behind a ClusterIP service on port 80; it is not the public demo entry point.

## GitOps with Argo CD

Argo CD is the Kubernetes desired-state controller. It is not provisioned by Terraform in this repository; an operator installs Argo CD, then applies the tracked Application manifests.

| Application | Git path and destination | Reconciliation settings |
| --- | --- | --- |
| <code>opentelemetry-demo</code> | <code>helm/opentelemetry-demo</code> from <code>main</code> to <code>otel-demo</code> | Automated sync, prune, self-heal, and CreateNamespace |
| <code>azure-webapp</code> | <code>helm/azure-webapp</code> from <code>main</code> to <code>default</code> | Automated sync, prune, self-heal, and CreateNamespace |

Terraform creates core in-cluster prerequisites, while Argo CD owns the Helm-rendered workload resources. Keeping these ownership lines explicit avoids competing reconcilers.

## Demo Workload

The primary workload is the official Astronomy Shop demo, packaged through the committed Helm wrapper. This repository’s contribution is the Azure foundation and GitOps delivery configuration, not application authorship.

The wrapper chart is version 0.1.0 with app version 3.0.0 and pins its upstream Helm dependency at 0.41.0 through the committed lockfile. The base profile retains 17 deployment objects:

<details>
<summary>17 retained base deployments</summary>

<br>

astronomy-db, cart, checkout, currency, email, flagd, frontend, frontend-proxy, kafka, load-generator, otel-collector, payment, product-catalog, quote, recommendation, shipping, and valkey-cart.
</details>

The <code>frontend-proxy</code> LoadBalancer is the public entry point on port 8080.

## CI and Delivery Automation

| Workflow | Trigger | Verified responsibility | Explicit boundary |
| --- | --- | --- | --- |
| <code>validate.yml</code> | Relevant pull requests; pushes to <code>main</code> and <code>feature/**</code> | Terraform format, backend-free init and validate; Helm dependency build, lint, and base-chart rendering | No Terraform plan/apply, security scan, Kubernetes server dry-run, or cluster deployment |
| <code>deploy-aks.yml</code> | Pushes to <code>main</code> affecting <code>app/**</code>, or manual workflow dispatch | OIDC Azure login, builds and pushes the canary with the short Git SHA, changes only the canary image tag in Helm values, bot-commits desired state | No AKS credential retrieval, kubectl, Helm upgrade, or direct Argo CD invocation |

The canary image is currently declared as one replica, ClusterIP, and port 80. Historical validation described an earlier two-replica/LoadBalancer self-heal exercise; that is kept as dated evidence only and is not represented as the current desired state.

### Terraform versus GitOps Ownership

| Terraform owns | Argo CD owns |
| --- | --- |
| Azure resource group, VNet/subnet, AKS, ACR, Azure role assignments, and in-cluster prerequisites | Helm-rendered canary and demo workloads, services, deployments, values-driven workload configuration, and reconciliation state |

This split makes cloud lifecycle and Kubernetes application lifecycle independently understandable. It also prevents the delivery workflow from becoming a second Kubernetes deployment controller.

## Security Posture

The controls below are source-backed; they are not a claim of a fully production-hardened environment.

| Control | Implemented evidence |
| --- | --- |
| GitHub delivery authentication | The canary workflow uses GitHub OIDC with Microsoft Entra ID rather than a long-lived Azure cloud credential. The Entra application/federation and <code>AcrPush</code> assignment are external setup. |
| Registry access | ACR admin access is disabled; the AKS kubelet identity receives <code>AcrPull</code> scoped to the project registry. |
| Network permission | The AKS control-plane identity receives <code>Network Contributor</code> scoped to the AKS subnet. |
| Terraform state access | The backend template uses Azure CLI/Microsoft Entra authentication and requires Storage Blob Data Contributor instead of a storage-account access key. |
| Repository hygiene | Local backend coordinates, Terraform variables, and kubeconfig paths are designed to remain ignored rather than committed. |

Private AKS/ACR, private endpoints, network policy, Key Vault, policy add-ons, Defender, image signing, SBOMs, and secret rotation are future hardening opportunities rather than implemented claims.

## Reuse and Customization

| Concern | Where to change it | Notes |
| --- | --- | --- |
| Azure region, environment tag, node count, VM size | Root variables and local <code>terraform.tfvars</code> | Main defaults are Central India, dev, two <code>Standard_D2s_v5</code> nodes. |
| Dedicated Kubernetes connection | <code>kubeconfig_path</code> input | Terraform Kubernetes-provider objects require a dedicated target-cluster kubeconfig. |
| Demo composition | <code>helm/opentelemetry-demo/values.yaml</code> | Base profile retains 17 deployment objects. |
| Public demo exposure | Demo values for <code>frontend-proxy</code> | The primary public entry point is port 8080. |
| Canary image/tag | <code>helm/azure-webapp/values.yaml</code> | Delivery workflow updates only the image tag. |
| GitOps source | <code>argocd/*.yaml</code> | The primary Applications track <code>main</code>. |

## Quick Start

### Prerequisites

- Azure CLI authenticated to a subscription where the operator can create the declared resources and role assignments.
- Terraform 1.6.0 or newer.
- kubectl, Helm, Git, and access to an Argo CD installation for the target cluster.
- A remote Terraform backend you own. The backend itself is intentionally external to this configuration.
- Storage Blob Data Contributor for the Terraform operator on the remote-state container when using the supplied AzureRM backend pattern.

### 1. Configure local inputs and backend

~~~bash
git clone <repository-url>
cd azure-aks-gitops-delivery

cp terraform.tfvars.example terraform.tfvars
cp backend.hcl.example backend.hcl
~~~

Set a dedicated target-cluster kubeconfig path in the ignored <code>terraform.tfvars</code>. Replace the placeholder backend coordinates in the ignored <code>backend.hcl</code>; the supplied example uses Azure CLI/Microsoft Entra authentication.

~~~bash
terraform init -reconfigure -backend-config=backend.hcl
terraform fmt -check -recursive
terraform validate
~~~

### 2. Bootstrap AKS before Kubernetes-provider resources

Terraform creates a small set of Kubernetes objects, so the dedicated kubeconfig must exist before the complete apply. The safe bootstrap pattern is:

~~~bash
terraform plan -target=module.aks -out=aks-bootstrap.tfplan
terraform apply aks-bootstrap.tfplan

PROJECT_KUBECONFIG=/secure/path/project.kubeconfig
az aks get-credentials \
  --resource-group "$(terraform output -raw resource_group_name)" \
  --name "$(terraform output -raw aks_cluster_name)" \
  --file "$PROJECT_KUBECONFIG"
~~~

Ensure <code>kubeconfig_path</code> in the ignored variables file is the same dedicated file, then complete the platform apply:

~~~bash
terraform plan -out=platform.tfplan
terraform apply platform.tfplan
~~~

### 3. Install Argo CD separately and apply tracked Applications

Argo CD installation/bootstrap is intentionally outside this Terraform configuration. After Argo CD is available in the target cluster, apply the tracked desired-state objects:

~~~bash
kubectl --kubeconfig "$PROJECT_KUBECONFIG" apply \
  -f argocd/opentelemetry-demo-application.yaml

kubectl --kubeconfig "$PROJECT_KUBECONFIG" apply \
  -f argocd/azure-webapp-application.yaml
~~~

### 4. Validate convergence and access the demo

~~~bash
kubectl --kubeconfig "$PROJECT_KUBECONFIG" get nodes

kubectl --kubeconfig "$PROJECT_KUBECONFIG" -n argocd \
  get applications

kubectl --kubeconfig "$PROJECT_KUBECONFIG" -n otel-demo \
  get deployments

kubectl --kubeconfig "$PROJECT_KUBECONFIG" -n otel-demo \
  get service frontend-proxy
~~~

Open the external address reported by <code>frontend-proxy</code> on port 8080.

### 5. Destroy only after review

Review the active workspace, planned resources, and output names before applying any destroy plan. Obtain explicit approval before applying the reviewed plan.

~~~bash
terraform workspace show
terraform output resource_group_name
terraform output aks_cluster_name

terraform plan -destroy -out=destroy.tfplan
terraform show -no-color destroy.tfplan
~~~

## Troubleshooting Playbook

| Symptom | Check | Likely boundary to inspect |
| --- | --- | --- |
| Argo Application is OutOfSync | Inspect the Application source revision, rendered path, and resource tree | Git branch/path/values and Argo reconciliation status |
| Pods are not Ready | Inspect deployments, events, and requests/limits in <code>otel-demo</code> | Capacity, image pulls, and chart values |
| LoadBalancer has no external address | Inspect <code>frontend-proxy</code> service and Azure load-balancer provisioning events | Service type, AKS networking, and Azure quota/network state |
| Terraform cannot manage Kubernetes handoff objects | Confirm <code>kubeconfig_path</code> points to the intended dedicated cluster file | Bootstrap order and target-cluster context |

## Design Decisions and Trade-offs

### Key Decisions

- **GitOps instead of CI-driven kubectl:** CI promotes the canary image through Git. Argo CD reconciles Kubernetes desired state. This keeps a reviewable desired-state history and avoids hidden imperative deployment steps in CI.
- **Demo workload as a deployment target, not product code:** A real multi-service demo exercises the delivery path while keeping platform work distinct from application authorship.
- **Terraform ownership limited to platform prerequisites:** Cloud resources and explicit in-cluster prerequisites stay with Terraform; Helm workload objects stay with Argo CD.

### Scope and Limitations

- This is a demonstrable engineering platform, not a high-availability production topology.
- The main profile uses a static two-node pool; no autoscaling, zone distribution, PDBs, or backup/DR configuration is declared.
- The demo public endpoint uses a LoadBalancer for accessibility; no ingress, custom domain, TLS termination, or WAF is configured here.

### Production-Hardening Opportunities

These are future options, not present-tense claims:

- Private AKS API, private ACR, Private Link/private endpoints, and egress control.
- Network policies, Azure Policy, Defender, workload admission controls, and hardened pod security settings.
- Key Vault CSI integration and a defined secret-rotation strategy.
- Availability-zone-aware node pools, autoscaling, PodDisruptionBudgets, backup/restore, and disaster-recovery design.
- Ingress with TLS and WAF, custom domains, and controlled public exposure.
- Multi-environment promotion, progressive delivery, image provenance/signing, SBOMs, and supply-chain scanning.

## Key Takeaways

- The repository documents a clear control-plane split: Terraform for Azure platform resources, GitHub Actions for canary image promotion, and Argo CD for workload reconciliation.
- Git is the delivery contract: CI updates the declared image version and Argo CD applies the resulting desired state.
- The project is a focused foundation for evolving GitOps delivery practices across environments.
