
# AWS EKS Add-ons and Their Uses — Interview Preparation

## 1. Core EKS Add-ons

### 1. Amazon VPC CNI (`aws-node`)

**Usage:** Provides Pod networking and assigns VPC IP addresses to Pods.

- Pods typically receive IP addresses from the VPC subnet CIDR ranges.
- Uses Elastic Network Interfaces (ENIs) and available IP addresses on EC2 worker nodes.
- IP exhaustion can prevent new Pods from starting.

**Interview answer:**  
"VPC CNI provides Pod networking and assigns VPC IP addresses to Pods."

### 2. Amazon EBS CSI Driver

**Usage:** Integrates Kubernetes with Amazon EBS for persistent block storage.

- Provisions, attaches, and mounts EBS volumes for Pods.
- Works with PersistentVolumes (PV) and PersistentVolumeClaims (PVC).
- Allows data to persist beyond an individual container's lifecycle.
- EBS volumes are normally attached to EC2 instances in the same Availability Zone.

**Interview answer:**  
"EBS CSI integrates Kubernetes storage with AWS EBS to provide persistent block storage."

### 3. EKS Pod Identity Agent

**Usage:** Enables supported Pods to obtain temporary AWS credentials for an associated IAM role.

- Used with an EKS Pod Identity association.
- Allows applications to access authorized AWS services such as S3, SQS, and Secrets Manager.
- Avoids storing long-lived AWS access keys in application code.

**Interview answer:**  
"Pod Identity securely provides IAM role credentials to Kubernetes workloads that need AWS access."

### 4. CoreDNS

**Usage:** Provides DNS-based service discovery inside Kubernetes.

- Resolves Kubernetes Service names into IP addresses.
- Allows frontend Pods to discover backend Services.
- Example: `backend-service.dev.svc.cluster.local`.
- DNS failures can prevent applications from discovering other services.

**Interview answer:**  
"CoreDNS resolves Kubernetes Service names so applications can communicate using DNS names instead of hardcoded IP addresses."

### 5. kube-proxy

**Usage:** Implements Kubernetes Service traffic-forwarding rules in a typical EKS configuration.

- Routes Service ClusterIP traffic to eligible backend Pods.
- Uses networking rules such as iptables or IPVS, depending on configuration.
- Helps distribute connections across Service backends.

**Interview answer:**  
"kube-proxy implements Service networking rules so traffic can reach backend Pods."

### 6. AWS Load Balancer Controller

**Usage:** Creates and configures AWS load balancers based on supported Kubernetes resources.

- Watches Ingress and supported Service resources.
- Creates and configures Application Load Balancers (ALBs) for HTTP/HTTPS routing.
- Can configure Network Load Balancers (NLBs) for supported Service configurations.

**Interview answer:**  
"The AWS Load Balancer Controller integrates Kubernetes Ingress and Service resources with AWS load balancers."

### 7. Metrics Server

**Usage:** Provides resource usage metrics, mainly CPU and memory, for Kubernetes workloads and nodes.

- Enables `kubectl top nodes` and `kubectl top pods`.
- Provides resource metrics for CPU/memory-based Horizontal Pod Autoscaler (HPA).
- Does not act as a long-term monitoring or historical metrics database.

**Interview answer:**  
"Metrics Server provides resource metrics for kubectl top and resource-based HPA scaling."

---

## 2. Other Important Components Used in EKS

| Component | Main usage |
|---|---|
| Amazon CloudWatch Observability add-on | Sends supported EKS logs and metrics to CloudWatch and supports Container Insights. |
| IAM OIDC Provider / IRSA | Allows Kubernetes ServiceAccounts to assume IAM roles through OpenID Connect. |
| External Secrets Operator | Retrieves secrets from systems such as AWS Secrets Manager and synchronizes them into Kubernetes Secrets. |
| Argo CD | Implements GitOps continuous delivery by reconciling Kubernetes deployments with desired configuration in Git. |
| Prometheus | Collects and stores time-series metrics for monitoring and alerting. |
| Grafana | Displays monitoring metrics in dashboards, commonly using Prometheus as a data source. |
| Cluster Autoscaler | Adjusts configured node groups when Pods cannot be scheduled or nodes are underutilized. |
| Karpenter | Provisions and consolidates worker nodes based on pending Pod requirements and configured constraints. |
| Amazon ECR integration | Allows nodes to retrieve container images from ECR using appropriate IAM permissions and network access. |

**Note:** Some components are EKS-managed add-ons; others are separately installed controllers, agents, or applications. They are not all installed by default.

---

## 3. Quick Revision — Remember These Seven

| Component | Remember it as |
|---|---|
| VPC CNI | Provides Pod networking and VPC IP addresses |
| EBS CSI Driver | Persistent EBS storage for Pods |
| Pod Identity Agent | Enables temporary AWS credentials for authorized Pods |
| CoreDNS | Resolves Kubernetes Service names |
| kube-proxy | Implements Service traffic forwarding |
| AWS Load Balancer Controller | Creates and configures ALBs and supported NLBs |
| Metrics Server | CPU/memory metrics for `kubectl top` and HPA |

---

## 4. Commands to Check These Components

### Check system components

```bash
kubectl get pods -n kube-system
```

### Check VPC CNI

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

### Check CoreDNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### Check EBS CSI Driver

```bash
kubectl get pods -n kube-system | grep ebs-csi
```

### Check Pod Identity Agent

```bash
kubectl get pods -n kube-system | grep eks-pod-identity
```

### Check AWS Load Balancer Controller

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Check Metrics Server

```bash
kubectl top nodes
kubectl top pods -A
```

---

## 5. One-Minute Interview Answer

"In EKS, the VPC CNI provides Pod networking and VPC IP addresses, CoreDNS handles DNS resolution, and kube-proxy implements Service routing in typical configurations. The EBS CSI driver provides persistent storage, while EKS Pod Identity provides temporary AWS credentials to authorized workloads. The AWS Load Balancer Controller integrates Kubernetes resources with AWS load balancers, and Metrics Server provides CPU and memory metrics for HPA scaling.

For monitoring and delivery, I use Prometheus and Grafana for observability, Argo CD for GitOps, and Cluster Autoscaler or Karpenter for node scaling."

