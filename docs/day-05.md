# Day 4 – Building the First Production Workloads

Today marked an important milestone in the homelab project. Up until this point, most of the work had focused on building the Kubernetes infrastructure itself. The cluster had been installed, networking had been configured, Longhorn had been deployed for persistent storage, and storage persistence had been verified by ensuring data survived pod deletion.

While these were all essential components, Kubernetes by itself provides very little value unless it is actually hosting applications. The objective today was therefore to begin transforming the cluster from infrastructure into a usable platform by deploying the first production-style workloads.

Rather than immediately jumping into CI/CD or GitOps, the decision was made to build the services that those technologies would eventually depend on. This meant deploying a self-hosted Git service, a dedicated database, and beginning work on the cluster's monitoring platform.

By the end of the day, the homelab had taken its first real steps towards becoming a complete DevOps platform.

---

# Objectives

The goals for today's session were carefully chosen to build upon the work completed over the previous few days.

The objectives were to:

* Deploy Gitea as the cluster's self-hosted Git server.
* Expose Gitea through Traefik using an Ingress resource.
* Deploy PostgreSQL using Longhorn persistent storage.
* Verify that Longhorn could reliably support real stateful applications.
* Begin deploying the monitoring stack using kube-prometheus-stack.
* Continue preparing the cluster for future GitOps and CI/CD workflows.

Rather than viewing these as independent deployments, each service was introduced because it will eventually become part of a much larger DevOps ecosystem.

---

# Deploying Gitea – Building the Centre of the DevOps Workflow

Every DevOps workflow begins with source code.

Whether deploying applications through GitHub, GitLab, Azure DevOps or Bitbucket, everything starts with a Git repository. Since one of the long-term goals of this homelab is to become completely self-hosted, relying on external Git hosting services would defeat part of that purpose.

For this reason, today's first task was to deploy **Gitea**, a lightweight, open-source Git hosting platform.

Gitea provides many of the same features found in GitHub, including:

* Git repositories
* Branch management
* Pull requests
* Issues
* User management
* Webhooks
* Repository permissions

Deploying Gitea inside Kubernetes means that every future project in the homelab can be stored locally without depending on third-party services.

This also lays the foundation for future integrations with Drone CI, ArgoCD and other GitOps tools that will be deployed later in the project.

---

# Publishing Gitea with Traefik

Deploying an application inside Kubernetes is only part of the process.

Applications running inside the cluster are normally inaccessible from outside unless they are exposed through a Kubernetes Service and an Ingress Controller.

Earlier in the project, Traefik had already been installed as the cluster's Ingress Controller.

Today's task was therefore to expose Gitea through an Ingress resource using the hostname:

```text
gitea.local
```

Once the Ingress resource was successfully applied, Kubernetes assigned the Traefik load balancer addresses and routed incoming requests to the Gitea service running inside the cluster.

Visiting `http://gitea.local` from the management workstation confirmed that routing through Traefik was functioning correctly.

Seeing the Gitea dashboard appear in the browser was an important milestone because it demonstrated that Kubernetes networking, Services and Ingress were all working together as expected.

The cluster was no longer simply running containers; it was now serving real applications.

---

# Problem Encountered – Invalid Ingress Manifest

The deployment was not successful on the first attempt.

When the Ingress manifest was applied, Kubernetes rejected it with the following error:

```text
cannot unmarshal string into Go struct field ServiceBackendPort.spec.rules.http.paths.backend.service.port.number of type int32
```

Initially, this looked like a Kubernetes API problem. However, after carefully reading the error message, it became clear that the issue was caused by the Ingress definition itself.

The service port had been defined as a string rather than an integer.

Because the Kubernetes API strictly validates resource definitions, the manifest could not be accepted.

After correcting the service port definition to use a numeric value, the Ingress resource was created successfully and Traefik immediately began routing requests to Gitea.

Although this was only a small configuration mistake, it reinforced an important lesson: Kubernetes manifests are strongly typed, and even small formatting errors can prevent resources from being created.

---

# Deploying PostgreSQL – Introducing Stateful Workloads

With Gitea running successfully, the next objective was to deploy PostgreSQL.

Unlike web applications, databases cannot simply be recreated if something goes wrong. They store the information that applications depend upon, making persistent storage one of the most critical parts of any Kubernetes environment.

This deployment served two purposes.

The first was to provide a database that future services can use.

The second was to verify that Longhorn could support real-world stateful applications rather than simple storage tests.

Deploying PostgreSQL represented the first genuine production-style workload within the cluster.

---

# Problem Encountered – ImagePullBackOff

The PostgreSQL deployment immediately encountered problems.

Instead of starting normally, Kubernetes repeatedly attempted to download the container image before eventually reporting an `ImagePullBackOff` error.

At first, this suggested that something was wrong with the deployment manifest.

However, examining the pod events told a completely different story.

The worker node was unable to download the PostgreSQL image from Docker Hub because DNS lookups and image downloads were intermittently timing out.

Since the homelab currently receives internet access through a laptop acting as a gateway, brief interruptions in connectivity directly affected Kubernetes' ability to pull container images.

Once internet connectivity stabilised, Kubernetes automatically retried the download and successfully cached the image on the worker node.

This demonstrated an important principle of Kubernetes troubleshooting: not every application failure originates from the application itself. Infrastructure issues such as networking or DNS can prevent workloads from ever starting.

---

# Problem Encountered – CrashLoopBackOff

Although the image was eventually downloaded successfully, PostgreSQL still refused to start.

This time the pod entered a `CrashLoopBackOff` state.

Unlike the previous issue, the container was now starting correctly before immediately exiting.

Inspecting the container logs revealed the root cause:

```text
initdb: directory "/var/lib/postgresql/data" exists but is not empty
It contains a lost+found directory.
```

This behaviour occurs because Longhorn formats newly created volumes using the ext4 filesystem.

The filesystem automatically creates a `lost+found` directory at the root of the volume.

PostgreSQL expects to initialise itself inside an empty directory, so it intentionally refuses to continue when it detects additional files.

The solution was to configure PostgreSQL to store its database inside a dedicated subdirectory by setting:

```text
PGDATA=/var/lib/postgresql/data/pgdata
```

Once PostgreSQL was directed to initialise inside this new directory, the database started successfully.

This problem provided valuable insight into how applications interact with persistent storage and demonstrated that storage-related issues often require understanding both Kubernetes and the behaviour of the application itself.

---

# Beginning the Monitoring Platform

With both Gitea and PostgreSQL deployed successfully, attention shifted towards observability.

As infrastructure grows, it becomes increasingly difficult to understand what is happening inside the cluster without proper monitoring.

Simply knowing that a pod is running is not enough.

Administrators need visibility into CPU usage, memory consumption, node health, storage utilisation and the overall condition of the cluster.

For this reason, the next logical component to deploy was the **kube-prometheus-stack**, which includes Prometheus, Grafana and several supporting components.

These tools will eventually provide a complete monitoring platform capable of collecting metrics from every node, pod and Kubernetes resource within the homelab.

---

# Problems During Monitoring Deployment

Although the Helm installation completed successfully, several components failed to start correctly.

Some containers entered an `ImagePullBackOff` state while others remained stuck in `ContainerCreating`.

Further investigation showed that these failures were once again related to image downloads from external registries such as Quay.io rather than problems with Kubernetes itself.

During troubleshooting, both cluster nodes temporarily reported a `NotReady` status before eventually recovering.

At the end of the session, Prometheus and several monitoring components were operational, while a few remaining services still required additional investigation.

Rather than forcing changes without understanding the root cause, the decision was made to pause and continue troubleshooting during the next session.

This reflects an important engineering principle: solving problems methodically produces far more reliable infrastructure than rushing to reach the next milestone.

---

# Lessons Learned

Today's work reinforced several important ideas about operating Kubernetes in the real world.

The first is that deploying applications is very different from deploying infrastructure. Every application has its own requirements, assumptions and failure modes that must be understood before it can run successfully.

Secondly, persistent storage introduces challenges that do not exist with stateless applications. Understanding how applications initialise data directories is just as important as understanding how Kubernetes mounts storage.

Finally, effective troubleshooting depends on following evidence rather than assumptions. Throughout the day, every problem was investigated by reading pod events, inspecting logs and understanding the underlying behaviour before making changes. This disciplined approach not only resolved the immediate issues but also provided a much deeper understanding of how Kubernetes behaves under real-world conditions.

---

# Looking Ahead

By the end of the day, the homelab had evolved from a Kubernetes cluster into the beginnings of a self-hosted DevOps platform. Gitea was successfully serving repositories through Traefik, PostgreSQL was running on Longhorn-backed persistent storage, and the first components of the monitoring stack had been deployed.

The next session will focus on completing the monitoring platform, exposing Grafana through Traefik, investigating the remaining image pull issues, and preparing the cluster for the introduction of GitOps with ArgoCD. Each new service brings the homelab one step closer to resembling the architecture of a modern production environment.
