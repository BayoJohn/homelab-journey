# Day 2 – Building a Multi-Node Kubernetes Cluster with Proxmox

## Overview

Today was focused on taking the homelab beyond a single virtual machine. Instead of running everything on one Ubuntu VM, I started building a proper Kubernetes cluster inside Proxmox by creating additional virtual machines that will act as worker nodes.

Along the way I also solved several networking issues between my Ubuntu PC, the Proxmox host, and the Kubernetes virtual machines.

---

## Objectives

* Deploy a sample application on Kubernetes
* Create a second Ubuntu VM
* Prepare the cluster for multiple nodes
* Troubleshoot networking
* Understand how Proxmox bridges virtual machines together

---

## Existing Infrastructure

At the beginning of Day 2 my setup looked like this:

```
Ubuntu Desktop (10.0.0.1)
        │
        │ Ethernet
        │
Proxmox Host (10.0.0.2)
        │
     vmbr0 Bridge
        │
Ubuntu VM (10.0.0.50)
Control Plane
```

The control plane was already running Kubernetes (k3s).

---

# Deploying the First Kubernetes Application

To verify the cluster was functioning correctly, I created a new namespace.

```bash
kubectl create namespace demo
```

Next, I deployed Nginx.

```bash
kubectl create deployment nginx \
  --image=nginx \
  -n demo
```

Then exposed it using a NodePort service.

```bash
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  -n demo
```

Initially the pod remained in the `ContainerCreating` state while the image was being downloaded.

After waiting a short while, Kubernetes successfully pulled the image and the pod entered the **Running** state.

This confirmed that:

* Kubernetes scheduling worked
* Container runtime worked
* Internet connectivity was functioning
* The control plane was healthy

---

# Planning the Cluster

Rather than leaving the homelab as a single-node cluster, I decided to build a proper Kubernetes environment.

Planned architecture:

```
Ubuntu Desktop
        │
        │
Proxmox
   │
   ├──────────────┐
   │              │
Control Plane   Worker 1
```

Eventually it will become:

```
Ubuntu Desktop
        │
        │
Proxmox
   │
   ├──────────────┐
   │              │
Control Plane   Worker 1
                  │
              Worker 2
```

When I purchase another Dell OptiPlex, the cluster can grow further by distributing workloads across multiple physical machines.

---

# Creating the First Worker Node

A new Ubuntu Server virtual machine was created inside Proxmox.

VM configuration:

* 2 vCPUs
* 2 GB RAM
* 32 GB Disk
* VirtIO Network Adapter
* Connected to `vmbr0`

Hostname:

```
k3s-worker1
```

IP Address:

```
10.0.0.51
```

---

# Installation Challenges

During installation, the virtual machine initially failed to boot because the Ubuntu installation ISO was not attached correctly.

After correcting the boot order and mounting the ISO, the installer launched successfully.

---

# Network Configuration

The installer was unable to obtain an IP address automatically.

Instead of using DHCP, I configured the worker node manually.

Configuration:

```
IP Address:
10.0.0.51/24

Gateway:
10.0.0.1

DNS:
8.8.8.8
```

This placed the worker node on the same subnet as:

* Ubuntu Desktop
* Proxmox
* Kubernetes Control Plane

---

# Verifying Connectivity

After installation I confirmed the worker node received the correct configuration.

```
hostname
```

Output:

```
k3s-worker1
```

Checking the IP:

```
ip addr
```

Result:

```
10.0.0.51
```

Routing table:

```
default via 10.0.0.1
```

Everything appeared correct.

---

# Internet Troubleshooting

Although the network configuration looked correct, internet access from the worker node was inconsistent.

Observed symptoms included:

* Slow package updates
* DNS resolution failures
* ICMP traffic behaving inconsistently
* SSL certificate validation failures due to incorrect system time

The issue was traced to the upstream internet connection being shared from the Ubuntu desktop over USB tethering, which introduced intermittent connectivity.

Time synchronization (Chrony) also could not synchronize because outbound connectivity was unreliable.

This remains an area for future improvement as the homelab evolves.

---

# Lessons Learned

Today reinforced several important concepts:

* Proxmox bridges behave like physical switches.
* Kubernetes nodes require reliable networking before joining a cluster.
* DNS problems often originate from general network connectivity issues.
* Incorrect system time can cause HTTPS and certificate validation failures.
* Building a homelab involves significant troubleshooting beyond simply installing software.

---

# Current Topology

```
Ubuntu Desktop (10.0.0.1)
        │
        │
Proxmox Host (10.0.0.2)
        │
     vmbr0
   ├──────────────┐
   │              │
Control Plane   Worker 1
10.0.0.50       10.0.0.51
```

---

# Next Steps

* Join `k3s-worker1` to the Kubernetes cluster.
* Verify both nodes appear in `kubectl get nodes`.
* Deploy workloads across multiple nodes.
* Explore pod scheduling and node placement.
* Begin deploying production-ready services such as Prometheus, Grafana, and Argo CD onto the multi-node cluster.

---

