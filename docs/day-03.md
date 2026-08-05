# Day 3 – Expanding My Kubernetes Homelab and Deploying Longhorn

One of the main reasons I chose to build a homelab instead of depending entirely on cloud platforms was that I wanted to experience the kinds of problems that happen in real infrastructure.

Cloud services often hide many of the underlying networking, storage, and hardware decisions. In a homelab, however, every broken connection, failed image pull, resource limitation, and configuration mistake becomes my responsibility.

Day 4 was the first time my Kubernetes environment truly began to feel like a real cluster.

Until this point, my k3s installation had been running entirely on one Ubuntu virtual machine. The machine acted as both the Kubernetes control plane and the only available node for running workloads. This was enough for learning basic commands and deploying simple applications, but it did not allow me to properly observe scheduling across multiple machines or experiment with distributed storage.

My goal for the day was to join the worker VM I had prepared earlier, confirm that Kubernetes could distribute workloads between both nodes, and install Longhorn as the cluster’s storage system.

As expected, the process did not go completely smoothly.

## Joining the Worker Node

The worker virtual machine had already been installed with Ubuntu Server and assigned the static IP address `10.0.0.51`. The control-plane node was running at `10.0.0.50`.

To add the worker to the cluster, I first retrieved the cluster’s node token from the control plane.

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

This token is used to authenticate new nodes before they are allowed to join the cluster.

I then moved to the worker machine and ran the k3s installation command, supplying the control plane’s API address and the token I had just copied.

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://10.0.0.50:6443 \
K3S_TOKEN=<node-token> sh -
```

After the installation completed, I returned to the control plane and checked the cluster nodes.

```bash
kubectl get nodes
```

The output showed both machines:

```text
NAME          STATUS   ROLES           VERSION
k3s-control   Ready    control-plane   v1.36.2+k3s1
k3s-worker1   Ready    <none>          v1.36.2+k3s1
```

Seeing both nodes report `Ready` was one of the most satisfying moments of the project so far.

For the first time, my Kubernetes environment was no longer a single machine running every component by itself. I now had a separate control plane responsible for managing the cluster and a worker node available to run workloads.

## Why `kubectl` Did Not Work on the Worker

After joining the worker, I tried running a Kubernetes command directly from it.

```bash
kubectl get pods
```

Instead of returning the list of Pods, the command failed with the following message:

```text
The connection to the server localhost:8080 was refused
```

My first assumption was that something had gone wrong during the installation.

After investigating, I realised that the worker did not have a Kubernetes administration configuration available to `kubectl`. Without a valid kubeconfig file, `kubectl` did not know the address of the Kubernetes API server or which credentials to use.

The API server runs as part of the control-plane components, but this does not mean `kubectl` can only be used directly on the control-plane machine. It can be used from any workstation that has a valid kubeconfig and network access to the API server.

In my current setup, the control plane already had the correct configuration, so I continued running administrative commands from there.

The worker’s main responsibility was different. It received Pod assignments from the Kubernetes scheduler, ran the required containers, and continuously reported its status back to the control plane.

That cleared up an important misunderstanding I had about the relationship between Kubernetes management tools and worker nodes.

## Watching Kubernetes Schedule Workloads

Once both nodes were connected, I wanted to confirm that Kubernetes was actually using the new worker rather than continuing to place everything on the control plane.

I listed all Pods in the cluster and included their node placement information.

```bash
kubectl get pods -A -o wide
```

The output showed several workloads running on `k3s-worker1`, including Longhorn components, CSI services, engine images, instance managers, and one of the Traefik service load-balancer Pods.

This was my first practical experience of watching the Kubernetes scheduler distribute workloads between different machines.

I had not manually told Kubernetes where each of those Pods should run. The scheduler evaluated the available nodes and placed the workloads automatically based on their requirements and the resources available in the cluster.

That was the point where the environment truly began to feel like Kubernetes rather than simply a collection of containers running on one VM.

## Installing Longhorn

With the worker successfully connected, I moved on to the next objective: distributed storage.

Kubernetes Pods are temporary by design. A Pod may be restarted, deleted, or recreated on another node at any time. This becomes a problem for applications that need to retain data, such as databases, monitoring systems, and file-based services.

Longhorn provides persistent block storage for Kubernetes. It manages storage volumes across the cluster and can replicate volume data between nodes, depending on the configured replica count. This allows application data to remain available when Pods are recreated or moved.

I installed Longhorn using its manifest file.

```bash
kubectl apply -f longhorn.yaml
```

At first, the installation appeared to be progressing normally. Longhorn created its namespace and began deploying the required components.

A few minutes later, however, almost everything started failing.

## When the Installation Began to Fall Apart

I checked the Longhorn namespace to see how the deployment was progressing.

```bash
kubectl get pods -n longhorn-system
```

Instead of seeing healthy Pods, I was met with a long list of errors:

```text
ImagePullBackOff
ErrImagePull
CrashLoopBackOff
```

Several CSI components had failed, engine-image Pods were not starting, and other Longhorn services were repeatedly restarting.

Seeing so many failed Pods at once was intimidating. My first thought was that the Longhorn installation itself had gone wrong and that I might need to remove everything and start again.

Instead of immediately reinstalling it, I decided to investigate the failures one at a time.

## Reading the Pod Events

I selected one of the failed engine-image Pods and described it.

```bash
kubectl describe pod engine-image-...
```

The event output revealed the actual problem:

```text
lookup registry-1.docker.io: Try again
```

Other Pods showed similar errors:

```text
lookup auth.docker.io: Try again
```

Some also reported connection timeouts.

These messages changed the direction of my troubleshooting. The Longhorn containers were not failing because of an internal Longhorn configuration problem. The nodes were having difficulty resolving Docker Hub addresses and downloading container images.

The real problem was the same weak point I had encountered earlier in the homelab: unstable internet connectivity through the USB-tethered connection.

Once I understood that, reinstalling Longhorn no longer made sense. Reinstalling the same manifests would not fix DNS or network connectivity.

## Letting Kubernetes Retry

One of the most important lessons from the day was learning when not to interfere.

Kubernetes does not permanently give up after a failed image pull. It continues retrying, usually with an increasing delay between attempts.

After the upstream network became more stable, the image downloads slowly began succeeding.

The engine-image Pods recovered first. The instance managers followed, and then the CSI components began starting successfully.

I did not need to delete the entire installation or recreate the cluster. Kubernetes continued reconciling the desired state and recovered many of the failed components automatically once the external problem disappeared.

Watching the system repair itself gave me a much better understanding of what Kubernetes reconciliation means in practice.

## The Final CSI Failure

After most of the Longhorn components recovered, one Pod continued restarting:

```text
csi-provisioner
```

Its logs initially showed repeated connection failures.

```text
Still connecting...
context deadline exceeded
```

At that stage, the provisioner was still waiting for the Longhorn CSI driver to become available.

After another restart, the log messages changed:

```text
Detected CSI driver driver.longhorn.io

Started node topology worker

attempting to acquire leader lease...
```

This was a much better sign. The provisioner had detected the CSI driver and was beginning its normal startup process.

A few minutes later, the Pod stopped restarting and entered the `Running` state.

## Confirming That Longhorn Was Healthy

Once the errors had cleared, I checked the Longhorn namespace again.

```bash
kubectl get pods -n longhorn-system
```

This time, the required components were running.

I also checked whether Longhorn recognised both Kubernetes nodes.

```bash
kubectl get nodes.longhorn.io
```

The output showed:

```text
NAME          READY
k3s-control   True
k3s-worker1   True
```

Finally, I verified the status of the Longhorn engine image.

```bash
kubectl get engineimages.longhorn.io
```

The engine image reported:

```text
STATE
deployed
```

At that point, Longhorn was operational across the two-node cluster.

The installation had looked completely broken at one stage, but the root cause was not the storage system itself. It was unreliable DNS and internet connectivity during the container-image download process.

## Increasing the Control Plane’s Memory

While monitoring the installation, I noticed that the control-plane VM had very little memory available.

It had originally been assigned approximately 2 GB of RAM, and Kubernetes, Longhorn, and the other system components were consuming most of it.

Rather than waiting for memory pressure to begin terminating processes or destabilising the cluster, I shut down the virtual machine and increased its memory allocation to 4 GB in Proxmox.

After restarting the VM, I checked its memory.

```bash
free -h
```

The system reported approximately:

```text
Mem: 3.3Gi
```

At first, the amount of used memory still appeared high. I later learned that Linux deliberately uses unused RAM for filesystem caching.

This cached memory improves performance and can be reclaimed when applications need it. Because of that, the `used` value by itself does not always indicate that a Linux system is running out of memory.

The more useful value is `available`, which estimates how much memory can still be allocated without swapping.

After the upgrade, the control plane had much more headroom, making it better prepared for the additional services I planned to install.

## What I Learned

Day 4 was not simply about joining a worker node or installing Longhorn. It forced me to think more carefully about how the different layers of the cluster interact.

I learned that a worker node does not automatically contain the configuration needed to administer the cluster with `kubectl`. Cluster management can be performed from any authorised machine, but it requires a valid kubeconfig and access to the Kubernetes API server.

I also learned that `ImagePullBackOff` does not always mean an image name is incorrect or unavailable. It may be caused by DNS failures, timeouts, authentication problems, rate limits, or general network instability.

The Pod events were much more useful than the status column alone. The status told me that the image pull had failed, but the events explained why.

Another major lesson was that Kubernetes is designed to keep retrying until the actual state matches the desired state. Once the network recovered, many of the failed Longhorn components repaired themselves without needing to be reinstalled.

Finally, I gained a better understanding of Linux memory reporting. A high value in the `used` column is not necessarily a warning sign because Linux uses free RAM as cache. The `available` value provides a more realistic picture of the memory that remains usable.

## Cluster Status at the End of Day 4

By the end of the session, the homelab had two healthy Kubernetes nodes, and Longhorn was running across both of them.

```text
Kubernetes Nodes
────────────────────────────────
k3s-control     Ready
k3s-worker1     Ready

Longhorn Storage
────────────────────────────────
Longhorn nodes          Ready
Engine image            Deployed
CSI components          Running
Instance managers       Running

Control-Plane Resources
────────────────────────────────
Previous RAM            2 GB
New RAM allocation      4 GB
Cluster status          Stable
```

This was the most significant expansion of the homelab so far.

The cluster now had a dedicated worker node, Kubernetes was distributing workloads between machines, and the foundation for persistent storage was in place. More importantly, the problems I encountered gave me practical experience reading events, interpreting logs, identifying network-related failures, and allowing Kubernetes to recover without making unnecessary changes.
