# Linux + AWS Disk Management Notes

Two tasks covered here:

| Task | Goal | Result |
|------|------|--------|
| **Task 1** | Attach a **new** EBS volume | 10 GB disk mounted at `/data` |
| **Task 2** | **Vertically expand** an existing EBS volume (LVM root disk) | Root EBS 20 GB → 30 GB |

> **Key point:** Task 1 *adds* new storage. Task 2 *increases the capacity* of an existing disk/filesystem.

---

# Task 1: Add a New Disk

## Step 1: Check current disks

SSH into the EC2 instance and run:

```bash
lsblk
```

![lsblk output before attaching the new volume](images/01-lsblk-before.png)

## Step 2: Create and attach the EBS volume

Create an EBS volume with these specs:

| Name | Volume ID | Type | Size | IOPS | Throughput |
|------|-----------|------|------|------|------------|
| extradisk | vol-0c925a52dd031c639 | gp3 | 10 GiB | 3000 | 125 |

Attach it to the EC2 instance (**Actions → Attach volume**) using a device name like `/dev/sdf`.

![EBS volume creation and attach screen](images/02-create-attach-volume.png)

Run `lsblk` again. The new disk appears as **`nvme1n1`**.

![lsblk showing the new nvme1n1 disk](images/03-lsblk-after-attach.png)

> ⚠️ **Important:** Never format the existing root disk.

## Step 3: Create a filesystem

`mkfs` formats the disk. Run it **only on a new/empty disk**, or when you intentionally want to erase the existing filesystem.

```bash
sudo mkfs -t xfs /dev/nvme1n1
```

| Part | Meaning |
|------|---------|
| `mkfs` | make filesystem |
| `-t xfs` | use the XFS filesystem |
| `/dev/nvme1n1` | disk to format |

## Step 4: Create a mount point

```bash
sudo mkdir /data
```

## Step 5: Mount the disk

```bash
sudo mount /dev/nvme1n1 /data
```

This connects:

```
/dev/nvme1n1 → XFS → /data
```

Anything created under `/data` is now stored on the new EBS volume. Example:

```bash
cd /data
sudo touch testfile
ls -l
```

`testfile` is now stored on the new disk.

## Step 6: Verify the disk

```bash
df -h
```

| Part | Meaning |
|------|---------|
| `df` | Disk filesystem usage |
| `-h` | Human-readable sizes (G, M, K instead of bytes) |

**Example output:**

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1     10G  104M  9.9G   2% /data
```

| Column | Meaning | Example |
|--------|---------|---------|
| Filesystem | Disk/filesystem being used | `/dev/nvme1n1` |
| Size | Total filesystem size | `10G` |
| Used | Space currently used | `104M` |
| Avail | Available space | `9.9G` |
| Use% | Percentage used | `2%` |
| Mounted on | Directory where it is mounted | `/data` |

## Step 7: Verify filesystem type and UUID

```bash
df -hT /data
```

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/nvme1n1   xfs    10G  104M  9.9G   2% /data
```

| Item | Value |
|------|-------|
| Device | `/dev/nvme1n1` |
| Filesystem type | `xfs` |
| Size | 10G |
| Used | 104M |
| Available | 9.9G |
| Mount point | `/data` |

Now get the UUID:

```bash
lsblk -f
```

| Option | Meaning |
|--------|---------|
| `lsblk` | Lists block devices/disks |
| `-f` | Shows filesystem info (type, label, UUID, mount point) |

**Our new disk:**

```
nvme1n1  xfs  b48917e7-0c1f-47f6-bfcb-e3b990779d1b  /data
```

| Item | Value |
|------|-------|
| Device | `/dev/nvme1n1` |
| Filesystem | XFS |
| UUID | `b48917e7-0c1f-47f6-bfcb-e3b990779d1b` |
| Mount point | `/data` |

> **Why is the UUID important?** It uniquely identifies the filesystem. We use it in `/etc/fstab` so Linux knows which disk to mount at `/data` automatically after a reboot.

## Complete flow so far

| Step | Task | Command / Action | Purpose |
|------|------|------------------|---------|
| 1 | Create EBS volume | AWS Console | Create additional storage |
| 2 | Attach EBS volume | Actions → Attach volume | Attach disk to EC2 |
| 3 | Identify disk | `lsblk` | Find the new disk |
| 4 | Create filesystem | `sudo mkfs -t xfs /dev/nvme1n1` | Create XFS filesystem |
| 5 | Create mount point | `sudo mkdir /data` | Directory for the disk |
| 6 | Mount disk | `sudo mount /dev/nvme1n1 /data` | Make disk accessible via `/data` |
| 7 | Verify storage | `df -h` | Check disk space |
| 8 | Verify filesystem | `df -hT /data` | Check filesystem type |
| 9 | Get UUID | `lsblk -f` | Find filesystem UUID |

---

## Persistent Mount using `/etc/fstab`

A manual `mount` is lost after reboot. To make it permanent:

1. **Get the disk UUID:**
   ```bash
   lsblk -f
   ```
2. **Edit `/etc/fstab`:**
   ```bash
   sudo vi /etc/fstab
   ```
3. **Add this line:**
   ```
   UUID=<disk-uuid>  /data  xfs  defaults,nofail  0  2
   ```
4. **Validate without rebooting:**
   ```bash
   sudo mount -a
   df -hT /data
   ```
5. `/data` will now mount automatically after every reboot.

## Remember for interviews

| Name type | Example |
|-----------|---------|
| AWS device name | `/dev/sdf` |
| Linux device name | `/dev/nvme1n1` |

### 🔥 Interview scenario

**Q: "You attached a 10 GB EBS volume to an EC2 instance. How do you make it persist after reboot?"**

> "I attach the EBS volume, verify it with `lsblk`, create the filesystem if required, mount it to the required directory, retrieve its UUID using `blkid` (or `lsblk -f`), and add the UUID to `/etc/fstab`. I then run `mount -a` to validate the configuration before rebooting."

---

## Troubleshooting: `/data` is 100% Full

1. **Check filesystem usage:**
   ```bash
   df -h /data
   ```
2. **Find which directories consume space:**
   ```bash
   sudo du -sh /data/*
   ```
3. **Find the largest files:**
   ```bash
   sudo find /data -type f -size +1G -exec ls -lh {} \;
   ```
4. **Investigate before deleting.** Then clean/archive files, rotate logs, or expand the EBS volume as appropriate.

---

# Task 2: Vertical Expansion of a Disk

> **Interview point:** EBS volume expansion works for both root and additional volumes, but **LVM-based root volumes need extra steps** compared with a simple standalone XFS data volume.

## LVM vs non-LVM disk

| Type | Layout | Our example |
|------|--------|-------------|
| **LVM-based** | EBS → Partition → LVM (PV → VG → LV) → Filesystem → Mount point | Root disk: `nvme0n1 → nvme0n1p4 → RootVG → rootVol → /` |
| **Non-LVM** | EBS → Disk/Partition → Filesystem → Mount point | Data disk: `nvme1n1 → XFS → /data` |

LVM is useful when you want flexible management and expansion of logical volumes.

## Step 1: Increase the EBS volume size in AWS

**EC2 → Volumes → select root volume → Actions → Modify volume**

Change **20 GiB → 30 GiB**, then wait until the modification is in progress/completed.

![Modify volume screen, 20 GiB to 30 GiB](images/04-modify-volume.png)

## Step 2: Verify Linux sees the larger disk

```bash
lsblk
```

![lsblk showing nvme0n1 at 30G](images/05-lsblk-disk-30g.png)

You should see:

```
nvme0n1        30G
└─nvme0n1p4   19.4G
```

📌 The EBS disk is now 30G, but the **partition is still ~19.4G**.

## Step 3: Grow the partition

The LVM physical volume is on `nvme0n1p4`, so grow partition 4:

```bash
sudo growpart /dev/nvme0n1 4
lsblk
```

![lsblk after growpart, partition grown](images/06-lsblk-after-growpart.png)

`nvme0n1p4` should now be close to 30G.

## Step 4: Resize the LVM physical volume

```bash
sudo pvresize /dev/nvme0n1p4
```

```
Physical volume "/dev/nvme0n1p4" changed
1 physical volume(s) resized or updated / 0 physical volume(s) not resized
```

Check it:

```bash
sudo pvs
```

![pvs output after pvresize](images/07-pvs.png)

## Step 5: Extend the logical volume

First identify it:

```bash
sudo lvs
```

![lvs output showing RootVG/rootVol](images/08-lvs.png)

The root LV is `/dev/mapper/RootVG-rootVol`. Extend it using all available free space:

```bash
sudo lvextend -l +100%FREE /dev/mapper/RootVG-rootVol
```

```
Size of logical volume RootVG/rootVol changed from 6.00 GiB (1536 extents) to 16.00 GiB (4096 extents).
Logical volume RootVG/rootVol successfully resized.
```

## Step 6: Grow the XFS filesystem

The root filesystem is XFS:

```bash
sudo xfs_growfs /
```

![xfs_growfs output](images/09-xfs-growfs.png)

Finally, verify:

```bash
df -h /
```

![df -h / showing the expanded root filesystem](images/10-df-root.png)

## Flow to remember

```
EBS 20G → 30G
      ↓
 Grow partition   (growpart)
      ↓
 pvresize
      ↓
 lvextend
      ↓
 xfs_growfs
      ↓
 Root filesystem expanded
```

This is a great real-world Linux + AWS interview scenario. It tests **EBS, partitions, LVM, and XFS** together.

---

# Quick Revision

## Task 1: Attach a new EBS volume

| # | Action |
|---|--------|
| 1 | Create new EBS volume (e.g., 10 GB) |
| 2 | Attach it to EC2 |
| 3 | `lsblk` → identify the new disk |
| 4 | `sudo mkfs -t xfs /dev/nvme1n1` |
| 5 | `sudo mkdir /data` |
| 6 | `sudo mount /dev/nvme1n1 /data` |
| 7 | `df -h` → verify |
| 8 | `lsblk -f` → get UUID |
| 9 | Add UUID to `/etc/fstab` for persistence |

**Result:** 10 GB → `/data`

```
EBS 10G → nvme1n1 → XFS → /data
```

## Task 2: Vertical expansion of existing EBS (LVM root)

| # | Action |
|---|--------|
| 1 | Modify existing EBS (e.g., 20 GB → 30 GB) |
| 2 | AWS expands the EBS volume |
| 3 | `lsblk` → verify new disk size |
| 4 | `sudo growpart /dev/nvme0n1 4` |
| 5 | `sudo pvresize /dev/nvme0n1p4` |
| 6 | `sudo lvextend -l +100%FREE /dev/mapper/RootVG-rootVol` |
| 7 | `sudo xfs_growfs /` |
| 8 | `df -h /` → verify |

**Result:** Root EBS 20 GB → 30 GB

```
EBS 20G → 30G → growpart → pvresize → lvextend → xfs_growfs
```
