# Day 2 – Expanding My Kubernetes Homelab with Proxmox

After spending Day 1 building the basic network foundation of my homelab, today’s goal was to begin expanding it beyond a single virtual machine.

At this point, I already had an Ubuntu Server VM running k3s as the Kubernetes control plane. However, running everything on one machine would not give me much experience with how a real Kubernetes cluster operates. I wanted to create an additional virtual machine that could eventually join the cluster as a worker node.

Before creating the worker, I decided to confirm that the existing Kubernetes installation was functioning properly by deploying a simple application.

## Testing the Kubernetes Control Plane

The existing control-plane node was running at `10.0.0.50`. To verify that Kubernetes could schedule and run workloads successfully, I created a separate namespace called `demo`.

```bash
kubectl create namespace demo
```

Using a separate namespace allowed me to keep the test application isolated from the default Kubernetes resources.

I then created an Nginx deployment inside the namespace.

```bash
kubectl create deployment nginx \
  --image=nginx \
  -n demo
```

After creating the deployment, I exposed it through a NodePort service.

```bash
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  -n demo
```

When I checked the pod for the first time, it remained in the `ContainerCreating` state. This initially looked like a problem, but Kubernetes was simply downloading the Nginx container image.

After waiting for a short period and checking again, the pod moved into the `Running` state.

That simple deployment confirmed several important things. Kubernetes was able to schedule the pod, the container runtime could download and start images, the control-plane node had internet access, and the k3s installation itself was healthy.

With the first application running successfully, I moved on to the main objective for the day: preparing the cluster for an additional node.

## Planning the Cluster Architecture

At the beginning of the session, the homelab looked like this:

```text
Ubuntu Desktop (10.0.0.1)
        │
        │ Direct Ethernet Connection
        │
Proxmox Host (10.0.0.2)
        │
        │ vmbr0
        │
K3s Control Plane (10.0.0.50)
```

The Ubuntu Desktop machine continued to act as the management workstation and internet gateway. It received internet access through USB tethering and forwarded that connection to the Proxmox environment.

The next stage was to add a worker node alongside the control plane.

```text
Ubuntu Desktop
        │
        │
Proxmox Host
        │
        ├── K3s Control Plane
        │
        └── K3s Worker 1
```

Eventually, I plan to add another worker and expand the cluster further.

```text
Ubuntu Desktop
        │
        │
Proxmox Host
        │
        ├── K3s Control Plane
        ├── K3s Worker 1
        └── K3s Worker 2
```

For now, all the virtual machines are hosted on the same physical Proxmox server. In the future, I would like to introduce another Dell OptiPlex and distribute the nodes across separate physical systems. That would make the environment more realistic and give me experience managing workloads across multiple hosts.

## Creating the First Worker Node

Inside Proxmox, I created a new Ubuntu Server virtual machine that would become the first Kubernetes worker node.

I assigned it two virtual CPU cores, 2 GB of RAM, and a 32 GB virtual disk. For networking, I used a VirtIO network adapter connected to the existing `vmbr0` bridge.

The hostname chosen for the machine was:

```text
k3s-worker1
```

Its planned static IP address was:

```text
10.0.0.51
```

Using another static address was important because Kubernetes nodes need to communicate reliably with one another. I did not want the worker node receiving a different address after a reboot and becoming unreachable from the control plane.

## The First Installation Problem

The first challenge appeared before Ubuntu had even started installing.

When I powered on the virtual machine, it failed to launch the installer. After checking the hardware configuration in Proxmox, I discovered that the Ubuntu Server ISO had not been attached correctly.

I mounted the ISO to the virtual CD/DVD drive and corrected the boot order so that the machine would boot from the installation media first.

After making those changes, the Ubuntu installer started successfully.

It was a small mistake, but it reminded me that many virtualization problems are not caused by the operating system itself. Sometimes the issue is simply an incorrect virtual hardware setting.

## Configuring the Worker Node Network

During installation, the worker node was unable to obtain an IP address automatically.

Because I was not using a DHCP server on the homelab network, this behaviour was expected. I therefore configured the network settings manually.

The worker node was assigned the following configuration:

```text
IP Address: 10.0.0.51/24
Gateway:    10.0.0.1
DNS Server: 8.8.8.8
```

The gateway was set to `10.0.0.1`, which is the address of the Ubuntu Desktop workstation. That machine was still responsible for forwarding traffic from the internal Ethernet network to the USB-tethered internet connection.

The completed network now contained the following systems:

```text
Ubuntu Desktop:        10.0.0.1
Proxmox Host:          10.0.0.2
K3s Control Plane:     10.0.0.50
K3s Worker 1:          10.0.0.51
```

All four systems were located on the same `10.0.0.0/24` subnet and were connected through the Proxmox bridge and the direct Ethernet link.

## Verifying the Installation

Once Ubuntu Server finished installing, I logged into the new virtual machine and began checking its configuration.

I first confirmed the hostname.

```bash
hostname
```

The output showed:

```text
k3s-worker1
```

I then checked the network interfaces.

```bash
ip addr
```

The system correctly displayed the static address:

```text
10.0.0.51
```

Finally, I inspected the routing table to confirm that the default route pointed to the Ubuntu workstation.

```bash
ip route
```

The important route was present:

```text
default via 10.0.0.1
```

From a configuration perspective, everything appeared correct. The worker node had the expected hostname, IP address, subnet, and gateway.

However, once I started testing external connectivity, I discovered that the network was still not completely stable.

## Troubleshooting Internet Connectivity

The worker node could communicate with systems inside the homelab, but internet access was inconsistent.

Package updates were sometimes extremely slow, and DNS lookups occasionally failed. Some network tests worked during one attempt and failed during the next. I also encountered SSL certificate validation errors because the worker node’s system clock was incorrect.

The time issue was similar to the one I had experienced on the control-plane VM during Day 1. When the system date is significantly wrong, HTTPS connections may fail because certificates appear to be either expired or not yet valid.

Normally, a time synchronization service such as Chrony would correct the clock automatically. However, Chrony also needed stable outbound connectivity to contact external time servers. Because the internet connection was unreliable, the machine could not consistently synchronize its clock.

The upstream connection for the entire homelab was still coming from USB tethering on the Ubuntu Desktop machine. Although the routing and NAT configuration allowed the virtual machines to reach the internet, the mobile connection itself introduced some instability.

This meant the virtual machines were configured correctly, but the internet connection they depended on was not always reliable.

For now, the setup was usable, but improving the upstream network would eventually become necessary, especially before deploying services that depend heavily on constant internet access.

## Understanding the Proxmox Bridge

One of the most useful things I learned today was how the Proxmox network bridge connects virtual machines.

The `vmbr0` bridge behaves much like a virtual network switch. The Proxmox host and every virtual machine attached to the bridge can communicate as though they were connected to the same physical switch.

This is why the control plane at `10.0.0.50` and the worker at `10.0.0.51` can communicate directly even though they are separate virtual machines.

The bridge does not automatically provide internet access, DHCP, or DNS. It simply connects the systems at the network layer. Services such as routing, NAT, address assignment, and name resolution must still be provided elsewhere.

In my setup, those responsibilities are currently handled manually, with the Ubuntu Desktop workstation acting as the gateway.

## Current Homelab Topology

By the end of Day 2, the environment looked like this:

```text
Ubuntu Desktop
10.0.0.1
Internet Gateway
        │
        │ Ethernet
        │
Proxmox Host
10.0.0.2
        │
        │ vmbr0
        │
        ├── K3s Control Plane
        │   10.0.0.50
        │
        └── K3s Worker 1
            10.0.0.51
```

The worker node had not yet joined the Kubernetes cluster, but the virtual machine had been created, installed, and assigned a stable network identity.

## What I Learned

Today helped me understand that creating a multi-node Kubernetes environment involves much more than installing k3s on multiple machines.

Before a worker can join the cluster, the underlying infrastructure must already be reliable. The nodes need predictable IP addresses, correct routing, working DNS, synchronized clocks, and stable communication with the control plane.

I also gained a better understanding of Proxmox bridges. They connect virtual machines like a physical network switch, but they do not automatically solve routing or internet-access problems.

Another important lesson was that DNS and certificate errors are not always separate problems. A DNS failure may be caused by unstable network connectivity, while an SSL error may result from an incorrect system clock rather than a broken website or package repository.

Most importantly, I continued practising a structured troubleshooting approach. Instead of treating every error as an isolated issue, I checked the hostname, IP configuration, routing table, local connectivity, internet access, DNS, and system time in order.

## Next Steps

The next task is to install k3s on `k3s-worker1` and join it to the existing control plane.

Once the node has joined successfully, I will confirm that both machines appear when I run:

```bash
kubectl get nodes
```

After that, I want to deploy workloads across the cluster and observe how Kubernetes schedules pods between the control plane and worker node.

I also plan to explore node labels, pod placement, and scheduling behaviour before introducing larger services such as Prometheus, Grafana, and Argo CD.

Day 2 moved the homelab from a single-server Kubernetes setup toward a proper multi-node environment. The worker had not joined the cluster yet, but the infrastructure needed for that next step was now in place.
