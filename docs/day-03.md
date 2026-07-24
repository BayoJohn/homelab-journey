# Day 4 — Expanding My Kubernetes Homelab: Joining a Worker Node and Deploying Longhorn

One of the reasons I decided to build a homelab instead of relying entirely on cloud providers was to experience the problems that happen in real infrastructure. Yesterday was the first day my cluster actually started behaving like a real production environment.

Up until now, my Kubernetes setup consisted of a single K3s control plane running on one Ubuntu virtual machine. While this worked perfectly for learning Kubernetes basics, it wasn't enough to understand how workloads move between machines or how distributed storage works.

The goal for the day was simple:

* Add a second machine as a worker node
* Verify that the cluster could schedule workloads across multiple nodes
* Install Longhorn as distributed storage
* Fix whatever broke along the way

As expected, quite a few things broke.

---

## Expanding the Cluster

The worker VM had already been prepared with Ubuntu 26.04 and network connectivity.

On the control plane, I retrieved the node token:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Then, on the worker machine, I joined it to the cluster:

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://10.0.0.50:6443 \
K3S_TOKEN=<node-token> sh -
```

A few moments later I verified everything from the control plane.

```bash
kubectl get nodes
```

Output:

```text
NAME          STATUS   ROLES           VERSION
k3s-control   Ready    control-plane   v1.36.2+k3s1
k3s-worker1   Ready    <none>          v1.36.2+k3s1
```

Seeing two nodes report **Ready** was a satisfying moment. For the first time my homelab wasn't just a single machine pretending to be Kubernetes—it was an actual cluster.

---

## Understanding the Role of the Worker

One thing that initially confused me was why I couldn't use `kubectl` directly on the worker.

Running:

```bash
kubectl get pods
```

returned:

```text
The connection to the server localhost:8080 was refused
```

At first I thought something was wrong.

After digging into it, I learned something important about K3s architecture.

The worker node isn't responsible for managing the Kubernetes cluster.

Instead, it simply:

* receives Pods from the scheduler,
* runs containers,
* reports its health back to the control plane.

The Kubernetes API server only exists on the control plane.

So every management command should be executed there.

That cleared up one misconception I'd been carrying for a while.

---

## Verifying Scheduling

To confirm that Kubernetes was actually making use of both nodes, I listed every Pod together with the node it was running on.

```bash
kubectl get pods -A -o wide
```

Filtering for the worker produced output similar to:

```text
longhorn-manager
longhorn-ui
csi-attacher
engine-image
instance-manager
traefik service load balancer
...
```

This showed that Kubernetes was already distributing workloads across both machines automatically.

That was my first real experience seeing the scheduler in action.

---

# Installing Longhorn

With the cluster functioning properly, the next task was installing Longhorn.

Longhorn provides distributed block storage for Kubernetes and allows Persistent Volumes to survive Pod rescheduling and node failures.

Installation itself was straightforward.

```bash
kubectl apply -f longhorn.yaml
```

Initially everything looked healthy.

Then, within a few minutes, almost every Longhorn Pod began failing.

---

# When Everything Started Breaking

Running:

```bash
kubectl get pods -n longhorn-system
```

showed a wall of errors.

```text
ImagePullBackOff

ErrImagePull

CrashLoopBackOff
```

Almost every CSI component had failed.

Seeing dozens of red Pods at once was honestly intimidating.

Instead of reinstalling everything, I started investigating each failure individually.

---

## Reading the Events

The first thing I checked was the events for one of the failing Pods.

```bash
kubectl describe pod engine-image-...
```

The error immediately stood out.

```text
lookup registry-1.docker.io: Try again
```

Other Pods showed:

```text
lookup auth.docker.io: Try again
```

and

```text
connection timed out
```

These weren't Longhorn problems.

They were DNS and network problems while pulling container images from Docker Hub.

---

## Waiting Instead of Panicking

This taught me something valuable.

Kubernetes will continuously retry failed image pulls.

After my network stabilized, the failed image downloads slowly began succeeding.

One by one the Pods recovered.

First the engine images.

Then the instance managers.

Then the CSI plugins.

Eventually nearly everything became healthy without reinstalling Longhorn.

---

## The Last Remaining Failure

One Pod kept restarting.

```
csi-provisioner
```

Its logs initially looked like this:

```text
Still connecting...
context deadline exceeded
```

Later, after another restart, the logs changed completely.

```text
Detected CSI driver driver.longhorn.io

Started node topology worker

attempting to acquire leader lease...
```

That told me the CSI driver had finally started correctly.

A few minutes later the Pod settled into Running state.

---

## Verifying Longhorn

Finally I confirmed the installation.

```bash
kubectl get pods -n longhorn-system
```

Every component eventually reported:

```text
Running
```

I also verified the Longhorn nodes.

```bash
kubectl get nodes.longhorn.io
```

Output:

```text
NAME          READY
k3s-control   True
k3s-worker1   True
```

The engine image was also deployed successfully.

```bash
kubectl get engineimages.longhorn.io
```

```text
STATE
deployed
```

At that point the storage system was fully operational across both nodes.

---

# Memory Upgrade

During the installation I noticed something concerning.

The control plane only had about **1.6 GB of RAM**, and Longhorn plus Kubernetes system components were consuming most of it.

Instead of waiting until resource pressure caused failures, I shut down the VM and increased its memory allocation to **4 GB** in Proxmox.

After booting again:

```bash
free -h
```

showed:

```text
Mem: 3.3Gi
```

The usage initially looked high, but after learning more about Linux memory management I realized most of the "used" memory was actually filesystem cache.

Linux intentionally uses available RAM for caching because unused RAM is wasted RAM.

The important value isn't **Used**, but **Available**, which now had plenty of headroom.

---

# Lessons Learned

Yesterday wasn't really about installing Longhorn.

It was about learning how to think like a systems engineer.

A few things stood out:

* A worker node isn't managed directly with `kubectl`; it simply runs workloads assigned by the control plane.
* ImagePullBackOff doesn't necessarily mean an image is missing—it can simply indicate DNS or network problems.
* Reading Pod events is often more useful than immediately reinstalling software.
* Kubernetes is remarkably resilient. Once network connectivity returned, many of the failed components recovered on their own.
* Linux memory usage can look alarming if you only look at the "Used" column. The "Available" value tells a much more accurate story.

---

## Cluster Status at the End of the Day

```text
Nodes:
- k3s-control
- k3s-worker1

Cluster Status:
✓ Both nodes Ready

Storage:
✓ Longhorn installed
✓ Engine images deployed
✓ CSI components healthy

Resources:
✓ Control plane upgraded from 2 GB → 4 GB RAM
✓ Cluster stable
```
