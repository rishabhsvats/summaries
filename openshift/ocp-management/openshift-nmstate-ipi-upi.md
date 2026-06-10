# NMState in IPI vs UPI: Complete Guide
### What is NMState?

NMState is a declarative network state management tool for Linux host networking. In OpenShift/Kubernetes, the kubernetes-nmstate operator allows you to configure host network interfaces declaratively using
Kubernetes resources.

### Key Characteristics

# Example NMState configuration
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: bond0-vlan100
spec:
desiredState:
interfaces:
- name: bond0
  type: bond
state: up
ipv4:
enabled: false
link-aggregation:
mode: 802.3ad
port:
- eth0
```

- eth1
- name: bond0.100
  type: vlan
state: up
vlan:
base-iface: bond0
id: 100
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.100.10
  prefix-length: 24

## What NMState Manages
- ✅ Physical network interfaces (eth0, ens1f0, etc.)
- ✅ Bonding (LACP, active-backup, etc.)
- ✅ VLANs (802.1Q tagging)
- ✅ Bridges (Linux bridge, OVS bridge)
- ✅ IP configuration (static, DHCP)
- ✅ Routes
- ✅ DNS configuration
- ✅ SR-IOV virtual functions

## What NMState Does NOT Manage
- ❌ Pod networking (that's CNI: OVN-Kubernetes, Calico, etc.)
- ❌ Kubernetes Services
- ❌ Ingress/Egress
- ❌ NetworkPolicy (application-level)

Scope: Host-level networking only (node network interfaces)

---
NMState in IPI

Role: Minimal to Optional

In IPI installations, NMState is generally not needed for basic cluster functionality because cloud providers handle host networking.

When NMState is NOT Needed (Most IPI Cases)

# IPI on AWS/Azure/GCP/vSphere
# Cloud provider provisions instances with:
# - Network interfaces (ENI, vNIC)
# - IP addresses (DHCP or cloud-assigned static)
# - Routes (default gateway via cloud metadata)
# - DNS (cloud-provided)

# Example: AWS EC2 instance
# - eth0: Primary ENI with cloud-assigned IP
# - Automatic default route via VPC Internet Gateway
# - DNS from VPC DHCP options

# OpenShift nodes "just work" with cloud networking
```bash
oc get nodes
# All nodes Ready, networking configured by cloud provider
```

## Why NMState isn't needed
- ✅ Cloud provider assigns IPs automatically (DHCP or metadata service)
- ✅ Single network interface per instance (typically) 
- ✅ Simple network topology (cloud-managed)
- ✅ No bonding/VLANs (cloud networking handles HA differently)
- ✅ Routes configured automatically by cloud-init

---
When NMState IS Useful in IPI (Advanced Cases)


**Use Case 1: Multiple Network Interfaces (Multi-NIC)**

# AWS example: Instance with multiple ENIs
# Primary ENI (eth0): Cluster networking
# Secondary ENI (eth1): Storage network or management

# IPI doesn't configure secondary interfaces automatically
# Use NMState to configure eth1:

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: secondary-nic-config
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
desiredState:
interfaces:
- name: eth1
  type: ethernet
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.1.0.10
  prefix-length: 24
routes:
config:
- destination: 192.168.0.0/16
  next-hop-interface: eth1
next-hop-address: 10.1.0.1
EOF
```

---
Use Case 2: SR-IOV with Cloud Instances

# AWS bare metal instances (i3.metal, c5.metal, etc.)
# Support SR-IOV for high-performance networking

# 1. Install SR-IOV Network Operator
```bash
oc apply -f sriov-network-operator.yaml
```

# 2. Use NMState with SR-IOV
# NMState can configure the PF (Physical Function)
# SR-IOV operator creates VFs (Virtual Functions)

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: sriov-pf-config
spec:
nodeSelector:
feature.node.kubernetes.io/network-sriov.capable: "true"
desiredState:
interfaces:
- name: ens1f0
  type: ethernet
state: up
ethernet:
sr-iov:
total-vfs: 8
EOF
```

---
Use Case 3: OpenShift Virtualization on IPI

# Running VMs on OpenShift (KubeVirt) on cloud instances
# VMs need bridge networking for network isolation

# Use NMState to create bridges on worker nodes:
```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: br-vlan100
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
desiredState:
interfaces:
- name: eth1
  type: ethernet
state: up
ipv4:
enabled: false
```

- name: br-vlan100
  type: linux-bridge
state: up
ipv4:
enabled: false
bridge:
options:
stp:
enabled: false
port:
- name: eth1.100
- name: eth1.100
  type: vlan
state: up
vlan:
base-iface: eth1
id: 100
EOF

# Now VMs can attach to br-vlan100 bridge
```bash
cat <<EOF | oc apply -f -
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
name: vlan100-network
spec:
config: |
{
"cniVersion": "0.3.1",
"name": "vlan100-network",
"type": "cnv-bridge",
"bridge": "br-vlan100"
}
EOF
```

---
Use Case 4: Custom Routing for Specific Workloads

# Route specific traffic (e.g., storage traffic) via secondary interface
```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: storage-network-routing
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
desiredState:
routes:
config:
- destination: 10.200.0.0/16  # Storage network CIDR
  next-hop-interface: eth1
next-hop-address: 10.1.0.1
table-id: 200
route-rules:
config:
- ip-from: 10.1.0.0/24
  route-table: 200
EOF
```

---
IPI NMState Deployment (Post-Install)

# NMState is NOT installed by default in IPI
# Deploy kubernetes-nmstate operator after cluster is up:

# 1. Install operator
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
name: openshift-nmstate
```

---
```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
name: openshift-nmstate
namespace: openshift-nmstate
spec: {}
```

---
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
name: kubernetes-nmstate-operator
namespace: openshift-nmstate
spec:
channel: stable
name: kubernetes-nmstate-operator
source: redhat-operators
sourceNamespace: openshift-marketplace
EOF
```

# 2. Create NMState instance
```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NMState
metadata:
name: nmstate
spec: {}
EOF
```

# 3. Verify deployment
```bash
oc get pods -n openshift-nmstate
# NAME                                    READY   STATUS
# nmstate-cert-manager-xxx                1/1     Running
# nmstate-handler-xxx (DaemonSet)         1/1     Running
# nmstate-webhook-xxx                     1/1     Running
```

# 4. Create NodeNetworkConfigurationPolicy (see examples above)

---
NMState in UPI

Role: Important to Critical (Especially Bare Metal)

## In UPI installations, NMState becomes much more relevant, particularly for
- Bare metal installations
- Complex network topologies
- On-premises deployments
- Edge locations
- Custom networking requirements

---

### UPI Installation-Time NMState (Agent-Based Installer)

For Agent-based installer (a UPI variant for bare metal), you can configure NMState at installation time.

agent-config.yaml with NMState

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
name: my-cluster
rendezvousIP: 192.168.1.10
```

# Per-host network configuration using NMState
hosts:
- hostname: master-0
  role: master
rootDeviceHints:
deviceName: /dev/sda
```
interfaces:
- name: eno1
  macAddress: 52:54:00:aa:bb:01
networkConfig:  # ← NMState configuration
interfaces:
- name: bond0
  type: bond
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.1.10
  prefix-length: 24
link-aggregation:
mode: 802.3ad
port:
- eno1
```

- eno2
- name: eno1
  type: ethernet
state: up
- name: eno2
  type: ethernet
state: up
```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 192.168.1.1
next-hop-interface: bond0
dns-resolver:
config:
server:
- 8.8.8.8
```

- 8.8.4.4

- hostname: master-1
  role: master
rootDeviceHints:
deviceName: /dev/sda
```
interfaces:
- name: eno1
  macAddress: 52:54:00:aa:bb:02
networkConfig:
interfaces:
- name: eno1
  type: ethernet
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.1.11
  prefix-length: 24
ipv6:
enabled: true
dhcp: false
address:
- ip: fd00::11
  prefix-length: 64
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 192.168.1.1
next-hop-interface: eno1
```

- destination: ::/0
  next-hop-address: fd00::1
next-hop-interface: eno1
```
dns-resolver:
config:
server:
- 8.8.8.8
```

- hostname: worker-0
  role: worker
rootDeviceHints:
deviceName: /dev/sda
```
interfaces:
- name: ens1f0
  macAddress: 52:54:00:aa:bb:10
networkConfig:
interfaces:
# VLAN for cluster traffic
```

- name: ens1f0.100
  type: vlan
state: up
vlan:
base-iface: ens1f0
id: 100
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.0.100.20
  prefix-length: 24
# VLAN for storage traffic
- name: ens1f0.200
  type: vlan
state: up
vlan:
base-iface: ens1f0
id: 200
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.0.200.20
  prefix-length: 24
- name: ens1f0
  type: ethernet
state: up
ipv4:
enabled: false
```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 10.0.100.1
next-hop-interface: ens1f0.100
```

- destination: 192.168.0.0/16  # Storage network
  next-hop-address: 10.0.200.1
next-hop-interface: ens1f0.200

## What happens

## 1. Agent ISO boots on bare metal hosts

## 2. Agent reads networkConfig (NMState format)

## 3. Configures network interfaces before installation

## 4. Installer uses configured networking for cluster deployment

---
Common UPI Bare Metal NMState Patterns


**Pattern 1: Bonding for High Availability**

# Dual-NIC bonding (LACP 802.3ad)
# Protects against single NIC failure
```
interfaces:
- name: bond0
  type: bond
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.1.10
  prefix-length: 24
link-aggregation:
mode: 802.3ad
options:
miimon: "100"
lacp_rate: fast
port:
- eno1
```

- eno2
- name: eno1
  type: ethernet
state: up
- name: eno2
  type: ethernet
state: up
```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 192.168.1.1
next-hop-interface: bond0
```

Use case: Production bare metal, network HA requirement

---
Pattern 2: Separate Network Planes (VLANs)

# Management VLAN (10)
# Cluster traffic VLAN (100)
# Storage traffic VLAN (200)
```
interfaces:
- name: ens1f0
  type: ethernet
state: up
ipv4:
enabled: false
```

- name: ens1f0.10    # Management
  type: vlan
state: up
vlan:
base-iface: ens1f0
id: 10
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.10.0.10
  prefix-length: 24

- name: ens1f0.100   # Cluster traffic (primary)
  type: vlan
state: up
vlan:
base-iface: ens1f0
id: 100
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.100.0.10
  prefix-length: 24

- name: ens1f0.200   # Storage
  type: vlan
state: up
vlan:
base-iface: ens1f0
id: 200
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.200.0.10
  prefix-length: 24

```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 10.100.0.1
next-hop-interface: ens1f0.100
metric: 100
```

- destination: 192.168.0.0/16  # Storage subnet
  next-hop-address: 10.200.0.1
next-hop-interface: ens1f0.200
metric: 50

Use case: Enterprise bare metal with network segmentation

---
Pattern 3: Dual-Stack (IPv4 + IPv6)

```
interfaces:
- name: eno1
  type: ethernet
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.1.10
  prefix-length: 24
ipv6:
enabled: true
dhcp: false
address:
- ip: fd00:1234::10
  prefix-length: 64
autoconf: false
```

```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 192.168.1.1
next-hop-interface: eno1
```

- destination: ::/0
  next-hop-address: fd00:1234::1
next-hop-interface: eno1

```
dns-resolver:
config:
server:
- 8.8.8.8
```

- 2001:4860:4860::8888

Use case: Modern deployments requiring IPv6 support

---
Pattern 4: Bridge for OpenShift Virtualization (KubeVirt)

# Worker nodes running VMs need bridge networking
```
interfaces:
- name: eno1
  type: ethernet
state: up
ipv4:
enabled: false
```

- name: br0
  type: linux-bridge
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 192.168.1.10
  prefix-length: 24
bridge:
options:
stp:
enabled: true
port:
- name: eno1

```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 192.168.1.1
next-hop-interface: br0
```

Use case: Bare metal workers hosting virtual machines

---
UPI Post-Install NMState (Traditional UPI)

## For traditional UPI (PXE boot, manual ignition), you configure networking via

## 1. Installation-time: Kernel parameters, NetworkManager keyfiles in ignition

## 2. Post-install: Deploy kubernetes-nmstate operator and manage declaratively

Installation-Time (Ignition Config)

# Add to ignition config (master.ign or worker.ign)
```
variant: fcos
version: 1.4.0
storage:
files:
- path: /etc/NetworkManager/system-connections/bond0.nmconnection
  mode: 0600
contents:
inline: |
[connection]
id=bond0
type=bond
interface-name=bond0
autoconnect=true
```

[bond]
mode=802.3ad
miimon=100

[ipv4]
method=manual
address=192.168.1.10/24
gateway=192.168.1.1
dns=8.8.8.8;8.8.4.4

- path: /etc/NetworkManager/system-connections/bond0-slave-eno1.nmconnection
  mode: 0600
contents:
inline: |
[connection]
id=bond0-slave-eno1
type=ethernet
interface-name=eno1
master=bond0
slave-type=bond
autoconnect=true

- path: /etc/NetworkManager/system-connections/bond0-slave-eno2.nmconnection
  mode: 0600
contents:
inline: |
[connection]
id=bond0-slave-eno2
type=ethernet
interface-name=eno2
master=bond0
slave-type=bond
autoconnect=true

## Limitations
- ❌ Must bake into ignition config (immutable after first boot)
- ❌ Complex for VLAN/bridge configs
- ❌ No declarative updates (can't change via API)
- ❌ Hard to maintain consistency across nodes

---
Post-Install (kubernetes-nmstate Operator)

# After UPI cluster is running:

# 1. Deploy kubernetes-nmstate operator (same as IPI)
```bash
oc apply -f nmstate-operator.yaml
```

# 2. Create NMState instance
```bash
oc apply -f nmstate-instance.yaml
```

# 3. Define network configuration policies
```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: workers-storage-vlan
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
desiredState:
interfaces:
- name: ens1f1.200
  type: vlan
state: up
vlan:
base-iface: ens1f1
id: 200
ipv4:
enabled: true
dhcp: true  # Or static
EOF
```

# 4. NMState operator ensures all matching nodes have this config
```bash
oc get nncp
# NAME                    STATUS
# workers-storage-vlan    Available
```

```bash
oc get nnce  # NodeNetworkConfigurationEnactment (per-node status)
# NODE       STATUS
# worker-0   Available
# worker-1   Available
# worker-2   Available
```

## Advantages
- ✅ Declarative management via Kubernetes API
- ✅ Automatic rollout to matching nodes
- ✅ Drift detection and remediation
- ✅ Easy to update (edit policy, operator applies changes)
- ✅ Status reporting per node

---
### Real-World Scenarios


**Scenario 1: IPI AWS with Jumbo Frames**

# AWS supports jumbo frames (MTU 9001) for improved performance
# Default MTU is 1500

# Deploy NMState operator
```bash
oc apply -f nmstate-operator.yaml
```

# Configure jumbo frames on all worker nodes
```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: jumbo-frames
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
desiredState:
interfaces:
- name: eth0
  type: ethernet
state: up
mtu: 9001
EOF
```

# Verify
```bash
oc get nncp jumbo-frames
# STATUS: Available
```

# Check on nodes
```bash
oc debug node/worker-0 -- chroot /host ip link show eth0
# eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001
```

IPI Role: NMState used for performance tuning, not basic connectivity

---
Scenario 2: UPI Bare Metal with LACP Bonding

# Production bare metal cluster
# 2x 10GbE NICs per server, LACP bonding to switch stack

# agent-config.yaml snippet
hosts:
- hostname: worker-prod-01
  role: worker
```
interfaces:
- name: ens1f0
  macAddress: 00:1b:21:ab:cd:01
```

- name: ens1f1
  macAddress: 00:1b:21:ab:cd:02
networkConfig:
```
interfaces:
- name: bond0
  type: bond
state: up
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.0.100.101
  prefix-length: 24
link-aggregation:
mode: 802.3ad
options:
miimon: "100"
lacp_rate: fast
xmit_hash_policy: layer3+4
port:
- ens1f0
```

- ens1f1
- name: ens1f0
  type: ethernet
state: up
mtu: 9000
- name: ens1f1
  type: ethernet
state: up
mtu: 9000
```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 10.0.100.1
next-hop-interface: bond0
dns-resolver:
config:
server:
- 10.0.100.53
```

UPI Role: NMState essential for production HA networking

---
Scenario 3: UPI Edge with Constrained Networking

# Edge deployment, single NIC, multiple services on VLANs
# Management: VLAN 10
# Cluster: VLAN 100
# OT/IoT devices: VLAN 300

hosts:
- hostname: edge-node-01
  role: master
```
interfaces:
- name: enp0s1
  macAddress: 52:54:00:ed:ge:01
networkConfig:
interfaces:
- name: enp0s1
  type: ethernet
state: up
ipv4:
enabled: false
mtu: 1500
```

- name: enp0s1.10
  type: vlan
state: up
vlan:
base-iface: enp0s1
id: 10
ipv4:
enabled: true
dhcp: false
address:
- ip: 172.16.10.11
  prefix-length: 24

- name: enp0s1.100
  type: vlan
state: up
vlan:
base-iface: enp0s1
id: 100
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.0.100.11
  prefix-length: 24

- name: enp0s1.300
  type: vlan
state: up
vlan:
base-iface: enp0s1
id: 300
ipv4:
enabled: true
dhcp: false
address:
- ip: 10.0.300.11
  prefix-length: 24

```
routes:
config:
- destination: 0.0.0.0/0
  next-hop-address: 10.0.100.1
next-hop-interface: enp0s1.100
metric: 100
```

- destination: 10.0.300.0/24
  next-hop-interface: enp0s1.300
metric: 50

UPI Role: NMState enables complex networking on constrained hardware

---
Scenario 4: OpenShift Virtualization on Bare Metal UPI

# Workers hosting VMs need bridge for VM networking
# VMs on isolated bridge (br-vms) separate from cluster networking

# Post-install NodeNetworkConfigurationPolicy
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
name: kubevirt-bridge
spec:
nodeSelector:
node-role.kubernetes.io/worker: ""
feature.node.kubernetes.io/kubevirt: "true"
desiredState:
interfaces:
- name: ens1f1  # Dedicated NIC for VM traffic
  type: ethernet
state: up
ipv4:
enabled: false
```

- name: br-vms
  type: linux-bridge
state: up
ipv4:
enabled: false
bridge:
options:
stp:
enabled: false
port:
- name: ens1f1

---
# NetworkAttachmentDefinition for VMs
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
name: vm-network
namespace: openshift-cnv
spec:
config: |
{
"cniVersion": "0.3.1",
"name": "vm-network",
"type": "cnv-bridge",
"bridge": "br-vms",
"ipam": {}
}
```

---
# VM using the bridge
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
name: test-vm
spec:
running: true
template:
spec:
domain:
devices:
interfaces:
- name: default
  masquerade: {}
```

- name: vm-net
  bridge: {}
networks:
- name: default
  pod: {}
- name: vm-net
  multus:
networkName: vm-network

UPI Role: NMState critical for VM networking isolation

---
NMState Operator Architecture

How kubernetes-nmstate Works

┌─────────────────────────────────────────────────────────────┐
│  User creates NodeNetworkConfigurationPolicy                 │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│  NMState Webhook validates policy                            │
│  - Syntax check                                              │
│  - Conflict detection                                        │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│  NMState Controller (watches NNCP)                           │
│  - Selects nodes via nodeSelector                           │
│  - Creates NodeNetworkConfigurationEnactment per node        │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│  NMState Handler (DaemonSet pod on each node)               │
│  - Reads NNCE for this node                                 │
│  - Calls nmstatectl apply with desiredState                 │
│  - Monitors network changes                                 │
│  - Reports status back to NNCE                              │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│  Node network interfaces updated                             │
│  - Bond created, VLANs configured, IPs assigned, etc.       │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│  Status propagated                                           │
│  - NNCE status: Available/Degraded/Progressing              │
│  - NNCP status: aggregated from all NNCEs                   │
└─────────────────────────────────────────────────────────────┘

---
Comparison Summary

┌────────────────────────────────┬───────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────────┬────────────────────────────────┐
│             Aspect             │                    IPI                    │                                        UPI Bare Metal                                        │           UPI Cloud            │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ NMState needed for basic       │ No                                        │ Yes (often)                                                                                  │ No (usually)                   │
│ install?                       │                                           │                                                                                              │                                │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ When NMState is useful         │ Multi-NIC, SR-IOV, KubeVirt, custom       │ Bonding, VLANs, complex topology, HA                                                         │ Similar to IPI (advanced       │
│                                │ routing                                   │                                                                                              │ cases)                         │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ Installation-time config       │ Not supported (cloud-init)                │ Yes (agent-config.yaml)                                                                      │ Not typical                    │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ Post-install operator          │ Optional (for advanced cases)             │ Recommended (declarative mgmt)                                                               │ Optional                       │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ Common use cases               │ - Jumbo frames<br>- SR-IOV<br>- KubeVirt  │ - LACP bonding<br>- VLAN segmentation<br>- Dual-stack<br>- KubeVirt bridges<br>- Edge        │ Same as IPI                    │
│                                │ bridges                                   │ constrained networking                                                                       │                                │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ Typical complexity             │ Low (simple cloud networking)             │ High (enterprise bare metal)                                                                 │ Low to medium                  │
├────────────────────────────────┼───────────────────────────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────────┼────────────────────────────────┤
│ Managed by                     │ Cloud provider + NMState (optional)       │ NMState (essential)                                                                          │ Cloud provider + NMState       │
│                                │                                           │                                                                                              │ (optional)                     │
└────────────────────────────────┴───────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────────┴────────────────────────────────┘

---
Key Takeaways

IPI

- NMState is optional for most IPI deployments
- Cloud providers handle basic networking automatically
- Use NMState for:
- Multiple network interfaces (multi-NIC)
- SR-IOV configurations
- OpenShift Virtualization (KubeVirt) bridges
- Custom routing or MTU requirements
- Deploy post-installation via operator

UPI (Bare Metal)

- NMState is critical for complex network topologies
- Essential for:
- Production HA (bonding)
- Network segmentation (VLANs)
- Multiple network planes (management, cluster, storage)
- Dual-stack IPv4/IPv6
- Edge deployments with limited NICs
- OpenShift Virtualization
- Configure at installation time (agent-config.yaml) or post-install (operator)
- Provides declarative, drift-resistant network management

UPI (Cloud)

- Similar to IPI — cloud networking usually sufficient
- NMState useful for same advanced cases as IPI
- Less critical than bare metal UPI

Bottom line: NMState's importance scales with networking complexity. Simple cloud deployments rarely need it; complex bare metal deployments can't live without it.
