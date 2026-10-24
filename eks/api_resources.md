# Kubernetes API Resources — Quick Reference

## 1. Important API Versions

| Resource | `apiVersion` | `kind` |
|---|---|---|
| Pod | `v1` | `Pod` |
| Service | `v1` | `Service` |
| ConfigMap | `v1` | `ConfigMap` |
| Secret | `v1` | `Secret` |
| Namespace | `v1` | `Namespace` |
| PersistentVolume | `v1` | `PersistentVolume` |
| PersistentVolumeClaim | `v1` | `PersistentVolumeClaim` |
| ServiceAccount | `v1` | `ServiceAccount` |
| Deployment | `apps/v1` | `Deployment` |
| StatefulSet | `apps/v1` | `StatefulSet` |
| DaemonSet | `apps/v1` | `DaemonSet` |
| ReplicaSet | `apps/v1` | `ReplicaSet` |
| Job | `batch/v1` | `Job` |
| CronJob | `batch/v1` | `CronJob` |
| HorizontalPodAutoscaler | `autoscaling/v2` | `HorizontalPodAutoscaler` |
| Ingress | `networking.k8s.io/v1` | `Ingress` |
| NetworkPolicy | `networking.k8s.io/v1` | `NetworkPolicy` |
| StorageClass | `storage.k8s.io/v1` | `StorageClass` |
| Role | `rbac.authorization.k8s.io/v1` | `Role` |
| RoleBinding | `rbac.authorization.k8s.io/v1` | `RoleBinding` |
| ClusterRole | `rbac.authorization.k8s.io/v1` | `ClusterRole` |
| ClusterRoleBinding | `rbac.authorization.k8s.io/v1` | `ClusterRoleBinding` |
| PodDisruptionBudget | `policy/v1` | `PodDisruptionBudget` |
| PriorityClass | `scheduling.k8s.io/v1` | `PriorityClass` |
| CustomResourceDefinition | `apiextensions.k8s.io/v1` | `CustomResourceDefinition` |

## 2. API Groups to Remember

### Core resources — `v1`
- Pod
- Service
- ConfigMap
- Secret
- Namespace
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- ServiceAccount

### Workloads — `apps/v1`
- Deployment
- StatefulSet
- DaemonSet
- ReplicaSet

### Batch workloads — `batch/v1`
- Job
- CronJob

### Networking — `networking.k8s.io/v1`
- Ingress
- NetworkPolicy

### RBAC — `rbac.authorization.k8s.io/v1`
- Role
- RoleBinding
- ClusterRole
- ClusterRoleBinding

### Autoscaling — `autoscaling/v2`
- HorizontalPodAutoscaler (HPA)

### Storage — `storage.k8s.io/v1`
- StorageClass

### Policy — `policy/v1`
- PodDisruptionBudget

### Scheduling — `scheduling.k8s.io/v1`
- PriorityClass

### API extensions — `apiextensions.k8s.io/v1`
- CustomResourceDefinition (CRD)

## 3. Example YAML Manifests

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

## 4. Commands to Verify API Versions

```bash
# List available API versions
kubectl api-versions

# List available resources and their API groups
kubectl api-resources

# View the Deployment schema
kubectl explain deployment

# View the Ingress schema
kubectl explain ingress

# View the HPA schema
kubectl explain hpa

# View a resource's API version and fields
kubectl explain deployment.apiVersion
```

## 5. Interview Quick Memory

Memorize these six API versions first:

1. `v1` — Pod, Service, ConfigMap, Secret, PV, PVC
2. `apps/v1` — Deployment, StatefulSet, DaemonSet, ReplicaSet
3. `batch/v1` — Job, CronJob
4. `networking.k8s.io/v1` — Ingress, NetworkPolicy
5. `rbac.authorization.k8s.io/v1` — Role, RoleBinding, ClusterRole, ClusterRoleBinding
6. `autoscaling/v2` — HorizontalPodAutoscaler (HPA)

**Remember:** `apiVersion` identifies the API group and version; `kind` identifies the Kubernetes resource type.
