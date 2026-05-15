# OpenShift automatic certificate rotation

## Overview

OpenShift internally manages the lifecycle of most certificates through dedicated operators and controllers. Rotation is triggered automatically based on validity thresholds — not fixed calendar schedules. The Machine Config Server (MCS) CA is the only internal certificate that does **not** rotate automatically.

---

## Certificates rotated automatically

### 1. etcd certificates

| Certificate | Role | Validity | Rotation trigger |
|---|---|---|---|
| etcd CA (`etcd-signer`) | Root of trust for all etcd TLS — signs all peer, server, and client certificates | 10 years | Near expiry |
| etcd peer certs | Mutually authenticate etcd members to each other during Raft consensus replication | 3 years | Near expiry or CA rotation |
| etcd server certs | Presented by each etcd member to authenticate itself to incoming clients, primarily the kube-apiserver | 3 years | Near expiry or CA rotation |
| etcd client certs (`etcd-client`) | Used by the kube-apiserver to identify and authenticate itself when reading and writing cluster state to etcd | 3 years | Near expiry or CA rotation |
| etcd metric certs (`etcd-metric-client`, `etcd-metric-signer`) | Secure the etcd metrics endpoint; all metric consumers connect to the etcd proxy using these certificates | 3 years | Near expiry or CA rotation |

etcd certificates are signed by the `etcd-signer` CA generated during the bootstrap process. Client secrets (`etcd-client`, `etcd-metric-client`, `etcd-metric-signer`, `etcd-signer`) are propagated to the `openshift-config` and `openshift-kube-apiserver` namespaces.

**Prerequisites:**
- etcd cluster at quorum (all 3 members healthy)
- `etcd` cluster operator `Available=True`
- API server reachable to propagate updated secrets

---

### 2. Kubelet / node certificates

| Certificate | Role | Validity | Rotation trigger |
|---|---|---|---|
| Bootstrap kubelet CA | Allows the kubelet to submit its very first CSR to the API server at node first-boot, before it has a real certificate | 10 years | Near expiry |
| `kube-apiserver-to-kubelet-signer` CA | Signs the kubelet's serving certificate; enables the API server to trust connections back to the kubelet for `logs`, `exec`, `attach`, and `debug` operations | 365 days | At 80% lifetime (≈292 days) |
| Kubelet client cert (node) | Authenticates the kubelet to the kube-apiserver for all node-to-control-plane API calls (e.g. status updates, pod lifecycle events) | 1 year | At 80% lifetime; every 30 days post-install |
| Kubelet client cert (in-cluster) | Short-lived credential used by in-cluster components to identify the kubelet during runtime operations; rotated frequently to limit exposure window | ~12 hours | Every 12 hours |

The kubelet uses the bootstrap certificate at `/etc/kubernetes/kubeconfig` for its first boot. The kube-controller-manager signs the resulting CSR, and the kubelet then manages its own certificate going forward. The `kube-apiserver-to-kubelet-signer` CA is renewed by the KAS Operator at 292 days; the old CA is removed at 365 days.

**Prerequisites:**
- **MachineConfigPool must not be paused** — MCO pushes the updated `kubelet-ca.crt` bundle to nodes; a paused pool blocks this delivery
- `kube-apiserver` operator `Available=True`
- kube-controller-manager running to auto-approve CSRs
- All nodes `Ready` and reachable by MCO
- Current `kube-apiserver-to-kubelet-signer` CA bundle propagated to node before client cert rotates

---

### 3. Service CA and service serving certificates

| Certificate | Role | Validity | Rotation trigger |
|---|---|---|---|
| Service CA (`signing-key`) | Internal cluster CA that issues TLS certificates to any service annotated for automatic certificate management; the trust anchor for all service-to-service communication within the cluster | 26 months | When < 13 months remain |
| Service serving certs (leaf) | Individual TLS server certificates issued to annotated services, enabling encrypted HTTPS between pods and services without manual certificate management | 2 years | Near expiry or on CA rotation |
| Monitoring certs (`openshift-monitoring`) | Secure Prometheus, Alertmanager, and related metrics endpoints; allow components to authenticate each other over TLS when scraping metrics | 2 years | On service CA rotation (every 13 months) |
| Logging certs (`openshift-logging`) | Secure log forwarding and aggregation pipelines (e.g. Fluentd to Elasticsearch); prevent plaintext log transmission across the cluster network | 2 years | On service CA rotation (every 13 months) |

The `service-ca` controller automatically re-issues all leaf serving certificates when the CA rotates. After a CA rotation, pods across the cluster must be restarted to pick up the new CA bundle.

**Prerequisites:**
- `service-ca` operator running and not degraded
- Service annotated with `service.beta.openshift.io/serving-cert-secret-name` (for leaf certs)
- API server reachable to update secrets and configmaps
- Pod restarts required post-rotation for workloads to trust the new CA

---

### 4. Control plane certificates

| Certificate | Role | Validity | Rotation trigger |
|---|---|---|---|
| kube-apiserver serving cert | Presented by the API server to all clients (CLI, controllers, kubelets) to prove its identity over HTTPS; without this, no client can securely connect to the control plane | Operator managed | Near expiry |
| kube-apiserver client cert | Used by the API server to authenticate itself when acting as a client — e.g. when calling aggregated API servers or admission webhooks | Operator managed | Near expiry |
| kube-controller-manager cert | Authenticates the controller manager to the API server; required for all control loops (Deployments, ReplicaSets, node lifecycle) to function | Operator managed | Near expiry |
| kube-scheduler cert | Authenticates the scheduler to the API server; required for pod scheduling decisions to be written back as binding objects | Operator managed | Near expiry |

All control plane certs are managed by their respective operators in namespaces `openshift-kube-apiserver`, `openshift-kube-controller-manager`, `openshift-kube-scheduler`, and `openshift-etcd`.

**Prerequisites:**
- Each operator must be `Available=True, Progressing=False, Degraded=False`
- etcd healthy (API server depends on etcd for secret storage)
- Cluster operators stable before and after rotation

---

### 5. Aggregated API server certificates

| Certificate | Role | Validity | Rotation trigger |
|---|---|---|---|
| Aggregated API CA | Root of trust for the aggregation layer; enables the kube-apiserver to verify the identity of extension API servers (e.g. `metrics-server`, OpenShift-specific APIs) | 30 days | Near expiry (controller-driven) |
| Aggregated API client certs | Used by the kube-apiserver to identify itself as a trusted front proxy when forwarding requests to aggregated API servers; without this, extension APIs reject the forwarded requests | 30 days | Near expiry (controller-driven) |

Rotated automatically by controllers in the `openshift-kube-apiserver-operator` namespace. The short 30-day validity is intentional — this is a high-trust boundary and frequent rotation limits the blast radius of a compromised credential.

**Prerequisites:**
- API server aggregation layer healthy
- Controller running in `openshift-kube-apiserver-operator`

---

### 6. OLM-managed operator certificates

| Certificate | Role | Rotation |
|---|---|---|
| Webhook certs (per operator CSV) | Secure the TLS channel between the kube-apiserver and an operator's admission or mutating webhook; both sides authenticate so only trusted requests are processed | OLM-driven |
| APIService certs (per operator CSV) | Secure the TLS channel between the kube-apiserver and an operator's custom API server registered via `APIService`; required for `kubectl` and `oc` commands targeting that API group to work | OLM-driven |

When installing Operators that include webhooks or API services in their `ClusterServiceVersion`, OLM creates and rotates the certificates automatically.

**Prerequisites:**
- OLM operator running and healthy
- Operator **not** in a proxy environment — proxy environments require manual certificate rotation
- Operator CSV defines a webhook or APIService

---

### 7. Machine Config Server (MCS) CA — manual only

| Certificate | Role | Automatic rotation |
|---|---|---|
| MCS serving cert | Presented by the Machine Config Server to new nodes at port 22623; the node verifies this cert before trusting and consuming its ignition config at first boot | No |
| MCS CA | Signs the MCS serving cert; embedded in the ignition config so that bootstrapping nodes know which CA to trust when reaching out to port 22623 for their OS configuration | No |

The MCS CA secures the ignition endpoint at port 22623, used only when new nodes first join the cluster. It is not automatically rotated because it lives partially outside the cluster — its public key is baked into ignition user-data at install time, and updating that reference requires either cloud API access (IPI) or manual intervention (UPI/bare metal).

**To rotate manually:**

```bash
# For MachineSet-backed clusters (IPI / cloud) — auto-updates user-data
oc adm ocp-certificates regenerate-machine-config-server-serving-cert

# For UPI / bare metal — rotate cert only, update user-data separately
oc adm ocp-certificates regenerate-machine-config-server-serving-cert --update-ignition=false
oc adm ocp-certificates update-ignition-ca-bundle-for-machine-config-server
```

For UPI and bare metal installs, the ignition user-data stored outside the cluster (e.g. in PXE configs or cloud init) must be manually updated with the new CA bundle after rotation.

---

## Global prerequisites for all automatic rotations

These conditions must hold for **any** certificate rotation to succeed:

### Cluster operators must be healthy

Before rotation begins, run:

```bash
oc adm wait-for-stable-cluster
```

All operators must report `Available=True, Progressing=False, Degraded=False`.

### MachineConfigPools must not be paused

A paused MCP blocks MCO from delivering updated certificate bundles to nodes. This specifically breaks the `kube-apiserver-to-kubelet-signer` CA rotation — the new CA is generated but never pushed, causing `oc logs`, `oc exec`, `oc debug`, and `oc attach` to fail within 12 hours.

```bash
# Check pool pause status
oc get mcp

# Unpause a pool
oc patch mcp <pool-name> --type=merge --patch '{"spec":{"paused":false}}'
```

### Nodes must be `Ready` and reachable

Nodes that are `NotReady` or unreachable by the MCO cannot receive updated certificate bundles. Certificate rotation will stall at those nodes.

### etcd quorum must be maintained

All control plane certificate rotations write updated secrets to etcd. A degraded etcd cluster (fewer than 2 of 3 members healthy) will stall any operator trying to persist rotated material.

### Time synchronisation (NTP) must be accurate

Certificate validity windows are time-based. Clock skew between nodes can cause valid certificates to be rejected or trigger premature rotation failures. NTP must be configured and accurate across all nodes.

---

## Quick reference: rotation windows

| Certificate group | Role summary | CA validity | Leaf validity | Auto-rotation threshold |
|---|---|---|---|---|
| etcd | Secures all etcd peer, server, and client communication | 10 years | 3 years | Near expiry |
| Kubelet bootstrap CA | First-boot node identity for CSR submission | 10 years | — | Near expiry |
| `kube-apiserver-to-kubelet-signer` | Enables API server to trust kubelet for exec/logs/attach | 365 days | — | 80% (≈292 days) |
| Kubelet client cert | Node-to-API-server authentication | — | 1 year / 12 hrs | 80% / every 12 hrs |
| Service CA | Internal cluster-wide TLS CA for service-to-service traffic | 26 months | 2 years | < 13 months remaining |
| Aggregated API | Front-proxy trust between API server and extension APIs | 30 days | 30 days | Near expiry |
| OLM / webhooks | Webhook and APIService TLS for operator extensions | OLM-managed | OLM-managed | OLM-driven |
| MCS CA | Node first-boot ignition trust anchor | — | — | **Manual only** |