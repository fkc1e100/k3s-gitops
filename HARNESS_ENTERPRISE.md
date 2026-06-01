# 🏦 Enterprise Agentic Harness: GKE & Regulatory Compliance (GitOps-Driven)

This document provides a comprehensive architectural blueprint, security model, and risk-management assessment for deploying the **Hermes Agentic Harness** inside a highly regulated **Google Cloud GKE (Google Kubernetes Engine)** production environment (such as banking, healthcare, or public sector).

---

## 🛡️ The Core Security Model: Read-Only Cluster / Write-Only Git

In an enterprise environment, allowing an AI agent direct mutation privileges (`kubectl create/patch/delete`) on a live cluster violates basic security compliance principles (like SOC 2, PCI DSS, and ISO 27001). 

To solve this, the Hermes GKE architecture mandates a **Strict GitOps Loop** where the agent functions solely as an **advisor**:

```mermaid
sequence-diagram
    autonumber
    actor Developer as Human Operator
    participant Agent as Hermes (GKE Pod)
    participant Git as Secure Source Manager / GitLab
    participant Argo as ArgoCD Controller
    participant GKE as Live GKE Cluster

    Agent->>GKE: Query health, logs & metrics (Read-Only)
    Note over Agent: Anomaly or request detected
    Agent->>Git: Open tracking Issue & create Branch
    Agent->>Git: Push proposed YAML changes as Pull Request
    Developer->>Git: Reviews PR (Policy-as-Code checks run in CI)
    Developer->>Git: Merges PR to main branch
    Argo->>Git: Observes repository drift
    Argo->>GKE: Synchronizes & Applies changes safely
```

1. **Read-Only Cluster Access:** The GKE ServiceAccount mapped to Hermes has **zero write permissions** inside the GKE API.
2. **Write-Only Git Access:** The agent’s only write authority is directed to your enterprise Git forge to propose changes.
3. **Human-In-The-Loop (HITL) Gatekeeping:** All agent actions must be merged by an authorized human operator.
4. **ArgoCD as the Sole Mutator:** ArgoCD is the only entity with write permissions to the cluster, ensuring auditability and state tracking.

---

## 📑 10 Critical Enterprise GKE Questions

An engineering team preparing to deploy this GitOps agentic harness on GKE must address the following ten architectural dimensions:

### 🌐 A. Platform Standpoint (GCP & Cloud Integrations)

#### 1. Identity & Least-Privilege Role Binding
* *Context:* Raw API keys or static SSH files are disallowed.
* *Question:* How will the team configure **GKE Workload Identity Federation** to bind the Hermes Kubernetes ServiceAccount directly to a GCP IAM Service Account? What minimal GCP IAM roles (e.g., Secret Manager Secret Accessor, Cloud Logging Writer) will be assigned?

#### 2. Externalized Secrets Management
* *Context:* Plaintext secrets must never be committed to Git.
* *Question:* How will sensitive API keys (e.g., Discord/Slack Bot Tokens, Git tokens, or LLM API keys) be retrieved? Will they use **Google Secret Manager** coupled with the **External Secrets Operator (ESO)** or the GCP CSI provider to inject secrets as ephemeral, memory-backed volumes?

#### 3. Private Network Egress Routing
* *Context:* Secure GKE nodes are deployed in private subnets with no public internet routing.
* *Question:* How will outbound internet connectivity (required to establish WebSocket channels to Discord/Slack or HTTPS to external LLMs) be established? Will the team deploy **Cloud NAT** with dedicated egress gateways, or route traffic through highly restricted corporate proxies with strict destination-allowlists?

---

### ☸️ B. Cluster Standpoint (GKE Infrastructure & Storage)

#### 4. GKE Autopilot vs. GKE Standard Compatibility
* *Context:* GKE Autopilot restricts custom node affinity rules and low-level system tolerations.
* *Question:* Will we run on **GKE Autopilot** or **GKE Standard**? If on Autopilot, how will we accommodate Autopilot's automated scheduling constraints, and how will we translate K3s-specific node-selector labels to GKE-compatible requests?

#### 5. High-Availability ReadWriteMany (RWX) Storage
* *Context:* During a GKE node or zone failure, the passive replica pod must immediately attach to the persistent workspace without data corruption.
* *Question:* Since GKE standard persistent disks are block-storage (RWO), what RWX storage class will be provisioned? Will we use **Google Cloud Filestore** (providing fully POSIX-compliant file locking for the underlying SQLite/state databases) or **Cloud Storage FUSE** (which offers lower costs but higher latency and different concurrency locks)?

#### 6. Regional Multi-Zonal Disaster Recovery
* *Context:* Production GKE clusters span multiple Availability Zones (AZs) for high availability.
* *Question:* If the zone hosting the active Hermes pod crashes, how quickly can the GKE controller unmap the Filestore volume and reschedule the pod onto a healthy node in an alternate zone (Recovery Time Objective)? Are the storage classes replicated regionally?

---

### 🔲 C. Namespace Standpoint (Security, Isolation & Policies)

#### 7. Zero-Mutation Namespace RBAC Enforcement
* *Context:* To prevent GKE privilege escalation, the agent must be technically unable to issue direct cluster writes.
* *Question:* How will we configure our Kubernetes RBAC to ensure that absolutely no write verbs (`create`, `update`, `patch`, `delete`) are granted to Hermes across any cluster namespaces? What automated audit logs will verify that the agent is restricted strictly to read-only verbs (`get`, `list`, `watch`)?

#### 8. Pod Security Standards (PSA) Compliance
* *Context:* GKE enforces strict Pod Security Admission profiles on production namespaces.
* *Question:* Does the Hermes image and deployment manifest fully comply with the GKE **`restricted` Pod Security Standard** (e.g., running with `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, and a `readOnlyRootFilesystem`)?

#### 9. Micro-Segmented Inbound & Outbound NetworkPolicies
* *Context:* Compromised pods must be prevented from traversing the internal cluster network.
* *Question:* What **NetworkPolicies** will be applied to the `hermes-system` namespace? Can we enforce a default-deny ingress policy (since Hermes operates entirely on outbound connections) and tightly restrict egress to GKE's DNS service, Secret Manager, and external APIs?

#### 10. Automated Pre-Commit Manifest Validation
* *Context:* LLMs can hallucinate or be targeted by prompt-injection attacks, potentially generating invalid or insecure YAML files.
* *Question:* What pre-flight validation checks will run inside Hermes before a PR is opened? Will we integrate automated validation tools (like `kubeconform`, `Kube-Score`, or a policy engine like `Kyverno` / `Conftest`) directly into the agent’s skill execution path to validate security and syntax compliance before any code is pushed to Git?

---

## 🔒 Outbound Network Security & LLM Infrastructure Selection

When designing your GKE egress NetworkPolicies, the method used to connect Hermes to its Large Language Model represents your primary security boundary:

| LLM Deployment Type | Outbound Network Boundary | Egress Policy Type | Compliance & Privacy Level |
| :--- | :--- | :--- | :--- |
| **Public SaaS APIs**<br/>*(e.g., Claude SaaS, Gemini Public)* | Direct connection to the public internet over HTTPS. | FQDN-based `CiliumNetworkPolicy` (requires GKE Dataplane V2). | **Moderate:** Sensitive cluster logs and metrics must leave the GCP network boundary. |
| **Cloud-Managed APIs**<br/>*(e.g., Gemini/Claude via Vertex AI)* | Internal endpoints over Google's private backbone network. | Private Google Access (VPC Service Controls + GKE Metadata limits). | **High:** Data never touches the public internet and is protected by Google VPC-SC perimeters. |
| **In-Cluster Self-Hosted**<br/>*(e.g., Gemma 4-31B in GKE via vLLM)* | Intra-cluster communication (No internet egress required). | Standard K8s `NetworkPolicy` (restricting egress solely to the local GPU namespace). | **Maximum (Air-Gapped):** Total data sovereignty; ideal for banking core operations. |

### 🛠️ GKE Dataplane V2 (Cilium) FQDN Policy Example (For Public APIs)
If using hosted APIs, standard Kubernetes IP-based NetworkPolicies cannot be used because API IP addresses change dynamically. You must use a GKE Dataplane V2 **`CiliumNetworkPolicy`** to restrict egress strictly by hostname:

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: allow-hermes-external-egress
  namespace: hermes-system
spec:
  endpointSelector:
    matchLabels:
      app: hermes-gateway
  egress:
    # 1. Allow DNS resolution through kube-dns
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
    # 2. Allow egress strictly to approved endpoints over HTTPS (Port 443)
    - toFQDNs:
        - matchName: "api.anthropic.com"          # Claude API
        - matchName: "generativelanguage.googleapis.com" # Gemini API
        - matchName: "api.discord.com"            # Discord API
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

---

## 🏛️ Compliant Git Platforms for Highly Regulated Industries

To enforce a GKE GitOps pipeline in banking, the hosting environment for your Git repositories must be private, audited, and sovereign:

### 1. Google Cloud Secure Source Manager (SSM)
* **The Cloud-Native Ideal:** Secure Source Manager is Google Cloud’s single-tenant, managed, regional source code repository service.
* **Banking Advantages:**
  * **VPC-SC Integration:** Fully enclosable inside a VPC Service Controls perimeter to prevent data exfiltration.
  * **Keyless Authentications:** Connects directly with GKE Workload Identity, eliminating Git Personal Access Tokens.
  * **Regional Residency:** Ensures code repositories are physically locked within specific geographic regions.

### 2. GitLab Self-Managed (In-Cluster / Private VPC)
* **The DevSecOps Standard:** GitLab is heavily favored by financial institutions because of its out-of-the-box compliance pipelines.
* **Banking Advantages:**
  * **Built-in Gates:** Can force mandatory SAST, DAST, secret scanning, and software supply-chain (SBOM) validation on all Hermes-proposed PRs.
  * **Air-Gap Capability:** Can be deployed fully air-gapped within GKE via Helm charts.

### 3. Gitea Enterprise / Forgejo
* **The Lightweight Option:** A Go-based Git forge that has achieved independent **SOC 2 Type II and SOC 3** compliance attestations.
* **Banking Advantages:**
  * **Low Resource Footprint:** Requires a fraction of the RAM/CPU of GitLab, allowing it to run cheaply inside a tiny GKE namespace.
  * **Sovereignty:** Supports complete air-gap configurations.
