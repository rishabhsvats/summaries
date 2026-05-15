# OpenShift Core Elements: Startup & Configuration

A layered breakdown of OpenShift's core architecture across the bootstrap, machine config, and core component layers.


 

---

## 1. Bootstrap Layer

The bootstrap layer is the most transient layer — it exists only to bring the cluster to life, then dissolves.

When an OpenShift installation begins, a **temporary bootstrap node** is provisioned first. This node hosts:
- A minimal etcd instance
- A temporary Kubernetes API server
- Machine Config Operator (MCO) manifests

Its sole job is to hand off control to the first set of control plane (master) nodes. Once those masters are stable and etcd is running as a quorum across them, the bootstrap node is decommissioned. This is a one-way gate — nothing in a running cluster remembers it.

### Ignition

The underlying **Ignition** system powers this. Ignition is a provisioning tool that reads declarative JSON configs (ignition configs) at first boot and wires up:
- Disk partitions
- Filesystem mounts
- systemd units
- Files

All of this happens before any user-space process runs. Every node — bootstrap, control plane, and worker — is burned into existence this way. You don't SSH in and configure; the machine configures itself from a spec.

---

## 2. Machine Config Layer

The machine config layer is where low-level node configuration lives as declarative, versioned objects.

The **Machine Config Operator (MCO)** is the component responsible for managing the OS-level configuration of every node in the cluster. It works through a few key objects:

### MachineConfig

A `MachineConfig` resource defines a desired state for a node's OS, including:
- Kernel arguments (e.g. `selinux=enforcing`)
- systemd unit files (custom services)
- File contents under `/etc` and other paths
- Entire filesystem overlays via ignition

### MachineConfigPool

A `MachineConfigPool` groups nodes that should share the same configuration (e.g. all workers vs. all masters) and controls rolling update behavior:
- How many nodes can be draining at once
- Pause and resume capability

### How updates work

When you apply a new `MachineConfig`, the MCO:
1. Renders it together with all other configs in that pool
2. Generates a new ignition config
3. Drains and reboots each affected node in turn

This is not a hot-patch — it's a **full node reboot** to apply changes. This guarantees that running OS state always matches the declared config.

### RHCOS (Red Hat CoreOS)

`CoreOS` (RHCOS) is the immutable OS beneath this layer. It's not a general-purpose OS you configure by hand; the MCO is the only sanctioned way to change it.

---

## 3. Core Component Layer

The core component layer is the permanent operational heart of OpenShift, running as **Cluster Operators**.

OpenShift wraps Kubernetes core components behind its own operator framework. Every capability — networking, storage, authentication, DNS, image registry, ingress — is managed by a dedicated Cluster Operator (CO).

### Cluster Version Operator (CVO)

The **CVO** is the supervisor of all Cluster Operators. It:
- Reads a release image
- Determines the desired state of every CO
- Drives them toward that state

This is how OpenShift upgrades work — you point the CVO at a new release, and it orchestrates the rolling update of every component.

### Kubernetes Core Operators (Blue tier)

| Operator | Manages | Notes |
|---|---|---|
| KAS Operator | `kube-apiserver` | Runs as static pod on masters |
| KCM Operator | `kube-controller-manager` | Runs as static pod on masters |
| Scheduler Operator | `kube-scheduler` | Runs as static pod on masters |
| etcd Operator | etcd quorum | Handles TLS rotation, member replacement |

### OpenShift-Added Operators (Teal tier)

| Operator | Manages | Notes |
|---|---|---|
| DNS Operator | CoreDNS-based DNS | Service discovery |
| Ingress Operator | HAProxy-based ingress | External traffic routing |
| Auth Operator | OAuth server | LDAP, OIDC, GitHub identity providers |
| Image Registry Operator | In-cluster registry | PVC-backed storage |
| Network Operator | OVN-Kubernetes / SDN | CNI layer |

---

## How the Three Layers Connect

```
Bootstrap Layer
  └── Seeds Machine Config + Core Component layers (one-time)

Machine Config Layer
  └── Governs OS-level reality on each node
        (kernel, filesystem, systemd via RHCOS + MCO)

Core Component Layer
  └── Governs Kubernetes and OpenShift capability
        (APIs, networking, auth, ingress via Cluster Operators + CVO)
```

The unifying principle across all three layers is **declarative, operator-driven reconciliation**. You declare what you want; the appropriate operator detects drift and corrects it — whether that's a node rebooting to apply a new `MachineConfig`, or the CVO rolling out a new release image component by component.

---

The Four Installation Methods
OpenShift can be installed using four primary methods: IPI (Installer Provisioned Infrastructure), UPI (User Provisioned Infrastructure), Assisted Installer, and Agent-based Installer.

# OpenShift auto-scaling behaviour by installation method

## UPI (User Provisioned Infrastructure)

UPI effectively disconnects MAPI from infrastructure at install time. You deleted the `MachineSet` manifests to avoid MAPI fighting your hand-provisioned nodes. So when you want to scale, you provision the node yourself (boot RHCOS, point it at `https://api-int.<cluster>:22623` for its ignition config), and then MCO handles the rest.

There is no auto-scaling in the traditional sense unless you bolt on something like the `cluster-api-provider-agent` or pre-register bare metal inventory with metal3 as a Day-2 operation.

## Assisted Installer

Assisted Installer gives you full auto-scaling if and only if you maintain a `BareMetalHost` inventory — essentially a pool of registered physical hosts with BMC credentials. The Cluster Autoscaler can then trigger metal3 to claim a host from that pool, provision RHCOS onto it, and MCO takes over once the node boots.

Without that inventory, you're back to manual provisioning.

## Agent-based Installer

Agent-based on vSphere is the smoothest story — the vSphere provider in MAPI knows how to clone VMs, so MachineSet-driven scaling works end-to-end even in a disconnected environment (as long as the mirror registry is reachable).

On bare metal with `platform: none`, it's the same constraint as UPI: you need to either pre-register `BareMetalHost` CRs with Ironic, or provision nodes manually and let MCO handle the config convergence.


