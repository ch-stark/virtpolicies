# 🧠 Brainstorming: RHACM Policy Strategy for OpenShift Virtualization

**Date:** December 12, 2025
**Topic:** Standardization and Governance of Virtualization using RHACM Policies
**Goal:** Define a "Infrastructure as Code" approach to managing Virtual Machines across the fleet, moving from manual hypervisor tweaking to declarative policy enforcement.

---

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
