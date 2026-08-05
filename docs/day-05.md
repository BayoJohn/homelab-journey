# Day 4 – Building the First Production Workloads

Until today, most of my homelab work had focused on building the Kubernetes environment itself.

I had installed the cluster, configured the network, added a worker node, deployed Longhorn, and tested persistent storage by confirming that data could survive after a Pod was deleted. Those steps were important, but they were still mostly about preparing the platform.

A Kubernetes cluster becomes truly useful when it begins hosting real applications.

My goal for today was to move beyond infrastructure testing and start deploying services that could eventually support a complete DevOps workflow. Instead of jumping directly into CI/CD or GitOps, I decided to begin with the services those systems would later depend on.

The plan was to deploy Gitea as a self-hosted Git platform, PostgreSQL as the first serious stateful workload, and the kube-prometheus-stack as the foundation of the cluster’s monitoring system.

By the end of the session, the homelab had started to feel less like a Kubernetes experiment and more like the beginning of a real self-hosted DevOps platform.

## Deploying Gitea

Every DevOps workflow starts with source code.

Whether a team uses GitHub, GitLab, Bitbucket, or Azure DevOps, the application code, infrastructure definitions, deployment manifests, and automation files all need to live in a Git repository.

Since one of my long-term goals is to make the homelab as self-hosted as possible, I decided to deploy Gitea inside the cluster.

Gitea is a lightweight, open-source Git hosting platform. It provides repositories, branches, pull requests, issues, user management, permissions, and webhooks without requiring the resources of a much larger platform.

Deploying it locally would give me a central location for storing future projects and Kubernetes manifests. It would also prepare the environment for later integrations with tools such as Drone CI and Argo CD.

Once the Gitea workload and Service had been created, the next challenge was making the application accessible from outside the cluster.

## Exposing Gitea Through Traefik

Applications running inside Kubernetes are not automatically available to devices outside the cluster.

A Service makes an application reachable within the Kubernetes network, while an Ingress resource defines how external HTTP requests should be routed to that Service.

Because Traefik was already installed as the cluster’s Ingress Controller, I created an Ingress rule for Gitea using the hostname:

```text
gitea.local
```

The idea was simple: requests for `gitea.local` would reach Traefik, and Traefik would forward them to the Gitea Service inside the cluster.

The Ingress did not work on the first attempt.

When I applied the manifest, Kubernetes rejected it with the following error:

```text
cannot unmarshal string into Go struct field ServiceBackendPort.spec.rules.http.paths.backend.service.port.number of type int32
```

At first glance, the message looked complicated. However, the important part was the reference to `port.number` and `int32`.

I had defined the backend service port as a string instead of a number. Kubernetes resource definitions are strongly typed, so a field expecting an integer cannot accept a quoted string.

After correcting the port value, I applied the manifest again. This time, Kubernetes accepted it, and Traefik began routing requests to the Gitea Service.

Opening `http://gitea.local` from my management workstation displayed the Gitea interface.

That was an important moment for the project. The cluster was no longer only running test Pods in the background. It was now serving a real application through Kubernetes networking, a Service, and an Ingress Controller.

It also reminded me that complicated-looking Kubernetes errors often become much easier to solve once I slow down and read exactly which field the API server is rejecting.

## Introducing PostgreSQL

After Gitea was reachable, I moved on to PostgreSQL.

This deployment was more significant than the earlier Nginx tests because PostgreSQL is a stateful application. A web server can usually be deleted and recreated without losing anything important. A database cannot.

PostgreSQL stores information that applications depend on. If its data disappears whenever the Pod restarts, then the deployment is useless.

This made it a good test for Longhorn.

I had already created test volumes and confirmed that Longhorn could preserve a simple file after Pod deletion. Running a real database would show whether the storage system could support an application with its own filesystem expectations and startup process.

The PostgreSQL deployment did not start successfully.

## The First PostgreSQL Failure

When I checked the Pod, Kubernetes reported:

```text
ImagePullBackOff
```

My first suspicion was that I had used the wrong image name or made an error in the deployment manifest.

Instead of changing the configuration immediately, I inspected the Pod events.

The events showed that the worker node was having trouble reaching Docker Hub. DNS lookups and image downloads were timing out intermittently.

The PostgreSQL configuration was not the problem. The container had not even started because the node could not reliably download the image.

The entire homelab was still receiving internet access through my Ubuntu management machine, which was sharing a USB-tethered mobile connection. Any interruption in that connection affected the Kubernetes nodes and prevented them from reaching external container registries.

Once the connection became stable again, Kubernetes retried the image pull automatically. The download eventually succeeded, and the image was cached on the worker node.

That solved the first problem, but PostgreSQL still did not remain running.

## The Second PostgreSQL Failure

After the image had been downloaded, the Pod changed from `ImagePullBackOff` to:

```text
CrashLoopBackOff
```

This was a different type of failure.

An `ImagePullBackOff` meant Kubernetes could not obtain the image. A `CrashLoopBackOff` meant the container was now starting, failing, and being restarted repeatedly.

I checked the PostgreSQL logs and found the actual cause:

```text
initdb: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory.
```

Longhorn had formatted the new volume using the ext4 filesystem. When ext4 creates a filesystem, it normally creates a directory called `lost+found` at the root.

PostgreSQL’s initialization process expects its data directory to be empty. When it saw the existing `lost+found` directory, it refused to initialise the database.

The Kubernetes volume was working correctly, and Longhorn had mounted it successfully. The failure came from the way PostgreSQL expected its data directory to be structured.

To solve the problem, I configured PostgreSQL to use a subdirectory inside the mounted volume:

```text
PGDATA=/var/lib/postgresql/data/pgdata
```

Instead of trying to initialise directly at the root of the ext4 volume, PostgreSQL created and used the `pgdata` directory.

After applying that change, the database started successfully.

This was one of the most useful lessons of the day because it showed that deploying stateful applications requires more than attaching a PersistentVolumeClaim. I also need to understand how the application uses its filesystem and what it expects to find when it starts.

## Confirming Longhorn with a Real Workload

Getting PostgreSQL running also gave me another level of confidence in Longhorn.

Earlier storage tests had shown that Longhorn could create a volume and preserve a file. PostgreSQL demonstrated that the storage system could support an actual stateful service with repeated reads, writes, initialization files, and a persistent data directory.

The PostgreSQL Pod could be recreated without treating the database volume as part of the container itself. The storage existed independently and could be mounted again whenever Kubernetes recreated the workload.

This was the first time Longhorn was being used for something closer to its real purpose rather than only as part of a controlled experiment.

## Beginning the Monitoring Stack

With Gitea and PostgreSQL running, I turned my attention to monitoring.

As the number of applications in the cluster increases, checking individual Pods manually will no longer be enough. I need a way to see what is happening across the entire environment.

I want to be able to monitor node health, CPU usage, memory consumption, Pod status, storage utilisation, and the general condition of the Kubernetes control plane.

For this reason, I began deploying `kube-prometheus-stack`.

The stack provides several connected monitoring components, including Prometheus for collecting and storing metrics and Grafana for visualising them. It also includes Kubernetes exporters, alerting components, and preconfigured monitoring resources.

I installed the stack using Helm. The chart installation completed, and Kubernetes began creating the required workloads.

However, the monitoring platform did not become fully healthy.

## More Image-Pull Problems

Several monitoring Pods entered `ImagePullBackOff`, while others remained in `ContainerCreating`.

Once again, the events showed that the cluster was struggling to download images from external registries, including Quay.io.

The problem was not limited to a single application or container image. The unstable upstream connection was now affecting multiple deployments.

During the troubleshooting process, both Kubernetes nodes temporarily entered a `NotReady` state before later recovering.

That was concerning because it showed that the network problem could affect more than image downloads. If a node cannot communicate reliably with the control plane or complete required system operations, Kubernetes may temporarily consider it unhealthy.

By the end of the session, Prometheus and several supporting components were running, but some parts of the monitoring stack still needed investigation.

I could have started deleting Pods, changing manifests, or reinstalling the Helm release, but none of those actions would have fixed an unstable internet connection.

I decided to stop making unnecessary changes and leave the remaining troubleshooting for the next session.

Part of operating infrastructure responsibly is knowing when the available evidence points to an external dependency rather than the application being deployed.

## What I Learned

Today showed me that deploying applications is very different from only building the Kubernetes infrastructure beneath them.

Infrastructure components may have common requirements, but every application brings its own assumptions and failure conditions.

Gitea required correct Service and Ingress configuration. A small type error in the manifest was enough for Kubernetes to reject the entire resource.

PostgreSQL required persistent storage, but attaching a volume was only part of the solution. I also had to understand how the database initialised its data directory and why an ext4 `lost+found` directory caused it to fail.

The monitoring stack introduced another layer of complexity because it consisted of several components pulling images from different registries. This made the weakness of the homelab’s current internet setup even more visible.

The most important lesson was to follow the evidence.

`ImagePullBackOff` led me to the Pod events and then to DNS and network timeouts. `CrashLoopBackOff` led me to the container logs and then to the PostgreSQL data-directory issue. The rejected Ingress manifest led me directly to a field with the wrong data type.

Each status was only the starting point. The real explanation came from events, logs, and careful inspection.

## Status at the End of the Day

By the end of the session, the homelab had moved beyond test deployments and was beginning to host services that could support a larger DevOps workflow.

```text
Gitea
────────────────────────────────
Deployment              Running
Service                 Available
Ingress                 Configured
Hostname                gitea.local
External access         Working through Traefik

PostgreSQL
────────────────────────────────
Container image         Downloaded
Persistent storage      Longhorn
Data directory          /var/lib/postgresql/data/pgdata
Status                  Running

Monitoring
────────────────────────────────
Helm release            Installed
Prometheus              Partially operational
Supporting components   Still recovering
Remaining issue         External image downloads

Cluster
────────────────────────────────
Control plane           Recovered
Worker node             Recovered
Main limitation         Unstable upstream internet connection
```

## Next Steps

The next session will focus on completing the monitoring deployment and confirming that every component in the `kube-prometheus-stack` is healthy.

Once Grafana is running properly, I plan to expose it through Traefik and begin exploring the dashboards provided for Kubernetes nodes, Pods, storage, and control-plane components.

I also need to continue investigating the image-pull failures and improve the reliability of the cluster’s internet connection.

After the monitoring layer is stable, the next major step will be introducing Argo CD and beginning the move toward a GitOps-based deployment workflow.

Today was the point where the homelab stopped being only a Kubernetes installation. It began becoming a platform capable of hosting the tools, databases, and monitoring services that a modern DevOps environment depends on.
