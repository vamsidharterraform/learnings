# Amazon EKS Cluster – Technical Notes

These notes cover the major sections of an Amazon EKS cluster and what each section represents from an AWS DevOps / production perspective.

---

## 1. Overview

The Overview section provides the high-level configuration and current status of the EKS cluster.

### Key Information

- **Cluster Name** – Unique name of the EKS cluster.
- **Status** – Current cluster state such as:
  - `CREATING`
  - `ACTIVE`
  - `UPDATING`
  - `DELETING`
  - `FAILED`
- **Kubernetes Version** – Version of Kubernetes running on the EKS control plane.
- **EKS Cluster ARN** – Unique AWS ARN of the cluster.
- **API Server Endpoint** – Kubernetes API server endpoint used by `kubectl`.
- **Platform Version** – AWS EKS platform version associated with the Kubernetes version.
- **Cluster IAM Role** – IAM role assumed by the EKS control plane.
- **OIDC Provider** – Used for IAM Roles for Service Accounts (IRSA).
- **Cluster creation time** – Timestamp when the control plane was created.

> **Important Concept**
> EKS is a managed Kubernetes control plane. AWS manages the Kubernetes API server and control-plane infrastructure, while we manage the workloads, node groups, networking configuration and Kubernetes resources.

---

## 2. Resources

The Resources section represents the infrastructure associated with the EKS cluster.

Typical resources include:

```
EKS Cluster
├── Node Groups
├── EC2 Instances
├── IAM Roles
├── Security Groups
├── VPC
├── Subnets
├── Load Balancers
├── EKS Add-ons
└── Kubernetes workloads
```

### Important EKS Resources

- EKS control plane
- Managed node groups
- Self-managed nodes, if used
- Fargate profiles, if used
- EKS add-ons
- IAM roles
- Security groups
- Load balancers
- VPC/subnets

> **Interview Point**
> The EKS control plane and worker nodes are separate components. Creating an EKS cluster does not automatically mean that worker nodes have been created.

---

## 3. Compute

The Compute section defines where Kubernetes workloads actually run.

### Managed Node Groups

A managed node group is a group of EC2 instances managed by EKS.

Typical configuration:

```
Node Group
├── IAM Node Role
├── EC2 instances
├── Instance type
├── AMI
├── Capacity type
├── Scaling configuration
└── Subnets
```

### Important Configuration

- **Instance type** – `t3.medium`, `m5.large`, `m6i.large`, etc.
- **Capacity type** – `ON_DEMAND`, `SPOT`
- Desired size
- Minimum size
- Maximum size
- AMI type
- Disk size
- Subnet placement
- Node IAM role

### Example

```
System Node Group
├── Desired: 2
├── Min: 2
├── Max: 4
└── ON_DEMAND

Application Node Group
├── Desired: 2
├── Min: 2
├── Max: 6
└── ON_DEMAND
```

### Why Separate Node Groups?

System workloads such as:
- CoreDNS
- AWS Load Balancer Controller
- Metrics components
- Monitoring agents
- EKS add-ons

can be isolated from application workloads. Application workloads can then scale independently.

---

## 4. Networking

EKS networking determines how the control plane, worker nodes, pods and external clients communicate.

Typical architecture:

```
                    Internet
                       |
                    Route 53
                       |
                      ALB
                       |
                 Kubernetes Service
                       |
                     Pods
                       |
                    VPC CNI
                       |
                 EC2 Node ENI
                       |
                      VPC
```

### VPC

The EKS cluster is deployed into a VPC.

Typical design:

```
VPC
├── Public Subnets
│   └── Load Balancers
│
└── Private Subnets
    ├── EKS Nodes
    ├── Applications
    └── Internal services
```

### Subnets

EKS should normally use subnets across multiple Availability Zones.

Example:

```
us-west-2a
├── Private subnet
└── Public subnet

us-west-2b
├── Private subnet
└── Public subnet

us-west-2c
├── Private subnet
└── Public subnet
```

This provides availability across AZs.

### VPC CNI

Amazon VPC CNI provides Kubernetes pods with AWS VPC networking. Pods receive IP addresses from the VPC networking infrastructure.

```
Pod
 ↓
VPC CNI
 ↓
ENI / Secondary IP
 ↓
VPC
```

### Important Networking Components

- VPC
- Public/private subnets
- Route tables
- Internet Gateway
- NAT Gateway
- Security Groups
- Network ACLs
- VPC CNI
- DNS
- Load Balancers

> **Interview Point**
> EKS networking is tightly integrated with the AWS VPC. With the Amazon VPC CNI, pods can use VPC IP addresses and communicate with AWS resources such as RDS according to routing and security controls.

---

## 5. Add-ons

EKS add-ons provide AWS-supported components required for cluster functionality.

Common add-ons:
- VPC CNI
- CoreDNS
- kube-proxy
- EBS CSI Driver
- AWS Load Balancer Controller
- CloudWatch Observability

### VPC CNI
Provides networking for Kubernetes pods.

### CoreDNS
Provides DNS resolution inside the Kubernetes cluster.

```
frontend-service
       ↓
CoreDNS
       ↓
backend-service
```

### kube-proxy
Maintains Kubernetes networking rules required for Service traffic.

### EBS CSI Driver
Allows Kubernetes to dynamically provision and attach Amazon EBS volumes.

Typical flow:

```
PVC
 ↓
StorageClass
 ↓
EBS CSI Driver
 ↓
EBS Volume
 ↓
Pod
```

### AWS Load Balancer Controller
Creates and manages AWS load balancers from Kubernetes resources.

```
Ingress
   ↓
AWS Load Balancer Controller
   ↓
Application Load Balancer
```

---

## 6. Capabilities

EKS capabilities provide additional AWS-managed functionality that can be enabled for the cluster.

Depending on the EKS version and AWS features enabled, capabilities can include AWS-managed functionality around:
- Compute
- Networking
- Storage
- Kubernetes management
- Cluster automation

> **Important Concept**
> Capabilities should be evaluated based on the workload requirement rather than enabling everything by default.

For example:

```
Application requires persistent storage
        ↓
EBS CSI

Application requires ALB
        ↓
AWS Load Balancer Controller

Application requires metrics
        ↓
CloudWatch / Prometheus
```

---

## 7. Access

The Access section controls how users and IAM identities authenticate and obtain access to the Kubernetes cluster.

Modern EKS access management can use EKS access entries.

Typical flow:

```
IAM User / IAM Role
        ↓
EKS Access Entry
        ↓
Kubernetes Access Policy
        ↓
Cluster permissions
```

### Important Concepts

- IAM authentication
- EKS access entries
- Kubernetes RBAC
- Access policies
- IAM roles
- `aws-auth` ConfigMap in older access models

### Example

A DevOps role could be granted administrative access:

```
GitLab IAM Role
      ↓
EKS Access Entry
      ↓
EKS/Kubernetes permissions
```

> **Interview Point**
> AWS IAM is primarily responsible for authentication, while Kubernetes RBAC and EKS access policies determine what the authenticated identity is allowed to do.

---

## 8. Observability

Observability provides visibility into the health and performance of the EKS environment.

Three major areas: **Metrics**, **Logs**, **Traces**.

### Metrics

Common tools:
- Prometheus
- Amazon CloudWatch
- Grafana

Monitor:
- CPU
- Memory
- Pod count
- Node utilization
- API server metrics
- Application metrics
- HPA metrics

### Logs

Common sources:
- Container logs
- Kubernetes logs
- EKS control-plane logs
- CloudWatch Logs
- Application logs
- ALB logs

### Monitoring Architecture

```
EKS
├── Application metrics
├── Node metrics
├── Kubernetes metrics
└── Control-plane logs
          ↓
    CloudWatch / Prometheus
          ↓
       Grafana
```

### Important EKS Control Plane Logs

Depending on what is enabled:
- API server
- Audit
- Authenticator
- Controller manager
- Scheduler

---

## 9. Update History & Backups

This section provides visibility into EKS cluster updates and backup-related operations.

### Update History

Useful for tracking:
- Kubernetes version upgrades
- Platform updates
- Configuration changes
- Add-on updates
- Cluster update status

Typical upgrade:

```
Current Kubernetes Version
        ↓
Check compatibility
        ↓
Upgrade EKS control plane
        ↓
Upgrade add-ons
        ↓
Upgrade node groups
        ↓
Validate workloads
```

### Production Upgrade Approach

Do not upgrade everything simultaneously without validation. A safer approach:

```
Control Plane
      ↓
Add-ons
      ↓
System Node Group
      ↓
Application Node Groups
      ↓
Application Validation
```

### Backups

Kubernetes application state should be considered separately from the EKS control plane.

```
Kubernetes configuration
        ↓
Git / GitOps

Persistent application data
        ↓
EBS / RDS / S3 backups

Cluster resources
        ↓
Infrastructure as Code
```

> Terraform should be used to recreate infrastructure rather than treating Terraform state itself as the application backup.

---

## 10. Certificate Authority

EKS provides a cluster certificate authority (CA) used to establish trust when communicating with the Kubernetes API server.

The Kubernetes API endpoint uses TLS:

```
kubectl
   ↓
HTTPS
   ↓
EKS API Server
   ↓
Certificate Authority
```

The CA certificate is required by Kubernetes clients to verify the identity of the API server. Terraform exposes it through the EKS cluster resource.

Example:

```hcl
resource "aws_eks_cluster" "main" {
  name = var.cluster_name
}
```

The cluster provides:
- `endpoint`
- `certificate_authority`

These can be consumed by tools such as:
- `kubectl`
- Terraform Kubernetes provider
- Helm provider
- Other Kubernetes clients

> **Important**
> The certificate authority is not the same as an application TLS certificate used by an ALB or Ingress.

```
EKS CA
└── Trust for Kubernetes API server

ACM Certificate
└── TLS for applications / ALB
```

---

## 11. Tags

Tags provide metadata for AWS resource identification, management and cost allocation.

Typical EKS tags:

```hcl
tags = {
  Name                   = "dev-app-cluster"
  Environment            = "dev"
  ManagedBy              = "Terraform"
  Terraform              = "true"
  karpenter.sh/discovery = "dev-app-cluster"
}
```

### Purpose

| Tag | Purpose |
|---|---|
| `Name` | Identifies the resource |
| `Environment` | Identifies Dev/Test/Prod |
| `ManagedBy` | Identifies the management tool |
| `Terraform` | Indicates Terraform-managed infrastructure |
| `karpenter.sh/discovery` | Used by Karpenter for cluster discovery |

Tags are useful for:
- Cost allocation
- Resource identification
- Automation
- Governance
- Environment separation
- Troubleshooting

---

## EKS Overall Architecture

```
                         AWS
                          |
                       VPC
                          |
          +---------------+---------------+
          |                               |
     Public Subnets                 Private Subnets
          |                               |
       ALB / NAT                    EKS Node Groups
                                          |
                         +----------------+----------------+
                         |                                 |
                  System Nodes                     Application Nodes
                         |                                 |
                  CoreDNS / CNI                    Application Pods
                  Controllers                            |
                         |                              Services
                         |                                 |
                         +---------------+-----------------+
                                         |
                                    VPC CNI / ENI
                                         |
                                      VPC
                                         |
                                  AWS Resources
                              RDS / S3 / EBS / etc.
```

## EKS Control Plane vs Worker Nodes

```
                EKS Cluster
                     |
          +----------+----------+
          |                     |
     Control Plane          Data Plane
          |                     |
   Managed by AWS         Managed by us
          |                     |
   API Server             Node Groups
   Scheduler              EC2 Instances
   Controller Manager     Kubernetes Pods
   etcd                    Applications
```

---

## Key Interview Statement

> EKS provides a managed Kubernetes control plane, while worker nodes or other compute capacity run the application workloads. The cluster integrates with AWS networking, IAM, storage, load balancing and observability services to provide a production Kubernetes platform.