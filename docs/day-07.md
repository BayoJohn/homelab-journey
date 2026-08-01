
# Day 3 - Rebuilding the Kubernetes Homelab After a Cluster Failure

## Overview

During the previous stage of the homelab project, the Kubernetes control plane became unstable after several configuration changes and repeated troubleshooting attempts. Although the cluster was partially operational, critical components such as the Kubernetes API, embedded datastore, and worker communication became unreliable.

Rather than continuing to repair an increasingly inconsistent environment, a decision was made to rebuild the cluster from scratch. This approach reduced technical debt, produced a cleaner architecture, and established a more resilient foundation for future workloads.

The objective of this session was not only to restore Kubernetes but also to redesign the deployment process so that future recovery would require only a few minutes through Proxmox snapshots.

---

# Why the Previous Cluster Was Abandoned

The previous deployment experienced multiple issues that accumulated over time.

## Problems Encountered

* Control plane instability after multiple configuration modifications.
* Embedded SQLite datastore inconsistencies requiring manual inspection.
* Virtual machine lock files preventing proper shutdown and deletion.
* Storage volumes remaining attached after VM deletion.
* Multiple networking inconsistencies while rebuilding Kubernetes.
* Difficulty determining whether failures originated from Kubernetes or the underlying virtual machine.

Although several repair procedures were attempted, the environment had become increasingly difficult to trust.

Rather than spending additional time repairing an unstable cluster, a full rebuild was considered the safer engineering decision.

---

# Recovery Strategy

Instead of recreating the previous environment exactly, several improvements were introduced.

## New Design Goals

* Clean Kubernetes installation
* Predictable static IP addresses
* Better VM lifecycle management
* Snapshot-based recovery
* Separation of infrastructure installation into distinct stages
* Validation after each major component

The deployment process was redesigned so that every major milestone could be verified independently before proceeding.

---

# Phase 1 – Removing the Old Environment

The failed control-plane virtual machine was removed from Proxmox.

During deletion another issue appeared:

* VM configuration locks
* Attached LVM volumes
* Locked logical volumes preventing deletion

These were manually released before recreating the virtual machine.

This reinforced an important lesson:

> Infrastructure cleanup is just as important as deployment.

---

# Phase 2 – Building a Fresh Control Plane

A new Ubuntu Server virtual machine was installed.

Configuration included:

| Component  | Value            |
| ---------- | ---------------- |
| Hostname   | k3s-control      |
| IP Address | 10.0.0.50        |
| Gateway    | 10.0.0.1         |
| DNS        | 1.1.1.1, 8.8.8.8 |

SSH connectivity was verified before installing Kubernetes.

---

# Phase 3 – Installing K3s

K3s was installed without the default Traefik ingress controller.

```bash
curl -sfL https://get.k3s.io | \
INSTALL_K3S_EXEC="server --disable traefik" sh -
```

Verification included:

* Node status
* System pods
* Cluster information
* API availability

The node successfully entered the Ready state.

---

# Phase 4 – Worker Node Deployment

Instead of manually reinstalling Ubuntu, the control-plane VM was cloned.

The clone was modified by changing:

* Hostname
* Static IP
* Machine identity
* SSH host keys

The worker joined the cluster using the node token generated on the control plane.

Final cluster state:

```text
k3s-control
k3s-worker1
```

---

# Phase 5 – Validating Scheduling

Before installing additional software, Kubernetes scheduling was tested.

An NGINX deployment was created with two replicas.

Kubernetes automatically scheduled pods across both nodes.

```text
Pod 1 → k3s-control

Pod 2 → k3s-worker1
```

This confirmed:

* Scheduler
* Networking
* Inter-node communication

were functioning correctly.

---

# Phase 6 – Installing Ingress NGINX

The NGINX Ingress Controller was installed.

Initially, image downloads repeatedly failed.

Problems included:

* ImagePullBackOff
* ErrImagePull
* Slow downloads from registry.k8s.io

Rather than reinstalling Kubernetes again, the root cause was investigated.

Eventually the controller images completed downloading successfully.

A simple NGINX application was exposed through Ingress.

The following request succeeded:

```text
Laptop

↓

webtest.local

↓

Ingress Controller

↓

Kubernetes Service

↓

NGINX Pod
```

This verified:

* DNS resolution
* NodePort
* Ingress rules
* Service routing
* Pod networking

---

# Phase 7 – Installing Longhorn

Longhorn was deployed as the cluster storage platform.

Initial deployment experienced:

* CSI components waiting for images
* ImagePullBackOff
* ContainerCreating delays

Rather than assuming installation failure, the cluster was monitored.

Eventually every CSI component became operational.

Healthy Longhorn components included:

* Longhorn Manager
* Longhorn UI
* CSI Provisioner
* CSI Resizer
* CSI Snapshotter
* CSI Attacher
* Instance Manager
* Engine Images

---

# Phase 8 – Installing Argo CD

GitOps capabilities were introduced through Argo CD.

Deployment included:

* Namespace creation
* Official installation manifests
* Ingress configuration
* Local DNS mapping

A login redirect issue occurred because Argo CD was serving HTTPS internally while NGINX Ingress was proxying requests.

The solution involved:

* Running the Argo CD server in insecure mode internally
* Allowing NGINX Ingress to terminate HTTP traffic

After applying the configuration, the Argo CD dashboard became accessible.

---

# Improving Platform Resilience

One of the most significant improvements compared to the previous deployment was the recovery strategy.

Instead of relying solely on backups, Proxmox snapshots were created after each stable milestone.

Current recovery points include:

```text
Snapshot 1

Fresh K3s Installation

↓

Snapshot 2

Ingress + Longhorn

↓

Snapshot 3

Argo CD Installed
```

This dramatically reduces recovery time.

Instead of rebuilding Kubernetes from scratch, the entire environment can now be restored within minutes.

---

# Lessons Learned

Several practical lessons emerged from this rebuild:

* Rebuilding is sometimes faster and safer than prolonged repair.
* Validate each infrastructure layer before installing additional software.
* Image download delays do not necessarily indicate installation failure.
* Cloning virtual machines significantly accelerates worker node deployment.
* Static addressing simplifies Kubernetes administration.
* Proxmox snapshots provide an effective disaster recovery mechanism for homelab environments.
* A staged deployment process makes troubleshooting substantially easier.

---

# Current Architecture

```text
                     Ubuntu Laptop
                           │
                     /etc/hosts
                           │
                    argocd.local
                    webtest.local
                           │
                 NGINX Ingress Controller
                           │
              ┌────────────┴────────────┐
              │                         │
           Argo CD                  Applications
              │
              │
     Kubernetes API Server
              │
      ┌───────┴────────┐
      │                │
k3s-control       k3s-worker1
      │                │
      └────────┬───────┘
               │
          Longhorn Storage
```

# What's Next

With the cluster now stable, the focus shifts from infrastructure creation to GitOps-driven application management. The next phase will involve connecting Argo CD to your Git repository so that application deployments are managed declaratively. From there, you'll begin deploying your portfolio site, Prometheus, Grafana, Gitea, Harbor, Jenkins, and other services entirely through Git, creating a reproducible and production-style Kubernetes workflow.

---
