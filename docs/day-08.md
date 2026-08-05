# Day 8 – Deploying Gitea, Fixing Longhorn Storage, and Building the Monitoring Stack

After rebuilding the Kubernetes cluster and introducing Argo CD, the next step was to begin turning the homelab into a proper self-hosted development platform.

Until now, most of the applications running in the cluster had either been temporary tests or infrastructure components. They helped confirm that Kubernetes networking, ingress, scheduling, and persistent storage were working, but I still needed a central place to store my application code and Kubernetes manifests.

That made Gitea the logical next service to deploy.

The plan appeared straightforward: install Gitea with Helm, store its data on Longhorn, expose it through the NGINX Ingress Controller, and then move on to the monitoring stack.

Instead, the deployment exposed a major weakness in my Longhorn configuration. Gitea, PostgreSQL, and Valkey all depended on persistent volumes, and none of them could start properly until the underlying storage problem was understood and corrected.

By the end of the session, I had not only deployed Gitea but also redesigned the cluster’s storage configuration and introduced a complete monitoring stack with Prometheus, Grafana, Alertmanager, kube-state-metrics, and Node Exporter.

## Why I Chose Gitea

The long-term goal of the homelab is to create a self-hosted DevOps environment where source code, application manifests, deployments, monitoring, and automation can all be managed locally.

Argo CD had already been installed, but it still needed a Git repository containing the desired state of the cluster. I could have connected it to GitHub, but part of the purpose of the project was to understand how these services work when I am responsible for hosting them myself.

Gitea provided the features I needed without requiring the amount of resources associated with larger platforms.

It would give me a central location for storing:

* application source code;
* Kubernetes manifests;
* Helm values files;
* infrastructure configuration;
* GitOps repositories;
* future CI/CD configuration.

Once Gitea was running, Argo CD could use it as the source of truth for application deployments.

## Preparing the Gitea Helm Deployment

I chose the official Gitea Helm chart rather than creating every Kubernetes resource manually.

The chart could deploy Gitea together with the supporting services it required, including PostgreSQL and Valkey. It also supported PersistentVolumeClaims, Kubernetes Services, ingress configuration, and SSH access for Git operations.

I added the Gitea Helm repository and refreshed the available charts.

```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
```

I then created a custom values file named:

```text
gitea-values.yaml
```

The file contained the settings needed for my environment, including Longhorn-backed persistent storage, the internal database, Valkey, ingress configuration, and a NodePort for Git-over-SSH access.

I did not want the Gitea administrator password written directly inside the Helm values file. Instead, I referenced an existing Kubernetes Secret named:

```text
gitea-admin
```

This allowed the chart to retrieve the administrator credentials from Kubernetes without placing the actual password inside the configuration committed to Git.

Once the values file was ready, I started the installation.

```bash
helm install gitea gitea-charts/gitea \
  --namespace gitea \
  --create-namespace \
  --values gitea-values.yaml
```

Helm accepted the deployment and began creating the resources.

That was where the problems started.

## Watching the Deployment Fail

When I checked the Gitea namespace, several Pods were stuck in different failure states.

```text
ContainerCreating
ImagePullBackOff
Init:CrashLoopBackOff
```

The PostgreSQL Pod was unable to start, and the Valkey StatefulSet was not completing its initialization.

Gitea depended on those services. Because the database was unavailable, Gitea’s initialization process continued waiting and restarting.

The failure had created a chain of dependencies:

```text
Persistent storage unavailable
            ↓
PostgreSQL cannot start
            ↓
Valkey initialization incomplete
            ↓
Gitea cannot complete startup
```

At first, the namespace looked like several applications had failed independently. In reality, they were all being affected by the same underlying problem.

## Following the Kubernetes Events

Rather than immediately removing the Helm release, I began inspecting the Pods and their events.

```bash
kubectl describe pod <pod-name> -n gitea
```

The important message was:

```text
AttachVolume.Attach failed
volume is not ready for workloads
```

This showed that Kubernetes had successfully created the workloads and attempted to start them. The problem occurred when the cluster tried to attach the persistent volumes.

I checked the PersistentVolumeClaims.

The claims existed and had been bound to PersistentVolumes, so Kubernetes had accepted the storage requests. However, the underlying Longhorn volumes were not becoming available to the Pods.

I then inspected the Longhorn volume resources directly.

Several of the new volumes remained in a state similar to:

```text
State:       Detached
Robustness:  Unknown
```

Longhorn had created the volume objects, but it could not prepare them for use by the workloads.

## Finding the Replica Mismatch

The breakthrough came when I inspected the replica configuration assigned to the new volumes.

Each volume had been created with:

```text
numberOfReplicas = 3
```

My Kubernetes cluster contained only two nodes:

```text
k3s-control
k3s-worker1
```

Longhorn was therefore being asked to place three replicas across an environment with only two available Kubernetes nodes.

With replica placement rules preventing every replica from being placed successfully, Longhorn repeatedly reported insufficient storage or an inability to schedule the required replicas.

The message initially made me think the disks had run out of capacity.

However, the problem was not the total number of free gigabytes. The cluster lacked enough eligible nodes to satisfy the requested replica layout.

This was an important distinction:

```text
Disk capacity problem
    = Not enough free space

Replica scheduling problem
    = Not enough eligible locations for the requested replicas
```

The default Longhorn configuration did not match the size of my cluster.

## Creating a StorageClass for the Homelab

I did not want to disable replication completely by reducing every volume to a single replica.

Instead, I created a new StorageClass designed specifically for the two-node environment.

I named it:

```text
longhorn-two-replicas
```

The important setting was:

```yaml
parameters:
  numberOfReplicas: "2"
```

This allowed Longhorn to maintain two replicas, with one scheduled on each Kubernetes node where possible.

The new configuration provided node-level redundancy while matching the infrastructure that was actually available.

However, there was an important limitation.

Both Kubernetes nodes were still virtual machines running on the same Proxmox host and physical storage device. Two Longhorn replicas could help if one VM, Kubernetes node, or replica failed, but they would not protect the data if the entire Proxmox host or its physical disk failed.

True physical redundancy would require placing the replicas on nodes hosted by separate physical machines with independent storage.

For the current homelab, the two-replica configuration was still a major improvement because it removed the impossible three-replica requirement and allowed stateful workloads to operate correctly.

## Removing the Failed Gitea Release

The original Gitea deployment had already created several failed resources and unusable storage volumes.

Rather than trying to modify every resource in place, I removed the failed Helm release and cleaned up the associated workload resources.

I then updated `gitea-values.yaml` so that the stateful components explicitly requested the new StorageClass.

```yaml
storageClass: longhorn-two-replicas
```

I applied the setting to the components that required persistent storage, particularly Gitea and PostgreSQL.

I also reviewed the requested volume sizes. The default allocations were larger than necessary for the current stage of the homelab, so I adjusted them to more reasonable values while leaving enough space for repositories, configuration data, and the database.

Once the configuration had been corrected, I deployed the chart again.

## Watching the Second Deployment Succeed

The second installation behaved very differently.

Kubernetes created new PersistentVolumeClaims using `longhorn-two-replicas`. Longhorn scheduled the required replicas, prepared the volumes, and attached them to the correct nodes.

PostgreSQL was now able to mount its data volume and start.

Valkey completed its initialization.

Gitea connected to its database and completed its own startup process.

When I checked the namespace again, the application components had entered the `Running` state.

The failure had not been caused by Gitea, PostgreSQL, Helm, or Kubernetes. It was caused by a storage policy that did not match the number of nodes available in the cluster.

Correcting the StorageClass solved the entire dependency chain.

## Dealing with More Image-Pull Problems

Storage was not the only issue encountered during the deployment.

Some containers also entered:

```text
ImagePullBackOff
```

The Pod events showed errors such as:

```text
lookup auth.docker.io: Try again
```

These messages pointed to the same intermittent DNS and internet connectivity problems I had experienced during earlier installations.

The nodes were occasionally unable to resolve Docker Hub endpoints or complete image downloads through the homelab’s shared internet connection.

This time, I did not mistake the errors for incorrect image names or broken Helm configuration.

Kubernetes continued retrying the downloads. Once the network became stable, the images were successfully pulled and cached on the nodes.

Where necessary, I recreated the affected Pods after confirming that the images were present locally. Kubernetes then started the containers using the cached images.

The experience reinforced the difference between a permanent configuration failure and a temporary external dependency problem.

Reinstalling the entire application would not have fixed DNS resolution.

## Exposing Gitea Through NGINX Ingress

Once all the Gitea components were healthy, I exposed the web interface through the NGINX Ingress Controller.

The internal hostname was:

```text
gitea.local
```

I updated the `/etc/hosts` file on my Ubuntu management workstation so that `gitea.local` resolved to the IP address through which the ingress controller was reachable.

The request path now looked like this:

```text
Ubuntu Management Workstation
            ↓
        gitea.local
            ↓
 NGINX Ingress Controller
            ↓
       Gitea Service
            ↓
        Gitea Pod
```

Opening the address in the browser displayed the Gitea interface.

Git operations over SSH were exposed separately through a NodePort service. This allowed the web dashboard to use normal HTTP ingress routing while SSH traffic reached the appropriate Gitea service port directly.

At this point, the homelab had its own working Git platform.

I could now create repositories locally and begin preparing the repository Argo CD would use to manage applications.

## Introducing Cluster Monitoring

After Gitea was stable, I moved on to observability.

The cluster was now running multiple important services, including Kubernetes system components, Longhorn, NGINX Ingress, Argo CD, Gitea, PostgreSQL, and Valkey.

Manually checking each Pod with `kubectl` was no longer enough.

I needed a way to observe:

* CPU and memory usage;
* node health;
* Pod status;
* storage consumption;
* PersistentVolume behaviour;
* Kubernetes object states;
* service availability;
* resource pressure across the cluster.

Instead of installing each monitoring component separately, I chose the `kube-prometheus-stack` Helm chart.

The chart bundles several widely used Kubernetes monitoring tools into one installation.

I added the Prometheus Community Helm repository and then installed the chart into a dedicated namespace.

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

The deployment introduced several components.

Prometheus would collect and store metrics.

Grafana would provide dashboards and visualisations.

Alertmanager would process and route alerts.

kube-state-metrics would expose information about Kubernetes objects such as Deployments, Pods, StatefulSets, and nodes.

Node Exporter would run across the cluster and collect operating-system and hardware metrics from each node.

## Monitoring Installation Problems

The monitoring deployment also encountered image-pull failures.

Several Pods initially reported:

```text
ErrImagePull
ImagePullBackOff
```

Once again, I inspected the events rather than assuming the Helm chart was broken.

The failures were caused by temporary DNS resolution and registry connectivity issues.

Some of the monitoring images were larger than the earlier test images, so the unreliable connection made the installation appear stuck for a long time.

I allowed Kubernetes to continue retrying.

Where an image had completed downloading but the existing Pod was still waiting in a backoff cycle, I recreated the Pod so Kubernetes could start it using the cached image.

Gradually, the monitoring components moved into the `Running` state.

I avoided removing and reinstalling the Helm release because the evidence showed that the resources were configured correctly. The problem was simply obtaining the required images.

## Exposing Grafana

Once the monitoring stack became healthy, I exposed Grafana through the existing NGINX Ingress Controller.

I assigned it the internal hostname:

```text
grafana.local
```

The hostname was added to the management workstation’s `/etc/hosts` file and pointed to the same reachable ingress address used by the other internal services.

Opening `grafana.local` displayed the Grafana login page.

For the first time, I could inspect the cluster through dashboards rather than relying entirely on command-line output.

Prometheus began collecting metrics, Node Exporter reported data from both Kubernetes nodes, and kube-state-metrics exposed information about the resources running inside the cluster.

The homelab now had a proper observability layer.

## What I Learned

Day 8 showed me how quickly a stateful application can expose weaknesses that are not obvious when running simple stateless workloads.

NGINX could run without persistent storage, so it never tested the replica configuration in a meaningful way. Gitea, PostgreSQL, and Valkey all required volumes, and their dependency on storage immediately revealed that the Longhorn defaults did not match my cluster.

I learned that a PersistentVolumeClaim being `Bound` does not guarantee that an application can use the volume. Kubernetes may successfully bind the claim while the storage platform still struggles to schedule replicas, attach the volume, or make it ready for a workload.

I also learned to interpret “insufficient storage” more carefully. It does not always mean the disks are full. In distributed storage systems, it may mean that the requested replicas cannot be placed according to the scheduling rules.

Creating a separate StorageClass was better than changing settings blindly for every volume. It gave me a reusable policy designed around the actual size of the cluster.

The monitoring installation reinforced another lesson from previous sessions: `ImagePullBackOff` describes the result of a failed image pull, not its cause. Pod events were still necessary to distinguish DNS failures from invalid images, authentication errors, or registry problems.

Most importantly, I could now see how the services were beginning to depend on one another.

```text
Gitea
  ↓ depends on
PostgreSQL and Valkey
  ↓ depend on
PersistentVolumeClaims
  ↓ depend on
Longhorn
  ↓ depends on
Healthy Kubernetes nodes and storage
```

A failure at the storage layer could prevent the entire application stack from starting.

## Platform Status at the End of Day 8

By the end of the session, the homelab had become much more than a basic Kubernetes cluster.

```text
Gitea
────────────────────────────────
Deployment method       Helm
Web hostname            gitea.local
Git SSH access          NodePort
Database                PostgreSQL
Cache                   Valkey
Persistent storage      Longhorn
Status                  Running

Longhorn
────────────────────────────────
Custom StorageClass     longhorn-two-replicas
Replica count           2
Kubernetes nodes        2
Volume provisioning     Working
Volume attachment       Working

Monitoring
────────────────────────────────
Prometheus              Running
Grafana                 Running
Alertmanager            Running
kube-state-metrics      Running
Node Exporter           Running on both nodes
Grafana hostname        grafana.local

Platform Services
────────────────────────────────
NGINX Ingress           Operational
Argo CD                 Operational
Gitea                   Operational
Monitoring stack        Operational
```

## Next Steps

The next stage will connect Argo CD to Gitea.

I plan to create a Git repository containing Kubernetes manifests and application definitions, then configure Argo CD to monitor that repository.

Once the connection is complete, changes pushed to Gitea will become the desired state of the cluster. Argo CD will detect those changes and synchronise the applications automatically.

This will move the homelab away from manually applying resources and toward a proper GitOps workflow.

I also plan to begin using Prometheus and Grafana to monitor the services already deployed, especially Kubernetes node health, Longhorn volume status, storage capacity, and application resource consumption.

Day 8 began as a Gitea installation, but it became a much deeper lesson in distributed storage and observability.

The cluster now has a self-hosted Git service, a storage policy designed around its actual size, and a monitoring platform capable of showing what is happening across the environment. Those components provide the foundation for the next stage: managing the entire platform through Git.
