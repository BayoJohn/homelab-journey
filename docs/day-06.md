# Day 6 – Investigating Severe Storage I/O Bottlenecks and Recovering a Corrupted Kubernetes Control Plane VM

Today was one of the most difficult troubleshooting sessions I have had in the homelab so far.

The Kubernetes control-plane virtual machine had become almost unusable. SSH connections would work briefly and then freeze, `kubectl` commands regularly timed out, and even simple tasks inside the VM became painfully slow.

The strange part was that the cluster had previously been working correctly. Nothing immediately pointed to a single obvious failure, so I needed to determine whether the problem was coming from the physical hard drive, the Proxmox host, the Ubuntu guest operating system, or one of the Kubernetes services running inside the VM.

My goal was not to start changing random settings or rebuild the cluster immediately. I wanted to follow the evidence and eliminate possible causes one at a time.

## Realising the Problem Was Storage-Related

I began by checking the overall performance of the Proxmox host.

The CPU statistics immediately revealed something unusual. More than 90% of the processor’s time was being spent in I/O wait, while actual user CPU usage remained below 8%.

That meant the processor was not busy doing useful work. It was mostly sitting idle and waiting for storage operations to complete.

The disk statistics made the problem even clearer.

The physical drive, `/dev/sda`, and several device-mapper volumes, including `dm-3`, `dm-4`, and `dm-6`, were constantly reaching 100% utilisation. Read latency was extremely high, ranging from about 135 milliseconds to more than 250 milliseconds.

The request queue also contained more than 200 outstanding operations at certain points.

Almost all the traffic consisted of reads, while write activity remained very low.

At that stage, I knew the VM was not slow because it lacked CPU power. The entire system was being held back by the storage layer.

## Considering the Possible Causes

There were several possible explanations for the behaviour.

The physical hard drive could have been failing. The host might have been running out of memory and constantly reading data from disk. Kubernetes or Longhorn might have been rebuilding or scanning volumes. The container runtime might have entered a loop. The guest filesystem could have become corrupted, or an application could have been performing an unusually large number of reads.

I did not yet know which explanation was correct, so I started with the simplest possibilities.

## Checking for Memory Pressure

The first thing I checked was whether the Proxmox host had enough available memory.

```bash
free -h
```

The host had approximately 15 GB of installed RAM, with around 8.5 GB still free. Swap was not being used.

This was important because a system with insufficient memory may constantly move data between RAM and disk, creating severe storage pressure.

That was not happening here.

The host had more than enough available memory to cache frequently accessed data, so memory pressure could be ruled out as the main cause of the problem.

## Looking for the Process Causing the Reads

I then used `iotop` to see whether one specific process was responsible for the disk activity.

I expected to find a process such as `k3s`, `containerd`, or a Longhorn component reading constantly from storage.

Instead, no single ordinary user process consistently explained the load.

The disks remained heavily utilised, but the activity appeared to be happening deeper in the system. It was likely connected to the virtual disk, kernel-level filesystem work, device-mapper operations, or processes running inside the VM.

This shifted my attention away from the Proxmox host itself and toward the Kubernetes control-plane virtual machine.

## Checking the Physical Disk

Before touching the VM filesystem, I needed to confirm that the physical drive was not failing.

I collected the drive’s SMART information.

```bash
smartctl -a /dev/sda
```

The results were reassuring.

The disk passed its overall SMART health assessment. It had no reallocated sectors, no pending sectors, no uncorrectable sectors, and no SATA CRC errors. The drive had not recorded any SMART failures, and its earlier self-tests had completed successfully.

The disk had accumulated roughly 15,000 power-on hours, so it was not new, but there was no evidence that it was physically damaged.

This did not mean the disk was fast. It was still a mechanical drive and could struggle under heavy random I/O. However, the data did not support the theory that the hardware was actively failing.

At that point, I ruled out physical disk failure as the primary cause.

## Focusing on the Control-Plane VM

The control-plane VM would boot successfully, but shortly afterward it became almost impossible to use.

SSH access was unreliable because the session would freeze before I could complete any meaningful investigation. Since troubleshooting from inside the guest was not practical, I continued from the Proxmox host.

I inspected the VM configuration to understand how much CPU and memory it had, which virtual disk it used, and how the storage was connected.

The next step was to inspect the Ubuntu filesystem directly from the hypervisor.

## Discovering Nested LVM

I first examined the partition structure of the VM’s virtual disk.

```bash
fdisk -l /dev/pve/vm-100-disk-0
```

The disk contained several partitions.

I initially attempted to run a filesystem check against the Linux partition, but `fsck` could not find an ext4 filesystem there.

The reason was that the partition did not contain the root filesystem directly. It contained an LVM physical volume.

This meant the storage layout had two layers of LVM.

Proxmox was already using LVM to store the VM’s virtual disk. Inside that virtual disk, Ubuntu had created another LVM volume group for its own root filesystem.

The structure looked roughly like this:

```text
Physical Hard Drive
        ↓
Proxmox LVM
        ↓
VM Virtual Disk
        ↓
Ubuntu Partition
        ↓
Ubuntu LVM Physical Volume
        ↓
ubuntu-vg
        ↓
ubuntu-lv
        ↓
ext4 Root Filesystem
```

Before I could inspect the ext4 filesystem, I first had to expose the partitions and activate Ubuntu’s internal LVM volumes.

## Accessing Ubuntu’s Internal Volumes

I created device mappings for the partitions inside the virtual disk using `kpartx`.

```bash
kpartx -av /dev/pve/vm-100-disk-0
```

I then scanned the system for LVM physical volumes.

```bash
pvscan
```

The scan discovered the VM’s internal volume group:

```text
ubuntu-vg
```

I activated it with:

```bash
vgchange -ay ubuntu-vg
```

Once the volume group was active, the logical volume containing Ubuntu’s root filesystem became accessible from the Proxmox host.

This was an important moment in the investigation. Even though the guest operating system was too unstable to administer normally, I could now access its root filesystem directly from the hypervisor.

## Running a Read-Only Filesystem Check

Before allowing `fsck` to modify anything, I performed a read-only analysis.

The scan detected several inconsistencies in the filesystem.

The problems included invalid inode extent trees, incorrect block-allocation information, mismatched free-block counts, incorrect inode counts, and other filesystem metadata errors.

The final result was clear:

```text
Filesystem still has errors
```

This confirmed that the Ubuntu root filesystem had become corrupted.

It was the first major fault I had found during the investigation.

However, I still did not know whether the corruption caused the severe I/O problem or whether it was simply another result of the same underlying issue.

## Repairing the Filesystem

Before starting the repair, I verified that the virtual machine was completely powered off. Running a filesystem repair against a mounted or active filesystem could have caused even more damage.

I then started a full automatic repair.

```bash
fsck.ext4 -fy /dev/ubuntu-vg/ubuntu-lv
```

The repair process corrected multiple issues, including extent trees, block allocation metadata, inode accounting, free-block counts, and journal-related inconsistencies.

When it finished, it reported:

```text
***** FILE SYSTEM WAS MODIFIED *****
```

Unlike the earlier read-only scan, the final result did not report any remaining filesystem errors.

The root filesystem was now internally consistent again.

## Booting the VM After Recovery

After the repair, I safely deactivated Ubuntu’s internal volume group and removed the temporary partition mappings.

I then started the VM again.

Ubuntu booted successfully and did not enter emergency filesystem recovery during startup. This confirmed that the repair had worked.

For a moment, it seemed possible that the corruption had been the main cause of the problem.

Unfortunately, the VM became unresponsive again shortly after boot.

SSH sessions continued to freeze, and the original storage bottleneck returned.

The filesystem had definitely been corrupted, and repairing it had been necessary, but the repair did not solve the performance issue.

## What the Investigation Confirmed

By the end of the session, I had ruled out several possible causes.

The Proxmox host had sufficient available memory and was not using swap. The physical hard drive passed its SMART checks and showed no signs of sector failure. The Ubuntu root filesystem had been corrupted, but it was successfully repaired and could boot normally again.

The most important unresolved fact was that the heavy disk activity only returned after the VM had finished booting and its services began starting.

That strongly suggested that something inside the Ubuntu VM was generating the read workload.

The most likely candidates were now Kubernetes and storage-related services such as:

```text
k3s
containerd
etcd
Longhorn
kubelet
```

Another possibility was that a damaged Kubernetes database, container snapshot, or Longhorn volume was being repeatedly read or recovered after startup.

The investigation had not yet identified the exact process, but it had narrowed the problem considerably.

## What I Learned

The most important lesson from today was that rebuilding should not always be the first response to a broken system.

Recreating the control plane might have been faster in the short term, but it would also have hidden the real cause and removed the opportunity to learn how the failure happened.

By following a structured process, I was able to separate several different layers of the problem.

I checked memory before blaming disk pressure. I inspected processes before assuming the VM itself was corrupted. I verified the physical drive before replacing hardware. I then worked through the nested LVM structure and repaired the guest filesystem directly from Proxmox.

Learning how to activate a VM’s internal LVM volumes from the hypervisor was especially valuable. It showed me that a virtual machine can still be investigated and repaired even when the operating system inside it is too unstable to access normally.

I also learned that finding one real problem does not necessarily mean I have found the root cause.

The filesystem corruption was genuine, but fixing it did not stop the storage bottleneck. It was part of the failure, not the complete explanation.

## Current Status

At the end of Day 6, the situation looked like this:

```text
Proxmox Host
────────────────────────────────
Available memory        Healthy
Swap usage              None
CPU bottleneck          No
Storage I/O wait        Extremely high

Physical Disk
────────────────────────────────
SMART status            Passed
Reallocated sectors     0
Pending sectors         0
Uncorrectable sectors   0
Hardware failure        Not detected

Control-Plane VM
────────────────────────────────
Root filesystem         Corrupted
Filesystem repair       Completed
Boot status             Successful
SSH responsiveness      Still unstable
Storage bottleneck      Still present
```

## Next Steps

The next stage of troubleshooting will focus on identifying exactly which service begins generating the excessive disk reads after Ubuntu boots.

I plan to stop the major services individually, beginning with k3s and the container runtime, while monitoring disk utilisation from the Proxmox host.

If stopping k3s causes the I/O wait to fall, I can then investigate its components more closely, including etcd data, containerd snapshots, Kubernetes logs, and Longhorn processes.

I also need to inspect the guest filesystem for unusually large directories, damaged container layers, repeated journal activity, and Longhorn replica or recovery operations.

Today did not completely solve the issue, but it moved the investigation from a broad hardware question to a much narrower problem inside the Kubernetes control-plane VM.

The system can now boot with a repaired filesystem. The remaining task is to identify what begins overwhelming the storage immediately after the operating system starts its services.
