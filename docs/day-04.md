# Day 4 – Understanding Kubernetes Persistent Storage with Longhorn

## Overview

Today's objective was not simply to deploy another Kubernetes component but to begin understanding one of the most important concepts in Kubernetes: **persistent storage**.

Up to this point, the cluster had been successfully built and Longhorn had been installed and was running correctly. However, simply having a storage solution installed is not enough. As DevOps engineers, we must verify that the storage platform behaves as expected and understand how Kubernetes interacts with it.

Today's work focused on learning how Kubernetes dynamically provisions storage, how Longhorn integrates with Kubernetes through the Container Storage Interface (CSI), and how PersistentVolumeClaims (PVCs), PersistentVolumes (PVs), and StorageClasses work together.

---

# Objectives

The goals for today's session were:

* Understand why Kubernetes requires persistent storage.
* Learn the difference between ephemeral and persistent storage.
* Examine the StorageClasses configured in the cluster.
* Ensure Longhorn was the default storage provider.
* Create a PersistentVolumeClaim.
* Observe Kubernetes dynamically provision storage through Longhorn.
* Inspect the PersistentVolume created by Kubernetes.
* Verify that Longhorn successfully created the underlying storage volume.
* Understand each layer involved in Kubernetes storage provisioning.

Unlike previous sessions that focused on infrastructure deployment, today's session focused primarily on understanding the architecture behind Kubernetes storage.

---

# Why Persistent Storage Matters

Containers are designed to be **ephemeral**, meaning they can be created, destroyed, and recreated at any time.

When a container is deleted, everything stored inside the container's writable filesystem is lost.

For stateless applications such as web servers or API gateways, this behaviour is acceptable because the application can simply start again without requiring any previous data.

However, many production applications cannot function this way.

Examples include:

* PostgreSQL
* MySQL
* MongoDB
* Gitea
* Nextcloud
* Jenkins
* MinIO

These applications store important data that must survive container restarts, node failures, upgrades, or rescheduling.

If a PostgreSQL Pod is deleted and its database exists only inside the container filesystem, every table and every record would disappear.

To solve this problem, Kubernetes separates compute from storage.

Instead of storing data inside the container, applications store data inside persistent volumes that exist independently of the Pod itself.

This design allows applications to be recreated while keeping their data intact.

---

# What is Longhorn?

Longhorn is a cloud-native distributed block storage platform built specifically for Kubernetes.

It is developed by Rancher and provides dynamically provisioned persistent storage using Kubernetes-native components.

Instead of manually creating storage volumes, Longhorn automatically provisions storage whenever a PersistentVolumeClaim is created.

Longhorn also provides production-grade storage capabilities such as:

* Dynamic volume provisioning
* Volume replication
* Snapshots
* Backups
* Volume expansion
* Replica rebuilding
* Automatic recovery after failures

These features make Longhorn an excellent learning platform because it introduces concepts commonly found in enterprise storage solutions while remaining relatively simple to operate.

In managed Kubernetes services, cloud providers offer similar functionality through services such as:

* AWS Elastic Block Store (EBS)
* Azure Managed Disks
* Google Persistent Disks

Longhorn allows these same concepts to be explored within a homelab environment.

---

# Understanding Kubernetes Storage Components

One of the most valuable lessons today was understanding that Kubernetes storage consists of multiple layers working together.

The storage workflow can be visualised as follows:

Application

↓

Pod

↓

PersistentVolumeClaim (PVC)

↓

StorageClass

↓

Longhorn CSI Driver

↓

PersistentVolume (PV)

↓

Longhorn Volume

↓

Physical Disk

Each component has a specific responsibility.

### Pod

The Pod runs the application.

It does not manage storage directly.

Instead, it requests storage through a PersistentVolumeClaim.

### PersistentVolumeClaim (PVC)

The PVC represents a request for storage.

It specifies requirements such as:

* Capacity
* Access mode
* StorageClass

The PVC does not create storage itself.

Instead, it asks Kubernetes to provide storage that satisfies its requirements.

### StorageClass

The StorageClass defines how Kubernetes should provision storage.

Because Longhorn was configured as the default StorageClass, Kubernetes automatically selected Longhorn whenever a PVC was created without explicitly specifying a storageClassName.

### CSI Driver

The Container Storage Interface (CSI) provides a standard method for Kubernetes to communicate with storage systems.

Rather than Kubernetes understanding every storage vendor individually, it communicates through the CSI standard.

Longhorn implements a CSI driver, allowing Kubernetes to request new storage volumes without needing to understand Longhorn's internal implementation.

### PersistentVolume (PV)

The PersistentVolume represents the actual storage resource allocated by Kubernetes.

Unlike the PVC, which is simply a request, the PV represents the real storage object that satisfies that request.

### Longhorn Volume

Finally, Longhorn creates and manages the actual storage volume on the cluster nodes.

This volume exists independently of the application and can be attached to Pods when required.

---

# Inspecting the StorageClasses

The first practical task was examining the StorageClasses configured within the cluster.

The cluster contained three StorageClasses:

* local-path
* longhorn
* longhorn-static

An unexpected discovery was that both **local-path** and **longhorn** were configured as default StorageClasses.

While Kubernetes can technically handle multiple default StorageClasses, doing so introduces unnecessary ambiguity because Kubernetes must determine which default should be used when a PersistentVolumeClaim does not specify one explicitly.

Production environments typically maintain a single default StorageClass to ensure predictable storage provisioning.

---

# Resolving Multiple Default StorageClasses

Inspection of the StorageClass YAML confirmed that both StorageClasses contained the annotation:

storageclass.kubernetes.io/is-default-class: "true"

The local-path StorageClass was patched to remove its default designation.

After verification, Longhorn became the cluster's only default StorageClass.

This ensures that all future PersistentVolumeClaims automatically use Longhorn unless another StorageClass is explicitly requested.

---

# Creating the PersistentVolumeClaim

A PersistentVolumeClaim named **demo-pvc** was created requesting:

* 2 GiB of storage
* ReadWriteOnce access

No StorageClass was specified.

Because Longhorn was now the default StorageClass, Kubernetes automatically selected Longhorn during provisioning.

This demonstrated Kubernetes' dynamic provisioning capabilities.

---

# Dynamic Provisioning Process

When the PersistentVolumeClaim was created, Kubernetes automatically initiated a sequence of operations.

The PVC was accepted by the Kubernetes API.

The PersistentVolume Controller recognised that no matching PersistentVolume already existed.

The controller contacted the Longhorn CSI provisioner.

The Longhorn CSI driver created a new storage volume.

Kubernetes automatically created a matching PersistentVolume.

Finally, the PersistentVolumeClaim transitioned into the **Bound** state.

This entire process occurred without manually creating a PersistentVolume.

One of Kubernetes' most powerful features is this ability to provision infrastructure dynamically based on application requirements.

---

# Inspecting the PersistentVolume

The newly created PersistentVolume was examined using `kubectl describe pv`.

Important observations included:

* The storage driver was `driver.longhorn.io`.
* The filesystem type was ext4.
* The volume had been provisioned through the Longhorn CSI driver.
* The PersistentVolume had automatically been associated with the PersistentVolumeClaim.

This inspection demonstrated that Kubernetes had successfully translated the storage request into a real storage resource.

---

# Inspecting the Longhorn Volume

The Longhorn custom resources were then inspected.

The new Longhorn volume existed successfully.

However, its state was:

* Detached
* Robustness: Unknown

This was expected behaviour.

Although storage had been provisioned, no Pod was yet using the PersistentVolumeClaim.

Without a consumer, Longhorn had no reason to attach the volume to any node.

The volume therefore remained detached until an application requests access.

---

# Replica Configuration Observation

During inspection of the PersistentVolume configuration, an important production consideration was discovered.

Longhorn was configured with:

numberOfReplicas = 3

However, the cluster currently consists of only two nodes.

Because Longhorn attempts to distribute replicas across different nodes, it cannot satisfy a three-replica configuration using only two nodes.

This will likely result in future volumes reporting a degraded state once attached.

Rather than immediately changing the replica count, the decision was made to leave the default unchanged temporarily.

Observing degraded behaviour provides valuable insight into how distributed storage systems behave when redundancy requirements cannot be fully satisfied.

Understanding these operational behaviours is an important aspect of learning Kubernetes storage administration.

---

# Challenges Encountered

The primary issue encountered today was discovering that two StorageClasses had been configured as defaults.

Although the cluster remained functional, this configuration could lead to unpredictable storage provisioning.

The issue was identified by inspecting the StorageClass resources directly.

After verifying the configuration, the unnecessary default designation was removed from the local-path StorageClass, leaving Longhorn as the single default storage provider.

No storage provisioning failures occurred after this change.

---

# Lessons Learned

Today's session reinforced several important Kubernetes concepts.

Persistent storage is independent of application Pods.

Applications should never rely on container filesystems for important data.

PersistentVolumeClaims represent requests rather than actual storage.

StorageClasses determine how storage is provisioned.

CSI drivers provide a standard interface between Kubernetes and storage systems.

PersistentVolumes are created automatically through dynamic provisioning when supported by the storage backend.

Longhorn volumes remain detached until an application actively mounts them.

Finally, understanding why each component exists is just as important as knowing how to create it.

---

# Next Steps

The next session will focus on using the newly provisioned storage.

A test Pod will mount the PersistentVolumeClaim and write data to the volume.

The Pod will then be deleted and recreated to verify that the data persists independently of the container.

This experiment will demonstrate the core purpose of persistent storage in Kubernetes and validate that Longhorn is functioning correctly within the cluster.
