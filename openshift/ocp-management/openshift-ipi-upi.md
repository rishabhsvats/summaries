# IPI vs UPI: Day 1+ Operational Differences

Complete breakdown of how IPI and UPI differ in post-installation configuration and operations across all aspects.

---

## 1. Machine Management & Scaling

### IPI Approach

Machine API is fully functional out of the box.


```yaml
# MachineSets are auto-created during installation
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
name: my-cluster-worker-us-east-1a
namespace: openshift-machine-api
spec:
replicas: 3
selector:
matchLabels:
machine.openshift.io/cluster-api-cluster: my-cluster
machine.openshift.io/cluster-api-machineset: my-cluster-worker-us-east-1a
template:
metadata:
labels:
machine.openshift.io/cluster-api-cluster: my-cluster
machine.openshift.io/cluster-api-machineset: my-cluster-worker-us-east-1a
spec:
providerSpec:
value:
# Cloud-specific config AUTO-POPULATED by installer
instanceType: m5.xlarge
ami:
id: ami-xxxxx
subnet:
id: subnet-xxxxx  # Installer-created subnet
securityGroups:
- id: sg-xxxxx    # Installer-created SG
iamInstanceProfile:
id: my-cluster-worker-profile  # Installer-created
```


**Day 1+ Operations:**

```bash
# Scaling - Simple
oc scale machineset my-cluster-worker-us-east-1a --replicas=5
```

```bash
# Add new MachineSet in different AZ - Simple
cat <<EOF | oc apply -f -
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
name: my-cluster-worker-us-east-1b
namespace: openshift-machine-api
spec:
replicas: 2
template:
spec:
providerSpec:
value:
instanceType: m5.2xlarge
# References installer-created infrastructure
subnet:
id: subnet-installer-created-1b
securityGroups:
- id: sg-installer-created
EOF
```

```bash
# Machine replacement - Automatic
oc delete machine my-cluster-worker-us-east-1a-xxxxx
# MachineSet controller creates replacement automatically
```

# Change instance type - Simple
```bash
oc patch machineset my-cluster-worker-us-east-1a \
--type=merge \
-p '{"spec":{"template":{"spec":{"providerSpec":{"value":{"instanceType":"m5.2xlarge"}}}}}}'
```

# Autoscaling - native
```bash
cat <<EOF | oc apply -f -
apiVersion: autoscaling.openshift.io/v1
kind: MachineAutoscaler
metadata:
name: my-cluster-worker-us-east-1a
namespace: openshift-machine-api
spec:
minReplicas: 3
maxReplicas: 10
scaleTargetRef:
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
name: my-cluster-worker-us-east-1a
```


## What IPI Provides
- ✅ MachineSets pre-configured with all cloud details
- ✅ References to installer-created infrastructure (subnets, SGs, IAM)
- ✅ Automatic machine replacement on failure
- ✅ Native autoscaling via ClusterAutoscaler
- ✅ Machine health checks
- ✅ Infrastructure components (machines, instances) are reconciled

---
### UPI Approach

Machine API is NOT configured by default.

# Day 1 - No MachineSets exist
```bash
oc get machinesets -n openshift-machine-api
# No resources found
```

# Machine management is manual

## Day 1+ Operations


**Option 1: Continue Manual Management (Common for UPI)**

# Scaling - Manual
# 1. Provision new VM via your tooling
```bash
terraform apply -var="worker_count=5"
```

# OR manually
```bash
aws ec2 run-instances \
--image-id ami-rhcos-xxx \
--instance-type m5.xlarge \
--subnet-id subnet-your-custom-subnet \
--security-group-ids sg-your-custom-sg \
--iam-instance-profile Name=your-custom-worker-profile \
--user-data file://worker.ign
```

# 2. Wait for node to join
```bash
oc get nodes -w
```

# 3. Approve CSRs (Certificate Signing Requests)
```bash
oc get csr
oc adm certificate approve <csr-name>
# Often two CSRs per node (client + server)
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
```

# 4. Label node (if needed)
```bash
oc label node <node-name> node-role.kubernetes.io/worker=
```

# Machine replacement - Manual
# 1. Cordon and drain
```bash
oc adm cordon <node-name>
oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

# 2. Terminate VM
```bash
aws ec2 terminate-instances --instance-ids i-xxxxx
```

# 3. Provision replacement (same as scaling above)

# Change instance type - Manual
# 1. Drain node
# 2. Terminate instance
# 3. Launch new instance with different type
# 4. Approve CSRs
# 5. Uncordon

Option 2: Post-Install Machine API Setup (Advanced)

# Manually create MachineSets after installation
```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
name: custom-worker-us-east-1a
namespace: openshift-machine-api
spec:
replicas: 3
selector:
matchLabels:
machine.openshift.io/cluster-api-cluster: upi-cluster
machine.openshift.io/cluster-api-machineset: custom-worker-us-east-1a
template:
spec:
providerSpec:
value:
# YOU must know/configure all cloud details
apiVersion: machine.openshift.io/v1beta1
kind: AWSMachineProviderConfig
instanceType: m5.xlarge
ami:
id: ami-xxxxx  # YOU determine this
subnet:
id: subnet-xxxxx  # YOUR custom subnet
securityGroups:
- id: sg-xxxxx  # YOUR custom SG
iamInstanceProfile:
id: your-worker-profile  # YOUR custom IAM
userDataSecret:
name: worker-user-data  # YOU create this
tags:
- name: kubernetes.io/cluster/upi-cluster
value: owned  # YOU set cluster ID
```

## Challenges
- ❌ No auto-configured MachineSets
- ❌ Must know all infrastructure IDs (subnet, SG, IAM, AMI)
- ❌ Must create userDataSecret with correct ignition
- ❌ Must tag resources correctly for cloud integrations
- ❌ If infrastructure changes, must update MachineSets manually

## What UPI Requires
- Manual VM lifecycle management OR
- Manual MachineSet creation with custom infrastructure references
- CSR approval automation (or manual approval)
- Custom scaling automation
- Manual node replacement procedures

---

## 2. Node Replacement & Upgrades

### IPI Approach

# Worker node upgrade - Automatic
# Machine Config Operator handles everything
```bash
oc get mcp
# NAME     CONFIG                    UPDATED   UPDATING   DEGRADED
# master   rendered-master-abc123    True      False      False
# worker   rendered-worker-def456    False     True       False
```

# Process is automatic:
# 1. MCO creates new machine config
# 2. Machines are drained one by one
# 3. Machine reboots with new config
# 4. Node rejoins cluster
# 5. Repeat for next node

# Control plane upgrade - Automatic
# Cluster Version Operator manages master nodes
# 1. Etcd operator drains/upgrades masters one by one
# 2. Static pods are updated
# 3. Machines reboot if needed

# Machine replacement (worker dies) - Automatic
# Machine health check detects failure
```bash
oc get machinehealthcheck -n openshift-machine-api
```

# MachineHealthCheck automatically:
# 1. Detects unhealthy node
# 2. Deletes Machine object
# 3. MachineSet controller creates replacement
# 4. New VM is provisioned automatically

---

### UPI Approach

# Worker node upgrade - Semi-automatic
# MCO creates machine configs, but node lifecycle is different

# If using Machine API (post-configured):
# - Similar to IPI
# - But YOU provisioned the infrastructure

# If using manual management:
# 1. MCO marks node as needing update
```bash
oc get nodes
# Shows nodes with outdated config
```

# 2. Cordon and drain manually
```bash
oc adm cordon worker-0
oc adm drain worker-0 --ignore-daemonsets --delete-emptydir-data
```

# 3. Node reboots automatically (MCO triggers reboot)
# OR manually: ssh core@worker-0 sudo systemctl reboot

# 4. Wait for node to come back
```bash
oc get nodes -w
```

# 5. Uncordon
```bash
oc adm uncordon worker-0
```

# 6. Repeat for next node

# Control plane upgrade - Manual COORDINATION
# Etcd operator still manages etcd upgrade, but:
# - If static pod update requires reboot, YOU manage timing
# - YOU ensure one master at a time
# - YOU monitor etcd quorum

# Recommended process:
```
for master in master-0 master-1 master-2; do
echo "Upgrading $master"
oc adm cordon $master
oc adm drain $master --ignore-daemonsets --delete-emptydir-data
# Wait for upgrade to apply
sleep 300
# Node reboots automatically or:
ssh core@$master sudo systemctl reboot
# Wait for node to return
oc wait --for=condition=Ready node/$master --timeout=600s
oc adm uncordon $master
done
```

# Machine replacement (worker dies) - Fully manual
# 1. Detect failure (monitoring, alerts)
# 2. Remove node from cluster
```bash
oc delete node worker-bad
```

# 3. Terminate VM
```bash
aws ec2 terminate-instances --instance-ids i-bad
```

# 4. Provision replacement
```bash
terraform apply
# OR
aws ec2 run-instances --user-data file://worker.ign ...
```

# 5. Approve CSRs
```bash
oc get csr | grep Pending | awk '{print $1}' | xargs oc adm certificate approve
```

# 6. Verify node joined
```bash
oc get nodes
```

---
3. Cluster Version Upgrades

### IPI Approach

# Upgrade process - Highly automated

# 1. Check available versions
```bash
oc adm upgrade
```

# 2. Trigger upgrade
```bash
oc adm upgrade --to=4.17.0
```

# Cluster Version Operator (CVO) handles:
# ✅ Control plane component updates (kube-apiserver, etcd, etc.)
# ✅ Control plane node updates (masters)
# ✅ Infrastructure operator updates (cloud integrations)
# ✅ Machine Config Operator updates worker configs
# ✅ Automatic rollout coordination
# ✅ Health checks between stages

# 3. Monitor upgrade
```bash
oc get clusterversion
oc get clusteroperators
oc adm upgrade
```

# Infrastructure impact:
# - Machine API updates worker VMs automatically
# - Cloud integrations update automatically
# - Load balancers, DNS, networking untouched (managed by installer)

---

### UPI Approach

# Upgrade process - mostly automated, but manual coordination

# 1. Check available versions (same as IPI)
```bash
oc adm upgrade
```

# 2. Trigger upgrade (same as IPI)
```bash
oc adm upgrade --to=4.17.0
```

# CVO handles control plane components (same as IPI)
# MCO handles machine configs (same as IPI)

# Manual considerations:

# Infrastructure compatibility checks (YOU must verify):
# - Load balancer configuration changes?
#   - Check release notes for new ports/health checks
#   - Update LB listeners/target groups manually if needed
# - DNS changes required?
#   - New DNS records needed? (rare, but possible)
# - Security group changes?
#   - New ports for new features? (e.g., new monitoring ports)
# - Storage driver updates?
#   - CSI driver changes may need manual intervention

# Example: 4.16 → 4.17 upgrade requires new LB health check
# IPI: Installer updates load balancer automatically
# UPI: YOU must update:

```bash
aws elbv2 modify-target-group \

--target-group-arn arn:aws:elasticloadbalancing:...:targetgroup/api/... \
--health-check-path /readyz \
--health-check-port 6443

# Worker node rollout:
# - If using Machine API: Automatic (like IPI)
# - If manual management: YOU control rollout timing
#   - Drain, wait for MCO update, reboot, uncordon
#   - YOU decide how many workers at a time

# Control plane rollout:
# - CVO handles component updates
# - But if you have custom load balancer setup:
#   - Monitor LB health checks during master reboots
#   - May need to adjust LB drain timeouts

# Post-upgrade verification (same workload, more infrastructure checks):
```bash
oc get clusteroperators  # All should be available
oc get nodes             # All should be ready
oc get mcp               # Should be updated, not degraded
```

# UPI-specific checks:
# - Verify load balancer health checks passing
# - Verify custom DNS still resolves
# - Check custom monitoring for infrastructure health
# - Review any manual infrastructure changes needed for next upgrade

## Key Differences

```
  ┌───────────────────────┬───────────────────────────────────┬─────────────────────────────────────────────────────────────┐
  │        Aspect         │                IPI                │                             UPI                             │
  ├───────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Control plane update  │ Automatic                         │ Automatic                                                   │
  ├───────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Worker update         │ Automatic (Machine API)           │ Automatic (if Machine API) or Manual                        │
  ├───────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Infrastructure update │ Automatic (LB, DNS, etc.)         │ Manual (YOU update LB, DNS, etc.)                           │
  ├───────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Pre-upgrade checks    │ CVO checks cluster health         │ CVO checks cluster + YOU check infrastructure compatibility │
  ├───────────────────────┼───────────────────────────────────┼─────────────────────────────────────────────────────────────┤
  │ Rollback              │ Platform handles machine rollback │ YOU may need to rollback infrastructure changes             │
  └───────────────────────┴───────────────────────────────────┴─────────────────────────────────────────────────────────────┘
```

---

## 4. Load Balancer Management

### IPI Approach

# Load balancers are fully managed by cloud provider integrations

# Day 1: Installer creates:
# - API load balancer (external)
# - API load balancer (internal)
# - Ingress load balancer (for Router)

# Day 2+: Automatic management

# Service type LoadBalancer - Automatic
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
name: my-app
spec:
type: LoadBalancer
selector:
app: my-app
ports:
- port: 80
targetPort: 8080
EOF
```

# What happens automatically:
# 1. Cloud provider controller detects Service
# 2. Creates cloud load balancer (ELB/ALB/NLB)
# 3. Configures target groups/backend pools
# 4. Adds worker nodes as targets
# 5. Updates Service with load balancer hostname/IP
# 6. Automatic health checks configured

```bash
oc get svc my-app
# NAME     TYPE           CLUSTER-IP      EXTERNAL-IP
# my-app   LoadBalancer   172.30.x.x      a1b2c3...elb.amazonaws.com
```

# Ingress (Router) changes - Automatic
# Default Ingress Controller uses installer-created LB
```bash
oc get ingresscontroller -n openshift-ingress-operator
```

# Add new IngressController with separate LB:
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.my-cluster.example.com
endpointPublishingStrategy:
type: LoadBalancerService
loadBalancer:
scope: Internal  # Cloud provider creates internal LB
EOF
```

# Cloud provider automatically:
# 1. Creates internal load balancer
# 2. Configures backend to router pods
# 3. Updates DNS (if using cloud DNS integration)

# Load balancer updates - Automatic
# - Node added/removed → LB targets updated automatically
# - Health check changes → Operator updates LB config
# - Router replicas scaled → LB targets updated

# Example: Scale routers
```
```bash
oc patch ingresscontroller default -n openshift-ingress-operator \
--type=merge \
-p '{"spec":{"replicas":5}}'
# Cloud provider LB automatically adds new router pods as targets
```

## What IPI Manages
- ✅ Load balancer creation
- ✅ Target/backend pool management
- ✅ Health check configuration
- ✅ Node membership updates
- ✅ Load balancer deletion on cluster destroy
- ✅ Security group updates for LB access

---
### UPI Approach

# Load balancers are your responsibility

# Day 1: YOU created:
# - API LB (external): api.cluster.example.com
#   - Target: master-0:6443, master-1:6443, master-2:6443
#   - Health: GET /readyz on 6443
# - API LB (internal): api-int.cluster.example.com
#   - Target: master-0:6443, master-0:22623, master-1:6443, master-1:22623, ...
#   - Health: GET /readyz on 6443, GET /healthz on 22623
# - Ingress LB: *.apps.cluster.example.com
#   - Target: worker-0:80, worker-0:443, worker-1:80, worker-1:443, ...
#   - Health: GET /healthz/ready on 1936

# Day 2+: Manual management

# Service type LoadBalancer - Depends on configuration

# Option 1: Cloud provider integration NOT configured (common UPI)
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
name: my-app
spec:
type: LoadBalancer  # This will PENDING forever
selector:
app: my-app
ports:
- port: 80
EOF
```

```bash
oc get svc my-app
# NAME     TYPE           CLUSTER-IP      EXTERNAL-IP
# my-app   LoadBalancer   172.30.x.x      <pending>   ← NO CLOUD INTEGRATION
```

# YOU must manually:
# 1. Create load balancer
```bash
aws elbv2 create-load-balancer \
--name my-app-lb \
--subnets subnet-xxx subnet-yyy \
--security-groups sg-xxx
```

# 2. Create target group
```bash
aws elbv2 create-target-group \
--name my-app-targets \
--protocol TCP \
--port 8080 \
--vpc-id vpc-xxx
```

# 3. Register worker nodes
```bash
aws elbv2 register-targets \
--target-group-arn arn:... \
--targets Id=i-worker-0 Id=i-worker-1 Id=i-worker-2
```

# 4. Create listener
```bash
aws elbv2 create-listener \
--load-balancer-arn arn:... \
--protocol TCP \
--port 80 \
--default-actions Type=forward,TargetGroupArn=arn:...
```

# 5. Update DNS
```bash
aws route53 change-resource-record-sets \
--hosted-zone-id Z123 \
--change-batch '{
"Changes": [{
"Action": "CREATE",
"ResourceRecordSet": {
"Name": "my-app.example.com",
"Type": "A",
"AliasTarget": {
"HostedZoneId": "...",
"DNSName": "my-app-lb-xxx.elb.amazonaws.com",
"EvaluateTargetHealth": true
}
}
}]
}'
```

# 6. Manually update Service (can't be type LoadBalancer without cloud integration)
# Change to NodePort or ExternalName instead

# Option 2: Post-install cloud provider configuration (advanced)
# Configure cloud provider credentials and infrastructure info
# Then LoadBalancer services work like IPI (but you configured it)

# Ingress (Router) changes - Manual COORDINATION

# Add new IngressController:
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.my-cluster.example.com
endpointPublishingStrategy:
type: HostNetwork  # Can't use LoadBalancerService without cloud integration
hostNetwork: {}
EOF
```

# If you want separate LB, YOU create it:
# 1. Create new load balancer
# 2. Point to worker nodes on router NodePort
# 3. Update DNS for internal.apps.my-cluster.example.com

# Node added/removed - Manual LB UPDATE
# New worker node added:
# 1. Provision worker (terraform/manual)
# 2. Approve CSRs
# 3. Update load balancer targets

```bash
aws elbv2 register-targets \
--target-group-arn arn:...:targetgroup/ingress/... \
--targets Id=i-new-worker
```

# Node removed:
```bash
aws elbv2 deregister-targets \
--target-group-arn arn:... \
--targets Id=i-old-worker
```

# Load balancer health check updates - Manual
# OpenShift changes health check endpoint? (e.g., upgrade)
# YOU must update LB config:

```bash
aws elbv2 modify-target-group \
--target-group-arn arn:... \
--health-check-path /readyz  # New path
--health-check-interval-seconds 10
```

# Control plane node replacement - Manual LB UPDATE
# Master node replaced:
# 1. Remove old master from API LB target groups
```bash
aws elbv2 deregister-targets \
--target-group-arn arn:...:targetgroup/api/... \
--targets Id=i-old-master
```

```bash
aws elbv2 deregister-targets \
--target-group-arn arn:...:targetgroup/api-internal/... \
--targets Id=i-old-master
```

# 2. Add new master to API LB target groups
```bash
aws elbv2 register-targets \
--target-group-arn arn:...:targetgroup/api/... \
--targets Id=i-new-master
```

```bash
aws elbv2 register-targets \
--target-group-arn arn:...:targetgroup/api-internal/... \
--targets Id=i-new-master
```

## What UPI Requires
- ❌ Manual load balancer creation for new services
- ❌ Manual target registration when nodes change
- ❌ Manual health check configuration updates
- ❌ Manual DNS updates for load balancer endpoints
- ❌ Manual coordination between cluster changes and LB config
- ❌ OR: Post-install cloud provider configuration (complex)

---

## 5. DNS Management

### IPI Approach

# DNS is fully automated

# Day 1: Installer creates:
# - Hosted zone (or uses existing)
# - api.cluster.example.com → API LB
# - api-int.cluster.example.com → Internal API LB
# - *.apps.cluster.example.com → Ingress LB

# Day 2+: Automatic DNS updates

# Route (Ingress) creation - Automatic DNS
```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
name: my-app
spec:
host: my-app.apps.my-cluster.example.com  # DNS already exists
to:
kind: Service
name: my-app
EOF
```

# DNS my-app.apps.my-cluster.example.com already resolves:
# *.apps.my-cluster.example.com → Ingress LB (created by installer)
# Router handles host-based routing (no DNS change needed)

# New IngressController domain - Automatic
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.my-cluster.example.com
endpointPublishingStrategy:
type: LoadBalancerService
EOF
```

# What happens automatically:
# 1. Cloud provider creates internal load balancer
# 2. Ingress Operator updates DNS (if cloud DNS integration active)
# 3. *.internal.apps.my-cluster.example.com → Internal LB

# API endpoint changes - Automatic
# API LB IP changes? (rare, but possible during maintenance)
# Cloud provider integration updates DNS record automatically

# Custom domain - requires DNS delegation (one-time setup)
# Want custom domain for apps?
# 1. Configure IngressController with custom domain
# 2. Delegate DNS to cluster's hosted zone (one-time manual step)

# In your DNS provider:
# *.apps.my-company.com  CNAME  router-default.apps.my-cluster.example.com
# OR
# *.apps.my-company.com  A  <ingress-lb-ip>

# Then Routes work automatically:
```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
name: my-app
spec:
host: my-app.apps.my-company.com  # Custom domain
to:
kind: Service
name: my-app
EOF
```

## What IPI Provides
- ✅ Automatic DNS zone management
- ✅ Automatic record creation for cluster endpoints
- ✅ Automatic updates when infrastructure changes
- ✅ Wildcard DNS for app routes
- ✅ DNS integration with cloud provider

---
### UPI Approach

# DNS is your responsibility

# Day 1: YOU created:
# - api.cluster.example.com → API LB IP/hostname
# - api-int.cluster.example.com → Internal API LB IP/hostname
# - *.apps.cluster.example.com → Ingress LB IP/hostname

# Day 2+: Manual DNS management

# Route creation - DNS already exists (wildcard)
```bash
cat <<EOF | oc apply -f -
apiVersion: route.openshift.io/v1
kind: Route
metadata:
name: my-app
spec:
host: my-app.apps.cluster.example.com
to:
kind: Service
name: my-app
EOF
```

# DNS resolves via existing wildcard:
# my-app.apps.cluster.example.com
#   → *.apps.cluster.example.com (wildcard you created)
#   → Ingress LB IP

# New IngressController domain - Manual DNS
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.cluster.example.com
endpointPublishingStrategy:
type: HostNetwork  # Or NodePort, since no cloud LB integration
EOF
```

# YOU must create DNS:
# 1. Get router pod IPs or worker node IPs
```bash
oc get pods -n openshift-ingress -o wide
# internal-router-xxx  worker-0  10.0.1.50
# internal-router-yyy  worker-1  10.0.1.51
```

# 2. Create DNS records manually
# Option A: Wildcard to worker IPs (if using HostNetwork)
# *.internal.apps.cluster.example.com  A  10.0.1.50
# *.internal.apps.cluster.example.com  A  10.0.1.51
# (Multi-value DNS or round-robin)

# Option B: Create separate load balancer (you manage it)
# *.internal.apps.cluster.example.com  A  <your-internal-lb-ip>

# AWS Route53 example:
```bash
aws route53 change-resource-record-sets \
--hosted-zone-id Z123 \
--change-batch '{
"Changes": [{
"Action": "CREATE",
"ResourceRecordSet": {
"Name": "*.internal.apps.cluster.example.com",
"Type": "A",
"TTL": 300,
"ResourceRecords": [
{"Value": "10.0.1.50"},
{"Value": "10.0.1.51"}
]
}
}]
}'
```

# API endpoint changes - Manual DNS UPDATE
# API load balancer IP changes?
# YOU must update DNS:

```bash
aws route53 change-resource-record-sets \
--hosted-zone-id Z123 \
--change-batch '{
"Changes": [{
"Action": "UPSERT",
"ResourceRecordSet": {
"Name": "api.cluster.example.com",
"Type": "A",
"TTL": 300,
"ResourceRecords": [{"Value": "<new-api-lb-ip>"}]
}
}]
}'
```

# Control plane node replacement - DNS unchanged
# Load balancer handles routing, DNS points to LB (no change needed)
# Unless you're using direct DNS to master IPs (bad practice)

# Worker node replacement - DNS coordination
# If using HostNetwork IngressController with DNS to worker IPs:
# 1. New worker added → Update DNS to include new IP
# 2. Old worker removed → Update DNS to remove old IP

# Custom domain - Manual DELEGATION
# Same as IPI, but you manage everything:
# *.apps.my-company.com  CNAME  <your-ingress-lb-hostname>
# OR
# *.apps.my-company.com  A  <your-ingress-lb-ip>

# DNS provider outage considerations:
# IPI: If cloud DNS fails, cluster DNS fails
# UPI: If YOUR DNS fails, cluster DNS fails
#      But you may have more DNS redundancy options

## What UPI Requires
- ❌ Manual DNS record creation
- ❌ Manual DNS updates when infrastructure changes
- ❌ Manual wildcard DNS setup for new domains
- ❌ Manual DNS for new load balancers
- ❌ Coordination between infrastructure and DNS changes

---

## 6. Certificate Management

### IPI Approach

# Certificates are automatically managed

# Day 1: Installer creates:
# - Root CA
# - API server certificates (with correct SANs for LB hostnames)
# - All internal certificates
# - Kubeconfig with embedded certs

# Day 2+: Automatic rotation and management

# API server certificate renewal - Automatic
# cluster-kube-apiserver-operator handles renewal
# Certificates rotate before expiration (typically 30 days before)

```bash
oc get secrets -n openshift-kube-apiserver | grep serving
# kube-apiserver-serving-cert-xxx  (rotates automatically)
```

# Check certificate expiration:
```bash
oc get secret kube-apiserver-serving-cert-xxx -n openshift-kube-apiserver -o json | \
jq -r '.data."tls.crt"' | base64 -d | openssl x509 -noout -enddate
# notAfter=Feb 10 12:00:00 2027 GMT
```

# Custom API certificates (named certificates) - Simple
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
name: my-api-cert
namespace: clusters  # HostedCluster namespace (for HyperShift)
# OR openshift-config (for standalone cluster)
type: kubernetes.io/tls
data:
tls.crt: <base64-cert>
tls.key: <base64-key>
```

---
```yaml
apiVersion: config.openshift.io/v1
kind: APIServer
metadata:
name: cluster
spec:
servingCerts:
namedCertificates:
- names:
- api.custom.example.com
servingCertificate:
name: my-api-cert
EOF
```

# Operator automatically:
# 1. Mounts certificate into kube-apiserver pods
# 2. Configures SNI routing for api.custom.example.com
# 3. Handles certificate rotation when you update the secret

# Ingress (Router) certificates - Simple
# Default: Self-signed cert (*.apps.cluster.example.com)

# Custom certificate:
```bash
oc create secret tls custom-ingress-cert \
--cert=apps-cert.pem \
--key=apps-key.pem \
-n openshift-ingress
```

```bash
oc patch ingresscontroller default -n openshift-ingress-operator \
--type=merge \
-p '{
"spec": {
"defaultCertificate": {
"name": "custom-ingress-cert"
}
}
}'
```

# Router automatically:
# 1. Picks up new certificate
# 2. Serves it for all *.apps routes
# 3. Reloads when certificate is updated

# Certificate rotation - Automatic
# Update secret with new certificate:
```bash
oc create secret tls custom-ingress-cert \
--cert=apps-cert-new.pem \
--key=apps-key-new.pem \
-n openshift-ingress \
--dry-run=client -o yaml | oc replace -f -
```

# Router detects change and reloads (no manual intervention)

# Internal certificates - fully automatic
# service-ca-operator manages all internal service certificates
# Automatic rotation, no user action needed

```bash
oc get secrets -n openshift-kube-apiserver | grep internal
# All rotate automatically before expiration
```

## What IPI Provides
- ✅ Automatic certificate generation with correct SANs
- ✅ Automatic certificate rotation
- ✅ Automatic API server certificate management
- ✅ Simple named certificate configuration
- ✅ Automatic ingress certificate handling
- ✅ Operator-driven certificate lifecycle

---
### UPI Approach

# Certificates are automatically managed (same as IPI)
# But initial generation must account for YOUR infrastructure

# Day 1: Installer creates certificates BUT:
# - SANs must match YOUR load balancer hostnames
# - You specified these in install-config.yaml

# install-config.yaml (UPI):
```yaml
apiVersion: v1
metadata:
name: my-cluster
platform:
none: {}  # UPI platform
networking:
machineNetwork:
- cidr: 10.0.0.0/16
controlPlane:
name: master
replicas: 3
compute:
- name: worker
replicas: 0  # UPI doesn't provision workers during install
# API URL (YOUR load balancer hostname):
apiServerURL: https://api.my-cluster.example.com:6443
```

# Installer generates certificates with SANs:
# - api.my-cluster.example.com (YOUR LB hostname)
# - api-int.my-cluster.example.com (YOUR internal LB hostname)
# - kubernetes, kubernetes.default, etc. (standard)

# Day 2+: Same automatic rotation as IPI

# Certificate rotation - Automatic (same as IPI)
# cluster-kube-apiserver-operator handles it
# No difference from IPI once cluster is running

# Custom API certificates - same as IPI
```bash
oc create secret tls my-api-cert \
--cert=cert.pem \
--key=key.pem \
-n openshift-config
```

```bash
oc patch apiserver cluster --type=merge -p '{
"spec": {
"servingCerts": {
"namedCertificates": [{
"names": ["api.custom.example.com"],
"servingCertificate": {"name": "my-api-cert"}
}]
}
}
}'
```

# Ingress certificates - same as IPI
```bash
oc create secret tls custom-ingress-cert \
--cert=apps-cert.pem \
--key=apps-key.pem \
-n openshift-ingress
```

```bash
oc patch ingresscontroller default -n openshift-ingress-operator \
--type=merge -p '{
"spec": {
"defaultCertificate": {
"name": "custom-ingress-cert"
}
}
}'
```

# Key difference: Load balancer coordination

# IPI: Load balancer automatically trusts cluster certificates
# UPI: YOU may need to configure load balancer SSL termination

# Example: AWS ALB with SSL termination (UPI)
# If terminating SSL at load balancer:

# 1. Upload certificate to AWS
```bash
aws acm import-certificate \
--certificate fileb://api-cert.pem \
--private-key fileb://api-key.pem \
--certificate-chain fileb://ca-chain.pem
```

# 2. Configure ALB listener with certificate
```bash
aws elbv2 modify-listener \
--listener-arn arn:... \
--certificates CertificateArn=arn:aws:acm:...
```

# 3. Ensure backend (kube-apiserver) trusts LB connection
# (Usually pass-through is simpler for API server)

# Certificate renewal with LB termination - Manual COORDINATION
# 1. Renew certificate
# 2. Update Kubernetes secret (automatic pickup)
# 3. Update load balancer certificate
```bash
aws acm import-certificate \
--certificate-arn arn:... \
--certificate fileb://api-cert-new.pem \
--private-key fileb://api-key-new.pem
```

## UPI Specific Considerations

```
  ┌─────────────────────────────┬───────────────────────────────────────────────┬───────────────────────────────────────────────┐
  │           Aspect            │                      IPI                      │                      UPI                      │
  ├─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Initial cert generation     │ Automatic with installer-managed LB hostnames │ Automatic with YOUR LB hostnames              │
  ├─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Certificate rotation        │ Automatic                                     │ Automatic                                     │
  ├─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ LB SSL termination          │ N/A (pass-through LB)                         │ If doing SSL termination, YOU update LB certs │
  ├─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Custom certificates         │ Same process                                  │ Same process                                  │
  ├─────────────────────────────┼───────────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Certificate-LB coordination │ Automatic                                     │ Manual if using LB SSL termination            │
  └─────────────────────────────┴───────────────────────────────────────────────┴───────────────────────────────────────────────┘
```

---

## 7. Storage Management

### IPI Approach

# Dynamic storage provisioning - Fully functional

# Day 1: Installer configures:
# - Cloud-specific CSI driver (AWS EBS, Azure Disk, GCP PD)
# - Default StorageClass
# - Cloud provider credentials for volume provisioning

```bash
oc get storageclass
# NAME            PROVISIONER             RECLAIMPOLICY
# gp3-csi (default)   ebs.csi.aws.com   Delete
```

# Day 2+: Automatic volume operations

# Create PVC - Automatic PROVISIONING
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: my-data
spec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 50Gi
storageClassName: gp3-csi
EOF
```

# What happens automatically:
# 1. CSI driver detects new PVC
# 2. Calls cloud API to create volume (EBS volume)
# 3. Tags volume with cluster information
# 4. Creates PV pointing to cloud volume
# 5. Binds PVC to PV

```bash
oc get pvc my-data
# NAME      STATUS   VOLUME                   CAPACITY   STORAGECLASS
# my-data   Bound    pvc-xxx-yyy-zzz         50Gi       gp3-csi
```

```bash
aws ec2 describe-volumes --filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned"
# Volume automatically created, tagged, and managed
```

# Volume expansion - Automatic
```bash
oc patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

# CSI driver automatically:
# 1. Detects size change
# 2. Calls cloud API to expand volume
# 3. If filesystem resize needed, triggers it (on pod restart)

# Volume deletion - Automatic
```bash
oc delete pvc my-data
```

# CSI driver automatically:
# 1. Deletes PV
# 2. Calls cloud API to delete volume (if ReclaimPolicy=Delete)
# 3. Removes cloud tags

# Snapshot - Automatic
```bash
cat <<EOF | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
name: my-data-snapshot
spec:
volumeSnapshotClassName: csi-aws-vsc
source:
persistentVolumeClaimName: my-data
EOF
```

# CSI driver automatically:
# 1. Calls cloud API to create snapshot (EBS snapshot)
# 2. Creates VolumeSnapshotContent
# 3. Marks VolumeSnapshot as ready

# Cross-AZ volume attachment - Automatic
# Pod scheduled in different AZ than volume?
# CSI driver detects and handles (may recreate pod in correct AZ or fail gracefully)

# Storage class customization - Simple
```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
type: io2  # AWS-specific parameter
iops: "10000"
encrypted: "true"
kmsKeyId: arn:aws:kms:...
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

# Works immediately (cloud provider integration active)

## What IPI Provides
- ✅ Pre-configured CSI drivers
- ✅ Cloud provider credentials configured
- ✅ Default StorageClass
- ✅ Automatic volume lifecycle (create, expand, delete)
- ✅ Automatic snapshot support
- ✅ Cloud tagging for volume ownership
- ✅ Volume topology awareness (AZ placement)

---
### UPI Approach

# Dynamic storage provisioning - Depends on configuration

# Day 1: Installer does NOT configure cloud provider integration
# (Because infrastructure is pre-provisioned, no cloud credentials given to installer)

```bash
oc get storageclass
# No resources found  ← NO DEFAULT STORAGE CLASS
```

# Day 2: YOU must configure storage

# Option 1: Configure cloud provider storage (post-install)

# Step 1: Create cloud credentials secret
# (Same credentials used for Machine API, if configured)
```bash
cat <<EOF | oc create -f -
apiVersion: v1
kind: Secret
metadata:
name: aws-creds
namespace: kube-system
type: Opaque
stringData:
aws_access_key_id: AKIA...
aws_secret_access_key: ...
EOF
```

# Step 2: Enable CSI driver operator
# (Usually auto-enabled, but verify)
```bash
oc get deployment aws-ebs-csi-driver-operator -n openshift-cluster-csi-drivers
```

# Step 3: Create cloud config
# Configure cluster to know its cloud infrastructure IDs
```bash
oc patch infrastructure cluster --type=merge -p '{
"spec": {
"platformSpec": {
"type": "AWS",
"aws": {}
}
}
}'
```

# Step 4: CSI driver needs cloud tags on infrastructure
# YOU must ensure your UPI infrastructure has proper tags:
# kubernetes.io/cluster/<cluster-id>: owned

# Tag your VPC, subnets, security groups:
```bash
aws ec2 create-tags \
--resources vpc-xxx subnet-xxx subnet-yyy sg-xxx \
--tags Key=kubernetes.io/cluster/my-upi-cluster,Value=owned
```

# Step 5: Create StorageClass
```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
name: gp3-csi
annotations:
storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
type: gp3
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

# NOW dynamic provisioning works (like IPI)
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: my-data
spec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 50Gi
storageClassName: gp3-csi
EOF
```

# Option 2: Use external provisioner (if no cloud integration)

```bash
helm install nfs-provisioner stable/nfs-client-provisioner \
--set nfs.server=nfs-server.example.com \
--set nfs.path=/exports/kubernetes
```

# 2. StorageClass created automatically by provisioner
```bash
oc get storageclass
# NAME        PROVISIONER
# nfs-client  cluster.local/nfs-provisioner
```

# Option 3: Manual static provisioning (no dynamic provisioning)

# Create volume manually (cloud or on-prem):
```bash
aws ec2 create-volume \
--availability-zone us-east-1a \
--size 50 \
--volume-type gp3 \
--tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=my-data}]'
```

# Create PV manually:
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
name: my-data-pv
spec:
capacity:
storage: 50Gi
accessModes:
- ReadWriteOnce
persistentVolumeReclaimPolicy: Retain
awsElasticBlockStore:
volumeID: vol-xxxxx  # Volume ID from above
fsType: ext4
EOF
```

# Create PVC:
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: my-data
spec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 50Gi
volumeName: my-data-pv  # Bind to specific PV
EOF
```

# Volume expansion - Manual (without CSI)
# 1. Expand cloud volume
```bash
aws ec2 modify-volume --volume-id vol-xxx --size 100
```

# 2. Wait for expansion complete
```bash
aws ec2 describe-volumes-modifications --volume-ids vol-xxx
```

# 3. Exec into pod and resize filesystem
```bash
oc exec -it <pod-using-volume> -- resize2fs /dev/xvdf
```

# 4. Update PV capacity
```bash
oc patch pv my-data-pv -p '{"spec":{"capacity":{"storage":"100Gi"}}}'
```

# 5. Update PVC capacity
```bash
oc patch pvc my-data -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'
```

# Volume deletion - Manual COORDINATION
```bash
oc delete pvc my-data
# If ReclaimPolicy=Retain (common for manual PVs)
oc delete pv my-data-pv
# Cloud volume still exists, delete manually:
aws ec2 delete-volume --volume-id vol-xxx
```

## What UPI Requires

```
  ┌────────────────────────┬─────────────────────────┬───────────────────────────┬───────────────────────────┐
  │         Aspect         │           IPI           │ UPI (without post-config) │  UPI (with post-config)   │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Default StorageClass   │ Automatic               │ None                      │ Manual creation           │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Cloud CSI driver       │ Pre-configured          │ Not configured            │ Manual configuration      │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Cloud credentials      │ Configured by installer │ Not configured            │ Manual secret creation    │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Dynamic provisioning   │ Works immediately       │ Not available             │ Works after configuration │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Volume expansion       │ Automatic               │ Manual                    │ Automatic (with CSI)      │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Snapshots              │ Automatic               │ Manual or not available   │ Automatic (with CSI)      │
  ├────────────────────────┼─────────────────────────┼───────────────────────────┼───────────────────────────┤
  │ Infrastructure tagging │ Automatic               │ Manual                    │ Manual                    │
  └────────────────────────┴─────────────────────────┴───────────────────────────┴───────────────────────────┘
```

---

## 8. Monitoring & Observability

### IPI Approach

# Cluster monitoring - Fully configured

# Day 1: Installer enables:
# - Prometheus Operator
# - Cluster Monitoring Operator
# - Prometheus (control plane + user workload)
# - Alertmanager
# - Grafana
# - Node exporter, kube-state-metrics

```bash
oc get pods -n openshift-monitoring
# prometheus-k8s-0
# prometheus-k8s-1
# alertmanager-main-0
# grafana-xxx
# node-exporter-xxx (DaemonSet on all nodes)
# ...
```

# Day 2+: Automatic monitoring

# Infrastructure metrics - Automatic
# Cloud provider metrics automatically collected
# - EC2 instance metrics (if using AWS CloudWatch integration)
# - Node metrics (CPU, memory, disk, network)
# - Control plane component metrics

# Access Prometheus:
```bash
oc get route prometheus-k8s -n openshift-monitoring
# https://prometheus-k8s-openshift-monitoring.apps.cluster.example.com
```

# Persistent storage for metrics - Simple
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
name: cluster-monitoring-config
namespace: openshift-monitoring
data:
config.yaml: |
prometheusK8s:
volumeClaimTemplate:
spec:
storageClassName: gp3-csi  # Cloud storage class
resources:
requests:
storage: 100Gi
EOF
```

# Monitoring stack automatically:
# 1. Creates PVC using cloud storage
# 2. Provisions EBS/Azure Disk/GCP PD volume
# 3. Mounts to Prometheus pods
# 4. Retains metrics according to retention policy

# Custom ServiceMonitors - works automatically
```bash
cat <<EOF | oc apply -f -
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
name: my-app-metrics
spec:
selector:
matchLabels:
app: my-app
endpoints:
- port: metrics
interval: 30s
EOF
```

# Prometheus automatically discovers and scrapes targets

# Alerting - configured
# Configure Alertmanager for notifications:
```bash
oc create secret generic alertmanager-main \
--from-file=alertmanager.yaml=alertmanager-config.yaml \
-n openshift-monitoring
```

# Alertmanager automatically:
# 1. Picks up new config
# 2. Reloads
# 3. Sends alerts via configured channels (email, Slack, PagerDuty)

# Logging (optional) - Simple TO ENABLE
# Install cluster logging operator:
```bash
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
name: cluster-logging
namespace: openshift-logging
spec:
channel: stable
name: cluster-logging
source: redhat-operators
sourceNamespace: openshift-marketplace
EOF
```

# Configure logging with cloud storage backend:
```bash
cat <<EOF | oc apply -f -
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
name: instance
namespace: openshift-logging
spec:
collection:
type: vector
logStore:
type: lokistack
lokistack:
name: logging-loki
```

---
```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
name: logging-loki
namespace: openshift-logging
spec:
size: 1x.small
storage:
schemas:
- version: v12
effectiveDate: "2022-06-01"
secret:
name: logging-loki-s3  # S3 bucket for log storage
type: s3
storageClassName: gp3-csi  # Cloud storage for WAL
EOF
```

# Cloud object storage for logs (S3/Azure Blob/GCS):
# IPI: Easy to configure, cloud integration works
# Loki automatically writes to S3/Azure/GCS

## What IPI Provides
- ✅ Monitoring stack pre-installed
- ✅ Cloud storage for metrics (via CSI)
- ✅ Cloud storage for logs (S3/Azure/GCS)
- ✅ Automatic metric collection from infrastructure
- ✅ Simple integration with cloud monitoring services
- ✅ Auto-configured service discovery

---
### UPI Approach

# Cluster monitoring - Same configuration as IPI
# (Monitoring is cluster-internal, not infrastructure-dependent)

# Day 1: Monitoring stack deployed same as IPI
```bash
oc get pods -n openshift-monitoring
# (Same as IPI)
```

# Day 2: Same as IPI EXCEPT storage considerations

# Persistent storage for metrics - depends on storage setup

# If you configured cloud storage (see Storage section):
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
name: cluster-monitoring-config
namespace: openshift-monitoring
data:
config.yaml: |
prometheusK8s:
volumeClaimTemplate:
spec:
storageClassName: gp3-csi  # Your configured storage class
resources:
requests:
storage: 100Gi
EOF
```

# If using manual/static storage:
# 1. Create volumes manually
```bash
aws ec2 create-volume --size 100 --availability-zone us-east-1a
aws ec2 create-volume --size 100 --availability-zone us-east-1b
```

# 2. Create PVs
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
name: prometheus-0
spec:
capacity:
storage: 100Gi
accessModes:
- ReadWriteOnce
awsElasticBlockStore:
volumeID: vol-xxx
fsType: ext4
```

---
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
name: prometheus-1
spec:
capacity:
storage: 100Gi
accessModes:
- ReadWriteOnce
awsElasticBlockStore:
volumeID: vol-yyy
fsType: ext4
EOF
```

# 3. Configure monitoring to use these PVs
# (More complex, need to ensure PVC binds to correct PVs)

# Logging with cloud storage - more complex

# S3 for Loki (UPI):
# 1. Create S3 bucket manually
```bash
aws s3 mb s3://my-cluster-logging
```

# 2. Create IAM policy for Loki access
```bash
aws iam create-policy \
--policy-name LokiS3Access \
--policy-document '{
"Version": "2012-10-17",
"Statement": [{
"Effect": "Allow",
"Action": ["s3:ListBucket", "s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
"Resource": [
"arn:aws:s3:::my-cluster-logging",
"arn:aws:s3:::my-cluster-logging/*"
]
}]
}'
```

# 3. Create IAM role for Loki pods
# (Complex: requires IRSA or static credentials)

# 4. Create secret with S3 credentials
```bash
oc create secret generic logging-loki-s3 \
-n openshift-logging \
--from-literal=bucketnames=my-cluster-logging \
--from-literal=endpoint=https://s3.us-east-1.amazonaws.com \
--from-literal=region=us-east-1 \
--from-literal=access_key_id=AKIA... \
--from-literal=access_key_secret=...
```

# 5. Configure LokiStack (same as IPI)
# Now it works

# OR use local storage for logs (if no cloud integration):
```bash
cat <<EOF | oc apply -f -
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
name: instance
namespace: openshift-logging
spec:
collection:
type: vector
logStore:
type: elasticsearch  # Local Elasticsearch instead of cloud Loki
elasticsearch:
storage:
storageClassName: local-storage  # Your local storage class
size: 200Gi
EOF
```

# Infrastructure monitoring - Manual INTEGRATION
# IPI: Cloud provider integration may provide additional infrastructure metrics
# UPI: No automatic cloud integration

# If you want CloudWatch/Azure Monitor/Stackdriver metrics:
# YOU must configure separately:

# Example: AWS CloudWatch agent on UPI
# 1. Create IAM role for CloudWatch access
# 2. Deploy CloudWatch agent as DaemonSet
# 3. Configure agent with cluster info
# 4. View metrics in CloudWatch console (separate from cluster monitoring)

# OR: Export metrics to external monitoring
# Configure Prometheus remote write:
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
name: cluster-monitoring-config
namespace: openshift-monitoring
data:
config.yaml: |
prometheusK8s:
remoteWrite:
- url: https://prometheus-remote.example.com/api/v1/write
basicAuth:
username:
name: prom-remote-creds
key: username
password:
name: prom-remote-creds
key: password
EOF
```

# Works same as IPI (cluster monitoring is infrastructure-agnostic)

## Key Differences

```
  ┌────────────────────────┬────────────────────────────────────────┬─────────────────────────────────────┐
  │         Aspect         │                  IPI                   │                 UPI                 │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Monitoring stack       │ Auto-installed                         │ Auto-installed (same)               │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Metrics storage        │ Easy cloud storage via CSI             │ Depends on storage configuration    │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Log storage (cloud)    │ Simple S3/Azure/GCS setup              │ Manual bucket creation + IAM setup  │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Infrastructure metrics │ May include cloud provider integration │ Manual cloud integration if desired │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Alerting               │ Same                                   │ Same                                │
  ├────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────┤
  │ Service discovery      │ Same                                   │ Same                                │
  └────────────────────────┴────────────────────────────────────────┴─────────────────────────────────────┘
```

---

## 9. Networking Configuration

### IPI Approach

# Networking - Fully integrated with cloud

# Day 1: Installer configures:
# - VPC with optimal CIDR ranges
# - Public + Private subnets across 3 AZs
# - Internet Gateway
# - NAT Gateways (for private subnet egress)
# - Route tables
# - Security groups with correct ingress/egress rules
# - Network Load Balancers
# - Cloud CNI integration (if using cloud CNI)

# Day 2+: Network changes

# Add firewall rule (security group) - via cloud provider
# OpenShift doesn't directly manage SGs, but cloud integration respects them

# Example: Expose custom port for NodePort service
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
name: my-app
spec:
type: NodePort
selector:
app: my-app
ports:
- port: 80
targetPort: 8080
nodePort: 30080  # Exposed on all nodes
EOF
```

# IPI: Worker security group already allows ingress on 30000-32767 (NodePort range)
# Traffic flows automatically

# Egress control - cloud NAT gateways
# Workers in private subnets use NAT Gateways (created by installer)
# All egress traffic goes through NAT

# Restrict egress to specific destinations:
# Use NetworkPolicy (cluster-level):
```bash
cat <<EOF | oc apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: deny-external
namespace: myapp
spec:
podSelector: {}
policyTypes:
- Egress
egress:
- to:
- namespaceSelector: {}  # Allow cluster-internal only
```

- to:
- podSelector:
matchLabels:
app: dns
ports:
- protocol: UDP
port: 53  # Allow DNS
EOF

# OR use cloud-level egress control (e.g., AWS VPC endpoints, NAT Gateway modifications)
# IPI: Infrastructure managed by cluster, coordinate with cloud admin

# VPC Peering / VPN - cloud configuration
# Connect cluster VPC to corporate network:

# AWS example:
# 1. Create VPC peering connection
```bash
aws ec2 create-vpc-peering-connection \
--vpc-id vpc-cluster \
--peer-vpc-id vpc-corporate
```

# 2. Accept peering
```bash
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id pcx-xxx
```

# 3. Update route tables
# IPI: Cluster-managed route tables need updates
# Find cluster route tables:
```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-cluster"
```

# Add routes to corporate CIDR:
```bash
aws ec2 create-route \
--route-table-id rtb-cluster-private-1a \
--destination-cidr-block 192.168.0.0/16 \
--vpc-peering-connection-id pcx-xxx
```

# Repeat for all AZ route tables

# 4. Update security groups
# Allow traffic from corporate CIDR:
```bash
aws ec2 authorize-security-group-ingress \
--group-id sg-cluster-workers \
--protocol tcp \
--port 443 \
--cidr 192.168.0.0/16
```

# Ingress controller customization - Automatic CLOUD LB
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.cluster.example.com
endpointPublishingStrategy:
type: LoadBalancerService
loadBalancer:
scope: Internal  # Cloud provider creates internal LB
providerParameters:
type: AWS
aws:
type: NLB  # Network Load Balancer
networkLoadBalancer:
subnets:  # Optional: specify subnets
names:
- cluster-private-us-east-1a
```

- cluster-private-us-east-1b
EOF

# Cloud provider automatically:
# 1. Creates internal NLB
# 2. Creates target groups
# 3. Registers router pods
# 4. Configures security groups

# Service type LoadBalancer with annotations - Automatic
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
name: my-app
annotations:
service.beta.kubernetes.io/aws-load-balancer-type: nlb  # AWS-specific
service.beta.kubernetes.io/aws-load-balancer-internal: "true"
service.beta.kubernetes.io/aws-load-balancer-subnets: subnet-xxx,subnet-yyy
spec:
type: LoadBalancer
selector:
app: my-app
ports:
- port: 443
targetPort: 8443
EOF
```

# Cloud provider creates NLB per annotations automatically

## What IPI Provides
- ✅ Optimized network topology (VPC, subnets, routing)
- ✅ Integrated NAT for egress
- ✅ Pre-configured security groups
- ✅ Automatic load balancer provisioning with cloud annotations
- ✅ Cloud CNI integration (if applicable)
- ✅ VPC/subnet tagging for cloud integrations

---
### UPI Approach

# Networking - you designed and created everything

# Day 1: YOU created:
# - VPC/Virtual Network (your CIDR choices)
# - Subnets (your topology)
# - Routing (your route tables)
# - NAT/Internet Gateway (your egress design)
# - Security groups/firewalls (your rules)
# - Load balancers (your configuration)

# Day 2+: Network changes

# Add firewall rule - Manual
# Want to expose NodePort 30080?

# YOU update security group:
```bash
aws ec2 authorize-security-group-ingress \
--group-id sg-your-workers \
--protocol tcp \
--port 30080 \
--cidr 0.0.0.0/0
```

# Egress control - your NAT configuration
# Workers use YOUR NAT Gateway/NAT instance

# Restrict egress at cloud level:
# 1. Update route table to remove default route
# 2. Add specific routes for allowed destinations
# 3. OR: Use network firewall (AWS Network Firewall, Azure Firewall)

```bash
aws ec2 create-route \
--route-table-id rtb-your-private \
--destination-cidr-block 52.0.0.0/8 \  # AWS service ranges only
--nat-gateway-id nat-xxx
```

# NetworkPolicy works same as IPI (cluster-level, infrastructure-agnostic)

# VPC Peering / VPN - your infrastructure
# Connect to corporate network:

# YOU created the VPC, so peering is straightforward:
# 1. Create VPC peering (same as IPI)
# 2. Update YOUR route tables (you control them)
# 3. Update YOUR security groups (you control them)

# Advantage: Full control over routing
# No conflict with installer-managed resources

# Ingress controller customization - Manual LB OR NODE IPS

# Option 1: Create separate load balancer manually
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
name: internal
namespace: openshift-ingress-operator
spec:
domain: internal.apps.cluster.example.com
endpointPublishingStrategy:
type: HostNetwork  # Or NodePort (no cloud LB integration)
hostNetwork: {}
EOF
```

# YOU create load balancer:
```bash
aws elbv2 create-load-balancer \
--name internal-ingress-lb \
--type network \
--scheme internal \
--subnets subnet-your-private-1a subnet-your-private-1b
```

# YOU create target group pointing to worker node IPs (HostNetwork mode)
```bash
aws elbv2 create-target-group \
--name internal-ingress-tg \
--protocol TCP \
--port 443 \
--vpc-id vpc-your-vpc \
--target-type ip  # HostNetwork uses node IPs
```

# Register worker node IPs:
```bash
aws elbv2 register-targets \
--target-group-arn arn:... \
--targets Id=10.0.1.50 Id=10.0.1.51 Id=10.0.1.52
```

# Option 2: Configure cloud provider integration (post-install)
# Then LoadBalancerService works like IPI
# (See Machine Management section for cloud provider setup)

# Service type LoadBalancer - No cloud integration (default)
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Service
metadata:
name: my-app
spec:
type: LoadBalancer
selector:
app: my-app
ports:
- port: 443
EOF
```

```bash
oc get svc my-app
# NAME     TYPE           EXTERNAL-IP
# my-app   LoadBalancer   <pending>  ← No cloud integration
```

# YOU create load balancer manually (see load balancer section above)

# Network changes coordination - full control
# Advantage: You can implement any network topology
# - Shared services VPC
# - Hub-and-spoke architecture
# - Custom DNS resolution (internal zones)
# - Multi-region connectivity
# - On-prem integration via Direct Connect/ExpressRoute

# Example: Custom DNS for internal services
# IPI: Limited to cluster DNS + cloud DNS integration
# UPI: YOU control DNS, can create any records

# Your internal DNS server can resolve:
# service.namespace.svc.cluster.local → from outside cluster
# (Configure your DNS to forward to cluster DNS or create static records)

## Key Differences

```
  ┌────────────────────────────┬────────────────────────────────────────┬─────────────────────────────────────────────────────────┐
  │           Aspect           │                  IPI                   │                           UPI                           │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Network topology           │ Installer-designed (opinionated)       │ Your design (complete freedom)                          │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Security group changes     │ Coordinate with installer-managed SGs  │ Full control, you manage all SGs                        │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Load balancer for services │ Automatic via cloud integration        │ Manual creation or post-install cloud config            │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Egress control             │ NAT Gateway (installer-managed)        │ YOUR NAT design (could be NAT instance, firewall, etc.) │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ VPC peering/VPN            │ Coordinate with installer resources    │ Full control, your infrastructure                       │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Network policy             │ Same                                   │ Same                                                    │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Routing changes            │ Coordinate with installer route tables │ Full control, your route tables                         │
  ├────────────────────────────┼────────────────────────────────────────┼─────────────────────────────────────────────────────────┤
  │ Custom topology            │ Limited (must fit installer model)     │ Unlimited (any design)                                  │
  └────────────────────────────┴────────────────────────────────────────┴─────────────────────────────────────────────────────────┘
```

---

## 10. Disaster Recovery & Backup

### IPI Approach

# Disaster recovery - Relies on cloud integrations

# Etcd backup - Same process (both IPI and UPI)
# Backup etcd data:
```bash
oc debug node/master-0
# chroot /host
# /usr/local/bin/cluster-backup.sh /home/core/backup
```

# This creates:
# - snapshot_<timestamp>.db (etcd snapshot)
# - static_kuberesources_<timestamp>.tar.gz (static pod manifests)

# Copy backup off cluster:
```bash
oc rsync master-0:/home/core/backup/ ./etcd-backup/
```

# Store in cloud object storage (IPI makes this easy):
# AWS S3:
```bash
aws s3 cp ./etcd-backup/ s3://my-cluster-backups/etcd/ --recursive
```

# Automated backups - use cloud storage
```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
name: etcd-backup
namespace: openshift-etcd
spec:
schedule: "0 2 * * *"  # Daily at 2 AM
jobTemplate:
spec:
template:
spec:
nodeSelector:
node-role.kubernetes.io/master: ""
tolerations:
- operator: Exists
containers:
- name: backup
image: quay.io/openshift/origin-cli:latest
command:
- /bin/bash
- -c
- |
  oc debug node/${MASTER_NODE} -- /bin/bash -c '
  chroot /host /usr/local/bin/cluster-backup.sh /tmp/backup
  aws s3 cp /tmp/backup/ s3://my-cluster-backups/etcd/$(date +%Y%m%d)/ --recursive
  '
env:
- name: MASTER_NODE
value: master-0
- name: AWS_ACCESS_KEY_ID
valueFrom:
secretKeyRef:
name: aws-creds
key: aws_access_key_id
- name: AWS_SECRET_ACCESS_KEY
valueFrom:
secretKeyRef:
name: aws-creds
key: aws_secret_access_key
EOF
```

```bash
velero install \
--provider aws \
--plugins velero/velero-plugin-for-aws:v1.9.0 \
--bucket my-cluster-velero-backups \
--secret-file ./credentials-velero \
--backup-location-config region=us-east-1
```

# Backup namespace:
velero backup create myapp-backup --include-namespaces myapp

# Velero automatically:
# - Backs up Kubernetes resources to S3
# - Creates volume snapshots (EBS snapshots via CSI)
# - Stores backup metadata in S3

# Restore from backup:
velero restore create --from-backup myapp-backup

# Velero automatically:
# - Restores Kubernetes resources
# - Creates volumes from snapshots
# - Restores PVCs

# Infrastructure snapshots - cloud provider
# IPI doesn't directly manage infrastructure snapshots, but:

# Take EC2 AMI of master nodes (for quick recovery):
```bash
aws ec2 create-image \
--instance-id i-master-0 \
--name "cluster-master-backup-$(date +%Y%m%d)" \
--description "Master node backup"
```

# Volume snapshots (automated via cloud):
# CSI driver can create snapshots:
```bash
cat <<EOF | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
name: myapp-data-snapshot
spec:
volumeSnapshotClassName: csi-aws-vsc
source:
persistentVolumeClaimName: myapp-data
EOF
```

# Snapshot stored in AWS (EBS snapshot), managed by cloud

# Cross-region DR - use cloud replication
# Replicate S3 backups to DR region:
```bash
aws s3 sync s3://my-cluster-backups/ s3://my-cluster-backups-dr/ --source-region us-east-1 --region us-west-2
```

# Replicate EBS snapshots to DR region:
```bash
aws ec2 copy-snapshot \
--source-region us-east-1 \
--source-snapshot-id snap-xxx \
--destination-region us-west-2
```

---

### UPI Approach

# Disaster recovery - Same tools, more manual coordination

# Etcd backup - same process as IPI
# (Etcd backup is cluster-internal, infrastructure-agnostic)

```bash
oc debug node/master-0
# chroot /host
# /usr/local/bin/cluster-backup.sh /home/core/backup
```

# Store backup - DEPENDS ON YOUR STORAGE

# If you configured cloud storage:
```bash
aws s3 cp ./etcd-backup/ s3://my-cluster-backups/etcd/ --recursive
```

# If no cloud integration:
# Store on network share:
scp -r ./etcd-backup/ backup-server:/backups/openshift/cluster/

# Automated backups - more complex (if no cloud integration)
```bash
cat <<EOF | oc apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
name: etcd-backup
namespace: openshift-etcd
spec:
schedule: "0 2 * * *"
jobTemplate:
spec:
template:
spec:
nodeSelector:
node-role.kubernetes.io/master: ""
tolerations:
- operator: Exists
volumes:
- name: backup-storage
nfs:  # Use NFS or other network storage
server: backup-server.example.com
path: /backups/openshift
containers:
- name: backup
image: quay.io/openshift/origin-cli:latest
volumeMounts:
- name: backup-storage
mountPath: /backup
command:
- /bin/bash
- -c
- |
  oc debug node/${MASTER_NODE} -- /bin/bash -c '
  chroot /host /usr/local/bin/cluster-backup.sh /tmp/backup
  cp -r /tmp/backup/* /backup/$(date +%Y%m%d)/
  '
EOF
```

# Application backup - Velero (more complex without cloud integration)

# Option 1: Velero with cloud storage (if you configured cloud provider)
# Same as IPI
```bash
velero install \
--provider aws \  # Use AWS provider with S3-compatible endpoint
--plugins velero/velero-plugin-for-aws:v1.9.0 \
--bucket velero-backups \
--secret-file ./credentials-velero \
--backup-location-config \
region=minio,s3ForcePathStyle="true",s3Url=http://minio.example.com:9000
```

```bash
velero install \
--provider restic \
--use-node-agent \
--no-default-backup-location
```

# Configure restic to backup to NFS:
```bash
cat <<EOF | oc apply -f -
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
name: default
spec:
provider: restic
objectStorage:
bucket: /backups/velero  # Local path on NFS mount
EOF
```

# Volume snapshots - more manual

# If you configured CSI driver:
# Same as IPI (automatic snapshots)

# If using manual storage:
# Take cloud snapshots manually:
```bash
aws ec2 create-snapshot \
--volume-id vol-myapp-data \
--description "myapp data backup $(date +%Y%m%d)"
```

# OR: Use volume-level tools (LVM snapshots, storage array snapshots)
lvcreate --size 1G --snapshot --name myapp-data-snap /dev/vg0/myapp-data

# Infrastructure snapshots - Fully manual
# Snapshot master nodes:
```bash
aws ec2 create-image \
--instance-id i-your-master-0 \
--name "master-0-backup-$(date +%Y%m%d)"
```

# Snapshot worker nodes:
```bash
for worker in i-worker-0 i-worker-1 i-worker-2; do
aws ec2 create-image \
--instance-id $worker \
--name "${worker}-backup-$(date +%Y%m%d)"
done
```

# Backup load balancer config:
```bash
aws elbv2 describe-load-balancers --load-balancer-arns arn:... > lb-config-backup.json
aws elbv2 describe-target-groups --load-balancer-arn arn:... > tg-config-backup.json
```

# Backup network config:
```bash
aws ec2 describe-vpcs --vpc-ids vpc-xxx > vpc-config.json
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxx" > subnets-config.json
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxx" > routes-config.json
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-xxx" > sg-config.json
```

# Store as Infrastructure as Code:
# Better approach: Use Terraform/CloudFormation
# Your UPI deployment is already IaC, so infrastructure is "backed up" as code

```bash
terraform state pull > terraform-state-backup.json
```

# Cross-region DR - more coordination

# Replicate S3 backups:
```bash
aws s3 sync s3://my-cluster-backups/ s3://my-cluster-backups-dr/ \
--source-region us-east-1 --region us-west-2
```

# Replicate EBS snapshots:
# List all cluster volumes:
```bash
aws ec2 describe-volumes \
--filters "Name=tag:kubernetes.io/cluster/my-cluster,Values=owned" \
--query 'Volumes[*].VolumeId' --output text | \
while read vol; do
snap_id=$(aws ec2 create-snapshot --volume-id $vol --query 'SnapshotId' --output text)
aws ec2 copy-snapshot \
--source-region us-east-1 \
--source-snapshot-id $snap_id \
--destination-region us-west-2
done
```

# Replicate infrastructure config:
# Re-run Terraform in DR region with backed-up state/config
cd terraform-dr
```bash
terraform apply -var="region=us-west-2"
```

# Disaster recovery procedure - MORE STEPS

# IPI cluster recovery:
# 1. Restore etcd from backup
# 2. Cluster recovers (infrastructure still managed by cloud provider)

# UPI cluster recovery:
# 1. Restore/recreate infrastructure (Terraform apply or manual)
# 2. Restore etcd from backup
# 3. Verify load balancers pointing to new masters
# 4. Verify DNS records updated
# 5. Verify security groups/firewalls configured
# 6. Restore applications (Velero restore)

## Key Differences

```
  ┌─────────────────────────────┬──────────────────────────────────────────┬───────────────────────────────────────────────┐
  │           Aspect            │                   IPI                    │                      UPI                      │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Etcd backup                 │ Same                                     │ Same                                          │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Application backup (Velero) │ Easy cloud integration                   │ May need S3-compatible or filesystem backend  │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Volume snapshots            │ Automatic (CSI)                          │ Depends on storage setup                      │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Infrastructure backup       │ Not needed (can recreate with installer) │ Must backup OR maintain as IaC                │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Cross-region DR             │ Cloud replication tools                  │ Cloud replication + infrastructure recreation │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Recovery complexity         │ Lower (infrastructure auto-managed)      │ Higher (infrastructure + cluster)             │
  ├─────────────────────────────┼──────────────────────────────────────────┼───────────────────────────────────────────────┤
  │ Backup storage              │ Easy cloud storage                       │ Depends on configuration                      │
  └─────────────────────────────┴──────────────────────────────────────────┴───────────────────────────────────────────────┘
```

---
## Summary Table: IPI vs UPI Day 1+ Operations

```
  ┌────────────────────────────┬────────────────────────────────────┬────────────────────────────────────────────┬──────────────────────────────────────────┐
  │         Operation          │                IPI                 │                UPI (Manual)                │          UPI (Post-Configured)           │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Machine scaling            │ oc scale machineset                │ Provision VM manually + approve CSR        │ oc scale machineset (if configured)      │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Node replacement           │ Automatic (health check)           │ Manual VM + CSR approval                   │ Automatic (if Machine API configured)    │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Cluster upgrade            │ oc adm upgrade + automatic         │ oc adm upgrade + verify LB/DNS/SG changes  │ oc adm upgrade + verify infrastructure   │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Load balancer for services │ Automatic cloud LB                 │ Manual LB creation                         │ Automatic (if cloud provider configured) │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ DNS for new domains        │ Automatic or simple delegation     │ Manual DNS records                         │ Automatic (if cloud DNS configured)      │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Certificate rotation       │ Automatic                          │ Automatic (cluster certs)                  │ Automatic                                │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Storage provisioning       │ Automatic (CSI)                    │ Manual volumes or static PVs               │ Automatic (if CSI configured)            │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Volume expansion           │ oc patch pvc                       │ Manual cloud resize + filesystem           │ oc patch pvc (if CSI configured)         │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Monitoring storage         │ Cloud storage (easy)               │ Manual storage or static PVs               │ Cloud storage (if configured)            │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Logging storage            │ S3/Azure/GCS (easy)                │ Manual bucket + IAM OR local storage       │ S3/Azure/GCS (if configured)             │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Networking changes         │ Coordinate with installer SGs      │ Full control, manage your SGs              │ Full control                             │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Disaster recovery          │ Cloud backups (straightforward)    │ Manual backups + infrastructure recreation │ Cloud backups (if configured)            │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Infrastructure as Code     │ Limited (conflicts with installer) │ Full control (Terraform/etc.)              │ Full control                             │
  ├────────────────────────────┼────────────────────────────────────┼────────────────────────────────────────────┼──────────────────────────────────────────┤
  │ Day 2 complexity           │ Low (automation built-in)          │ High (manual coordination)                 │ Medium (depends on configuration)        │
  └────────────────────────────┴────────────────────────────────────┴────────────────────────────────────────────┴──────────────────────────────────────────┘
```

---
## Conclusion

## IPI Day 1+
- Strengths: Automation, simplicity, cloud-native integrations work out of the box
- Limitations: Less flexibility, must work within installer's model, harder to customize infrastructure

## UPI Day 1+
- Strengths: Complete control, custom topologies, integration with existing infrastructure, can implement any design
- Limitations: More manual work, must maintain infrastructure separately, requires deep expertise

Best practice for UPI: Treat infrastructure as code (Terraform/CloudFormation) and automate as much as possible. UPI gives you the freedom to build robust automation tailored to your needs.

