# Day 9 – The Day My Homelab Started Feeling Like a Platform

Today started with a small annoyance.

Every service in the homelab had its own address, and every time I wanted to check something, I had to remember where it lived. Grafana was open in one tab, Gitea in another, Longhorn somewhere else, and Prometheus usually buried among several terminal windows.

Nothing was technically wrong with that setup, but it still felt unfinished. I had built several useful services, yet there was no simple way to move between them.

So the first thing I did was deploy Homepage.

Homepage is not the most complex application in the cluster, and it did not teach me a new Kubernetes concept. Still, it changed the way the entire environment felt.

Instead of treating each service as a separate project, I could now open one dashboard and reach Gitea, Grafana, Longhorn, Argo CD, Prometheus, and the rest of the platform from the same place.

It was a small improvement, but it made the homelab feel organised for the first time. There was finally a front door.

I could have stopped there and considered the day productive.

Instead, I decided to shut down one of the nodes.

## Testing What Happens When a Node Disappears

I had spent several days building a two-node Kubernetes cluster, but I had never properly tested what would happen if one of those nodes failed.

I understood the theory. Kubernetes constantly watches the state of its nodes. When a node becomes unavailable, workloads managed by controllers such as Deployments can eventually be recreated elsewhere.

I had read about that behaviour many times, but I had not watched it happen in my own environment.

The worker node was running several cluster services and application workloads, so shutting it down felt uncomfortable. That was exactly why I wanted to do it.

I powered off `k3s-worker1` and began watching the cluster.

After a short period, Kubernetes marked the worker as:

```text
NotReady
```

Pods that had been running there became unavailable. For workloads controlled by Deployments, Kubernetes eventually began creating replacement Pods on the remaining control-plane node.

Watching that happen made the scheduler and controller logic feel much less abstract.

The cluster was not simply “moving” the old containers. It was recognising that the desired number of replicas no longer existed and creating new Pods on a node that was still available.

The stateless applications recovered reasonably well.

Then I powered the worker back on.

At first, the cluster appeared to return to normal. Both nodes became available again, Pods started running, and nothing looked seriously broken from the usual status commands.

But Gitea was no longer working properly.

## Everything Looked Healthy, but Gitea Was Broken

The strange part was that Gitea did not appear completely down.

Its Pods were running. PostgreSQL was available. Kubernetes was not showing an obvious scheduling failure, and the application interface could still be reached.

Users simply could not log in.

That made the problem more difficult to understand because there was no single failed Pod pointing directly to the cause.

I started checking the services Gitea depended on.

PostgreSQL appeared healthy, so I moved on to Valkey, the Redis-compatible service used by the deployment.

That was where the real problem was hiding.

During the worker-node outage, the Valkey cluster had lost quorum. One of its members disappeared, and when Kubernetes later recreated the Pod, it returned with a different network identity.

Kubernetes was satisfied because the expected number of Pods existed again. Valkey was not.

The cluster still remembered information about the previous member, while the recreated Pod joined with a new identity. From the outside, all the Pods looked alive. Internally, however, the Valkey nodes no longer agreed on the state of the cluster.

That was the first time I had personally encountered a distributed application that appeared healthy at the Kubernetes level while being broken at the application level.

A green Pod status did not mean the service inside it was functioning correctly.

## Rebuilding the Valkey Cluster

The next few hours were spent trying to understand exactly what Valkey believed had happened.

I inspected the StatefulSet and compared the cluster node IDs. I checked readiness probes, reviewed slot assignments, restarted Pods, read logs, and compared the old addresses stored in the cluster configuration with the IP addresses of the recreated Pods.

At first, I kept thinking in terms of Kubernetes.

Which Pod was failing?

Which container needed restarting?

Which object had Kubernetes created incorrectly?

Eventually, it became clear that Kubernetes had done what it was supposed to do. It had restored the Pods.

The application running inside those Pods was the part that could not recover cleanly.

Valkey was still holding onto the identity of a node that no longer existed. Restarting the same Pods did not repair that disagreement because the broken cluster state was being preserved.

The cleanest solution was to rebuild the Valkey cluster rather than continue trying to force the existing members to agree.

After recreating it, I checked the cluster state again.

```text
cluster_state: ok
cluster_slots_ok: 16384
```

Shortly afterward, Gitea login began working again.

That was the point where the entire failure finally made sense.

Gitea had not been broken by its web container or its database. Authentication was failing because a supporting distributed service had lost quorum during the node outage and had not recovered correctly when the missing node returned.

It was a deeper failure than a normal Pod restart, and it taught me far more than another successful Helm installation would have.

## Completing the Development Workflow

Once Gitea was stable again, I moved on to the part of the platform I had been looking forward to building: continuous integration.

I deployed Drone Server inside Kubernetes and connected it to Gitea using OAuth.

The first successful login was unexpectedly satisfying. Instead of being redirected to GitHub or another public service, Drone sent me to the Git server running inside my own cluster.

The source-code platform and the CI system were now connected entirely within the homelab.

Drone Server handled the user interface, repository integration, and pipeline coordination, but it still needed somewhere to execute the actual jobs.

For that, I deployed the Drone Kubernetes Runner.

This changed the way builds would run compared with some of my earlier CI experiments.

Previously, a CI runner might create Docker containers directly on the host. In this setup, Drone sends a request to Kubernetes, and Kubernetes creates temporary Pods for the pipeline steps.

The runner does not need to manage the underlying containers itself. It asks Kubernetes to schedule them, provide resources, and remove them when the build is complete.

The flow now looked like this:

```text
Developer pushes code to Gitea
              ↓
Drone detects the repository event
              ↓
Drone creates a pipeline job
              ↓
Kubernetes Runner requests build Pods
              ↓
Kubernetes schedules and runs the pipeline
```

That was an important architectural shift.

Kubernetes was no longer only hosting the CI platform. It had also become the execution environment for the builds themselves.

## Giving the Pipelines Somewhere to Push Images

After connecting Gitea and Drone, the next missing piece became obvious.

The pipelines could build container images, but I still needed somewhere private to store them.

That led to Harbor.

Harbor would give the homelab its own container registry, allowing Drone to build images and push them internally. Argo CD could then deploy workloads that referenced those images.

Installing Harbor would complete a much larger chain:

```text
Gitea → Drone → Harbor → Argo CD → Kubernetes
```

Harbor is not a lightweight application. It includes several services of its own, such as the registry, portal, core service, database, Redis, and job service.

Before installing it, I checked the available Longhorn storage and realised that the cluster was already becoming constrained.

My existing two-replica StorageClass was appropriate for important workloads that needed node-level redundancy, but using two replicas for every Harbor volume would consume storage quickly.

I had to choose between forcing the more resilient configuration and potentially running out of usable capacity, or accepting lower redundancy for the registry.

I created a separate single-replica Longhorn StorageClass specifically for Harbor.

This meant Harbor’s volumes would have only one Longhorn replica. If the node holding that replica failed permanently before the data could be recovered, the registry data could be lost.

It was not the configuration I would choose for an important production registry, but it was a practical decision for the hardware currently available in the homelab.

The registry could always be rebuilt from source repositories and CI pipelines if necessary. That made it a better candidate for reduced replication than services containing unique or difficult-to-recreate data.

I also disabled Trivy temporarily.

Vulnerability scanning would be useful later, but it would require additional resources and container images. At this stage, getting a stable registry running was more important than enabling every optional feature.

After applying those decisions, the Harbor services began starting successfully.

Harbor Core, Registry, Portal, PostgreSQL, Redis, and Jobservice all became operational.

For the first time, the homelab had its own private container registry.

## Cleaning Up What I Had Built

By this point, the cluster had grown significantly, but the files used to create it were scattered across my home directory.

Some Helm values files were stored in temporary folders. Several manifests had duplicate versions. A few configurations still contained values that should never be committed directly to Git.

The cluster was beginning to resemble a real platform, but the repository behind it did not.

Before finishing for the day, I started cleaning everything up.

I created separate directories for the major services and moved their Helm values and Kubernetes manifests into the appropriate locations. I removed outdated copies, replaced passwords and tokens with placeholders, and began organising the repository as though another person might need to understand it later.

The difference was not immediately visible from inside the cluster, but it mattered.

A platform is not reproducible if the only working configuration exists in shell history or in random files scattered across one machine.

The repository needed to explain how the environment had been built.

By the end of the cleanup, the project looked less like a folder of experiments and more like an infrastructure codebase.

## What Changed Today

At the start of the day, I had a Kubernetes cluster running several independent services.

By the end of it, those services had begun forming a connected workflow.

Gitea stored the source code and Kubernetes configuration.

Drone watched the repositories and created CI jobs.

The Kubernetes Runner executed those jobs as Pods.

Harbor stored the resulting container images.

Argo CD handled the deployment side of the process.

Longhorn provided storage to the stateful services.

Prometheus collected metrics, while Grafana made them visible.

Loki and Alloy handled the logging side of the environment.

Homepage provided one place to access the platform.

The architecture was no longer just a list of applications I had installed.

The parts were beginning to depend on and support one another.

```text
Source Code
   Gitea
      ↓
Continuous Integration
   Drone
      ↓
Container Images
   Harbor
      ↓
Continuous Delivery
   Argo CD
      ↓
Runtime Platform
   Kubernetes
      ↓
Storage and Observability
   Longhorn, Prometheus, Grafana, Loki and Alloy
```

The node-failure test also exposed the cost of that growing dependency.

When Valkey lost quorum, Gitea authentication stopped working even though most of the visible application Pods still appeared healthy.

The more connected the platform becomes, the more important it is to understand the behaviour of each supporting service during failure.

## End of Day 9

Today was not memorable because of the number of applications I installed.

The most valuable part was watching the platform fail in a way I had not expected.

Shutting down one worker node showed that stateless Kubernetes workloads could be recreated elsewhere, but it also exposed the limitations of a distributed stateful service that could not automatically recover its cluster membership.

Recovering Valkey forced me to look beyond Pod status and examine what the application itself believed about the cluster.

Deploying Drone showed me how Kubernetes could serve as the execution engine for CI pipelines rather than only hosting the CI server.

Installing Harbor forced me to make a real infrastructure trade-off between storage resilience and available capacity.

Even the repository cleanup reflected a change in how I was approaching the project. I was no longer creating files only to make the next deployment work. I was beginning to organise the environment so that it could be understood, reproduced, and repaired later.

A week earlier, most of my progress could be measured by whether I had learned a new `kubectl` command.

Now the more important questions were different.

What happens when a node disappears?

Which services recover automatically?

Which applications preserve outdated cluster membership?

Where should build jobs run?

Where should images be stored?

Which data requires replication, and which data can be recreated?

Those are no longer questions about installing Kubernetes.

They are questions about operating a platform.

The homelab is still small, and several parts of it are limited by the hardware underneath them. Both Kubernetes nodes still run on the same Proxmox host, storage capacity is restricted, and some of the resilience exists only at the virtual-machine level.

But today was the first time the environment felt like more than a collection of tools.

It felt like a system I could continue building, breaking, understanding, and improving.
