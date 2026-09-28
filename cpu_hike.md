# EKS Interview Prep: Production Node Running Out of CPU

> **Interview question:** *"A production EKS node is running out of CPU. What will you do?"*

---

## 1. TL;DR: The Answer in One Line

**Diagnose → Protect production → Scale/Mitigate → Find root cause → Prevent recurrence**

| Phase | What you do |
|-------|-------------|
| **Diagnose** | Find which node, which pods, and whether pods are actually Pending |
| **Protect** | Add capacity / scale / roll back so users aren't impacted |
| **Mitigate** | Take the *least disruptive* action (never a blind reboot) |
| **Root cause** | Bad release? Traffic spike? Bad requests/limits? Runaway process? |
| **Prevent** | Right-size resources, tune HPA, add node autoscaling, add alerts |

---

## 2. Key Concept: CPU Utilization vs CPU Requests

This is the most important distinction in the whole answer.

| Term | Meaning |
|------|---------|
| **CPU utilization** | How much CPU is *actually being used* right now |
| **CPU requests** | How much CPU pods have *reserved*. The scheduler uses this to decide if a new pod fits |

> A node can show **90% utilization** and still accept pods, or show **low utilization** and reject pods because its *requests* are already fully allocated.

**Always check both:** `actual CPU usage` + `CPU requests/limits`

---

## 3. Step-by-Step Troubleshooting

### Step 1: Confirm the problem

```bash
kubectl top nodes
kubectl top pods -A --sort-by=cpu
```

**Ask yourself:** Is the CPU being consumed by **application pods**, **system pods**, or **something running directly on the node**?

### Step 2: Check if pods are failing to schedule

```bash
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pending-pod> -n <namespace>
```

Look at the **scheduler events**, for example:

```
0/4 nodes are available: 2 Insufficient cpu
```

✅ **What this tells you:** it's not just a monitoring alert. There is **real scheduling pressure**.

### Step 3: Check node capacity and allocation

```bash
kubectl describe node <node-name>
```

Look at:

```
Capacity:              cpu: 4
Allocatable:           cpu: 3900m
Allocated resources:   CPU Requests: 3800m
```

→ Only ~100m of schedulable CPU left, even if actual usage looks fine.

### Step 4: Find the workload causing it

```bash
kubectl top pods -A --sort-by=cpu
```

Example output:

```
NAMESPACE          POD                    CPU
employee-backend   backend-xxx            1800m
employee-backend   backend-yyy            1600m
prometheus         prometheus-xxx          300m
```

Now investigate the backend:

```bash
# Check requests and limits
kubectl describe pod backend-xxx -n employee-backend
#   requests: cpu: 500m
#   limits:   cpu: 2

# Check application logs
kubectl logs backend-xxx -n employee-backend --tail=200

# Check for recent changes
kubectl rollout history deployment/backend -n employee-backend
```

💡 This is where troubleshooting becomes **root-cause analysis**, not just "add more capacity."

### Step 5: Check HPA (Horizontal Pod Autoscaler)

```bash
kubectl get hpa -A
kubectl describe hpa <hpa-name> -n <namespace>
```

Example situation:

```
CPU utilization:   92%
Target:            70%
Current replicas:  4
Desired replicas:  8
```

**The chain reaction:**

```
Traffic increased
      ↓
Backend CPU increased
      ↓
HPA increased replicas
      ↓
More pods required
      ↓
Node has insufficient CPU
      ↓
Pods become Pending
```

> ⚠️ **HPA alone cannot fix this.** It adds pods, not nodes. You need **node-level scaling**.

### Step 6: Check Cluster Autoscaler / Karpenter

**With Cluster Autoscaler:**

```
HPA → More pods → Pending pods → Cluster Autoscaler
   → Increase node group → AWS ASG → New EC2 → Pods schedule
```

**With Karpenter:**

```
HPA → Pending pods → Karpenter
   → Provision suitable EC2 → Node joins EKS → Pod schedules
```

**Current project status:**

| Component | Status |
|-----------|--------|
| Metrics Server | ✅ Installed |
| HPA | ✅ Installed |
| Cluster Autoscaler | ❌ Not installed |
| Karpenter | ❌ Not installed |

🎯 **Important interview point:** without Cluster Autoscaler or Karpenter, a **Managed Node Group will NOT add a node automatically** just because CPU is high. Scaling has to be done manually.

---

## 4. Immediate Production Mitigation

Don't wait for a full RCA before protecting the application. Choose based on the situation:

| Option | When to use | How |
|--------|-------------|-----|
| **A. Increase node capacity** | Nodes are full and pods are Pending | Increase Managed Node Group desired/max capacity (e.g., 2 → 4 nodes) |
| **B. Scale the workload** | Traffic is the cause | `kubectl scale deployment backend --replicas=8 -n employee-backend` *(normally let HPA handle this)* |
| **C. Reduce unnecessary workloads** | Non-critical batch/cron jobs are consuming resources | Temporarily scale down or suspend them |
| **D. Roll back a bad deployment** | CPU spiked right after a release | `kubectl rollout undo deployment/backend -n employee-backend` |

**Rollback signal to watch for:**

```
Deployment → CPU spike → Error rate increases
```

---

## 5. ⚠️ Interview Trap: Don't Immediately Reboot the EC2

If a node is at 100% CPU, rebooting may **hide the symptom** but doesn't reveal the **cause**.

**First establish:**

```
What is consuming CPU?
        ↓
      Why?
        ↓
Application traffic?  Bad deployment?  Bad configuration?
Insufficient resources?  Runaway process?
```

Then take the **least disruptive** mitigation.

---

## 6. If the Problem Is at the Linux Level

If Kubernetes metrics don't explain the CPU usage, access the node via **SSM Session Manager**, then run:

```bash
top
ps aux --sort=-%cpu | head -20
vmstat 1 5

# Kernel / system activity
journalctl -k --since "30 min ago"
```

Also investigate the container runtime and processes if necessary.

---

## 7. Permanent Fix (After Stabilizing)

Do the RCA and document it. Example:

| | |
|---|---|
| **Root cause** | Backend deployment introduced inefficient processing |
| **Immediate mitigation** | Added node capacity + rolled back deployment |
| **Permanent fix** | Code optimization, correct CPU requests/limits, HPA tuning, node autoscaling, monitoring/alerting |

---

## 8. The Flow to Memorize

```
              HIGH CPU ALERT
                    │
                    ▼
            kubectl top nodes
                    │
                    ▼
        Which node is affected?
                    │
                    ▼
          kubectl top pods -A
                    │
                    ▼
     Which workload consumes CPU?
                    │
                    ▼
        Check requests / limits
                    │
                    ▼
                Check HPA
                    │
                    ▼
         Are pods Pending?
           ┌────────┴────────┐
          NO                YES
           │                 │
           ▼                 ▼
    Investigate app     Insufficient CPU
    / deployment                │
                                ▼
                  Cluster Autoscaler / Karpenter?
                                │
                                ▼
                        Add EC2 capacity
                                │
                                ▼
                         Pods schedule
                                │
                                ▼
                               RCA
                                │
                                ▼
                     Permanent prevention
```

---

## 9. Strong Interview Answer (Say This)

> "First I confirm the node-level utilization with `kubectl top nodes` and identify the consuming workloads using `kubectl top pods -A --sort-by=cpu`. I then check the node's allocatable resources and pod CPU requests and limits, and determine whether pods are Pending due to *Insufficient cpu*.
>
> I check HPA to understand whether increased traffic caused replica growth, and then verify whether Cluster Autoscaler or Karpenter is available to provision additional nodes.
>
> For immediate mitigation, depending on the situation, I may increase node capacity, scale the affected workload, suspend a non-critical workload, or roll back a problematic deployment.
>
> Once the application is stable, I investigate the root cause and implement permanent fixes such as right-sizing requests/limits, HPA tuning, node autoscaling, and application optimization."

**Framework:** Diagnose → Protect production → Scale/Mitigate → Identify root cause → Prevent recurrence.

---

## 10. Quick-Recall Cheat Sheet

| Goal | Command |
|------|---------|
| Node CPU usage | `kubectl top nodes` |
| Top CPU pods | `kubectl top pods -A --sort-by=cpu` |
| Find Pending pods | `kubectl get pods -A --field-selector=status.phase=Pending` |
| Why is a pod Pending? | `kubectl describe pod <pod> -n <ns>` |
| Node capacity vs requests | `kubectl describe node <node>` |
| Pod requests/limits | `kubectl describe pod <pod> -n <ns>` |
| App logs | `kubectl logs <pod> -n <ns> --tail=200` |
| Recent releases | `kubectl rollout history deployment/<name> -n <ns>` |
| Check HPA | `kubectl get hpa -A` / `kubectl describe hpa <name> -n <ns>` |
| Manual scale | `kubectl scale deployment <name> --replicas=<n> -n <ns>` |
| Roll back | `kubectl rollout undo deployment/<name> -n <ns>` |
| Linux-level CPU | `top`, `ps aux --sort=-%cpu \| head -20`, `vmstat 1 5` |
| Kernel logs | `journalctl -k --since "30 min ago"` |

---

## 11. Likely Follow-Up Questions

**Q: Node CPU is 90% but pods still schedule. Why?**
The scheduler goes by CPU *requests*, not actual utilization. Requests may still have room.

**Q: HPA is scaling up but pods stay Pending. Why?**
HPA adds pods, not nodes. With no Cluster Autoscaler or Karpenter, the node group can't grow.

**Q: What's the difference between Cluster Autoscaler and Karpenter?**
Cluster Autoscaler grows an existing node group (via the AWS ASG). Karpenter provisions suitable EC2 instances directly in response to Pending pods.

**Q: Would you reboot the node?**
Not first. Find out what's consuming CPU and why, then choose the least disruptive fix.