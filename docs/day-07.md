# Day 7 – Rebuilding My Kubernetes Homelab After a Cluster Failure

After spending the previous session investigating severe storage I/O problems and repairing a corrupted filesystem, I had to make a difficult decision about the future of the cluster.

The control-plane virtual machine could boot again, but the Kubernetes environment itself was no longer reliable. SSH sessions still froze, API requests timed out, worker communication had become inconsistent, and several rounds of troubleshooting had left the system in a state that was increasingly difficult to understand.

I could have continued trying to repair each problem individually. However, even if I managed to make the cluster appear healthy again, I would not have been completely confident that the underlying configuration was clean.

At that point, rebuilding was no longer an admission of failure. It was the safer engineering decision.

My goal for Day 7 was therefore not only to restore Kubernetes but to rebuild the homelab in a more controlled way. I wanted a clean installation, clearly defined deployment stages, validation after every major change, and reliable recovery points that would prevent me from repeating the entire process whenever something broke.

## Why I Stopped Repairing the Previous Cluster

The previous cluster had accumulated several problems over time.

The control plane had become unstable after repeated configuration changes and troubleshooting attempts. The embedded k3s datastore required manual inspection, the VM filesystem had become corrupted, and communication between the control plane and worker node was no longer dependable.

The Proxmox side of the environment had also become messy. VM lock files interfered with shutdown and deletion, logical volumes remained attached after failed operations, and it became increasingly difficult to tell whether each new error came from Kubernetes, Ubuntu, Proxmox, or the storage layer underneath them.

The longer I continued repairing the same environment, the more technical debt I introduced.

Even when one issue was resolved, I could not be certain that another hidden inconsistency would not appear later. A cluster may look healthy because its nodes report `Ready`, but that does not necessarily mean its datastore, storage, networking, and configuration are all trustworthy.

I decided that a clean rebuild would give me something more valuable than another temporary fix: a known-good foundation.

## Redesigning the Recovery Strategy

I did not want to recreate the previous environment in exactly the same way.

The original cluster had grown one component at a time, but I had not created clear recovery points between those stages. When the control plane failed, there was no simple way to return to the last stable version of the infrastructure.

This time, I divided the rebuild into separate phases.

The first phase would contain only a clean Ubuntu Server installation and a working k3s control plane. The next phase would introduce the worker node and validate cluster scheduling. After that, I would add ingress, persistent storage, and finally Argo CD.

Each phase had to be tested before I moved to the next one.

I also decided to create Proxmox snapshots after important milestones. These snapshots would not replace proper application backups, but they would give me a fast way to recover the lab’s virtual machines after a failed configuration change.

Instead of spending hours reinstalling everything, I could return to a recent known-good state and continue from there.

## Removing the Failed Environment

The rebuild began by removing the old control-plane virtual machine.

Even this process presented another challenge.

Proxmox reported configuration locks, and some of the VM’s logical volumes remained attached. These locks prevented the machine from being cleanly removed until I manually released the affected resources.

This was another reminder that infrastructure cleanup deserves the same level of care as infrastructure deployment.

Deleting a VM from the interface does not always mean every related resource has disappeared. Virtual disks, logical volumes, snapshots, and lock files may remain behind and interfere with future deployments.

Once I confirmed that the old VM and its associated storage had been removed correctly, I began creating the replacement control plane.

## Building a Fresh Control Plane

I installed a new Ubuntu Server virtual machine and gave it the same predictable network identity I had used previously.

```text
Hostname:     k3s-control
IP address:   10.0.0.50
Gateway:      10.0.0.1
DNS servers:  1.1.1.1, 8.8.8.8
```

The Ubuntu management machine at `10.0.0.1` continued to serve as the gateway between the homelab network and the internet connection.

Before installing Kubernetes, I verified the basics.

The VM could communicate with the Proxmox host, reach the management workstation, resolve domain names, access the internet, and accept SSH connections.

I deliberately avoided installing additional software until those checks passed.

One of the lessons from the failed environment was that Kubernetes should not be used to hide an unstable operating system or network. If the VM cannot maintain reliable connectivity before Kubernetes is installed, adding Kubernetes will only make troubleshooting more complicated.

## Installing a Clean K3s Control Plane

I installed k3s without its bundled Traefik Ingress Controller.

```bash
curl -sfL https://get.k3s.io | \
INSTALL_K3S_EXEC="server --disable traefik" sh -
```

I disabled Traefik because I planned to install the NGINX Ingress Controller separately. This gave me more control over the ingress layer and prevented two controllers from competing for the same role.

Once the installation completed, I checked the node status.

```bash
kubectl get nodes
```

I also inspected the system Pods.

```bash
kubectl get pods -A
```

The control-plane node entered the `Ready` state, the Kubernetes API responded normally, and the core k3s components were running.

This became the first stable milestone of the new environment.

Before moving forward, I created a Proxmox snapshot of the clean installation. If a later component damaged the cluster, I would no longer need to reinstall Ubuntu and k3s from the beginning.

## Creating the Worker from a Clone

Instead of manually installing another Ubuntu Server VM, I cloned the fresh control-plane machine.

Cloning saved time, but I also knew that the clone could not be used safely without changing the identity it had inherited from the original VM.

I changed the hostname, assigned a new static IP address, regenerated the SSH host keys, and ensured the machine identity was no longer the same as the control plane.

The worker was configured as:

```text
Hostname:    k3s-worker1
IP address:  10.0.0.51
```

I then retrieved the node token from the control plane.

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On the worker, I used the token and the control plane’s API address to join the cluster.

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://10.0.0.50:6443 \
K3S_TOKEN=<node-token> sh -
```

After the installation completed, I returned to the control plane and checked the nodes.

```bash
kubectl get nodes
```

The cluster now contained:

```text
k3s-control
k3s-worker1
```

Both nodes reported `Ready`.

The cluster had been restored, but I did not want to assume that node registration alone meant everything was working.

## Testing Kubernetes Scheduling

Before installing Longhorn, ingress, or any other major platform component, I created a simple NGINX deployment with two replicas.

The purpose was to test whether the scheduler could place workloads across the nodes and whether both machines could run Pods successfully.

When I inspected the deployment using the wide output format, Kubernetes had distributed the replicas across the cluster.

```text
Pod 1 → k3s-control
Pod 2 → k3s-worker1
```

This confirmed that the scheduler was functioning and that both nodes could receive workloads.

It also demonstrated that the control plane and worker could communicate correctly through the cluster network.

The test was intentionally simple. If NGINX could not run reliably across two nodes, there would have been no reason to introduce more complicated services.

Once this test passed, I moved on to the ingress layer.

## Installing the NGINX Ingress Controller

Because Traefik had been disabled during the k3s installation, the cluster did not yet have an Ingress Controller.

I installed the NGINX Ingress Controller to handle HTTP traffic entering the cluster.

The installation did not become healthy immediately.

Several Pods reported:

```text
ImagePullBackOff
ErrImagePull
```

Downloads from `registry.k8s.io` were extremely slow, and some attempts timed out.

This behaviour was familiar from the earlier cluster. The homelab was still dependent on an unstable upstream internet connection, so failed image pulls did not automatically mean the Kubernetes installation or ingress manifest was incorrect.

Instead of deleting the deployment or reinstalling the cluster again, I inspected the Pod events and allowed Kubernetes to continue retrying.

Eventually, the required images finished downloading and the ingress controller became operational.

## Testing the Full Ingress Path

After the controller was running, I deployed a simple NGINX application and exposed it through an Ingress resource.

I configured the hostname:

```text
webtest.local
```

The local request path now looked like this:

```text
Ubuntu Management Workstation
            ↓
       webtest.local
            ↓
   NGINX Ingress Controller
            ↓
      Kubernetes Service
            ↓
         NGINX Pod
```

When I opened `webtest.local` from the management workstation, the NGINX page loaded successfully.

This single test validated several parts of the environment at once.

The local hostname resolved correctly, traffic reached the appropriate Kubernetes node, the ingress controller accepted the request, the Ingress rule matched the hostname, the Service forwarded the traffic, and the Pod returned the response.

The cluster was now doing more than running workloads internally. It could serve applications through a structured ingress layer.

## Reinstalling Longhorn

With networking and ingress confirmed, I began rebuilding the persistent-storage layer.

I installed Longhorn across the two-node cluster.

As before, the deployment initially showed several delayed or failed components. CSI Pods waited for images, engine components remained in `ContainerCreating`, and some workloads reported `ImagePullBackOff`.

This time, I did not interpret every red status as proof that the installation was broken.

I checked the Pod events and confirmed that many of the delays came from slow external image downloads. Kubernetes continued retrying, and the components gradually became healthy as the required images arrived.

Eventually, the Longhorn deployment included healthy instances of its major components:

```text
Longhorn Manager
Longhorn UI
CSI Provisioner
CSI Resizer
CSI Snapshotter
CSI Attacher
Instance Manager
Engine Image
```

Once both Kubernetes nodes were recognised by Longhorn and its CSI components were running, I considered the storage phase complete.

I then created another Proxmox snapshot.

At this point, I had a recovery state containing a clean two-node Kubernetes cluster, a working ingress controller, and distributed storage.

## Introducing Argo CD

The final major component for the day was Argo CD.

The long-term goal of the homelab is to manage applications declaratively through Git. Instead of manually applying every Kubernetes manifest, I want the desired state of the cluster to live in a repository.

Argo CD will continuously compare that desired state with what is actually running and apply changes when necessary.

I created the Argo CD namespace and applied the official installation manifests.

Once the Pods were running, I configured an Ingress resource and added a local DNS mapping so I could reach the dashboard using:

```text
argocd.local
```

The first attempt resulted in a redirect problem.

Argo CD expected to serve HTTPS internally, while the NGINX Ingress Controller was proxying traffic differently. The browser was redirected repeatedly instead of reaching the login page.

To resolve this, I configured the Argo CD server to run in insecure mode inside the cluster.

In this context, insecure mode did not mean the service had been exposed publicly without protection. It meant Argo CD would accept plain HTTP traffic from the internal ingress layer rather than attempting to terminate TLS itself.

The NGINX Ingress Controller could then handle the incoming request and forward it correctly to Argo CD.

After applying the configuration, the dashboard became accessible through `argocd.local`.

That became the next stable milestone, so I created another Proxmox snapshot.

## Building Recovery into the Platform

The most important improvement in this rebuild was not a new Kubernetes application.

It was the recovery process.

The environment now had snapshots representing clear stages of development:

```text
Snapshot 1
Fresh Ubuntu and K3s control plane
        ↓
Snapshot 2
Two-node cluster, NGINX Ingress and Longhorn
        ↓
Snapshot 3
Argo CD installed and accessible
```

If a future experiment damages Argo CD, I can return to the state before it was installed.

If a storage or ingress change destabilises the cluster, I can return to the clean k3s installation without reinstalling Ubuntu.

These snapshots reduce recovery time dramatically, but they are not a complete replacement for proper backups. A snapshot stored on the same physical disk cannot protect the environment if that disk fails.

For the homelab, however, they provide a practical way to recover quickly from configuration mistakes while I continue building a separate backup strategy for application data and cluster configuration.

## What I Learned

The biggest lesson from Day 7 was that rebuilding is sometimes more responsible than continuing to repair.

Troubleshooting remains important, and the previous investigation taught me a great deal about storage, LVM, filesystem recovery, and virtual-machine internals. However, there comes a point where an environment has changed so many times that restoring confidence is more difficult than recreating it.

I also learned the value of validating infrastructure in layers.

I verified Ubuntu networking before installing k3s. I checked the control plane before adding the worker. I tested scheduling before installing ingress. I validated ingress before introducing Longhorn, and I confirmed storage health before installing Argo CD.

This staged approach made each problem easier to isolate because fewer components had changed between successful tests.

Cloning also made worker deployment much faster, but it reinforced the importance of changing machine-specific information. A cloned VM must not retain the same hostname, network address, machine identity, or SSH host keys as its source.

The repeated image-pull delays taught me not to confuse slow external dependencies with failed installations. `ImagePullBackOff` is a symptom, and Pod events are still necessary to determine whether the cause is DNS, connectivity, authentication, rate limiting, or an invalid image.

Most importantly, I learned to include recovery in the design rather than treating it as something to consider only after the next failure.

## Architecture at the End of Day 7

By the end of the rebuild, the homelab architecture looked like this:

```text
                    Ubuntu Management Workstation
                              │
                    Local hostname mappings
                              │
                   ┌──────────┴──────────┐
                   │                     │
              argocd.local          webtest.local
                   │                     │
                   └──────────┬──────────┘
                              │
                  NGINX Ingress Controller
                              │
                   ┌──────────┴──────────┐
                   │                     │
                Argo CD             Applications
                   │
                   │
              Kubernetes API
                   │
             ┌─────┴─────┐
             │           │
       k3s-control   k3s-worker1
             │           │
             └─────┬─────┘
                   │
             Longhorn Storage
```

The platform now had two healthy Kubernetes nodes, working ingress routing, distributed persistent storage, and an accessible Argo CD installation.

## Current Status

```text
Kubernetes
────────────────────────────────
Control plane           Ready
Worker node             Ready
Pod scheduling          Verified
Inter-node networking   Working

Ingress
────────────────────────────────
Controller              NGINX Ingress
Test hostname           webtest.local
External routing        Verified

Storage
────────────────────────────────
Provider                Longhorn
CSI components          Running
Cluster nodes           Recognised

GitOps
────────────────────────────────
Platform                Argo CD
Hostname                argocd.local
Dashboard access        Working

Recovery
────────────────────────────────
Clean k3s snapshot      Created
Ingress/storage state   Created
Argo CD state           Created
```

## Next Steps

With the infrastructure stable again, the next phase will shift from manually creating cluster resources to managing them through Git.

I plan to connect Argo CD to my Git repository and define the applications declaratively. Instead of running `kubectl apply` for every deployment, Argo CD will read the manifests from the repository and keep the cluster synchronised with them.

The first GitOps-managed workload will likely be my portfolio application. After that, I can begin introducing Prometheus, Grafana, Gitea, Harbor, Jenkins, and other services through the same workflow.

Day 7 was not simply a return to where the project had been before the failure. The rebuilt environment is cleaner, easier to understand, and much faster to recover.

The previous cluster taught me how systems fail. This rebuild taught me how to design the next version so that failure is easier to survive.
