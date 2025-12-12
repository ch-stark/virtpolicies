# 🧠 Strategy: RHACM Policy Architecture for OpenShift Virtualization

**Date:** December 12, 2025
**Subject:** Standardization and Governance of Virtualization using RHACM
**Goal:** Define an "Infrastructure as Code" approach to managing Virtual Machines across the fleet, transitioning from manual hypervisor configuration to declarative policy enforcement.

---

## 🏗️ Part 1: Strategic Rollout (The Phases)
*A phased approach to building the foundation and enabling workloads.*

### Phase 1: Infrastructure & Lifecycle (Day 0/1)
*Focus: Ensuring the platform is capable of running VMs consistently across all clusters.*

* **1. The "Operator" Policy**
    * **Goal:** Ensure OpenShift Virtualization is installed and healthy.
    * **Specs:** Create `Namespace` (`openshift-cnv`), `OperatorGroup`, and `Subscription` (Channel: `stable`).
    * **Decision Point:** Determine if upgrades should be automatic or gated via manual approval.

* **2. The "HyperConverged" Config Policy**
    * **Goal:** Standardize the virtualization engine behavior.
    * **Specs:** Deploy `HyperConverged` CR and monitor for drift.
    * **Key Settings:** `liveMigrationConfig` (bandwidth/parallelism), `workloadUpdateStrategy` (LiveMigrate vs. Evict), and `featureGates`.

* **3. Networking Prerequisites (The "Plumbing")**
    * **Goal:** Establish L2 bridging (essential for VMs, unlike containers).
    * **Specs:** Enforce `NodeNetworkConfigurationPolicy` (NNCP) for Linux Bridges (`br-ex`) and `NetworkAttachmentDefinition` (Multus).
    * **Risk:** NNCP application can disrupt node networking; requires careful RHACM Rollout Strategies.

### Phase 2: Workload Enablement (Day 2)
*Focus: Making it easy for developers/admins to consume VMs.*

* **4. Golden Image Distribution (The "Catalog")**
    * **Goal:** Uniform availability of "RHEL 9" and "Windows 2022" templates.
    * **Specs:** Sync `Template`, `BootSource`, and `DataVolume` objects referencing a central registry.
    * **Cleanup:** Prune default "Community" templates to reduce noise.

* **5. Storage Class Defaults**
    * **Goal:** Ensure VMs use storage optimized for features like Live Migration.
    * **Specs:** Annotate specific StorageClasses (e.g., `ocs-storagecluster-ceph-rbd`) as the default for KubeVirt.

### Phase 3: Governance & Guardrails
*Focus: Protecting the cluster from "Noisy Neighbor" VMs.*

* **6. Resource Quotas & Limits**
    * **Goal:** Prevent rogue VMs from exhausting node resources.
    * **Specs:** Enforce `LimitRange` in user namespaces; `MustNotHave` VMs in the `default` namespace.

* **7. MAC Address Governance**
    * **Goal:** Prevent physical network IP collisions.
    * **Specs:** Use OPA/Kyverno to reject manually defined MAC addresses unless the user has `network-admin` privileges.

### Phase 4: Operations (Handling "Copy Update")
*Focus: Managing the disconnect between updated Templates and running VMs.*

* **8. The "Recreate" Strategy (Stateless/Cattle)**
    * **Mechanism:** GitOps / ArgoCD.
    * **Logic:** When the `Template` updates, trigger a re-sync to delete/recreate VM pods. Best for workers and CI agents.

* **9. The "In-Place" Strategy (Stateful/Pets)**
    * **Mechanism:** RHACM + Ansible Automation Platform (AAP).
    * **Logic:** RHACM detects policy violation -> Triggers Ansible Job -> Job performs `dnf update` via SSH/WinRM. Best for DBs and Windows Servers.

---

## 📚 Part 2: The Policy Catalog
*A detailed technical inventory of policies to be implemented.*

### I. Security & Lifecycle
*Governs the creation, protection, and retirement of Virtual Machines.*

| Policy Name | Mode | Description |
| :--- | :--- | :--- |
| **Enforce VM Delete Protection** | `Enforce` | Applies `kubevirt.io/vm-delete-protection: "True"` to all non-system VMs to prevent accidental UI/CLI deletion. |
| **Audit Missing Protection** | `Inform` | Reports on any VM resource lacking the delete protection label to identify compliance gaps. |
| **Restrict Label Modification** | `OPA` | Blocks low-privilege accounts from removing protection labels; restricts this action to `supervmadmin`. |
| **Auto-Prune Stale VMs** | `Enforce` | Automatically deletes VMs labeled `pipeline-created: "true"` if the age exceeds 24 hours. |
| **Standardize HA Run Strategy** | `Enforce` | Mandates `runStrategy: Always` for critical VMs to ensure rescheduling upon node failure. |

### II. Resource & Performance Optimization
*Ensures VMs get required compute without destabilizing the cluster.*

| Policy Name | Target | Description |
| :--- | :--- | :--- |
| **Mandate Dedicated CPU** | High-Perf | Enforces `dedicatedCpuPlacement: true` for specialized instance types (e.g., CX series). |
| **Enforce vNUMA Topology** | Latency | Requires vNUMA topology mapping for large VMs to align memory/CPU on the physical host. |
| **Control CPU Overcommit** | `HCO` | Limits `vmiCPUAllocationRatio` to balance density against performance risks. |
| **Standardize Instances** | Cluster | Mandates use of approved `VirtualMachineClusterInstancetype` to baseline vCPU/Memory sizing. |

### III. Storage Policies
*Optimizes I/O performance and enables advanced features.*

| Policy Name | Description |
| :--- | :--- |
| **Enforce RWX for Migration** | Mandates `ReadWriteMany` (RWX) or shared Block/RWO for PVCs intended for live migration. |
| **Prefer Block Volume Mode** | Enforces `VolumeMode: Block` where supported (e.g., Ceph RBD) to bypass filesystem overhead. |
| **Windows-Optimized Storage** | Enforces specific StorageClasses for Windows VMs (e.g., disabled caching, specific sector sizes). |
| **Configure CDI Scratch** | Enforces `scratchSpaceStorageClass` in `HyperConverged` CR to speed up image imports. |

### IV. Network & Connectivity
*Manages traffic flow, hardware acceleration, and guest introspection.*

| Policy Name | Description |
| :--- | :--- |
| **Dedicated Migration Net** | Verifies `spec.liveMigrationConfig.network` is set to offload migration traffic to a secondary Multus network. |
| **Standardize VirtIO** | Enforces `virtio` NIC models (rejecting e1000e/rtl8139) for max throughput. |
| **Protect KubeMacPool** | Audits against `mutatevirtualmachines.kubemacpool.io=ignore` to ensure unique MAC assignment. |
| **Require Guest Agent** | Audits VMs to ensure QEMU Guest Agent is "Ready" (required for IP reporting and consistent snapshots). |

---

## 📋 Part 3: Execution Guide

### Prioritization Matrix

| Policy Idea | Impact | Complexity | Priority |
| :--- | :--- | :--- | :--- |
| **Operator Install** | High (Critical) | Low | **P0** |
| **HyperConverged CR** | High (Critical) | Low | **P0** |
| **Network (Multus/Bridge)** | High | High (Risk of outage) | **P1** |
| **Golden Image Sync** | Medium | Medium | **P2** |
| **VM Update Strategy** | Medium | High (Requires Ansible) | **P3** |

### Implementation Notes

1.  **The "Singleton" Challenge:** Policies targeting the `HyperConverged` Custom Resource (e.g., CPU Overcommit, Migration Network) must use a **patch** strategy. Since there is only one HCO per cluster, overwriting it completely will wipe out local settings.
2.  **Gatekeeper Integration:** Standard ACM ConfigurationPolicies cannot easily restrict *who* changes a label (Label Modification Policy). This requires an Admission Controller like OPA Gatekeeper or Kyverno.
3.  **Compliance Logic:** For "Auto-Prune Stale VMs," ensure the `complianceType` is set to `MustNotHave`. The object selector (e.g., `pipeline-created: "true"`) is a critical safety mechanism to avoid deleting permanent production VMs.
