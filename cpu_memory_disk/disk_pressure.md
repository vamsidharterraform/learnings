# 💽 "Disk Full / Disk Pressure on EC2 or EKS": Interview Quick Reference

> **Interview question:** *"A production server (or EKS node) is running out of disk space. What do you do?"*

---

## 🎯 1. The Framework (memorize this)

```
CONFIRM  →  FIND WHAT'S EATING IT  →  CLASSIFY  →  MITIGATE  →  VERIFY  →  RCA & PREVENT
```

| Phase | Goal | Key tools |
|---|---|---|
| **1. Confirm** | Which filesystem? Space, inodes, or I/O? | `df -h`, `df -i`, `iostat` |
| **2. Find** | Which directory / file / process? | `du`, `find`, `lsof` |
| **3. Classify** | Logs, deleted-but-open files, images, data growth, or inodes? | patterns below |
| **4. Mitigate** | Free space **safely**, or expand the volume | truncate, clean, EBS resize |
| **5. Verify** | Space recovered, app healthy, no re-fill | `df -h`, app checks |
| **6. RCA & Prevent** | Stop it filling again | rotation, alarms, sizing |

---

## 🗣️ 2. The 60-Second Answer (say this)

> "First I confirm **which filesystem** is full with `df -h`, and I also check **inodes** with `df -i`, because a disk can be 'full' with space still free. Then I find the culprit with `du` sorted by size, and `find` for large files.
>
> I **classify** it: runaway **logs**, **deleted files still held open** by a process (seen with `lsof +L1`), **container images and layers**, genuine **data growth**, or **inode exhaustion**.
>
> For mitigation I free space **safely**. I **truncate** active logs instead of deleting them, vacuum the journal, clear package caches, or prune unused images. If the usage is legitimate, I **expand the EBS volume online** and grow the filesystem. I **never blindly `rm -rf`**, especially anything a database or running process owns.
>
> Then I do the **RCA** and **prevent recurrence** with log rotation, CloudWatch disk alarms, right-sized volumes, and shipping logs off the box."

---

## 🧩 3. Core Concepts (interviewers test these)

### 3.1 "Disk pressure" can mean three different things

| Type | Symptom | Check |
|---|---|---|
| 🗂️ **Space exhausted** | `No space left on device` | `df -h` |
| 🔢 **Inodes exhausted** | `No space left on device` but `df -h` shows free space | `df -i` |
| ⏱️ **I/O saturated** | Slow app, high `wa`, disk not full | `iostat -xz 1 5` |

📌 **Say:** *"I always check **space, inodes, and I/O**. They look similar but have different fixes."*

### 3.2 `df` vs `du` mismatch: the classic trap

| Tool | Reports |
|---|---|
| `df` | What the **filesystem** says is used |
| `du` | What **visible files** add up to |

If `df` shows **much more** than `du`, the likely cause is **deleted files still held open by a running process**. The space isn't freed until the process closes the file.

```bash
lsof +L1                    # files deleted but still open (link count 0)
```
**Fix:** restart the process holding it (or truncate via `/proc/<PID>/fd/<N>`).

### 3.3 Why `rm` on a log file might not help

If an app has the log **open**, `rm` removes the name but the space stays used. **Truncate instead:**

```bash
: > /var/log/app/app.log          # or: truncate -s 0 /var/log/app/app.log
```

### 3.4 Other things to know

- **ext4 reserves ~5% of blocks for root**, so non-root users hit "full" slightly earlier.
- **XFS can't be shrunk**, and **EBS volumes can't be shrunk** either. You can only grow.
- **`tmpfs` (`/dev/shm`, `/tmp` on some systems) counts against RAM**, not disk.

---

## 🔎 4. Linux Triage: Step by Step

```bash
# 1️⃣ Confirm: space, inodes, mounts
df -h                                    # which filesystem is full?
df -i                                    # inode usage
lsblk ; mount | grep -E 'xfs|ext4'       # layout and filesystem types

# 2️⃣ Find the biggest consumers (stay on one filesystem with -x)
du -xh / --max-depth=1 2>/dev/null | sort -h | tail
du -xh /var --max-depth=2 2>/dev/null | sort -h | tail
find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null
ncdu -x /                                # interactive, if installed

# 3️⃣ Deleted-but-open files
lsof +L1

# 4️⃣ Common culprits
journalctl --disk-usage                  # systemd journal size
du -sh /var/log/* | sort -h | tail
du -sh /var/lib/docker /var/lib/containerd /var/cache 2>/dev/null
ls /var/crash /tmp 2>/dev/null | head    # core dumps, temp files

# 5️⃣ Inodes: find directories with huge file counts
for d in /var/*; do echo "$(find "$d" -xdev 2>/dev/null | wc -l) $d"; done | sort -n | tail

# 6️⃣ If it's an I/O problem instead
iostat -xz 1 5                           # watch await, %util, queue size
```

### 📖 Reading `iostat -xz`

| Column | Meaning | 🚨 Red flag |
|---|---|---|
| `r/s`, `w/s` | Read / write IOPS | Hitting the volume's IOPS limit |
| `await` | Avg ms per I/O request | Consistently high (tens to hundreds of ms) |
| `%util` | How busy the device is | Near 100% |
| `aqu-sz` | Average queue length | Growing → saturated |

---

## 🧭 5. Classify the Root Cause

| Pattern you see | Likely cause | Action |
|---|---|---|
| 📜 `/var/log` huge | **No log rotation** / debug logging left on | Truncate, add `logrotate`, reduce log level |
| 📓 Journal GBs in size | **Journald unbounded** | `journalctl --vacuum-size=500M`, set `SystemMaxUse` |
| 👻 `df` ≫ `du` | **Deleted-but-open** files | `lsof +L1`, restart that process |
| 🔢 `df -i` at 100% | **Millions of tiny files** (sessions, mail, cache, temp) | Delete the files, fix the app, consider a larger inode ratio at format time |
| 🐳 `/var/lib/docker` or `/var/lib/containerd` large | **Unused images / layers / stopped containers** | Prune safely (see below) |
| 💣 Core dumps in `/var/crash` or `/` | App crashes | Remove, fix crashes, limit core size |
| 🗄️ Database dir growing | **Genuine growth**, WAL/binlogs, no archiving | Archive/purge per DB policy, **expand the volume** |
| 📦 Package cache | `yum`/`dnf`/`apt` cache | `dnf clean all` / `apt clean` |
| 🌱 Slow steady growth | Natural data growth | **Expand the volume**, add alarms |

---

## ☁️ 6. AWS-Specific Checks (senior-level points)

| Check | Why it matters |
|---|---|
| **Disk usage isn't in CloudWatch by default** | Install the **CloudWatch Agent** for `disk_used_percent` and `inodes_free`. Otherwise you find out when the app dies. |
| **EBS volume expansion is online** | **Elastic Volumes** let you grow size, type and IOPS **without detaching or downtime**. |
| **Modification cooldown** | After a volume modification, you generally must wait (about **6 hours**) before modifying the same volume again. Plan the size with headroom. |
| **gp2 vs gp3** | **gp2** has burst credits (`BurstBalance`) and IOPS tied to size. **gp3** has a baseline of **3,000 IOPS / 125 MB/s** independent of size, and is usually cheaper. Prefer gp3. |
| **I/O metrics** | `VolumeQueueLength`, `BurstBalance` (gp2), `VolumeReadOps`/`WriteOps` to prove an I/O bottleneck. |
| **Instance-level EBS limits** | Small instance types have an **EBS bandwidth/IOPS cap** that can bottleneck even a fast volume. |
| **Snapshot before risky changes** | Take an **EBS snapshot** before resizing or deleting anything uncertain. |
| **Instance store** | **Ephemeral**: data is lost when the instance stops. Don't put anything you need there. |

### 📏 Expanding an EBS volume (the standard procedure)

```bash
# 1️⃣ Grow the volume (no downtime)
aws ec2 modify-volume --volume-id vol-xxxx --size 100

# 2️⃣ On the instance: see the new size
lsblk

# 3️⃣ Grow the partition (example: nvme0n1, partition 1)
sudo growpart /dev/nvme0n1 1

# 4️⃣ Grow the filesystem
sudo xfs_growfs -d /                    # XFS (Amazon Linux 2023 default)
sudo resize2fs /dev/nvme0n1p1           # ext4

# 5️⃣ Confirm
df -h
```

---

## 🛠️ 7. Mitigation Options (safest first)

| # | Action | Command | ⚠️ Caveat |
|---|---|---|---|
| 1 | **Truncate** big active logs | `: > /var/log/app/app.log` | Don't `rm` an open file. Space won't free. |
| 2 | **Compress / rotate** old logs | `logrotate -f /etc/logrotate.conf`, `gzip` | Check the config first |
| 3 | **Vacuum journal** | `journalctl --vacuum-size=500M` | Loses older journal entries |
| 4 | **Clear package caches** | `dnf clean all` / `apt clean` | Safe |
| 5 | **Prune unused images** | `docker system prune` / `crictl rmi --prune` | Only unused ones, and know your workloads |
| 6 | **Restart process** holding deleted files | `systemctl restart <svc>` | Brief disruption, so use rolling restarts |
| 7 | **Move old data** to S3 | `aws s3 cp/sync` then delete locally | Verify the upload before deleting |
| 8 | **Expand the EBS volume** | See section 6 | Best for **legitimate** growth |

### 🚫 Don'ts (interviewers love these)

- ❌ **Don't `rm -rf` blindly.** Never delete DB data files, WAL/redo logs, or anything a running process owns.
- ❌ **Don't delete an open log file.** Truncate it instead.
- ❌ **Don't manually delete `/var/lib/docker` or `/var/lib/containerd` contents.** Use the runtime's prune commands.
- ❌ **Don't just expand the disk** without finding **why** it filled, or it will fill again.
- ❌ **Don't reboot first.** It won't free space, and a full root disk can even cause boot or service problems.

---

## ☸️ 8. If It's an EKS Node / Pod

### 8.1 Key concepts

| Concept | Meaning |
|---|---|
| **`DiskPressure`** | A **node condition** set by the kubelet when disk is low. The node gets a `disk-pressure` taint, and new pods stop scheduling there. |
| **Kubelet eviction thresholds (defaults)** | `nodefs.available < 10%`, `imagefs.available < 15%`, `nodefs.inodesFree < 5%` |
| **Image garbage collection** | Kubelet starts removing unused images when disk passes **85%** and tries to get down to **80%** |
| **Ephemeral storage** | Container writable layers, `emptyDir`, logs. It's **not persistent**. |
| **Ephemeral-storage request/limit** | Lets you cap per-pod local disk use. Exceeding the limit means the pod is **evicted**. |
| **Default root volume** | Managed node groups default to a small root volume (**20 GiB**), which is easy to fill. Size it in the launch template. |

### 8.2 Triage flow

```bash
kubectl describe node <node>                         # look for DiskPressure=True, taints, evictions
kubectl get pods -A | grep -E 'Evicted'              # evicted pods
kubectl get events -A --sort-by=.lastTimestamp | grep -i -E 'evict|ephemeral|disk'
kubectl describe pod <pod> -n <ns>                   # "The node was low on resource: ephemeral-storage"

# On the node (via SSM)
df -h ; df -i
sudo du -xh /var/lib/containerd --max-depth=2 | sort -h | tail
sudo du -sh /var/log/pods /var/lib/kubelet 2>/dev/null
sudo crictl images                                   # image sizes
sudo crictl rmi --prune                              # remove unused images
```

### 8.3 Typical EKS causes

| Cause | Fix |
|---|---|
| 🐳 **Too many / large images** cached on the node | Image GC, `crictl rmi --prune`, slimmer images |
| 📜 **Container logs** filling `/var/log/pods` | Kubelet log rotation (`containerLogMaxSize`, `containerLogMaxFiles`), ship logs off-node |
| 📂 **`emptyDir` abuse** (apps writing temp data) | Set `ephemeral-storage` limits, use a PVC or EFS for real data |
| 🧱 **Root volume too small** | Increase the node volume size in the launch template and **roll the node group** |
| 🧹 **Pods writing to container layer** | Fix the app, mount a volume |

### 8.4 Persistent volumes (a different problem)

If a **PVC** is filling up (a database, for example), not node disk:

```bash
kubectl exec -it <pod> -n <ns> -- df -h
kubectl get pvc -A
```
**Fix:** expand the PVC (needs a StorageClass with `allowVolumeExpansion: true`, and the **EBS CSI driver**), then the filesystem grows online.

### 8.5 Mitigation for nodes

| Situation | Action |
|---|---|
| Node in DiskPressure | **Cordon and drain** the node, replace it (the ASG / node group makes a fresh one) |
| Recurring | Bigger root volume, image GC tuning, log shipping |
| Pods evicted for ephemeral storage | Raise limit **after** understanding the write pattern |
| Need quick relief on a live node | `crictl rmi --prune`, clear logs, then investigate |

---

## ✅ 9. Verify Recovery

```bash
df -h ; df -i ; iostat -xz 1 3
```

- [ ] Usage **well below** the threshold (aim for under 70–80%)
- [ ] **Inodes** healthy
- [ ] `lsof +L1` shows **no large deleted-open files**
- [ ] **Application responding**, no write errors in logs
- [ ] Node **`DiskPressure=False`** and pods rescheduling normally (EKS)
- [ ] **Watch the growth rate.** If it refills in hours, the root cause isn't fixed.

---

## 🛡️ 10. Prevent Recurrence

| Area | Action |
|---|---|
| 📊 **Monitoring** | CloudWatch Agent `disk_used_percent` and `inodes_free`, Prometheus `node_filesystem_avail_bytes` |
| 🚨 **Alerting** | Alarm at **80% (warning)** and **90% (critical)**, plus **predictive** "full in N hours" alerts |
| 🔄 **Log hygiene** | `logrotate`, journald `SystemMaxUse`, sensible log levels, **ship logs to CloudWatch / OpenSearch** |
| 📐 **Sizing** | Right-size volumes with headroom, **gp3**, separate volumes for `/var/log` and data |
| 🧱 **Separation** | Keep app data on a **dedicated EBS volume**, not the root volume |
| ☸️ **Kubernetes** | `ephemeral-storage` requests/limits, LimitRange, kubelet image GC and log rotation, bigger node root volumes |
| 🏗️ **IaC** | Define volume sizes in **Terraform** so they're reviewed and consistent |
| 🗑️ **Lifecycle** | Archive old data to **S3** with lifecycle rules |
| 📚 **Docs** | RCA + runbook, with the fix in the backlog |

**RCA example:**
*Root cause:* debug logging left on in production with no rotation →
*Mitigation:* truncated logs, restarted service, expanded volume →
*Permanent fix:* logrotate, log level policy, disk alarms at 80%/90%, logs shipped to CloudWatch.

---

## 💬 11. Likely Follow-Up Questions

**Q: `df` says 100% but `du` shows much less. Why?**
**Deleted files still open** by a process. Find them with `lsof +L1` and restart the process.

**Q: "No space left on device" but `df -h` shows free space?**
**Inodes exhausted.** Check `df -i`, and find the directory with millions of small files.

**Q: Can you expand an EBS volume without downtime?**
Yes, with **Elastic Volumes**. Modify the volume, then `growpart` and `xfs_growfs`/`resize2fs`. You can't shrink it, though.

**Q: Can you shrink an EBS volume?**
No. You'd create a smaller volume, copy the data, and swap.

**Q: How do you handle a full disk on a database server?**
Don't delete data files. Free space from **old logs, binlogs/WAL (per the DB's own purge process)**, or old backups. Then **expand the volume**. Take a **snapshot first**.

**Q: Why use `truncate` instead of `rm` on a log?**
The app still holds the file open. `rm` doesn't free the space, but truncating does.

**Q: What does DiskPressure do on a Kubernetes node?**
The kubelet taints the node, stops new pods being scheduled there, starts image GC, and **evicts pods** if usage stays above thresholds.

**Q: Disk isn't full but the app is slow. What now?**
Possibly **I/O saturation**. Check `iostat` (`await`, `%util`), EBS `BurstBalance`/`VolumeQueueLength`, and instance EBS limits. Consider **gp3** with more IOPS/throughput.

**Q: How is disk pressure different from memory pressure?**
Both can lead to evictions on K8s. Disk is **persistent and slow-growing**, so you can often **expand or clean up**. Memory is **volatile and incompressible**, so excess means a kill.

---

## ⚡ 12. One-Page Cheat Sheet

| Goal | Command |
|---|---|
| Space per filesystem | `df -h` |
| Inode usage | `df -i` |
| Biggest directories | `du -xh / --max-depth=1 \| sort -h \| tail` |
| Large files | `find / -xdev -type f -size +500M -exec ls -lh {} \;` |
| Deleted-but-open files | `lsof +L1` |
| Truncate a log safely | `: > /var/log/app/app.log` |
| Journal size / vacuum | `journalctl --disk-usage` ; `journalctl --vacuum-size=500M` |
| Force log rotation | `logrotate -f /etc/logrotate.conf` |
| Package cache cleanup | `dnf clean all` / `apt clean` |
| Disk I/O stats | `iostat -xz 1 5` |
| Block device layout | `lsblk` |
| Grow EBS volume | `aws ec2 modify-volume --volume-id <id> --size <GiB>` |
| Grow partition / FS | `growpart /dev/nvme0n1 1` ; `xfs_growfs -d /` or `resize2fs <dev>` |
| Node disk condition | `kubectl describe node <node>` (look for `DiskPressure`) |
| Evicted pods | `kubectl get pods -A \| grep Evicted` |
| Images on node | `crictl images` ; `crictl rmi --prune` |
| PVC usage | `kubectl exec <pod> -n <ns> -- df -h` |

---

### 🏁 Closing line

> **"I check space, inodes, and I/O to know which kind of disk problem it is, find the consumer with `du` and `lsof`, free space safely by truncating rather than deleting, and expand the EBS volume online if the growth is legitimate. Then I root-cause it and prevent recurrence with log rotation, disk alarms, right-sized gp3 volumes, and, on EKS, ephemeral-storage limits and image GC. I never `rm -rf` blindly, and I never reboot first."**