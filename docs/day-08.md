
# Day 8 – Deploying Gitea, Building Persistent Storage, and Establishing Cluster Monitoring

## Overview

The primary objective of today's work was to transform the Kubernetes cluster into a self-hosted development platform by deploying **Gitea** as an internal Git server. Unlike previous deployments, this installation relied heavily on persistent storage, making it the first real workload that tested the reliability of the newly built Longhorn storage cluster.

Although the deployment initially appeared straightforward, it exposed several weaknesses in the storage configuration. Numerous debugging sessions were required before Gitea could successfully start. The troubleshooting process significantly improved the resilience of the homelab by correcting Longhorn's replication strategy and validating persistent storage across both cluster nodes.

After Gitea was successfully deployed, the cluster was further enhanced with a complete monitoring stack consisting of Prometheus, Grafana, Alertmanager, kube-state-metrics, and Node Exporter. This provided real-time visibility into cluster health, resource utilization, and storage performance.

---

# Objective

The objectives for today's work were:

* Deploy Gitea using Helm.
* Configure persistent storage using Longhorn.
* Integrate Gitea with the NGINX Ingress Controller.
* Configure an administrative account securely using Kubernetes Secrets.
* Troubleshoot Longhorn storage provisioning failures.
* Improve storage resilience by redesigning replica placement.
* Deploy the Kubernetes monitoring stack.
* Prepare the cluster for future GitOps workflows using Argo CD.

---

# Installing Gitea

Unlike the previous lightweight NGINX application, Gitea is a stateful application requiring persistent storage for repositories, configuration files, and its backend database.

The official Helm chart was chosen because it automatically deploys:

* Gitea
* PostgreSQL
* Valkey (Redis-compatible cache)
* Kubernetes Services
* Persistent Volume Claims
* Ingress support

The Helm repository was added first:

```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
```

After confirming the latest chart version, a custom values file (`gitea-values.yaml`) was created.

Instead of embedding sensitive credentials directly inside the Helm configuration, an existing Kubernetes Secret named `gitea-admin` was referenced. This allowed the administrator username and password to remain outside the configuration file, following Kubernetes security best practices.

The deployment also configured:

* Persistent storage using Longhorn
* Internal PostgreSQL database
* Internal Valkey cluster
* NGINX Ingress
* SSH NodePort for Git operations

Deployment was initiated using:

```bash
helm install gitea gitea-charts/gitea \
    -n gitea \
    --create-namespace \
    -f gitea-values.yaml
```

---

# First Deployment Failure

Unlike previous deployments, the Gitea installation did not complete successfully.

Almost immediately, several pods became stuck in various states:

* `ContainerCreating`
* `ImagePullBackOff`
* `Init:CrashLoopBackOff`

The PostgreSQL pod never started.

The Valkey StatefulSet also failed to initialize.

Since Gitea depends on both services before completing its own initialization, the Gitea pod continuously restarted while waiting for the database to become available.

The cluster therefore entered a dependency failure chain where every component waited on another component that had not successfully started.

---

# Investigating the Storage Failure

Initial inspection focused on Kubernetes events.

Running:

```bash
kubectl describe pod
```

revealed repeated messages similar to:

```text
AttachVolume.Attach failed
volume is not ready for workloads
```

This indicated that Kubernetes itself was functioning correctly.

Instead, Longhorn was unable to attach newly created Persistent Volumes.

The next step was to inspect the Longhorn volume objects directly.

Several newly created volumes were found in the following state:

* Detached
* Unknown robustness

Although the PVCs were successfully created and bound, Longhorn could not make the underlying block devices available to the workloads.

---

# Root Cause Analysis

Further investigation eventually revealed the real cause.

Every newly provisioned Longhorn volume was configured with:

```text
numberOfReplicas = 3
```

However, the cluster contained only:

* 1 Control Plane node
* 1 Worker node

Longhorn therefore attempted to create three independent replicas despite having only two physical machines.

Because sufficient replica placement was impossible, Longhorn reported:

```text
insufficient storage
```

This error was slightly misleading.

The problem was **not** disk capacity.

The problem was insufficient nodes to satisfy the requested replica count.

Consequently, volumes remained detached and could never become available to Kubernetes workloads.

---

# Improving Storage Resilience

Rather than disabling replication entirely, a more resilient configuration was designed.

A new StorageClass named:

```text
longhorn-two-replicas
```

was created.

This StorageClass reduced:

```yaml
numberOfReplicas: 2
```

which perfectly matched the two-node cluster.

This provided several advantages:

* every volume still has redundancy;
* one replica is stored on each physical node;
* either node can fail without immediate data loss;
* Longhorn can successfully schedule every replica.

This represented a significant architectural improvement over the default configuration.

---

# Redeploying Gitea

The previous failed deployment was completely removed.

The Helm values file was updated so both Gitea and PostgreSQL explicitly requested:

```yaml
storageClass: longhorn-two-replicas
```

Additionally, unnecessary storage allocations were reviewed.

Persistent volumes were resized appropriately for the homelab environment while preserving sufficient capacity for repositories and databases.

After redeployment, Kubernetes successfully provisioned new Persistent Volumes using the updated StorageClass.

This time:

* PostgreSQL attached successfully.
* Valkey StatefulSets initialized.
* Gitea initialization completed.
* All application pods transitioned to the **Running** state.

This confirmed that the storage redesign had completely resolved the deployment failure.

---

# Temporary Docker Hub Resolution Issues

While deploying Gitea, several containers intermittently entered the `ImagePullBackOff` state.

The failure messages indicated DNS resolution problems when contacting Docker Hub authentication services, for example:

```text
lookup auth.docker.io: Try again
```

Rather than indicating a permanent failure, these errors were caused by temporary network or DNS resolution issues during image downloads.

The same behaviour had been observed previously during the installation of the NGINX Ingress Controller.

Once the images were successfully downloaded, restarting the affected pods allowed Kubernetes to reuse the locally cached images, after which all containers started normally.

This reinforced an important operational lesson: transient image pull failures do not necessarily require redeployment. In many cases, allowing the image to finish downloading or recreating the pod is sufficient.

---

# Exposing Gitea Through NGINX Ingress

With all application components running successfully, Gitea was exposed internally using the NGINX Ingress Controller.

The deployment was configured to respond to:

```text
http://gitea.local
```

while Git operations over SSH were made available through a dedicated NodePort service.

The workstation's `/etc/hosts` file was updated to map the internal cluster IP to the hostname, allowing browsers and Git clients to access Gitea using a friendly domain name rather than an IP address.

This provided an experience similar to hosting a production Git server on a private network.

---

# Deploying the Monitoring Stack

With the platform components operational, the next objective was to introduce comprehensive observability into the Kubernetes cluster.

Rather than installing Prometheus, Grafana, Alertmanager, and related components individually, the **kube-prometheus-stack** Helm chart was selected. This chart bundles the most widely adopted Kubernetes monitoring tools into a single deployment, simplifying installation while maintaining production-grade defaults.

The Prometheus Community Helm repository was added, after which the monitoring stack was installed into a dedicated namespace:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
    -n monitoring
```

The installation deployed the following components:

* Prometheus Operator
* Prometheus Server
* Alertmanager
* Grafana
* kube-state-metrics
* Node Exporter

---

# Monitoring Deployment Challenges

Similar to the earlier Gitea deployment, several monitoring components initially encountered `ErrImagePull` and `ImagePullBackOff` errors. Investigation showed that these failures were again caused by temporary DNS resolution issues when accessing Docker Hub registries.

Rather than reinstalling the monitoring stack, the affected pods were allowed to retry their downloads. Where necessary, images were manually pulled and the affected pods recreated. Once the required images had been cached locally, all monitoring components transitioned successfully to the `Running` state.

This approach avoided unnecessary redeployments and demonstrated the importance of distinguishing transient network issues from genuine configuration errors.

---

# Final Monitoring Architecture

By the end of the deployment, the monitoring namespace contained fully operational instances of:

* Prometheus for metrics collection
* Grafana for visualization
* Alertmanager for alert processing
* kube-state-metrics for Kubernetes object metrics
* Node Exporter running on both cluster nodes for hardware and operating system metrics

Grafana was then exposed through the existing NGINX Ingress Controller, enabling convenient browser-based access via the internal hostname `grafana.local`.

This completed the observability layer of the homelab, providing real-time insight into node health, pod status, storage performance, and cluster resource utilization.

---

# Outcome

By the conclusion of today's work, the homelab had evolved significantly beyond a basic Kubernetes cluster. It now included a fully operational self-hosted Git service backed by replicated persistent storage and a comprehensive monitoring platform capable of observing both infrastructure and workloads.

More importantly, the debugging process led to a fundamental improvement in the storage architecture. Identifying the mismatch between Longhorn's default three-replica configuration and the available two-node cluster resulted in the creation of a custom StorageClass that better aligned with the physical infrastructure while still providing redundancy. This change greatly improved the reliability of stateful workloads.

The platform is now ready for the next phase of development, where Gitea will serve as the central Git repository and Argo CD will be configured to continuously synchronize Kubernetes manifests from Gitea, completing the transition to a fully self-hosted GitOps workflow.
