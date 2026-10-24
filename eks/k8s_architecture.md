
# Kubernetes / EKS Architecture — Interview Preparation Guide

The most important thing to understand is how the control plane makes decisions, how worker nodes run Pods, and how networking, storage, and AWS integration work together.

## 1. Kubernetes Architecture Overview

### EKS Control Plane
*Managed by AWS*

| Component | Responsibility |
|---|---|
| `kube-apiserver` | API entry point |
| `etcd` | Cluster state store |
| `kube-scheduler` | Selects nodes for Pods |
| `kube-controller-manager` | Reconciles desired and actual state |

### Worker Nodes — Data Plane

| Component | Responsibility |
|---|---|
| `kubelet` | Manages Pods on the node |
| `containerd` | Runs containers |
| `kube-proxy` | Service networking rules |
| VPC CNI | Pod networking |
| Application Pods | Frontend, backend, monitoring |

### Supporting Components and AWS Integrations

| Component | Responsibility |
|---|---|
| CoreDNS | Kubernetes service discovery |
| EBS CSI Driver | Persistent block storage |
| EKS Pod Identity Agent | AWS credentials for authorized Pods |
| AWS Load Balancer Controller | Provisions and configures ALBs from Kubernetes resources |

## 2. Control Plane Components

### 1. kube-apiserver

**What it does:** The API server is the entry point to the Kubernetes API. `kubectl`, kubelets, controllers, and other clients communicate with the cluster through it.

- Authenticates and authorizes requests.
- Validates resource configurations.
- Handles API operations and coordinates persistence of cluster state in etcd.

**If it fails:** New management operations and control-plane reconciliation may be interrupted. Existing containers may continue running, but Kubernetes management and recovery capabilities are affected.

**Interview answer:**  
"The API server is the central gateway to the Kubernetes cluster. It validates requests and coordinates access to the cluster state stored in etcd."

### 2. etcd

**What it does:** A distributed key-value store that holds Kubernetes' persistent cluster state, including Deployments, Services, Secrets, ConfigMaps, and other API objects.

- Uses the Raft consensus algorithm.
- Requires a quorum for reliable cluster-state operations.
- In EKS, AWS manages the etcd infrastructure; customers do not administer it directly.

**If it fails:** The API server cannot reliably read or persist cluster state, impairing cluster management and reconciliation.

**Interview answer:**  
"etcd is the persistent store for Kubernetes cluster state. The API server uses it to read and write resource information."

### 3. kube-scheduler

**What it does:** Chooses an appropriate node for a newly created, unscheduled Pod.

It considers:

- Available CPU and memory.
- Node selectors and affinity.
- Taints and tolerations.
- Topology and other scheduling constraints.

**If it fails:** Newly created Pods may remain `Pending` without being assigned to a node. Already-running Pods can continue running.

**Interview answer:**  
"The scheduler selects a suitable node for each unscheduled Pod. It does not start the container; the kubelet does that."

### 4. kube-controller-manager

**What it does:** Runs controllers that continually compare the desired state with the observed state and take corrective action.

Examples:

- Deployment and ReplicaSet controllers maintain the requested number of replicas.
- Node controller monitors node health.
- EndpointSlice controller updates Service endpoint information.

**If it fails:** Reconciliation and automated recovery may be delayed. Existing Pods can continue running, but the cluster may not respond correctly to changes or failures.

**Interview answer:**  
"Controllers continuously reconcile the actual state with the desired state, helping Kubernetes maintain the configuration declared by the user."

### 5. Cloud Integration in EKS

Kubernetes cloud integration connects Kubernetes resources with AWS infrastructure.

A key distinction for interviews:

- **Cloud Controller Manager:** Provides cloud-provider integration for supported Kubernetes functions.
- **AWS Load Balancer Controller:** A separate controller that watches Kubernetes Ingress and Service resources and configures AWS load balancers.
- **EKS managed control plane:** AWS manages the underlying control-plane infrastructure.

**Interview answer:**  
"In EKS, AWS manages the control plane. The AWS Load Balancer Controller integrates Kubernetes routing resources with AWS ALBs, while cloud-provider integration handles its own supported responsibilities."

## 3. Worker Node Components

### 6. kubelet

The kubelet is the agent on each worker node.

- Watches the API server for Pods assigned to its node.
- Instructs the container runtime to create and manage containers.
- Executes configured health probes.
- Reports Pod and node status.

**If it fails:** The node may become `NotReady`, and Pod management on that node can be affected. Kubernetes may eventually replace affected workloads elsewhere, depending on health detection, capacity, and workload configuration.

**Interview answer:**  
"The kubelet ensures the Pods assigned to its node are running according to their specifications and reports their status to the control plane."

### 7. kube-proxy

In a typical EKS configuration, kube-proxy runs on each node and implements Kubernetes Service networking rules.

- Watches Services and EndpointSlices.
- Programs packet-forwarding rules, commonly using iptables or IPVS where configured.
- Enables traffic sent to a Service ClusterIP to reach the appropriate backend Pod.

**If it fails:** Existing rules may continue working, but new Service changes or endpoint updates may not be reflected correctly.

**Interview answer:**  
"kube-proxy configures node-level networking rules so traffic sent to a Kubernetes Service can reach its backend Pods."

**Note:** Some EKS clusters use alternative networking implementations, so kube-proxy is not universally required in every possible configuration.

### 8. Container Runtime — containerd

**What it does:** Runs containers on worker nodes.

- Pulls container images.
- Creates and starts containers.
- Stops containers and manages their lifecycle through the Container Runtime Interface (CRI).

**Interview answer:**  
"The kubelet manages the desired Pod state, while containerd performs the container runtime operations needed to run the application."

### 9. Pod

A Pod is the smallest deployable unit in Kubernetes.

- Contains one or more containers.
- Containers in the same Pod share its network namespace and IP address.
- Containers can share configured volumes.
- Pods are replaceable; their IP addresses can change when they are recreated.

**Interview answer:**  
"A Pod is the deployment unit in Kubernetes. It runs one or more closely related containers that share networking and can share storage volumes."

## 4. Important EKS Add-ons

| Component | Main responsibility | Useful command |
|---|---|---|
| Amazon VPC CNI (`aws-node`) | Provides Pod networking using VPC networking resources | `kubectl get pods -n kube-system -o wide` |
| CoreDNS | Resolves Kubernetes service names | `kubectl get pods -n kube-system -l k8s-app=kube-dns` |
| EBS CSI Driver | Provisions and attaches EBS-backed persistent volumes | `kubectl get pods -n kube-system \| grep ebs-csi` |
| EKS Pod Identity Agent | Enables supported Pods to obtain credentials for associated IAM roles | `kubectl get pods -n kube-system \| grep eks-pod-identity` |
| AWS Load Balancer Controller | Reconciles Ingress and supported Service resources with AWS load balancers | `kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100` |
| Metrics Server | Provides resource metrics used by `kubectl top` and CPU/memory-based HPA | `kubectl top nodes` |
| kube-proxy | Configures Service networking rules in typical deployments | `kubectl get pods -n kube-system -o wide` |

### Important Distinction: VPC CNI vs CoreDNS vs kube-proxy

- **VPC CNI:** Provides networking and IP addresses for Pods.
- **CoreDNS:** Resolves service names, such as `employee-service.dev.svc.cluster.local`.
- **kube-proxy:** Implements Service traffic-forwarding rules in a typical cluster networking setup.

These components solve different problems; they are not interchangeable.

## 5. How a Pod Gets Created — Real-Time Example

Suppose you deploy a React frontend to EKS.

1. You run `kubectl apply -f deployment.yaml`.
2. The API server authenticates, authorizes, and validates the request.
3. The desired state is persisted in the cluster state store.
4. The Deployment controller creates a ReplicaSet, which creates Pods.
5. The scheduler selects suitable worker nodes for unscheduled Pods.
6. The kubelet on each selected node instructs containerd to pull the image and start the containers.
7. The VPC CNI provides Pod networking, and readiness probes determine when the application is ready.
8. Services and EndpointSlices reflect the available backends. If an Ingress is configured, the AWS Load Balancer Controller configures the ALB routing.

## 6. Troubleshooting Commands for Interviews

### Control-plane connectivity and API access

```bash
kubectl cluster-info
kubectl get --raw='/readyz?verbose'
```

### Node health and versions

```bash
kubectl get nodes -o wide
kubectl describe node <node-name>
```

### Pod scheduling and status

```bash
kubectl get pods -A -o wide
kubectl describe pod <pod-name> -n <namespace>
```

### Service discovery and routing

```bash
kubectl get svc -A
kubectl get endpointslice -A
kubectl get ingress -A
```

### CPU and memory metrics

```bash
kubectl top nodes
kubectl top pods -A
```

### Recent events

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

## 7. One-Minute Interview Answer

"Kubernetes has two main parts: the control plane and the worker nodes. In EKS, AWS manages the control plane. The API server is the entry point to the Kubernetes API, and etcd stores the persistent cluster state. The scheduler selects a node for each unscheduled Pod, while controllers reconcile the actual state with the desired state.

On worker nodes, kubelet manages the Pods, and containerd runs their containers. In a typical configuration, kube-proxy implements Service networking rules, while the VPC CNI provides Pod networking and CoreDNS handles service discovery. For AWS integration, the EBS CSI driver manages persistent EBS storage, EKS Pod Identity provides IAM credentials to associated workloads, and the AWS Load Balancer Controller configures ALBs from Kubernetes resources.

When troubleshooting, I check node health, Pod status, events, Services, EndpointSlices, logs, and metrics to identify the failing component."

## 8. Quick Summary Table

| Component | Location | Main usage |
|---|---|---|
| `kube-apiserver` | Control plane | API entry point; authentication and validation; coordinates etcd access |
| `etcd` | Control plane | Stores persistent cluster state |
| `kube-scheduler` | Control plane | Chooses the node for an unscheduled Pod |
| `kube-controller-manager` | Control plane | Reconciliation loops: desired vs actual state |
| `cloud-controller-manager` | Control plane | Kubernetes-to-cloud integration |
| `kubelet` | Every worker node | Manages Pods and reports status |
| `kube-proxy` | Every worker node in typical configurations | Service ClusterIP to Pod IP forwarding through networking rules |
| `containerd` | Every worker node | Pulls images and runs containers |
| VPC CNI | Every worker node | Provides Pod networking and VPC IP addresses |
| CoreDNS | Cluster | Service-name resolution |
| EBS CSI Driver | Cluster | Persistent EBS storage |
| Pod Identity Agent | Every worker node where installed | Enables supported Pods to obtain associated IAM-role credentials |
