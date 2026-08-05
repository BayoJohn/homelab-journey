# Day 4 – Understanding Kubernetes Persistent Storage with Longhorn

After successfully joining the worker node to my k3s cluster and getting Longhorn running, I could have simply moved on to installing another application. However, I realised that having Longhorn installed did not necessarily mean I understood what it was doing.

Seeing all the Longhorn Pods in a `Running` state only confirmed that the software had started successfully. It did not prove that Kubernetes could request storage from it, that Longhorn could create a volume, or that application data would survive after a Pod was deleted.

Today, I decided to slow down and understand one of the most important parts of running applications on Kubernetes: persistent storage.

Until now, most of the workloads I had deployed were stateless. If an Nginx Pod was deleted, Kubernetes could simply create another one without causing any serious problem. The replacement Pod would use the same container image and continue serving the application.

That approach does not work for every application.

Databases, CI/CD platforms, file-sharing systems, object-storage services, and source-code platforms all need somewhere reliable to keep their data. If their files exist only inside a container, deleting the container may also delete everything the application has stored.

My goal for the day was therefore not just to create a storage resource. I wanted to follow the entire process from the moment Kubernetes receives a request for storage to the point where Longhorn creates the actual volume.

## Why Containers Need Persistent Storage

Containers are designed to be temporary.

Kubernetes can stop a container, delete its Pod, move the workload to another node, or create a replacement at any time. This flexibility is one of the reasons Kubernetes is so useful, but it also means that the filesystem inside a container should not be trusted as permanent storage.

For a stateless web server, this is usually not a problem. The Pod can disappear and another one can start using the same image.

For a database, the situation is completely different.

If PostgreSQL stores all its database files only inside the container filesystem, deleting the Pod could mean losing every database, table, and record stored there. The application might restart successfully, but it would return with an empty database.

The same concern applies to applications such as MySQL, MongoDB, Jenkins, Gitea, Nextcloud, and MinIO. Their data needs to exist independently of the containers that use it.

Kubernetes solves this by separating the application from its storage.

A Pod can be deleted and recreated, while the storage volume continues to exist. When the replacement Pod starts, Kubernetes can attach the same volume and make the original data available again.

This was the behaviour I wanted to begin testing with Longhorn.

## Understanding What Longhorn Provides

Longhorn is a distributed block-storage system designed specifically for Kubernetes.

Instead of manually creating a disk every time an application needs storage, Longhorn integrates with Kubernetes and creates volumes automatically. It can also replicate data, rebuild failed replicas, create snapshots, expand volumes, and support backups.

This makes it useful for a homelab because it allows me to experiment with storage concepts that are normally provided by cloud platforms.

For example, managed Kubernetes environments can use storage services such as Amazon EBS, Azure Managed Disks, or Google Persistent Disk. In my own environment, Longhorn performs a similar role by providing block storage from the disks attached to my cluster nodes.

The important part was understanding how Kubernetes communicates with Longhorn.

Kubernetes does not directly understand the internal design of every storage platform. Instead, it uses the Container Storage Interface, commonly called CSI.

Longhorn provides a CSI driver that acts as the connection between Kubernetes and the Longhorn storage system. When Kubernetes needs a volume, it sends the request through the CSI driver, and Longhorn handles the actual creation and management of that storage.

## Following the Kubernetes Storage Chain

One of the most useful things I learned today was that creating storage in Kubernetes involves several connected objects.

The process can be represented like this:

```text
Application
    ↓
Pod
    ↓
PersistentVolumeClaim
    ↓
StorageClass
    ↓
Longhorn CSI Driver
    ↓
PersistentVolume
    ↓
Longhorn Volume
    ↓
Physical Disk
```

Before today, terms such as PersistentVolumeClaim, PersistentVolume, and StorageClass seemed closely related, but I did not fully understand where one ended and another began.

The Pod runs the application, but it does not normally ask Longhorn for storage directly. Instead, it refers to a PersistentVolumeClaim.

A PersistentVolumeClaim, or PVC, is the application’s request for storage. It describes what the application needs, including the capacity, access mode, and sometimes the StorageClass that should be used.

The PVC is not the disk itself. It is only the request.

The StorageClass tells Kubernetes how that request should be fulfilled. It identifies the storage provisioner and contains settings that control how new volumes should be created.

In my cluster, the Longhorn StorageClass uses the Longhorn CSI driver. When Kubernetes sees a PVC that should use Longhorn, it passes the request to `driver.longhorn.io`.

Longhorn then creates the underlying volume, while Kubernetes creates a PersistentVolume object to represent that storage inside the cluster.

The PersistentVolume, or PV, is therefore the Kubernetes representation of the real storage resource that was created to satisfy the claim.

Understanding this chain made the storage process feel much less mysterious.

## Inspecting the Available StorageClasses

I began the practical work by checking the StorageClasses installed in the cluster.

```bash
kubectl get storageclass
```

The cluster contained three StorageClasses:

```text
local-path
longhorn
longhorn-static
```

The `local-path` StorageClass was installed as part of k3s. It provisions storage from the local filesystem of a Kubernetes node.

The `longhorn` StorageClass was created by the Longhorn installation and supports dynamic provisioning through the Longhorn CSI driver.

The `longhorn-static` StorageClass is intended for working with existing Longhorn volumes rather than automatically creating new ones in the same way as the standard Longhorn class.

While inspecting the output, I noticed something unexpected: both `local-path` and `longhorn` were marked as default StorageClasses.

A default StorageClass is automatically selected when a PVC does not explicitly provide a `storageClassName`.

Having more than one default is technically possible, but it makes the behaviour less obvious. Kubernetes may select the most recently created default class for claims that do not name one directly. I wanted storage provisioning in the cluster to be predictable, so I decided that Longhorn should be the only default.

## Making Longhorn the Default Storage Provider

I inspected the StorageClass definitions and confirmed that both contained the default-class annotation:

```text
storageclass.kubernetes.io/is-default-class: "true"
```

I removed the default designation from `local-path` by changing its annotation to `false`.

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

After applying the change, I checked the StorageClasses again.

```bash
kubectl get storageclass
```

Longhorn was now the only class marked as default.

This meant that any future PersistentVolumeClaim created without a `storageClassName` would automatically use Longhorn.

Making this change also helped me understand that installing a new storage system does not automatically mean every application will begin using it. The default StorageClass determines which provisioner handles claims that do not request a specific class.

## Creating My First PersistentVolumeClaim

With Longhorn configured as the only default StorageClass, I created a PersistentVolumeClaim named `demo-pvc`.

The claim requested 2 GiB of storage and used the `ReadWriteOnce` access mode.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

I deliberately left out the `storageClassName`.

This allowed me to test whether Kubernetes would automatically select Longhorn as the default provisioner.

I saved the configuration and applied it to the cluster.

```bash
kubectl apply -f demo-pvc.yaml
```

After creating the claim, I checked its status.

```bash
kubectl get pvc
```

The claim eventually entered the `Bound` state.

That single status represented several operations happening behind the scenes.

Kubernetes accepted the PVC through the API server. It recognised that no existing PersistentVolume satisfied the request. Because the claim did not specify a StorageClass, Kubernetes selected the default Longhorn class.

The Longhorn CSI provisioner then received the request and created a new volume. Kubernetes created a matching PersistentVolume and bound it to `demo-pvc`.

I had only created a claim, but Kubernetes and Longhorn had handled the rest of the process automatically.

That was my first clear demonstration of dynamic storage provisioning.

## Inspecting the PersistentVolume

After confirming that the claim was bound, I checked the PersistentVolume that Kubernetes had created.

```bash
kubectl get pv
```

I then described the volume to view its details.

```bash
kubectl describe pv <persistent-volume-name>
```

Several details confirmed that the request had been handled by Longhorn.

The volume used the CSI driver:

```text
driver.longhorn.io
```

The filesystem type was:

```text
ext4
```

The PersistentVolume also contained a claim reference pointing back to `demo-pvc`.

This showed that the PVC and PV were now connected. The claim represented what the application had requested, while the PersistentVolume represented the storage Kubernetes had allocated to satisfy that request.

The PV name itself had been generated automatically. I did not need to create or name the PersistentVolume manually.

This was one of the main benefits of dynamic provisioning.

## Checking the Volume Inside Longhorn

The next step was to confirm that the volume also existed from Longhorn’s perspective.

I inspected the Longhorn volume resources.

```bash
kubectl get volumes.longhorn.io -n longhorn-system
```

The new volume had been created successfully, but its state was shown as detached.

Its robustness was also reported as unknown.

At first, that looked like another possible failure. However, I realised that no Pod was currently using the claim.

Longhorn had created the storage, but Kubernetes had not asked it to attach the volume to a node. There was no application consuming `demo-pvc`, so there was no reason for the volume to be mounted anywhere.

The detached state was therefore expected.

Once a Pod references the PVC, Kubernetes will schedule the Pod onto a node and request that Longhorn attach the volume to that node. The Longhorn CSI driver will then mount the filesystem so the container can use it.

This helped me understand that creating a PVC and attaching a volume are separate stages.

The storage can exist before any application begins using it.

## Discovering the Replica Configuration

While inspecting the Longhorn volume configuration, I noticed that the default replica count was set to three.

```text
numberOfReplicas = 3
```

My cluster currently contains only two Kubernetes nodes.

Ideally, Longhorn distributes replicas across different nodes so that losing one node does not destroy every copy of the data. With only two nodes, it cannot place three replicas on three separate machines.

Depending on the replica scheduling and anti-affinity settings, Longhorn may be unable to schedule all three replicas as intended. Once the volume is attached and becomes active, this could cause it to report a degraded state because the requested redundancy level has not been fully achieved.

The simplest solution would be to reduce the default replica count to two.

However, I decided not to change it immediately.

I wanted to observe how Longhorn reports an unsatisfied replica requirement. Seeing the volume become degraded would help me understand the difference between a volume being available and a volume having its desired level of redundancy.

A degraded volume may still function, but it does not have all the replicas required by its configuration. That means the application may continue working while the storage system warns that its fault tolerance has been reduced.

This is the kind of behaviour that would be important to recognise in a real production environment.

## What I Learned

Today’s work was less about installing software and more about understanding what happens after the software has been installed.

Before this session, I knew that PVCs were used to request storage, but I did not fully understand how the request travelled through Kubernetes and eventually became a Longhorn volume.

I now understand that a PersistentVolumeClaim is only a request. It describes the storage an application needs but does not represent the actual disk.

The StorageClass determines which storage provider should fulfil that request. The CSI driver provides the standard communication layer between Kubernetes and the storage system.

The PersistentVolume represents the allocated storage inside Kubernetes, while Longhorn manages the actual volume and its replicas on the cluster nodes.

I also learned that a provisioned volume does not need to be attached immediately. Longhorn can create the volume and leave it detached until a Pod requests access to it.

Another important discovery was the effect of having multiple default StorageClasses. The cluster still worked, but the provisioning behaviour was less predictable than I wanted. Making Longhorn the only default gave me clearer control over where future claims would be provisioned.

Finally, I began to understand the relationship between storage availability and redundancy. A volume can remain usable even when its desired replica count has not been fully satisfied, but that condition reduces fault tolerance and should not be ignored.

## Current Storage Status

By the end of the session, Longhorn was the only default StorageClass in the cluster.

The `demo-pvc` claim had successfully requested 2 GiB of storage without explicitly naming a StorageClass. Kubernetes automatically selected Longhorn, created a matching PersistentVolume, and bound the claim to it.

Longhorn also created the underlying volume successfully. It remained detached because no Pod was using it yet.

The storage provisioning process was working as expected.

```text
StorageClass
────────────────────────────────
Default provider        Longhorn

PersistentVolumeClaim
────────────────────────────────
Name                    demo-pvc
Requested capacity      2 GiB
Access mode             ReadWriteOnce
Status                  Bound

PersistentVolume
────────────────────────────────
Provisioner             driver.longhorn.io
Filesystem              ext4
Provisioning type       Dynamic

Longhorn Volume
────────────────────────────────
State                   Detached
Current consumer        None
Default replicas        3
Available nodes         2
```

## Next Steps

The next stage will be to create a test Pod that mounts `demo-pvc`.

The Pod will write a file to the Longhorn volume. I will then delete the Pod and create a replacement that mounts the same claim.

If the file remains available after the original Pod has been removed, it will prove that the data exists independently of the container.

That experiment will move the lesson from storage provisioning to actual data persistence and demonstrate the main reason PersistentVolumes are necessary in Kubernetes.
