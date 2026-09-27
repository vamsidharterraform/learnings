
# EKS Kubernetes Upgrade Runbook
## Kubernetes 1.33 → 1.34

**Cluster:** `prod-app-cluster`  
**AWS Region:** `us-west-2`  
**Upgrade:** Kubernetes `1.33 → 1.34`  
**Infrastructure:** Terraform  
**Node Groups:** `system`, `application`  
**Networking:** AWS VPC CNI  
**Storage:** EBS CSI / GP2  
**GitOps:** Argo CD  
**Monitoring:** Prometheus / Grafana / CloudWatch  
**Ingress:** AWS Load Balancer Controller

---

# 1. Upgrade Architecture

The upgrade was performed in two independent phases:

```text
                    EKS Upgrade
                         |
             +-----------+-----------+
             |                       |
       Control Plane             Worker Nodes
          1.33 → 1.34              1.33 → 1.34
             |                       |
       EKS API Server          Managed Node Groups
             |                       |
             |              +--------+--------+
             |              |                 |
             |           system          application
             |              |                 |
             +--------------+-----------------+
                            |
                    Kubernetes workloads
```

## Important principle

The EKS control plane and worker nodes are upgraded separately.

```text
Control Plane Upgrade
        ↓
Validate Cluster
        ↓
Upgrade Managed Node Groups
        ↓
Validate Nodes
        ↓
Validate Applications
```

Upgrading the control plane does **not automatically mean that all worker nodes immediately become the new Kubernetes version**.

---

# 2. Phase 0 – Pre-Upgrade Preparation

Before starting the upgrade, verify:

- Current Kubernetes version
- Cluster status
- Node versions
- Node groups
- EKS add-ons
- Kubernetes workloads
- PVC/PV
- CRDs
- Deprecated APIs
- EKS Upgrade Insights
- Application health
- Terraform state

---

# 3. Check Current Cluster Version

```bash
aws eks describe-cluster \
  --name prod-app-cluster \
  --query 'cluster.{Version:version,Status:status}' \
  --output table
```

Expected before upgrade:

```text
Version    Status
---------  ------
1.33       ACTIVE
```

---

# 4. Check Kubernetes Nodes

```bash
kubectl get nodes -o wide
```

Check Kubernetes versions:

```bash
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion,OS:.status.nodeInfo.osImage"
```

The nodes were running:

```text
Kubernetes: 1.33.13
OS: Amazon Linux 2023
```

---

# 5. Check Node Groups

```bash
aws eks list-nodegroups \
  --cluster-name prod-app-cluster
```

Check each node group:

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name system \
  --query 'nodegroup.{Status:status,Version:version,Desired:scalingConfig.desiredSize,Min:scalingConfig.minSize,Max:scalingConfig.maxSize}' \
  --output table
```

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name application \
  --query 'nodegroup.{Status:status,Version:version,Desired:scalingConfig.desiredSize,Min:scalingConfig.minSize,Max:scalingConfig.maxSize}' \
  --output table
```

---

# 6. Check EKS Add-ons

```bash
aws eks list-addons \
  --cluster-name prod-app-cluster
```

Installed add-ons:

```text
amazon-cloudwatch-observability
aws-ebs-csi-driver
eks-pod-identity-agent
vpc-cni
```

Check versions:

```bash
aws eks describe-addon \
  --cluster-name prod-app-cluster \
  --addon-name vpc-cni \
  --query 'addon.{Version:addonVersion,Status:status}' \
  --output table
```

The VPC CNI version was:

```text
v1.23.0-eksbuild.1
```

The version was compatible with the target Kubernetes version, so no unnecessary downgrade was performed.

---

# 7. Check Kubernetes Workloads

Before upgrading:

```bash
kubectl get pods -A
```

Look for:

```text
Running
Completed
Pending
CrashLoopBackOff
ImagePullBackOff
```

A useful command:

```bash
kubectl get pods -A \
  --field-selector=status.phase!=Running,status.phase!=Succeeded
```

Also check:

```bash
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A
kubectl get jobs -A
kubectl get cronjobs -A
```

---

# 8. Check Storage

Check PVCs:

```bash
kubectl get pvc -A
```

Check PVs:

```bash
kubectl get pv
```

This is especially important for stateful workloads because EBS volumes are AZ-aware.

---

# 9. Check CRDs

```bash
kubectl get crd \
  -o custom-columns="NAME:.metadata.name,VERSION:.spec.versions[*].name"
```

Important CRDs included:

```text
External Secrets
AWS Load Balancer Controller
Prometheus Operator
CloudWatch Observability
AWS VPC CNI Network Policy
```

Important point:

> A CRD using `v1alpha1` or `v1beta1` does not automatically mean that it is a deprecated Kubernetes built-in API. CRDs are owned by their respective controllers/operators and must be evaluated based on the controller version and compatibility.

---

# 10. Check API Versions Used by Workloads

```bash
kubectl get deploy,sts,ds,job,cronjob,svc,ingress,hpa,pdb,networkpolicy \
  -A -o yaml | \
  grep 'apiVersion:' | \
  sort | uniq -c
```

The cluster contained current APIs such as:

```text
v1
apps/v1
autoscaling/v2
networking.k8s.io/v1
policy/v1
storage.k8s.io/v1
```

---

# 11. EKS Upgrade Insights

Before upgrading, check EKS Upgrade Insights.

The important checks we verified were:

```text
EKS add-on version compatibility       PASSING
kube-proxy version skew               PASSING
Kubelet version skew                  PASSING
Cluster health issues                 PASSING
Amazon Linux 2 compatibility          PASSING
```

The Amazon Linux 2 check was passing because the nodes were running:

```text
Amazon Linux 2023
```

and not AL2.

---

# 12. Terraform Configuration

The Terraform root module contained:

```hcl
module "eks" {
  source = "./modules/eks"

  cluster_name              = var.cluster_name
  cluster_version           = var.cluster_version
  cluster_nodegroup_version = var.cluster_nodegroup_version
  vpc_id                    = module.vpc.vpc_id
  subnet_ids                = module.vpc.private_subnet_ids
  node_groups               = var.node_groups
}
```

Two separate variables were maintained:

```hcl
variable "cluster_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.33"
}

variable "cluster_nodegroup_version" {
  description = "nodegroup Kubernetes version"
  type        = string
  default     = "1.33"
}
```

This separation is important because:

```text
cluster_version
        ↓
EKS control plane

cluster_nodegroup_version
        ↓
Managed worker nodes
```

---

# 13. Phase 1 – Terraform Control Plane Upgrade

First change:

```hcl
cluster_version = "1.34"
```

Keep:

```hcl
cluster_nodegroup_version = "1.33"
```

This intentionally upgrades only the control plane.

Run:

```bash
terraform plan
```

The expected important change:

```text
aws_eks_cluster.main
1.33 → 1.34
```

The node groups should not change in this phase.

---

# 14. Understand the Terraform Plan

The plan showed:

```text
aws_eks_cluster.main
version: 1.33 → 1.34
```

Other related resources were recalculated as part of Terraform's dependency graph.

For example:

```text
data.tls_certificate.eks
```

could be:

```text
known after apply
```

This does not mean Terraform is creating a new certificate.

It means the data source will be resolved during apply because the dependent EKS cluster information changes.

The OIDC provider and External Secrets IRSA trust relationship were also recalculated.

---

# 15. Apply Control Plane Upgrade

```bash
terraform apply
```

After completion:

```bash
aws eks describe-cluster \
  --name prod-app-cluster \
  --query 'cluster.{Version:version,Status:status}' \
  --output table
```

Expected:

```text
Version    Status
---------  ------
1.34       ACTIVE
```

---

# 16. Important Observation After Control Plane Upgrade

After the control plane upgrade:

```bash
kubectl get nodes
```

still showed worker nodes running:

```text
v1.33.13
```

This was expected.

The situation was:

```text
Control Plane → 1.34
Workers       → 1.33
```

This demonstrates the separation between EKS control-plane upgrades and managed node-group upgrades.

---

# 17. Validate Cluster After Control Plane Upgrade

```bash
kubectl get pods -A
```

Verify important components:

```text
CoreDNS
AWS VPC CNI
EBS CSI
AWS Load Balancer Controller
External Secrets
Argo CD
Prometheus
Grafana
Metrics Server
CloudWatch
```

Also:

```bash
kubectl get nodes
```

and:

```bash
kubectl get events -A \
  --sort-by='.lastTimestamp'
```

Look for unexpected errors.

---

# 18. Phase 2 – Upgrade Managed Node Groups

After confirming the control plane was healthy, change:

```hcl
cluster_nodegroup_version = "1.34"
```

Now the configuration becomes:

```hcl
variable "cluster_version" {
  default = "1.34"
}

variable "cluster_nodegroup_version" {
  default = "1.34"
}
```

---

# 19. Terraform Plan for Worker Upgrade

Run:

```bash
terraform plan
```

The expected changes were:

```text
module.eks.aws_eks_node_group.main["application"]
1.33 → 1.34

module.eks.aws_eks_node_group.main["system"]
1.33 → 1.34
```

Importantly:

```text
No replacement of the node group
```

The managed node groups were updated in place.

---

# 20. Managed Node Group Rolling Upgrade

The node groups use:

```hcl
update_config {
  max_unavailable = 1
}
```

This provides controlled rolling replacement.

Conceptually:

```text
Old node
   ↓
Cordoned/drained by EKS managed update process
   ↓
New node created
   ↓
Workloads rescheduled
   ↓
Old node removed
   ↓
Next node
```

For a standard EKS Managed Node Group upgrade, we generally don't manually perform the entire cordon/drain process ourselves.

EKS manages the managed node-group rolling update.

---

# 21. Monitor Node Group Upgrade

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name system \
  --query 'nodegroup.{Status:status,Version:version}' \
  --output table
```

Application:

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name application \
  --query 'nodegroup.{Status:status,Version:version}' \
  --output table
```

Monitor Kubernetes nodes:

```bash
kubectl get nodes -o wide -w
```

After the upgrade:

```text
v1.34.11
```

was visible on the worker nodes.

---

# 22. Validate Node Groups

```bash
kubectl get nodes \
  -L workload \
  -L topology.kubernetes.io/zone
```

Our final topology during the incident was:

```text
SYSTEM
  us-west-2a
  us-west-2c

APPLICATION
  us-west-2b
  us-west-2c
```

All nodes were:

```text
Ready
v1.34.11
```

---

# 23. Production Issue Encountered During Upgrade

## Prometheus Became Pending

During the managed node-group rolling upgrade:

```text
prometheus-kube-prometheus-stack-prometheus-0
```

became:

```text
0/2 Pending
```

This was an important production-style incident.

---

# 24. First Troubleshooting Step

Run:

```bash
kubectl describe pod \
  prometheus-kube-prometheus-stack-prometheus-0 \
  -n monitoring
```

The scheduler reported:

```text
0/6 nodes are available:
1 node(s) were unschedulable,
2 node(s) didn't match PersistentVolume's node affinity,
3 node(s) didn't match Pod's node affinity/selector.
```

Later, as nodes were being replaced:

```text
0/5 nodes
```

and eventually:

```text
0/4 nodes
```

with PV and pod affinity/selector mismatch messages.

---

# 25. Check Prometheus PVC

```bash
kubectl get pvc -n monitoring
```

The Prometheus PVC was:

```text
STATUS: Bound
CAPACITY: 10Gi
ACCESS MODE: RWO
STORAGECLASS: gp2
```

This immediately showed that the PVC itself was not simply "broken."

---

# 26. Inspect the PV

```bash
kubectl describe pv \
  pvc-76b45025-495a-4aed-9e9f-4f316e2ece59
```

Important result:

```text
Capacity: 10Gi
Access Mode: RWO
StorageClass: gp2
```

Most importantly:

```text
Node Affinity:

topology.kubernetes.io/zone in [us-west-2b]
topology.kubernetes.io/region in [us-west-2]
```

Therefore the EBS volume was tied to:

```text
us-west-2b
```

---

# 27. Check Prometheus Pod Affinity

```bash
kubectl get pod \
  prometheus-kube-prometheus-stack-prometheus-0 \
  -n monitoring \
  -o jsonpath='{.spec.affinity}'
```

The pod had pod anti-affinity, but that was not the primary blocker.

---

# 28. Check Prometheus Node Selector

```bash
kubectl get pod \
  prometheus-kube-prometheus-stack-prometheus-0 \
  -n monitoring \
  -o jsonpath='{.spec.nodeSelector}'
```

Result:

```json
{"workload":"system"}
```

This was the critical constraint.

Prometheus required:

```text
workload=system
```

while its EBS volume required:

```text
us-west-2b
```

---

# 29. Check System Nodes and AZs

```bash
kubectl get nodes \
  -l workload=system \
  -L topology.kubernetes.io/zone
```

Result:

```text
system → us-west-2a
system → us-west-2c
```

There was:

```text
NO system node in us-west-2b
```

---

# 30. Check Application Nodes

```bash
kubectl get nodes \
  -l workload=application \
  -L topology.kubernetes.io/zone
```

Result:

```text
application → us-west-2b
application → us-west-2c
```

There was an application node in `2b`, but Prometheus could not use it because:

```text
workload=application
```

didn't satisfy:

```text
workload=system
```

---

# 31. Verify Node Group Subnets

Check system node group:

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name system \
  --query 'nodegroup.subnets' \
  --output table
```

Check application:

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name application \
  --query 'nodegroup.subnets' \
  --output table
```

Both node groups used:

```text
subnet-09f79d5322d92eac1
subnet-072d3ca2f29fed313
subnet-056c291e2cf4c474b
```

---

# 32. Map Subnets to AZs

```bash
aws ec2 describe-subnets \
  --subnet-ids \
  subnet-09f79d5322d92eac1 \
  subnet-072d3ca2f29fed313 \
  subnet-056c291e2cf4c474b \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' \
  --output table
```

Result:

```text
subnet-09f79d5322d92eac1 → us-west-2a → 10.0.1.0/24
subnet-056c291e2cf4c474b → us-west-2b → 10.0.2.0/24
subnet-072d3ca2f29fed313 → us-west-2c → 10.0.3.0/24
```

Therefore the system node group had access to all three AZs.

But access to all three AZs does **not** guarantee that a node exists in every AZ.

---

# 33. Check System Node Group Scaling

```bash
aws eks describe-nodegroup \
  --cluster-name prod-app-cluster \
  --nodegroup-name system \
  --query 'nodegroup.scalingConfig' \
  --output table
```

Result:

```text
desiredSize = 2
minSize     = 2
maxSize     = 2
```

Therefore EKS only maintained two system nodes.

They happened to be:

```text
2a
2c
```

and not:

```text
2b
```

---

# 34. Root Cause

The complete dependency chain was:

```text
EKS Managed Node Group Upgrade
             ↓
Old worker node replaced
             ↓
Prometheus needed rescheduling
             ↓
Prometheus nodeSelector
workload=system
             ↓
Prometheus PVC uses EBS
             ↓
EBS PV has AZ affinity
us-west-2b
             ↓
Available system nodes:
2a + 2c
             ↓
No system node in 2b
             ↓
No node satisfies BOTH constraints
             ↓
Prometheus Pending
```

In short:

```text
Pod scheduling constraint
        +
EBS AZ topology constraint
        +
Node-group AZ distribution
        =
Scheduling failure
```

---

# 35. Why This Is a Production Issue

This is a realistic production failure because the infrastructure can look healthy:

```text
EKS Cluster       → ACTIVE
Nodes             → Ready
Node versions     → Correct
PVC               → Bound
PV                → Bound
EBS               → Existing
Prometheus        → Healthy configuration
```

Yet the workload can still be:

```text
Pending
```

because Kubernetes must satisfy **all scheduling constraints simultaneously**.

---

# 36. Important Lesson About EBS

EBS volumes are Availability Zone scoped.

For example:

```text
EBS Volume
    ↓
us-west-2b
```

It cannot simply attach to a node in:

```text
us-west-2a
```

or:

```text
us-west-2c
```

Therefore stateful workloads using EBS must be designed with storage topology in mind.

---

# 37. Lab Fix We Discussed

The immediate lab approach was to increase the system node-group capacity.

Current:

```text
desired = 2
min     = 2
max     = 2
```

Temporarily:

```bash
aws eks update-nodegroup-config \
  --cluster-name prod-app-cluster \
  --nodegroup-name system \
  --scaling-config desiredSize=3,minSize=2,maxSize=3
```

Then monitor:

```bash
kubectl get nodes \
  -l workload=system \
  -L topology.kubernetes.io/zone \
  -w
```

And monitor Prometheus:

```bash
kubectl get pod \
  -n monitoring \
  prometheus-kube-prometheus-stack-prometheus-0 \
  -w
```

Important:

> Increasing desired capacity does not guarantee that the new node will land in a particular AZ. The node group can use all configured subnets, but node placement is not automatically "one node per AZ."

The cluster recreation was then chosen to reproduce the scenario cleanly before implementing the permanent Terraform design.

---

# 38. Production-Grade Design Considerations

Several approaches can address this type of issue.

## Option 1 – Ensure system capacity exists across required AZs

For example:

```text
System nodes:

2a → system
2b → system
2c → system
```

This allows a Prometheus pod requiring:

```text
workload=system
```

to find a node in the EBS volume's AZ.

However, simply setting three nodes is not a strict guarantee of one node per AZ.

---

## Option 2 – Use dedicated monitoring nodes

A dedicated monitoring node group can be used:

```text
Monitoring Node Group

monitoring=true
        |
        +-- Prometheus
        +-- Grafana
        +-- Alertmanager
```

This provides workload isolation and predictable capacity.

---

## Option 3 – Review Prometheus scheduling constraints

Instead of:

```yaml
nodeSelector:
  workload: system
```

the monitoring workload can be allowed to use an appropriate set of nodes.

However, this should be an intentional architecture decision rather than a quick production workaround.

---

# 39. Things NOT to Do During the Incident

Do not immediately delete:

```bash
kubectl delete pvc ...
```

Do not delete the PV just to make the pod schedule.

Do not manually try to move an EBS volume between AZs.

Do not blindly remove the node selector.

First determine:

```text
Pod constraints
+
PV topology
+
Available nodes
+
AZ distribution
```

Then select the appropriate architectural fix.

---

# 40. Final Upgrade Validation

After both control plane and node-group upgrades:

```bash
aws eks describe-cluster \
  --name prod-app-cluster \
  --query 'cluster.{Version:version,Status:status}' \
  --output table
```

Check nodes:

```bash
kubectl get nodes -o wide
```

Check node versions:

```bash
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,VERSION:.status.nodeInfo.kubeletVersion"
```

Check pods:

```bash
kubectl get pods -A
```

Check workloads:

```bash
kubectl get deployments -A
kubectl get statefulsets -A
kubectl get daemonsets -A
```

Check storage:

```bash
kubectl get pvc -A
kubectl get pv
```

Check services:

```bash
kubectl get svc -A
```

Check ingress:

```bash
kubectl get ingress -A
```

Check events:

```bash
kubectl get events -A \
  --sort-by='.lastTimestamp'
```

---

# 41. Final Upgrade Checklist

## Pre-Upgrade

- [ ] Confirm cluster version
- [ ] Confirm cluster status
- [ ] Check node versions
- [ ] Check node groups
- [ ] Check node-group scaling
- [ ] Check EKS add-ons
- [ ] Check workloads
- [ ] Check PVC/PV
- [ ] Check CRDs
- [ ] Check API versions
- [ ] Review EKS Upgrade Insights
- [ ] Confirm Terraform state
- [ ] Confirm backup/rollback strategy

## Control Plane Upgrade

- [ ] Change `cluster_version`
- [ ] Run `terraform plan`
- [ ] Confirm only intended control-plane changes
- [ ] Run `terraform apply`
- [ ] Verify EKS version
- [ ] Verify cluster `ACTIVE`
- [ ] Verify workloads

## Worker Upgrade

- [ ] Change `cluster_nodegroup_version`
- [ ] Run `terraform plan`
- [ ] Confirm node groups update in place
- [ ] Verify `max_unavailable`
- [ ] Apply Terraform
- [ ] Monitor node-group status
- [ ] Monitor nodes
- [ ] Monitor pods
- [ ] Check scheduling events

## Post-Upgrade

- [ ] All nodes Ready
- [ ] All nodes target Kubernetes version
- [ ] EKS add-ons healthy
- [ ] CoreDNS healthy
- [ ] CNI healthy
- [ ] EBS CSI healthy
- [ ] Load Balancer Controller healthy
- [ ] External Secrets healthy
- [ ] Argo CD healthy
- [ ] Prometheus healthy
- [ ] Grafana healthy
- [ ] PVCs Bound
- [ ] Ingress healthy
- [ ] Application smoke tests successful
- [ ] No unexpected Kubernetes events

---

# 42. Interview Explanation

### Question

**How would you perform an EKS Kubernetes version upgrade in production?**

### Answer

> "I would perform the upgrade in controlled phases. First I would validate the current EKS version, node groups, add-ons, workloads, storage, CRDs and EKS Upgrade Insights. I would also verify that the workloads are using supported Kubernetes APIs.
>
> I would then upgrade the EKS control plane first using Terraform and validate the cluster health. The worker nodes may still remain on the previous Kubernetes version, which is expected.
>
> After validating the control plane, I would update the managed node-group Kubernetes version in Terraform and perform a rolling node-group upgrade. I would use the node-group update configuration such as `max_unavailable=1` to control the rollout and monitor node and pod scheduling throughout the process.
>
> Finally, I would validate node versions, add-ons, workloads, storage, ingress and application functionality.
>
> During the upgrade I encountered a Prometheus scheduling issue. The Prometheus pod had `nodeSelector=workload: system`, while its EBS-backed PVC was bound to a PV with node affinity for `us-west-2b`. The system nodes were in `us-west-2a` and `us-west-2c`, while the only node in `us-west-2b` belonged to the application node group. Therefore Kubernetes couldn't find a node satisfying both the pod's node selector and the PV's AZ constraint. This demonstrated why AZ-aware capacity planning is important for stateful workloads during EKS node replacement."

---

# 43. Key Interview Takeaways

| Topic | What to remember |
|---|---|
| EKS upgrade order | Control plane → worker nodes |
| Control plane | Can be upgraded independently |
| Worker nodes | Managed separately |
| Node upgrade | EKS performs managed rolling replacement |
| `max_unavailable=1` | Controls rollout disruption |
| EBS | AZ-scoped |
| RWO | Volume attached for read/write use by a single node at a time |
| PV node affinity | Can restrict scheduling to an AZ |
| `nodeSelector` | Restricts which nodes a pod can use |
| Subnets | Having subnets in 3 AZs doesn't guarantee nodes in all 3 |
| Stateful workloads | Need topology-aware capacity planning |
| Pending pod | Check scheduler events first |
| Troubleshooting | Pod constraints + PV constraints + node labels + AZ |
| Terraform | Separate control-plane and node-group versions |
| EKS Insights | Validate upgrade compatibility before changing production |
| AL2023 | Current nodes were already using AL2023 |
| CRDs | Controller-owned APIs must be evaluated separately from built-in Kubernetes API deprecations |
| Production mindset | Don't delete storage before identifying the scheduling constraint |

---

# 44. Most Important Production Lesson

The biggest lesson from this upgrade was:

```text
A successful EKS upgrade is not just:

Control Plane = 1.34
Nodes = 1.34
```

It is:

```text
EKS Version
     +
Node Version
     +
Add-ons
     +
Kubernetes APIs
     +
Node Capacity
     +
AZ Distribution
     +
Storage Topology
     +
Pod Scheduling Constraints
     +
Application Health
```

All of these must be considered together during a production EKS upgrade.