# 🧠 Brainstorming: RHACM Policy Strategy for OpenShift Virtualization

**Date:** December 12, 2025
**Topic:** Standardization and Governance of Virtualization using RHACM Policies
**Goal:** Define a "Infrastructure as Code" approach to managing Virtual Machines across the fleet, moving from manual hypervisor tweaking to declarative policy enforcement.

---
existing blog:
https://github.com/ch-stark/blogvmdeleteprotection/blob/main/index.md



## 🏗️ Phase 1: Infrastructure & Lifecycle (Day 0/1)
*Focus: Ensuring the platform is capable of running VMs consistently across all clusters.*

### 1. The "Operator" Policy
* **Goal:** Ensure OpenShift Virtualization is installed and healthy on target clusters.
* **Policy Elements:**
    * Create `Namespace` (`openshift-cnv`).
    * Create `OperatorGroup` (Targeting `openshift-cnv`).
    * Create `Subscription` (Channel: `stable`, Approval: `Automatic`).
* **Discussion Points:**
    * *Do we version-lock the Operator (Manual approval) or allow auto-upgrades?*

### 2. The "HyperConverged" Config Policy
* **Goal:** Standardize the behavior of the virtualization engine.
* **Policy Elements:**
    * Deploy `HyperConverged` CR.
    * **Drift Check:** Ensure no one changes `liveMigrationConfig` locally.
* **Key Settings to Standardize:**
    * `liveMigrationConfig` (Bandwidth limits, parallel migrations).
    * `workloadUpdateStrategy` (LiveMigrate vs. Evict).
    * `featureGates` (Enable/Disable GPU passthrough, etc.).

### 3. Networking Prerequisites (The "Plumbing")
* **Goal:** VMs need L2 bridging. Containers usually don't. We must ensure the bridge exists.
* **Policy Elements:**
    * Check/Create `NodeNetworkConfigurationPolicy` (NNCP) to create a Linux Bridge (`br-ex`) on workers.
    * Enforce `NetworkAttachmentDefinition` (Multus) in the default namespace so VMs can attach to it.
* **Risk:** *Applying NNCP can momentarily disrupt node networking. How do we roll this out safely using RHACM Rollout Strategies?*

---

## 📦 Phase 2: Workload Enablement (Day 2)
*Focus: Making it easy for developers/admins to consume VMs.*

### 4. Golden Image Distribution (The "Catalog")
* **Goal:** Developers should see the same "RHEL 9" and "Windows 2022" templates on every cluster.
* **Policy Elements:**
    * Sync `Template` objects to the `openshift` namespace.
    * Sync `BootSource` / `DataVolume` definitions (referencing a central registry/nexus).
    * **Cleanup:** Remove default "Community" templates that we don't support.

### 5. Storage Class Defaults
* **Goal:** VMs often require specific storage features (RWX for Live Migration, Block mode for performance).
* **Policy Elements:**
    * Annotate a specific StorageClass (e.g., `ocs-storagecluster-ceph-rbd`) as the default for KubeVirt.

---

## 🛡️ Phase 3: Governance & Guardrails
*Focus: Protecting the cluster from "Noisy Neighbor" VMs.*

### 6. Resource Quotas & Limits
* **Goal:** Prevent a rogue VM from eating all CPU/RAM on a node.
* **Policy Elements:**
    * **MustHave:** `LimitRange` in user namespaces with default memory requests for VMs.
    * **MustNotHave:** VMs in the `default` namespace.

### 7. MAC Address Governance
* **Goal:** Prevent IP collisions in the physical network.
* **Policy Elements:**
    * **OPA/Kyverno Policy:** Reject VMs with manually defined MAC addresses unless the user has the `network-admin` role.

---

## 🔄 Phase 4: Day 2 Operations (Handling "Copy Update")
*Focus: Managing the disconnect between updated Templates and running VMs.*

### 8. The "Recreate" Strategy (For Stateless/Cattle)
* **Goal:** Ensure VMs referencing a template are always running the latest version.
* **Mechanism:** GitOps / ArgoCD.
* **Policy Approach:**
    * Treat VMs as disposable.
    * When the `Template` is updated in the policy, trigger a re-sync in ArgoCD to delete/recreate the VM pods.
    * *Best for:* Worker nodes, CI/CD agents, calculation nodes.

### 9. The "In-Place" Strategy (For Stateful/Pets)
* **Goal:** Update the OS/App inside long-running VMs without destroying the VM object.
* **Mechanism:** RHACM integration with **Ansible Automation Platform (AAP)**.
* **Policy Approach:**
    * **Pre-hook/Post-hook:** When RHACM detects a policy violation (or on a schedule), trigger an Ansible Job Template.
    * The Ansible job connects to the VM (via SSH/WinRM) and performs `dnf update` or applies patches.
    * *Best for:* Databases, Legacy Applications, Windows Servers.

---

## 📋 Prioritization Matrix

| Policy Idea | Impact | Complexity | Priority |
| :--- | :--- | :--- | :--- |
| **Operator Install** | High (Critical) | Low | **P0** |
| **HyperConverged CR** | High (Critical) | Low | **P0** |
| **Network (Multus/Bridge)** | High | High (Risk of outage) | **P1** |
| **Golden Image Sync** | Medium | Medium | **P2** |
| **VM Update Strategy** | Medium | High (Requires Ansible) | **P3** |




Further ideas:

This is a technically sound and comprehensive list of policies. You have covered the "Big Three" of infrastructure (Compute, Network, Storage) alongside Governance/Lifecycle.

I have reviewed your list for technical accuracy and "made it nice" by standardizing the terminology, removing the trailing commas/typos, and organizing it into a clean, professional catalog suitable for documentation or a presentation.

RHACM Policy Catalog: OpenShift Virtualization
I. Security & Lifecycle Policies
Governs the creation, protection, and retirement of Virtual Machines.

Enforce VM Delete Protection

Mode: Enforce

Description: Iterates through all non-system namespaces and automatically applies the kubevirt.io/vm-delete-protection: "True" label to all existing VirtualMachine objects. This acts as a safety barrier against accidental deletion via the UI or CLI.

Audit Missing Delete Protection

Mode: Inform

Description: Monitors the cluster and reports on any VirtualMachine resource that lacks the delete protection label. This provides visibility into the security posture and compliance gaps before strictly enforcing the label cluster-wide.

Restrict Modification of Protection Labels

Mechanism: Gatekeeper / OPA

Description: Blocks specific low-privilege service accounts (e.g., standard operators or developers) from removing the delete protection label. Modification is restricted to requests originating from a designated administrative group (e.g., supervmadmin).

Auto-Prune Stale/Temporary VMs

Mode: Enforce (ComplianceType: MustNotHave)

Description: Automatically deletes Virtual Machines tagged as temporary (e.g., labeled pipeline-created: "true") if their creation timestamp exceeds a defined threshold (e.g., 24 hours), preventing resource sprawl.

Standardize High-Availability Run Strategy

Mode: Enforce

Description: Mandates that critical Virtual Machines utilize runStrategy: Always. This ensures VMs are automatically rescheduled and recreated on a healthy node if the original node fails or is recycled.

II. Resource & Performance Optimization
Ensures VMs get the compute resources they need without destabilizing the cluster.

Mandate Dedicated CPU Placement

Target: High-Performance Workloads

Description: Enforces that VMs using specialized instance types (e.g., CX series) have dedicatedCpuPlacement: true set. This guarantees CPU pinning and isolates the workload from "noisy neighbors."

Enforce vNUMA Topology

Target: Latency-Sensitive Workloads

Description: Requires that large or high-performance VMs include configuration for vNUMA topology mapping (e.g., spec.domain.cpu.numa). This ensures memory and CPU resources are locally aligned on the physical host for maximum throughput.

Control CPU Overcommitment

Target: HyperConverged CR

Description: Enforces a maximum limit on the vmiCPUAllocationRatio within the HyperConverged Cluster Resource. This strictly balances VM density against performance risks by preventing excessive over-provisioning.

Standardize Instance Types

Description: Mandates that all new VMs utilize an approved VirtualMachineClusterInstancetype. This enforces a baseline configuration profile, standardizing vCPU and memory sizing across managed clusters.

Automate Resource Limits in Quota Namespaces

Description: Enforces automatic calculation of memory limits in namespaces subject to ResourceQuotas. It checks for the alpha.kubevirt.io/auto-memory-limits-ratio label or validates that manual limits satisfy the minimum 100Mi overhead rule.

III. Storage Policies
Optimizes I/O performance and enables advanced features like Live Migration.

Enforce RWX Access Mode for Migration

Description: Mandates that all VM Persistent Volume Claims (PVCs) intended for live migration utilize the ReadWriteMany (RWX) access mode (or supported Block/RWO shared backends), ensuring data accessibility across nodes during migration.

Prefer Block Volume Mode

Description: Ensures that storage provisioned for VM disks utilizes VolumeMode: Block where supported (e.g., ODF/Ceph RBD). Block mode bypasses the filesystem layer, offering superior I/O performance.

Mandate Windows-Optimized Storage Class

Description: Enforces the usage of a specific StorageClass for Windows VMs (e.g., one with disabled caching or specific sector sizes) to prevent performance degradation and filesystem corruption.

Configure CDI Scratch Space

Target: HyperConverged CR

Description: Enforces a specific scratchSpaceStorageClass to dedicate high-performance storage for temporary Containerized Data Importer (CDI) operations, speeding up image imports and uploads.

Enable Persistent Reservation (SCSI)

Target: HyperConverged CR

Description: Ensures the persistentReservation feature gate is enabled. This allows LUN-backed block mode disks to be shared among multiple VMs, a requirement for Windows Failover Clustering.

IV. Network & Connectivity Policies
Manages traffic flow, hardware acceleration, and guest introspection.

Enforce Dedicated Migration Network

Target: HyperConverged CR

Description: Verifies the presence of the spec.liveMigrationConfig.network field. This ensures live migration traffic is offloaded to a secondary Multus network, preventing saturation of the primary CNI/Pod network.

Standardize on VirtIO Interfaces

Description: Enforces the use of the virtio NIC model (rejecting e1000e or rtl8139) to maximize network throughput and ensure compatibility with non-x86 architectures like IBM Z and ARM64.

Protect KubeMacPool Assignment

Description: Audits namespaces to ensure KubeMacPool is not disabled (checks for mutatevirtualmachines.kubemacpool.io=ignore). This safeguards the automatic assignment of unique MAC addresses to avoid network conflicts.

Enforce SR-IOV Node Policies

Target: High-Throughput Clusters

Description: Enforces the existence of SriovNetworkNodePolicy objects on relevant nodes. This guarantees that hardware-accelerated SR-IOV Virtual Functions (VFs) are configured and exposed for low-latency VM networking.

Require QEMU Guest Agent Readiness

Description: audits VMs to ensure the QEMU Guest Agent is installed and reporting "Ready." This agent is required for IP address reporting, graceful shutdowns, and application-consistent snapshots.

Implementation Notes & Review
The "Singleton" Challenge: Several of your policies (8, 14, 15, 16) target the HyperConverged Custom Resource. Since there is usually only one of these per cluster, your ACM Policy must be careful to patch the resource rather than overwrite it, otherwise, you might inadvertently wipe out other local settings.

Gatekeeper Integration (Policy #3): You correctly identified that ACM's standard ConfigurationPolicy cannot easily prevent modification of a label by a specific user. This definitely requires an Admission Controller like OPA Gatekeeper or Kyverno.

Compliance Type: For Policy #4, ensuring the complianceType is set to Mustnothave is crucial. Be careful with the object selector so you don't delete VMs that should exist but just happen to be old. The label selector (pipeline-created: "true") is a critical safety mechanism here.

Would you like me to generate the actual RHACM YAML manifest for any of these specific policies?














