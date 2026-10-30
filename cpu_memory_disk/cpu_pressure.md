# 🔥 "EC2 CPU is at 95%+": Senior DevOps Quick Reference

> **Interview question:** *"An EC2 instance (or EKS node) is at full CPU. What do you do?"*

---

## 🎯 1. The Framework (memorize this)

```
CONFIRM  →  IDENTIFY  →  CLASSIFY  →  MITIGATE  →  VERIFY  →  RCA & PREVENT
```

| Phase | Goal | Key tools |
|---|---|---|
| **1. Confirm** | Is it real, and how bad? | `uptime`, `nproc`, CloudWatch |
| **2. Identify** | Which process / pod? | `top`, `pidstat`, `kubectl top` |
| **3. Classify** | CPU-bound, I/O-bound, memory-bound, or capacity? | `vmstat`, `mpstat`, `iostat`, `free` |
| **4. Mitigate** | Protect users, least disruptive action | scale, rollback, throttle, restart service |
| **5. Verify** | Confirm recovery | `top`, `uptime`, app health |
| **6. RCA & Prevent** | Make sure it never repeats | right-size, autoscale, alarms |

---

## 🗣️ 2. The 60-Second Answer (say this)

> "First I confirm the alert and check **load average against core count** with `uptime` and `nproc`. Then I find the culprit with `top` or `ps aux --sort=-%cpu`, down to the thread level if needed.
>
> I then **classify** the problem with `vmstat` and `mpstat`: high `us` means application code, high `sy` means kernel or system calls, high `wa` means a disk bottleneck, and high `st` means noisy neighbour or exhausted burst credits. I also rule out memory pressure and OOM kills.
>
> On AWS I check CloudWatch, and on T-series instances the **CPU credit balance**. If it's a legitimate traffic spike, I scale out with the ASG or HPA. If it's a bad release, I **roll back**. If it's a runaway process, I stop it gracefully after confirming the impact.
>
> I **don't reboot blindly**, because that hides the cause. After stabilizing, I do an RCA and prevent recurrence with right-sizing, autoscaling, and alarms."

---

## 🧠 3. Core Concepts (the 4 classic questions)

### Q1. CPU utilization vs load average?

| | CPU Utilization | Load Average |
|---|---|---|
| **What it is** | % of time CPUs are busy | Avg number of tasks **running + waiting** (including disk-blocked) over 1, 5, 15 min |
| **Scale** | 0–100% | Compare against **number of cores** (`nproc`) |
| **Example** | 95% busy | Load 8 on 4 cores = overloaded, tasks are queueing |

📌 **Rule of thumb:** load ≈ cores is fully busy, and load well above cores means saturation.
📈 **Trend:** `8, 3, 1` = just started and rising. `1, 3, 8` = recovering.

### Q2. How do I find the top CPU process?

```bash
top                                  # P = sort by CPU, M = by memory, 1 = per core, q = quit
ps aux --sort=-%cpu | head -10
ps -p <PID> -o pid,ppid,user,%cpu,%mem,stat,etime,cmd
top -H -p <PID>                      # per-thread view
pidstat 1 5                          # per-process CPU over time
```
☕ **Java:** take thread dumps with `jstack <PID>` (take 3, a few seconds apart) and map hot thread IDs (hex) to stack traces.

### Q3. What does `wa` in vmstat mean?

**I/O wait.** The CPU is idle but waiting on disk or network storage. High `wa` means the problem is **storage, not compute**, so scaling CPU won't help. Check `iostat -xz 1 5` (`%util`, `await`) and EBS limits (IOPS/throughput, burst balance).

### Q4. High CPU but low load average?

| Possible cause | How to check |
|---|---|
| **Short spike**, since load average is smoothed over 1/5/15 min | Watch `vmstat 1`, which is real-time |
| **Few busy threads** pinning a core, so load is low | `top` with `1`, `mpstat -P ALL 1` |
| **Steal time (`st`)**: noisy neighbour or burst credits exhausted | `vmstat`/`top` `st` column, CloudWatch `CPUCreditBalance` |
| **Interrupts / softirq (`si`/`hi`)**: network-heavy load | `mpstat`, `/proc/interrupts` |

---

## 🔎 4. Linux Triage: Step by Step

```bash
# 1️⃣ Confirm
uptime ; nproc

# 2️⃣ Find the culprit
top                                   # then P, 1
ps aux --sort=-%cpu | head -10

# 3️⃣ Classify
mpstat -P ALL 1                       # per-core breakdown
vmstat 1 5                            # r b us sy id wa st
pidstat 1 5                           # who's using CPU over time

# 4️⃣ Rule out memory
free -h
ps aux --sort=-%mem | head -10
dmesg -T | grep -i -E 'oom|killed process'

# 5️⃣ Rule out disk
df -h
iostat -xz 1 5

# 6️⃣ Logs
journalctl -u <service> -n 100
journalctl -u <service> -f
journalctl --since "30 minutes ago" -p err
```

### 📖 Reading `vmstat 1 5`

| Column | Meaning | Red flag |
|---|---|---|
| `r` | Runnable processes | Consistently > cores |
| `b` | Blocked (uninterruptible) | > 0 often → I/O problem |
| `us` | User CPU (app code) | High → app / code problem |
| `sy` | System CPU (kernel) | High → syscalls, network, context switching |
| `id` | Idle | Near 0 → saturated |
| `wa` | I/O wait | High → disk bottleneck |
| `st` | Steal | High → hypervisor contention / credit exhaustion |
| `si` / `so` | Swap in / out | Non-zero → memory pressure |

### 🧭 Quick Diagnosis Table

| Symptom | Likely cause | Next step |
|---|---|---|
| High `us`, one process | App bug, infinite loop, hot code | Thread dump / profiler, check recent deploy |
| High `sy` | Heavy syscalls, network, context switches | `strace -c -p <PID>`, `perf top` |
| High `wa` | Slow disk / EBS limits | `iostat`, check EBS volume metrics |
| High `st` | Noisy neighbour / T-instance out of credits | CloudWatch credits, change instance type |
| Swapping + OOM | Memory leak causing thrashing | `free -h`, `dmesg`, fix leak or add RAM |
| Many processes, load ≫ cores | Real traffic / capacity shortage | Scale out |

---

## ☁️ 5. AWS-Specific Checks (what separates seniors from juniors)

| Check | Why it matters |
|---|---|
| **CloudWatch `CPUUtilization`** | Hypervisor-level view. Compare with the OS view and check how long it's been high. |
| **`CPUCreditBalance` / `CPUSurplusCreditsCharged`** (T2/T3/T4g) | Burstable instances **throttle to baseline** when credits hit 0, which is a classic "mystery" cause. Fix: move to M/C family or enable Unlimited (watch the cost). |
| **Instance type fit** | Compute-heavy → **C-family**. Memory-heavy → **R-family**. Use **AWS Compute Optimizer** for rightsizing. |
| **EBS metrics** (`VolumeQueueLength`, burst balance, IOPS) | gp2 burst exhaustion causes high `wa`. Consider **gp3**. |
| **Status checks** | Distinguish an OS-level problem from an instance/host problem. |
| **ASG / ALB metrics** | Check `RequestCount`, `TargetResponseTime`, and 5xx to see user impact and whether it's traffic. |
| **Memory isn't in CloudWatch by default** | Install the **CloudWatch Agent** for mem/disk metrics. |
| **Access** | Use **SSM Session Manager** (no SSH or open port 22 needed). |

---

## 🛠️ 6. Corrective Actions by Root Cause

| Root cause | Action |
|---|---|
| 🐛 Application CPU spike / bad code | Roll back the release, profile, fix code |
| ☕ Java thread problem | `jstack` thread dumps, check GC (`jstat -gcutil`) |
| 🐍 Runaway script / infinite loop | Fix the script, add timeouts |
| 📈 Excess traffic | **Scale out** (ASG policy, HPA), add caching/CDN, rate limiting |
| 💽 I/O bottleneck | Upgrade EBS (gp3, more IOPS), tune queries |
| 🧱 Insufficient capacity | Scale out or resize the instance type |
| 🎟️ T-instance credits exhausted | Move to a fixed-performance family or Unlimited mode |

### Stopping a process (carefully)

```bash
systemctl restart <service>     # preferred for managed services
kill <PID>                      # graceful (SIGTERM) first
kill -9 <PID>                   # last resort (SIGKILL), only after confirming impact
```
⚠️ **Never kill an important production process without understanding the impact.** Consider draining it from the load balancer first.

### 🚫 Interview trap: don't reboot first
Rebooting clears the symptom and **destroys the evidence** (and any in-memory state). Always ask: *what is consuming CPU, and why?* Then choose the **least disruptive** fix.

---

## ☸️ 7. If It's an EKS Node

### Key concept: utilization ≠ requests

| Term | Meaning |
|---|---|
| **CPU utilization** | CPU actually being used now |
| **CPU requests** | CPU reserved by pods. The **scheduler** uses this to place pods |

A node can be at 90% utilization and still accept pods, or sit idle and reject pods because its **requests** are fully allocated. Always check both.

### Triage flow

```bash
kubectl top nodes                                         # which node?
kubectl top pods -A --sort-by=cpu                         # which pods?
kubectl get pods -A --field-selector=status.phase=Pending # scheduling pressure?
kubectl describe pod <pod> -n <ns>                        # look for: "Insufficient cpu"
kubectl describe node <node>                              # Allocatable vs Allocated requests
kubectl get hpa -A                                        # is HPA scaling?
kubectl logs <pod> -n <ns> --tail=200
kubectl rollout history deployment/<name> -n <ns>         # recent release?
```

### The chain reaction

```
Traffic ↑ → Pod CPU ↑ → HPA adds replicas → Nodes full → Pods Pending
```

> ⚠️ **HPA adds pods, not nodes.** You also need **Cluster Autoscaler or Karpenter**. Without one, a Managed Node Group will **not** grow automatically.

| Autoscaler | How it works |
|---|---|
| **Cluster Autoscaler** | Grows an existing node group through the AWS ASG |
| **Karpenter** | Provisions right-sized EC2 directly in response to Pending pods (faster, more flexible) |

### Mitigation options

| Option | When |
|---|---|
| ➕ Increase node group desired/max | Nodes full, pods Pending |
| 📊 Scale the workload | Traffic-driven (let HPA do it) |
| ⏸️ Suspend non-critical jobs | Batch/cron stealing CPU |
| ⏪ `kubectl rollout undo deployment/<name> -n <ns>` | CPU spiked right after a deploy |

If Kubernetes metrics don't explain it, use **SSM** into the node and run the Linux triage from section 4.

---

## ✅ 8. Verify Recovery

```bash
top ; uptime ; vmstat 1 5
```

- [ ] CPU utilization back to normal
- [ ] Load average recovering (1-min dropping)
- [ ] Memory healthy, no swapping
- [ ] Application responding, error rate and latency normal
- [ ] No unexpected restarts or pods stuck Pending
- [ ] CloudWatch graph confirms the trend

---

## 🛡️ 9. Prevent Recurrence (RCA mindset)

| Area | Action |
|---|---|
| **Autoscaling** | ASG target tracking (e.g. 60% CPU), HPA, Karpenter or Cluster Autoscaler |
| **Right-sizing** | Correct requests/limits, Compute Optimizer, right instance family |
| **Alerting** | CloudWatch alarms on CPU > 80% for 5+ min, credit balance, `wa`, Pending pods |
| **Observability** | CloudWatch Agent, Prometheus/Grafana, APM, centralized logs |
| **Release safety** | Canary / blue-green, automated rollback, load tests before prod |
| **Protection** | Rate limiting, caching, WAF for abusive traffic |
| **Documentation** | Write the RCA and runbook, and add the fix to the backlog |

**RCA example:**
*Root cause:* new release introduced inefficient processing →
*Mitigation:* rolled back + added capacity →
*Permanent fix:* code optimization, right-sized requests/limits, autoscaling, alerts.

---

## 💬 10. Likely Follow-Up Questions

**Q: CPU is 90% but pods still schedule. Why?**
The scheduler uses **requests**, not actual usage.

**Q: HPA is scaling but pods are Pending. Why?**
HPA adds pods, not nodes. There's no Cluster Autoscaler or Karpenter, or the node group has hit its max.

**Q: CPU is high but the app is fine. Do you act?**
Investigate and trend it. High utilization isn't automatically a problem, but check latency, queueing (load vs cores), and headroom for spikes.

**Q: Load is high but CPU is low?**
Tasks are blocked on I/O (`wa`, `b` column). Check disk, NFS/EFS, and network storage.

**Q: How do you debug a Java app at 100% CPU?**
`top -H -p <PID>` to find the hot thread, convert its ID to hex, then match it in the `jstack` dump. Also check GC activity.

**Q: Reboot or terminate the instance?**
Not first. Capture evidence, then mitigate. In an ASG, drain it or detach it, and let it replace the instance only after you've captured the evidence.

---

## ⚡ 11. One-Page Cheat Sheet

| Goal | Command |
|---|---|
| Load vs cores | `uptime` ; `nproc` |
| Live top CPU | `top` (then `P`, `1`) |
| Top CPU processes | `ps aux --sort=-%cpu \| head -10` |
| Process details | `ps -p <PID> -o pid,ppid,user,%cpu,%mem,stat,etime,cmd` |
| Per-thread CPU | `top -H -p <PID>` |
| Per-core breakdown | `mpstat -P ALL 1` |
| CPU / memory / IO snapshot | `vmstat 1 5` |
| Per-process over time | `pidstat 1 5` |
| Memory | `free -h` |
| OOM events | `dmesg -T \| grep -i -E 'oom\|killed process'` |
| Disk space / IO | `df -h` ; `iostat -xz 1 5` |
| Service logs | `journalctl -u <svc> -n 100` / `-f` |
| Recent errors | `journalctl --since "30 minutes ago" -p err` |
| Java threads | `jstack <PID>` |
| Syscall summary | `strace -c -p <PID>` |
| K8s node / pod CPU | `kubectl top nodes` ; `kubectl top pods -A --sort-by=cpu` |
| Pending pods | `kubectl get pods -A --field-selector=status.phase=Pending` |
| Node allocation | `kubectl describe node <node>` |
| Rollback | `kubectl rollout undo deployment/<name> -n <ns>` |

---

### 🏁 Closing line

> **"Confirm, identify, classify, mitigate with the least disruption, verify, then root-cause and prevent. I never reboot blindly, and I always check both utilization and capacity (requests, credits, and limits)."**