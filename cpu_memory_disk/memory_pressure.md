# 🧠 "Memory Pressure / High Memory on EC2 or EKS": Interview Quick Reference

> **Interview question:** *"A production server (or EKS node) is running out of memory. What do you do?"*

---

## 🎯 1. The Framework (memorize this)

```
CONFIRM  →  IDENTIFY  →  CLASSIFY  →  MITIGATE  →  VERIFY  →  RCA & PREVENT
```

| Phase | Goal | Key tools |
|---|---|---|
| **1. Confirm** | Is it real memory pressure, or just cache? | `free -h`, `vmstat`, CloudWatch Agent |
| **2. Identify** | Which process / pod is the consumer? | `ps`, `top`, `kubectl top` |
| **3. Classify** | Leak, spike, misconfiguration, or genuine capacity shortage? | trend over time, logs, `dmesg` |
| **4. Mitigate** | Free memory safely, protect users | restart, scale out, resize, rollback |
| **5. Verify** | Memory stable, no OOM, app healthy | `free -h`, `dmesg`, app checks |
| **6. RCA & Prevent** | Stop it recurring | limits, alarms, profiling, right-sizing |

---

## 🗣️ 2. The 60-Second Answer (say this)

> "First I **confirm it's real pressure** by looking at **`available` memory in `free -h`**, not `free`, because Linux uses spare RAM for cache. I check **swap activity** (`si/so` in `vmstat`) and **OOM kills** in `dmesg`.
>
> Then I **identify the consumer** with `ps aux --sort=-%mem` or, on EKS, `kubectl top pods`. I **classify** it: is memory **growing steadily** (a leak), **jumping suddenly** (a traffic or batch spike), or **hitting a wrong limit** (misconfiguration)?
>
> For **mitigation**, I take the least disruptive action: gracefully restart the leaking service, scale out behind the load balancer, roll back a bad release, or resize the instance. I **don't just add swap or drop caches** as a fix.
>
> After stabilizing, I do the **RCA**, using heap dumps or profiling, and **prevent recurrence** with correct memory requests/limits, CloudWatch alarms, autoscaling, and right-sizing."

---

## 🧩 3. Core Concepts (get these right, interviewers test them)

### 3.1 `free` vs `available`: the #1 trap

```bash
free -h
```

| Column | Meaning | Look at it? |
|---|---|---|
| **free** | Completely unused RAM | ❌ Misleading, it's normally small |
| **buff/cache** | Disk cache the kernel can **reclaim instantly** | ℹ️ Not a problem |
| **available** | RAM usable by apps **without swapping** | ✅ **This is the real number** |

📌 **Say:** *"Low `free` is normal on Linux. I worry when **`available` is low** and **swap is being used**."*

### 3.2 Healthy vs unhealthy signals

| Signal | Healthy | 🚨 Problem |
|---|---|---|
| `available` | Comfortable headroom | Near zero or shrinking |
| Swap `si`/`so` (vmstat) | 0 | Constantly non-zero (**thrashing**) |
| OOM events in `dmesg` | None | `Out of memory: Killed process ...` |
| Process RSS over time | Stable | **Climbs steadily** (leak) |
| `wa` (I/O wait) | Low | High, since swapping hammers the disk |

### 3.3 What the OOM killer does

When RAM and swap are exhausted, the **kernel picks a process and sends SIGKILL**.
It scores each process by memory use (plus `oom_score_adj`) and kills the **largest**, which is often your **main app or database**, not the actual culprit.

---

## 🔎 4. Linux Triage: Step by Step

```bash
# 1️⃣ Confirm real pressure
free -h                                  # look at "available"
vmstat 1 5                               # si/so = swap in/out, wa = I/O wait
cat /proc/meminfo | head -20            # MemAvailable, SwapFree, Slab

# 2️⃣ Find the consumer
ps aux --sort=-%mem | head -10
top                                      # press M to sort by memory
ps -p <PID> -o pid,ppid,user,%mem,rss,vsz,etime,cmd
pmap -x <PID> | tail -5                  # memory map summary

# 3️⃣ Check for OOM kills
dmesg -T | grep -i -E 'oom|killed process'
journalctl -k | grep -i oom              # survives longer than dmesg after reboot

# 4️⃣ Is it growing? (leak test)
watch -n 5 'ps -o rss,vsz,cmd -p <PID>'  # does RSS climb with no drop?
sar -r 1 5                               # memory history (needs sysstat)

# 5️⃣ Check other consumers
df -h /dev/shm /tmp                      # tmpfs counts as memory!
slabtop -o | head                        # kernel slab (rare, but real)

# 6️⃣ Logs
journalctl -u <service> -n 100
journalctl --since "30 minutes ago" -p err
```

### 📖 RSS vs VSZ (quick clarity)

| Term | Meaning | Use it? |
|---|---|---|
| **RSS** | Physical RAM actually used by the process | ✅ Use this |
| **VSZ** | Virtual address space reserved (often huge) | ❌ Not real usage |

---

## 🧭 5. Classify the Root Cause

| Pattern you see | Likely cause | Action |
|---|---|---|
| 📈 RSS **climbs steadily**, never drops | **Memory leak** | Heap dump / profiler, fix code, restart as stopgap |
| ⚡ **Sudden jump** with traffic or batch job | **Spike / capacity** | Scale out, throttle, queue the work |
| 🔁 Process **restarts repeatedly** | OOM kill loop | Check `dmesg`, raise limit *after* understanding why |
| ☕ Java app near limit | **Heap vs container/instance mismatch** | Tune `-Xmx` or `-XX:MaxRAMPercentage`, leave headroom |
| 🗄️ Database using all RAM | Misconfigured buffers or caches | Tune `shared_buffers`, `innodb_buffer_pool_size` |
| 🧱 Many processes, each moderate | Too many workers / connections | Cap worker count, use connection pooling |
| 💾 `tmpfs` / `/dev/shm` full | Files in RAM-backed filesystem | Clean up, move data to disk |
| 🔄 Heavy swapping, high `wa` | Real undersizing | Resize instance (R-family), scale out |

---

## ☁️ 6. AWS-Specific Checks (senior-level points)

| Check | Why it matters |
|---|---|
| **Memory is NOT in CloudWatch by default** | Install the **CloudWatch Agent** to get `mem_used_percent`, swap, and disk. Otherwise you're blind. |
| **Instance family fit** | Memory-heavy workloads belong on **R-family** (or X for extreme cases), not C/T. |
| **AWS Compute Optimizer** | Gives data-driven right-sizing recommendations (needs memory metrics from the agent). |
| **ASG scaling on memory** | Scale on a **custom CloudWatch memory metric** (the default CPU policy won't catch it). |
| **No swap by default on most AMIs** | So memory exhaustion goes **straight to OOM kill**. Adding swap is a *stopgap*, not a fix. |
| **Access** | Use **SSM Session Manager**, with no open SSH port. |

---

## 🛠️ 7. Mitigation Options (least disruptive first)

| # | Action | When | ⚠️ Caveat |
|---|---|---|---|
| 1 | **Graceful restart** of the leaking service (`systemctl restart <svc>`) | Confirmed leak, need quick relief | Hides the cause, so capture a **heap dump first** |
| 2 | **Scale out** (ASG / more replicas) | Traffic-driven pressure | Each instance still needs enough RAM |
| 3 | **Roll back** the release | Memory jumped right after a deploy | Keep the bad build for analysis |
| 4 | **Resize / change instance type** (e.g. M → R) | Genuine capacity shortage | Needs stop/start or a replacement via ASG |
| 5 | **Throttle / rate-limit** or pause batch jobs | Spike from non-critical work | Communicate the impact |
| 6 | **Add temporary swap** | Emergency only, to avoid OOM while you fix | Slow, can cause thrashing, and it's not a real fix |

### 🚫 Don'ts (interviewers love these)

- ❌ **Don't `echo 3 > /proc/sys/vm/drop_caches` as a "fix".** Cache isn't the problem, and it hurts performance.
- ❌ **Don't blindly `kill -9`** a production process. Try graceful first and understand the impact.
- ❌ **Don't just raise the limit** without asking *why* usage grew. You'll hide a leak.
- ❌ **Don't reboot first.** You lose the evidence (heap, logs, state).

---

## ☸️ 8. If It's an EKS Node / Pod

### 8.1 Key concepts

| Concept | Meaning |
|---|---|
| **Memory request** | What the scheduler **reserves**, used for placement |
| **Memory limit** | The **hard cap**. Exceeding it means the container is **OOMKilled** (exit code **137**) |
| **Memory is incompressible** | Unlike CPU, it **can't be throttled**, so excess means a kill |

### 8.2 OOMKilled vs Evicted (a very common question)

| | **OOMKilled** | **Evicted** |
|---|---|---|
| **Who acts** | Linux **kernel** (cgroup limit) | **kubelet** (node under memory pressure) |
| **Trigger** | Container exceeds **its own limit** | **Node** `memory.available` falls below threshold |
| **Signal** | `Reason: OOMKilled`, exit code **137** | Pod status **Evicted** |
| **Scope** | One container | Pods chosen by QoS and usage over requests |
| **Fix** | Raise limit or fix leak | Right-size requests, add nodes |

### 8.3 QoS classes (who dies first)

| QoS | Condition | Eviction order |
|---|---|---|
| **BestEffort** | No requests or limits | 💀 **First** |
| **Burstable** | Requests < limits | ⚠️ Middle (those over their requests first) |
| **Guaranteed** | Requests = limits | 🛡️ **Last** |

📌 **Say:** *"Critical production pods should be **Guaranteed** for memory, with requests equal to limits."*

### 8.4 Triage flow

```bash
kubectl top nodes                                   # node memory
kubectl top pods -A --sort-by=memory                # top memory pods
kubectl describe node <node>                        # look for "MemoryPressure=True", evictions
kubectl describe pod <pod> -n <ns>                  # "Last State: Terminated, Reason: OOMKilled"
kubectl get events -A --sort-by=.lastTimestamp | grep -i -E 'oom|evict'
kubectl get pods -A | grep -E 'Evicted|OOMKilled|CrashLoop'
kubectl logs <pod> -n <ns> --previous               # logs from before the crash
kubectl rollout history deployment/<name> -n <ns>   # recent release?
```

### 8.5 ☕ Java in containers (frequent follow-up)

- Container memory is **more than the heap**: heap + metaspace + thread stacks + native memory.
- If `-Xmx` equals the container limit, the pod **will OOMKill** even though the heap looks fine.
- ✅ Use `-XX:MaxRAMPercentage=70` (or similar) and leave headroom.
- Diagnose with `jcmd <PID> GC.heap_info`, `jmap -histo:live <PID>`, and GC logs.

### 8.6 K8s mitigation

| Situation | Action |
|---|---|
| Pod OOMKilled, **limit too low** | Raise limit **after** confirming real need |
| Pod OOMKilled, **leak** | Roll back or fix, capture a heap dump |
| Node MemoryPressure, nodes full | Add nodes (managed node group, **Cluster Autoscaler / Karpenter**) |
| Many BestEffort pods | Set requests/limits, enforce with **LimitRange** |
| Right-size automatically | **VPA** in recommendation mode to suggest values |

> ⚠️ **HPA on memory is tricky.** Memory often doesn't drop after scale-out, so many teams scale on CPU or custom metrics (RPS, queue depth) instead.

---

## ✅ 9. Verify Recovery

```bash
free -h ; vmstat 1 5 ; dmesg -T | tail
```

- [ ] `available` memory healthy and **stable** (not creeping down)
- [ ] **Swap activity** (`si/so`) back to 0
- [ ] **No new OOM** entries in `dmesg` or pod events
- [ ] Application **responding**, error rate and latency normal
- [ ] **No restart loops** (`kubectl get pods`, `systemctl status`)
- [ ] Watch for **30+ minutes**, because a leak may come back

---

## 🛡️ 10. Prevent Recurrence

| Area | Action |
|---|---|
| 📊 **Monitoring** | CloudWatch Agent memory metrics, Prometheus `container_memory_working_set_bytes` |
| 🚨 **Alerting** | Alarm on **memory > 80% for 5+ min**, OOM events, pod restarts, MemoryPressure |
| 🎯 **Limits** | Set requests and limits on **every** pod, enforce with LimitRange / ResourceQuota |
| 🛡️ **QoS** | Critical services as **Guaranteed** |
| 📐 **Right-sizing** | Compute Optimizer, VPA recommendations, R-family where needed |
| 🔬 **Profiling** | Heap dumps, memory profilers, load tests before release |
| 🚀 **Release safety** | Canary / blue-green with auto rollback |
| 📚 **Docs** | RCA + runbook, with the fix tracked in the backlog |

**RCA example:**
*Root cause:* new release cached unbounded query results →
*Mitigation:* rolled back + restarted pods →
*Permanent fix:* bounded cache with TTL, memory limits tuned, alert on working-set growth.

---

## 💬 11. Likely Follow-Up Questions

**Q: `free` shows only 200 MB free. Is that a problem?**
Not necessarily. Check **`available`**. Linux uses spare RAM as cache, which is reclaimable.

**Q: How do you tell a leak from a spike?**
Trend it. A **leak** shows steadily climbing RSS that never drops, even when load drops. A **spike** correlates with traffic or a job and recovers.

**Q: Should you enable swap?**
As an **emergency buffer**, maybe. It isn't a fix: it's slow, causes thrashing, and masks problems. Kubernetes nodes traditionally run with swap off.

**Q: Pod keeps getting OOMKilled. Do you just double the limit?**
No. First check **why** with usage trends, logs, and heap analysis. Raise the limit only if the workload legitimately needs it.

**Q: High memory but the app is fine. Act?**
Monitor and trend it. Check headroom for spikes and whether usage is cache (fine) or RSS (watch).

**Q: How is memory pressure different from CPU pressure?**
CPU is **compressible**, so it gets throttled and slows down. Memory is **incompressible**, so excess means a **kill**.

**Q: Which process does the OOM killer pick?**
Generally the one with the **highest memory score** (adjusted by `oom_score_adj`), which isn't always the culprit.

---

## ⚡ 12. One-Page Cheat Sheet

| Goal | Command |
|---|---|
| Real free memory | `free -h` (read **available**) |
| Swap activity | `vmstat 1 5` (`si`/`so`) |
| Memory details | `cat /proc/meminfo` |
| Top memory processes | `ps aux --sort=-%mem \| head -10` |
| Process RSS and details | `ps -p <PID> -o pid,%mem,rss,vsz,etime,cmd` |
| Memory map | `pmap -x <PID>` |
| OOM events | `dmesg -T \| grep -i -E 'oom\|killed process'` |
| Memory history | `sar -r 1 5` |
| RAM-backed storage | `df -h /dev/shm /tmp` |
| Service logs | `journalctl -u <svc> -n 100` |
| Java heap info | `jcmd <PID> GC.heap_info` ; `jmap -histo:live <PID>` |
| K8s node / pod memory | `kubectl top nodes` ; `kubectl top pods -A --sort-by=memory` |
| OOMKilled reason | `kubectl describe pod <pod> -n <ns>` |
| Pre-crash logs | `kubectl logs <pod> -n <ns> --previous` |
| Evictions / OOM events | `kubectl get events -A \| grep -i -E 'oom\|evict'` |
| Rollback | `kubectl rollout undo deployment/<name> -n <ns>` |

---

### 🏁 Closing line

> **"I confirm real pressure using `available` memory and swap activity, find the consumer, classify it as a leak, spike, or misconfiguration, mitigate with the least disruption while preserving evidence, then root-cause and prevent it with proper limits, monitoring, and right-sizing. I never treat cache as a problem, and I never reboot first."**