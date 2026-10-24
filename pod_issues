# Pod Issues: Causes, Identification and Fixes

## Contents

1. [First 3 commands for any pod issue](#1-first-3-commands-for-any-pod-issue)
2. [Pod lifecycle: where each issue appears](#2-pod-lifecycle-where-each-issue-appears)
3. [Pending](#3-pending-pod-not-scheduled)
4. [ContainerCreating](#4-containercreating-scheduled-but-not-started)
5. [ImagePullBackOff / ErrImagePull](#5-imagepullbackoff--errimagepull)
6. [Init container issues](#6-init-container-issues)
7. [CrashLoopBackOff](#7-crashloopbackoff)
8. [OOMKilled](#8-oomkilled)
9. [Container config and create errors](#9-container-config-and-create-errors)
10. [Running but not Ready](#10-running-but-not-ready-01)
11. [Running and Ready but no traffic](#11-running-and-ready-but-no-traffic)
12. [Evicted](#12-evicted)
13. [Terminating stuck](#13-terminating-stuck)
14. [Unknown / NodeLost / NotReady node](#14-unknown--node-notready)
15. [Pods not created at all (ReplicaSet / quota / admission)](#15-pods-not-created-at-all)
16. [Exit code cheat sheet](#16-exit-code-cheat-sheet)
17. [Status to first-check quick map](#17-status--first-check-quick-map)

---

## 1. First 3 Commands for Any Pod Issue

```bash
kubectl get pods -n <ns> -o wide                      # status, node, IP, restarts
kubectl describe pod <pod> -n <ns>                    # Events section at the bottom is the key
kubectl logs <pod> -n <ns> [-c <container>] [--previous]
```

> **Rule:** `describe` tells you **why Kubernetes could not start it**. `logs` tells you **why the app failed after it started**.

---

## 2. Pod Lifecycle: Where Each Issue Appears

```
 Pod created
    │
    ▼
 Pending ─────────────► scheduler can't place it (CPU/mem, taints, PVC, affinity)
    │
    ▼  (node assigned)
 ContainerCreating ───► no pod IP (CNI), volume attach/mount, missing Secret/ConfigMap, image pulling
    │
    ▼
 Init:0/1 ────────────► init container running / failing
    │
    ▼
 PodInitializing
    │
    ▼
 Running ─────────────► CrashLoopBackOff, OOMKilled, probe failures (0/1 Ready)
    │
    ▼
 Terminating / Completed / Evicted
```

---

## 3. Pending (Pod Not Scheduled)

The pod has **no node assigned**. Check `Events` in `kubectl describe pod` for `FailedScheduling`.

| Cause | Event message / how to identify | Fix |
|-------|--------------------------------|-----|
| **Insufficient CPU** | `0/3 nodes are available: 3 Insufficient cpu` | Lower CPU `requests`, add nodes, enable Cluster Autoscaler/Karpenter |
| **Insufficient memory** | `... 3 Insufficient memory` | Lower memory `requests`, add bigger/more nodes |
| **Requests larger than any node** | Pod requests more than the biggest node's allocatable (`kubectl describe node`) | Reduce requests or use a larger instance type |
| **Too many pods on node** | `Too many pods` | Raise max pods (prefix delegation), add nodes |
| **Taint not tolerated** | `node(s) had untolerated taint {key: value}` | Add a `toleration`, or remove the taint, or use another node group |
| **nodeSelector / label mismatch** | `node(s) didn't match Pod's node affinity/selector` | Fix label or selector (`kubectl get nodes --show-labels`) |
| **Node affinity rule too strict** | Same as above | Relax to `preferredDuringScheduling` |
| **Pod anti-affinity** | `node(s) didn't match pod anti-affinity rules` | Add nodes, relax the rule, use `topologySpreadConstraints` |
| **PVC not bound** | `pod has unbound immediate PersistentVolumeClaims` | Check `kubectl get pvc`, StorageClass exists, EBS CSI healthy, provisioner permissions |
| **Volume AZ conflict** | `node(s) had volume node affinity conflict` | EBS is single-AZ: schedule in that AZ, use `volumeBindingMode: WaitForFirstConsumer` |
| **Node not Ready or unschedulable (cordoned)** | `node(s) were unschedulable` | `kubectl uncordon <node>` and fix node health |
| **ResourceQuota / LimitRange exceeded** | `exceeded quota` in events | Increase quota or reduce requests |
| **Host port conflict** | `node(s) didn't have free ports` | Stop using `hostPort`, use a Service |
| **Missing PriorityClass or preemption pending** | `preemption: ... no preemption victims found` | Fix priority, add capacity |
| **Cluster Autoscaler / Karpenter not scaling** | Pod Pending but no new node appears | Check autoscaler logs, ASG max size, instance quota, subnet IPs, taints on node group |
| **Scheduler down (rare on EKS)** | Pod Pending with **no** events at all | Check cluster health, AWS Health dashboard |

```bash
kubectl describe pod <pod> -n <ns> | grep -A10 Events
kubectl describe node <node> | grep -A8 "Allocated resources"
kubectl get nodes --show-labels
kubectl describe node <node> | grep Taints
```

---

## 4. ContainerCreating (Scheduled but Not Started)

Node is assigned, but the pod sandbox or containers can't be set up.

| Cause | Event message / how to identify | Fix |
|-------|--------------------------------|-----|
| **No pod IPs available (VPC CNI)** | `failed to assign an IP address to container` / `FailedCreatePodSandBox` | Subnet exhausted: enable **prefix delegation**, use a secondary CIDR (custom networking), add bigger subnets, tune `WARM_IP_TARGET` |
| **Node hit max pods (ENI limit)** | Same error, node already has many pods | Larger instance type, prefix delegation, more nodes |
| **`aws-node` (CNI) pod unhealthy** | `kubectl get pods -n kube-system -l k8s-app=aws-node`; logs show errors | Check `aws-node` logs, IAM permissions on node role, update the add-on |
| **CNI cannot reach API / ENI attach fails** | `NetworkPluginNotReady` or ENI errors in `ipamd` logs | Fix node IAM policy (`AmazonEKS_CNI_Policy`), check EC2 ENI limits |
| **Secret not found** | `MountVolume.SetUp failed ... secret "x" not found` | Create the Secret in the **same namespace**, fix the name |
| **ConfigMap not found** | `configmap "x" not found` | Create the ConfigMap in the same namespace |
| **EBS volume attach fails** | `AttachVolume.Attach failed` | Check EBS CSI controller logs and its IAM role (Pod Identity), volume state in EC2 |
| **Volume stuck attached to another node** | `Multi-Attach error ... already used by pod` / `volume is already attached` | Old pod must terminate first; use RollingUpdate with `Recreate` strategy for RWO volumes, or use EFS for RWX |
| **EBS volume in a different AZ** | Mount timeout, volume can't attach | Schedule pod in the volume's AZ |
| **EBS attach limit per instance reached** | `AttachVolume.Attach failed ... too many volumes` | Spread pods across nodes, use larger instance types |
| **Mount timeout** | `Unable to attach or mount volumes: timed out waiting for the condition` | Check CSI node plugin pod on that node, security groups, EFS mount targets |
| **EFS mount failure** | `mount.nfs: Connection timed out` | Open NFS port 2049 in security groups, create mount targets in each AZ |
| **Image still pulling (large image)** | Events show `Pulling image` for a long time | Smaller image, image caching, regional ECR |
| **Pod sandbox creation failure (runtime)** | `Failed to create pod sandbox` | Check containerd/kubelet logs on the node, disk space, node health |
| **Node disk full / inode exhaustion** | Node condition `DiskPressure` | Clean images/logs, bigger root volume |
| **Security Groups for Pods misconfig** | Branch ENI not available | Check `ENABLE_POD_ENI`, SecurityGroupPolicy, instance type support |

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node -o wide
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100
kubectl get pvc,pv -n <ns>
kubectl logs -n kube-system -l app=ebs-csi-controller -c ebs-plugin --tail=100
```

---

## 5. ImagePullBackOff / ErrImagePull

`ErrImagePull` is the first failure. `ImagePullBackOff` means Kubernetes is waiting between retries.

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Wrong image name or typo** | `repository does not exist or may require authorization` | Correct the image name |
| **Tag does not exist** | `manifest unknown` / `not found` | Use a valid tag, avoid `latest` surprises |
| **Private registry, no credentials** | `pull access denied` / `unauthorized` | Create `imagePullSecret` and reference it in the pod or ServiceAccount |
| **ECR: node role lacks permission** | `no basic auth credentials` | Attach `AmazonEC2ContainerRegistryReadOnly` to the node role |
| **ECR in different account/region** | Access denied | Add ECR repository policy for cross-account, use correct region |
| **No network path to registry** | `i/o timeout` / `dial tcp ... timeout` | Private subnets need **NAT Gateway** or VPC endpoints |
| **ECR VPC endpoints incomplete** | Timeouts only in private clusters | Create `ecr.api`, `ecr.dkr` interface endpoints and the **S3 gateway endpoint** (image layers come from S3) |
| **Docker Hub rate limit** | `toomanyrequests: You have reached your pull rate limit` | Use ECR (pull-through cache), authenticate to Docker Hub |
| **Wrong CPU architecture** | Pull succeeds, then `exec format error` | Build multi-arch images (amd64 vs arm64 / Graviton) |
| **`imagePullPolicy: Never` and image absent** | `ErrImageNeverPull` | Use `IfNotPresent` or `Always` |
| **Invalid image reference** | `InvalidImageName` | Fix the format (`registry/repo:tag`) |
| **Image too large / slow pull** | Timeouts, long `Pulling image` | Slim the image, multi-stage builds, increase node bandwidth |
| **Security group / NACL blocks egress 443** | Timeouts | Allow outbound HTTPS from nodes |

```bash
kubectl describe pod <pod> -n <ns> | grep -A8 Events
aws ecr describe-images --repository-name <repo>
```

---

## 6. Init Container Issues

Init containers run **in order, one at a time, and must all succeed** before the main containers start.

| Status | Cause | How to identify | Fix |
|--------|-------|-----------------|-----|
| `Init:0/1` (stuck) | Init container waiting on a dependency (DB, service) that is not ready | `kubectl logs <pod> -c <init-container>` shows retry/wait loop | Fix the dependency, correct the host/port, add a timeout |
| `Init:0/1` (stuck) | Init container waiting on DNS that fails | Logs show `could not resolve host` | Check CoreDNS, Service name and namespace |
| `Init:Error` | Init command exited non-zero | `kubectl describe pod` → init container `Exit Code` | Fix the script/command |
| `Init:CrashLoopBackOff` | Init container keeps failing and restarting | `kubectl logs <pod> -c <init> --previous` | Fix the script, wrong args, missing permissions |
| `Init:ImagePullBackOff` | Init image can't be pulled | Events | Same as section 5 |
| `Init:CreateContainerConfigError` | Missing Secret/ConfigMap for the init container | Events | Create the missing object |
| `Init:OOMKilled` | Init container exceeded memory limit | `Reason: OOMKilled` | Raise init container limits |
| `Init:Error` (permission) | Cannot write to volume (`chown`/`chmod` fails) | Logs show `permission denied` | Set `securityContext.fsGroup`, run as proper user |
| Stuck at `PodInitializing` | Main container image pulling or volume mount in progress | Events | See sections 4 and 5 |
| Init never finishes (restartPolicy) | Init container is a long-running process | It never exits | Init containers must **exit**; move long-running work to the main container or a sidecar |

```bash
kubectl describe pod <pod> -n <ns>                  # look at the "Init Containers:" block
kubectl logs <pod> -n <ns> -c <init-container-name>
kubectl logs <pod> -n <ns> -c <init-container-name> --previous
```

---

## 7. CrashLoopBackOff

The container **starts, then exits**, repeatedly. Kubernetes restarts it with growing delays (10s, 20s, 40s, up to 5 min).

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Application error / exception on startup** | `kubectl logs --previous` shows stack trace | Fix the code or config |
| **Missing or wrong environment variable** | Logs: `env var X is required` | Add the variable, check ConfigMap/Secret keys |
| **Cannot connect to dependency (DB, cache, API)** | Logs: `connection refused` / `timeout` | Verify Service name, port, DNS, security groups, credentials |
| **Wrong command or args** | Exit code `127` (command not found) or `126` (not executable) | Fix `command` / `args`, check the Dockerfile `ENTRYPOINT` |
| **Main process exits immediately** | Exit code `0` but pod keeps restarting | Container needs a foreground process; for jobs use `Job` or `restartPolicy: OnFailure` |
| **OOMKilled** | `Last State: Terminated, Reason: OOMKilled`, exit `137` | See section 8 |
| **Liveness probe failing** | Events: `Liveness probe failed ... Killing container` | Increase `initialDelaySeconds`, add a `startupProbe`, fix probe path/port |
| **Wrong container port or bind address** | App binds to `127.0.0.1` instead of `0.0.0.0` | Bind to `0.0.0.0` |
| **Permission denied (non-root, read-only FS)** | Logs: `permission denied` | Fix `securityContext`, `fsGroup`, writable `emptyDir` |
| **Config file missing / bad mount** | `no such file or directory` | Check volume mounts and `subPath` |
| **Wrong CPU architecture** | `exec format error` | Rebuild image for the correct architecture |
| **Bad image / entrypoint script has CRLF line endings** | `/bin/sh^M: bad interpreter` | Convert scripts to LF |
| **Secrets/IAM missing at startup (AWS SDK)** | `AccessDenied` / `NoCredentialProviders` | Fix Pod Identity association or IRSA, check ServiceAccount |
| **Segmentation fault** | Exit code `139` | Fix the app or native library |
| **Crash after SIGTERM misconfig** | Exit code `143` repeatedly | Check shutdown handling |
| **Dependency not ready (race at startup)** | Fails first, works later | Add retries, readiness gating, init container |

```bash
kubectl logs <pod> -n <ns> --previous
kubectl describe pod <pod> -n <ns> | grep -A12 "Last State"
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.containerStatuses[*].lastState}'
```

---

## 8. OOMKilled

Kernel killed the container for exceeding its **memory limit** (exit code **137**).

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Memory limit too low** | `Reason: OOMKilled` in `Last State`; `kubectl top pod` near the limit | Raise `limits.memory` and `requests.memory` |
| **Memory leak in app** | Memory grows steadily until kill | Profile and fix the leak, add restarts as a stopgap |
| **JVM heap larger than container limit** | Java app killed even with a "big enough" heap | Use `-XX:MaxRAMPercentage`, leave headroom for non-heap memory |
| **Traffic spike or large payloads** | Kills coincide with load | Scale with HPA, add limits per request, raise memory |
| **Node-level memory pressure** | Pod evicted or node `MemoryPressure=True` | Fix node sizing, set right requests so scheduler packs correctly |
| **Requests far below real usage** | Many pods packed on one node, then OOM | Set requests close to actual usage |

> Remember: **CPU limit = throttling (slow)**. **Memory limit = kill (OOMKilled)**.

---

## 9. Container Config and Create Errors

| Status / error | Cause | How to identify | Fix |
|----------------|-------|-----------------|-----|
| `CreateContainerConfigError` | Secret or ConfigMap referenced by `env`/`envFrom` doesn't exist, or key missing | Events: `secret "x" not found` / `couldn't find key y` | Create the object, fix the key name, same namespace |
| `CreateContainerConfigError` | `runAsNonRoot: true` but image runs as root | `container has runAsNonRoot and image will run as root` | Set `runAsUser` or fix the image's `USER` |
| `CreateContainerError` | Invalid command/entrypoint, or mount issue | Events show runtime error | Fix `command`, volume mounts, `subPath` |
| `RunContainerError` | Runtime cannot start container (bad entrypoint, permissions) | Events / kubelet logs | Fix the image or securityContext |
| `ErrImageNeverPull` | `imagePullPolicy: Never` with no local image | Events | Change the pull policy |
| `InvalidImageName` | Malformed image string | Events | Fix the image reference |
| `ServiceAccount not found` | Pod references a missing ServiceAccount | `error looking up service account` (at ReplicaSet `FailedCreate`) | Create the ServiceAccount |
| `Error` | Container exited non-zero (one-off Job/Pod) | `kubectl logs`, exit code | Fix the app |
| `Completed` | Container exited 0 (normal for Jobs) | Expected for Jobs | For Deployments, the main process must stay in the foreground |
| `DeadlineExceeded` | Job/Pod ran longer than `activeDeadlineSeconds` | `kubectl describe` | Increase deadline or fix the slowness |

---

## 10. Running but Not Ready (0/1)

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Readiness probe failing** | Events: `Readiness probe failed: HTTP probe failed with statuscode: 404/500` | Fix probe path, port, scheme (HTTP vs HTTPS) |
| **Probe port or path wrong** | `connection refused` in event | Match the port the app actually listens on |
| **App slow to start** | Probe fails only at startup | Add `startupProbe` or raise `initialDelaySeconds` |
| **Probe timeout too short** | `context deadline exceeded` | Increase `timeoutSeconds`, `failureThreshold` |
| **Dependency (DB) down, health endpoint checks it** | App returns 503 | Fix dependency or make readiness check shallow |
| **Readiness gate not satisfied (ALB)** | Condition `target-health...` is `False` | Check ALB target health, security groups, health check path |
| **Sidecar not ready** | One container of 2 not ready (`1/2`) | `kubectl describe pod`, check the failing container |

> **Readiness failure** = removed from Service endpoints, **no restart**.
> **Liveness failure** = container **restarted**.

---

## 11. Running and Ready but No Traffic

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Service selector doesn't match pod labels** | `kubectl get endpointslices` is empty | Align selector with `kubectl get pods --show-labels` |
| **Wrong `targetPort`** | Endpoints exist but connection refused | Match the container's actual port |
| **App binds to localhost** | `curl` to pod IP fails, `localhost` works inside pod | Bind to `0.0.0.0` |
| **NetworkPolicy blocking** | Traffic works without policies | Review `kubectl get networkpolicy`, allow the source |
| **Security group blocks ALB → pod** | Target `unhealthy`, `Request timed out` | Allow ALB SG to pod/node SG on the app port |
| **Wrong ALB health check path/port** | Target `unhealthy`, `Health checks failed` | Set `healthcheck-path` and port annotations |
| **Ingress/IngressClass wrong** | No ALB address on the Ingress | Check `ingressClassName`, controller logs |
| **kube-proxy problem** | ClusterIP unreachable, pod IP reachable | Check kube-proxy pods and logs on the node |
| **CoreDNS problem** | Name lookup fails, IP works | Check CoreDNS pods and `nslookup` from a debug pod |
| **Pod terminating but still in ALB** | 502s during rollout | `preStop` sleep, deregistration delay, readiness gates |

---

## 12. Evicted

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Node memory pressure** | Status `Evicted`, message `The node was low on resource: memory` | Right-size requests/limits, add nodes |
| **Node disk pressure (`nodefs`/`imagefs`)** | `low on resource: ephemeral-storage` | Larger root volume, cleanup logs/images, set ephemeral-storage limits |
| **Pod exceeded ephemeral storage limit** | `Pod ephemeral local storage usage exceeds the total limit` | Reduce writes to container FS, use volumes, raise limit |
| **PID pressure** | Node condition `PIDPressure` | Fix process leaks, raise limits |
| **Node-pressure with BestEffort pods** | BestEffort pods evicted first | Set requests/limits to get Burstable or Guaranteed QoS |
| **Cleanup** | Many `Evicted` pods listed | `kubectl delete pod --field-selector=status.phase=Failed -n <ns>` |

> **Eviction order:** BestEffort → Burstable (exceeding requests) → Guaranteed (last).

---

## 13. Terminating Stuck

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Finalizer not removed** | `metadata.finalizers` present in `kubectl get pod -o yaml` | Fix the owning controller, then as last resort patch out the finalizer |
| **App ignores SIGTERM** | Waits the full `terminationGracePeriodSeconds`, then SIGKILL | Handle SIGTERM, use `exec` form in entrypoint |
| **Long `preStop` hook** | Pod stays Terminating for the hook duration | Shorten the hook or the grace period |
| **Volume cannot detach/unmount** | CSI events, `Unable to detach` | Check EBS CSI controller and node plugin |
| **Node is NotReady/unreachable** | Pod stuck while node is down | Node recovers or is removed, then force delete |
| **Force delete (last resort)** | | `kubectl delete pod <pod> -n <ns> --grace-period=0 --force` |

---

## 14. Unknown / Node NotReady

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **Node lost (EC2 stopped, crashed, network)** | `kubectl get nodes` shows `NotReady`; pods `Unknown` | Check EC2 status, replace the node, ASG will recreate |
| **kubelet down** | `describe node` shows `Kubelet stopped posting node status` | Restart kubelet, check logs through SSM |
| **Node cannot reach API server** | Heartbeats stop | Check security groups, route tables, endpoint access (public/private) |
| **CNI not ready** | Node condition `NetworkUnavailable` / `NetworkPluginNotReady` | Check `aws-node` pod on that node |
| **DiskPressure / MemoryPressure / PIDPressure** | `kubectl describe node` conditions | Free resources or resize |
| **Node role not authorized** | Node can't join: `Unauthorized` | Fix Access Entry or `aws-auth` for the node role |
| **Pods rescheduling after node failure** | Normal after the eviction timeout (~5 min by default) | Wait, or cordon and drain |

```bash
kubectl get nodes
kubectl describe node <node> | grep -A10 Conditions
```

---

## 15. Pods Not Created at All

No pod appears. Check the **ReplicaSet/Deployment events**, not the pod.

| Cause | How to identify | Fix |
|-------|-----------------|-----|
| **ResourceQuota exceeded** | `kubectl describe rs <rs>` → `FailedCreate: exceeded quota` | Raise quota or free resources |
| **LimitRange violation** | `FailedCreate: ... maximum/minimum cpu/memory` | Adjust requests/limits |
| **Admission webhook rejection** | `FailedCreate: admission webhook "x" denied the request` | Fix the manifest, or the webhook policy (Kyverno/OPA/Gatekeeper) |
| **Pod Security Standards violation** | `violates PodSecurity "restricted"` | Add required `securityContext` fields |
| **ServiceAccount missing** | `serviceaccount "x" not found` | Create it |
| **Webhook unreachable** | `failed calling webhook ... connection refused/timeout` | Fix the webhook service, or its `failurePolicy` |
| **Invalid manifest** | `kubectl apply` returns validation errors | Fix YAML, `kubectl apply --dry-run=server` |
| **Deployment paused or replicas = 0** | `kubectl rollout status`, `kubectl get deploy` | Resume, scale up |
| **HPA scaled to min 0 or stuck** | `kubectl describe hpa` | Fix metrics, min replicas |

```bash
kubectl describe rs <replicaset> -n <ns>
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

---

## 16. Exit Code Cheat Sheet

| Exit code | Meaning | Typical cause |
|-----------|---------|---------------|
| `0` | Success | Process finished (fine for Jobs, wrong for long-running apps) |
| `1` | General application error | Unhandled exception, bad config |
| `2` | Misuse of shell command | Bad script syntax |
| `126` | Command cannot execute | Not executable, permission issue |
| `127` | Command not found | Wrong `command`/`args`, missing binary |
| `137` | **SIGKILL** (128+9) | **OOMKilled**, or liveness kill, or grace period expired |
| `139` | SIGSEGV (128+11) | Segmentation fault |
| `143` | SIGTERM (128+15) | Graceful termination signal received |

---

## 17. Status → First-Check Quick Map

| Pod status | Think first | First command |
|------------|-------------|---------------|
| `Pending` | Scheduling: resources, taints, PVC, affinity | `kubectl describe pod` → Events |
| `ContainerCreating` | CNI (IPs), volumes, Secret/ConfigMap | `kubectl describe pod` → Events, `aws-node` logs |
| `ImagePullBackOff` / `ErrImagePull` | Image name/tag, registry auth, network to registry | `kubectl describe pod` → Events |
| `Init:*` | Init container logs and exit code | `kubectl logs <pod> -c <init>` |
| `CrashLoopBackOff` | App crash, exit code, probes | `kubectl logs --previous` |
| `OOMKilled` | Memory limit vs real usage | `kubectl describe pod` → Last State, `kubectl top` |
| `CreateContainerConfigError` | Missing Secret/ConfigMap/key, securityContext | `kubectl describe pod` |
| `Running 0/1` | Readiness probe | `kubectl describe pod` → probe events |
| `Evicted` | Node pressure | `kubectl describe node` |
| `Terminating` (stuck) | Finalizers, SIGTERM handling, volume detach | `kubectl get pod -o yaml` |
| `Unknown` | Node NotReady | `kubectl get nodes` |
| No pods created | Quota, webhook, PSA | `kubectl describe rs` |

### One-line interview answer

> "I start with `kubectl describe pod` and read the Events. If the pod is **Pending**, it's a scheduling issue: resources, taints, affinity, or an unbound PVC. If it's **ContainerCreating**, I look at the CNI for IP exhaustion, volume attach, or missing Secrets. If it's **ImagePullBackOff**, I check the image, registry auth and network path. If it's **CrashLoopBackOff**, I read `logs --previous` and the exit code: 137 means OOMKilled, 127 means a bad command. For **Init** states I read the init container logs. For **Running 0/1**, it's the readiness probe. And if the pod isn't created at all, I check the ReplicaSet events for quota or admission webhook failures."