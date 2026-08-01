

# Day 6 – Investigating Severe Storage I/O Bottlenecks and Recovering a Corrupted Kubernetes Control Plane VM

## Objective

Today's goal was to investigate why the Kubernetes control plane virtual machine had become almost unusable. The VM was extremely slow, SSH sessions would freeze shortly after connecting, and Kubernetes commands such as `kubectl` frequently timed out. Since the cluster had previously been functioning correctly, it was necessary to determine whether the issue originated from the underlying hardware, the Proxmox host, the guest operating system, or the Kubernetes services running inside the VM.

---

## Initial Symptoms

The troubleshooting process began by analysing the system's performance metrics. The results immediately indicated that the problem was related to storage rather than CPU processing power.

The CPU spent over **90% of its time in I/O wait**, meaning that instead of executing instructions, it was stalled while waiting for data to be read from storage. At the same time, user CPU utilisation remained below 8%, confirming that the processor itself was not overloaded.

Disk monitoring tools showed that the physical hard drive (`/dev/sda`) and several logical volumes (`dm-3`, `dm-4`, and `dm-6`) were continuously operating at **100% utilisation**. Read latency ranged between approximately **135 ms and 256 ms**, with request queues exceeding **200 outstanding operations**. Nearly all disk activity consisted of read operations while write activity remained almost nonexistent.

These observations suggested that the storage subsystem was unable to keep up with the number of read requests being generated.

---

## Formulating Possible Causes

Based on these performance metrics, several potential causes were considered.

Possible explanations included:

* A failing hard drive
* Insufficient RAM causing excessive disk reads
* Kubernetes components repeatedly scanning data
* Longhorn storage continuously rebuilding volumes
* Container runtime issues
* Filesystem corruption
* An application performing excessive sequential reads

Rather than immediately rebuilding the cluster, each possibility was investigated systematically.

---

# Step 1 – Verifying Memory Usage

The first diagnostic step was to determine whether memory pressure was forcing Linux to constantly retrieve data from disk.

Using the `free -h` command, the Proxmox host reported:

* Approximately **15 GB** of installed RAM
* Roughly **8.5 GB** of free memory
* No swap usage

This confirmed that memory was not the bottleneck. Because plenty of RAM was still available, Linux should have been able to cache frequently accessed files instead of repeatedly reading them from disk.

As a result, insufficient memory was ruled out as the primary cause.

---

# Step 2 – Identifying Disk Activity

Next, the system's disk activity was examined using `iotop`.

The objective was to determine whether a particular process was generating the excessive disk reads observed earlier.

Although disk utilisation remained extremely high, no single user process consistently appeared responsible for the workload. This suggested that the activity was likely occurring deeper within the storage stack, involving kernel processes, virtual machine storage, or filesystem operations.

This shifted attention away from ordinary user applications and toward the virtual machine itself.

---

# Step 3 – Verifying Physical Disk Health

Since the storage subsystem appeared heavily loaded, the next concern was whether the physical hard drive had begun to fail.

SMART diagnostics were collected using:

```bash
smartctl -a /dev/sda
```

The results were encouraging.

The drive successfully passed its SMART health assessment and showed no evidence of impending hardware failure.

Important observations included:

* Zero reallocated sectors
* Zero pending sectors
* Zero uncorrectable sectors
* Zero SATA CRC communication errors
* No SMART errors recorded
* Previous self-tests completed successfully

Although the disk was approximately 15,000 power-on hours old, there were no indications of physical damage.

This allowed hardware failure to be eliminated as the primary cause of the performance problem.

---

# Step 4 – Investigating the Ubuntu Control Plane VM

Attention then shifted to the Kubernetes control plane virtual machine.

The VM would boot successfully but quickly became unresponsive.

SSH sessions consistently froze shortly after login, making it impossible to investigate from inside the guest operating system.

Because remote access was unreliable, the investigation continued directly from the Proxmox host.

The VM configuration was inspected to understand its virtual hardware allocation, including CPU cores, memory allocation, storage configuration, and disk controller type.

---

# Step 5 – Exploring the Guest Disk Structure

The virtual disk belonging to the Ubuntu VM was inspected using:

```bash
fdisk -l /dev/pve/vm-100-disk-0
```

This revealed that the VM contained multiple partitions.

However, attempting to run `fsck` directly against the Linux partition failed because the partition actually contained another LVM physical volume rather than an ext4 filesystem.

This explained why the filesystem could not be checked immediately.

The Ubuntu VM used **nested LVM**, meaning:

* Proxmox LVM stored the virtual disk.
* Inside the virtual disk, Ubuntu created its own LVM volume group.
* The ext4 filesystem existed inside Ubuntu's logical volume.

This required several additional steps before filesystem recovery could begin.

---

# Step 6 – Activating Ubuntu's Internal LVM

The partition mappings were created using:

```bash
kpartx
```

The internal LVM structures were then discovered with:

```bash
pvscan
```

This revealed an additional volume group:

```text
ubuntu-vg
```

The volume group was activated using:

```bash
vgchange -ay ubuntu-vg
```

Once activated, the logical volume containing Ubuntu's root filesystem became available.

This allowed direct access to the guest filesystem without requiring the VM itself to boot successfully.

---

# Step 7 – Performing a Read-Only Filesystem Analysis

Before making any modifications, a read-only filesystem check was performed.

The analysis immediately detected multiple inconsistencies.

Examples included:

* Invalid inode extent trees
* Incorrect block allocation information
* Free block count mismatches
* Incorrect inode counts
* Filesystem metadata inconsistencies

Most importantly, the filesystem check concluded with:

```text
Filesystem still has errors
```

This confirmed that the Ubuntu filesystem had indeed become corrupted.

---

# Step 8 – Repairing the Filesystem

After verifying that the virtual machine was completely powered off, a full repair was initiated using:

```bash
fsck.ext4 -fy /dev/ubuntu-vg/ubuntu-lv
```

The repair process automatically corrected the detected inconsistencies.

The operation repaired:

* Extent trees
* Block allocation metadata
* Free block accounting
* Inode accounting
* Filesystem journal

The repair completed successfully and reported:

```text
***** FILE SYSTEM WAS MODIFIED *****
```

Unlike the previous read-only scan, no remaining filesystem errors were reported after the repair.

This indicated that the filesystem had been restored to a consistent state.

---

# Step 9 – Boot Verification

Once the repair had completed, Ubuntu's internal volume group was safely deactivated before restarting the virtual machine.

The VM successfully booted into Ubuntu without requiring filesystem recovery during startup.

This confirmed that the filesystem repair had been successful.

However, despite the successful repair, the original performance issue remained.

The operating system still became unresponsive shortly after boot, and SSH sessions continued to freeze.

---

# Findings

The investigation successfully ruled out several potential causes.

Confirmed findings included:

* The physical hard drive is healthy.
* The Proxmox host has sufficient available memory.
* The virtual machine's filesystem had become corrupted.
* The filesystem corruption was successfully repaired.
* Ubuntu is now capable of booting normally.

However, the continued freezing after boot indicates that filesystem corruption was not the root cause of the storage bottleneck.

The evidence now suggests that one or more services inside the Ubuntu VM—most likely Kubernetes components such as `k3s`, `etcd`, Longhorn, or another storage-intensive process—are generating excessive disk activity after startup, causing the system to become unresponsive.

---

# Lessons Learned

This troubleshooting session reinforced several important principles of infrastructure administration.

Rather than immediately rebuilding the virtual machine, a structured diagnostic process was followed to eliminate potential causes one by one. Hardware health, memory availability, storage performance, filesystem integrity, and virtual storage configuration were each verified independently.

An especially valuable lesson was learning how to recover a guest operating system directly from the hypervisor. By activating the guest's internal LVM from the Proxmox host, it was possible to inspect and repair the filesystem even though the VM itself could no longer be administered normally.

Although the root cause of the performance issue has not yet been fully identified, the investigation significantly narrowed the scope of the problem. Future troubleshooting will focus on identifying which Kubernetes or storage-related service begins generating excessive disk reads after the operating system finishes booting. This approach avoids unnecessary rebuilding and demonstrates the importance of evidence-based troubleshooting in real-world infrastructure environments.
