# Day 5 – When the Homelab Started Feeling Like a Platform

There are days when you install a piece of software, verify that it's running, and call it a day.

Then there are days like today.

I didn't set out to deploy half a dozen services or spend hours debugging distributed systems. The plan was much simpler: continue building the homelab, understand the cluster a little better, and maybe add another component before the day was over.

Instead, today became one of those days where every new discovery led to another question, and every answer uncovered something I hadn't fully understood before.

By the time I finally shut everything down for the night, the homelab no longer felt like a Kubernetes cluster with a few applications running on top of it.

It had started to feel like an actual platform.

---

# A Small Dashboard That Changed Everything

One thing had been bothering me for a while.

Every time I wanted to check the health of the cluster, I found myself opening Grafana in one tab, Gitea in another, Longhorn somewhere else, and Prometheus in yet another browser window. It worked, but it felt disorganized.

So I decided to deploy **Homepage**.

It wasn't the most technically challenging application I've installed, but it instantly changed the way I interacted with the homelab. Instead of remembering URLs or keeping browser tabs permanently open, I now had a single place where every service could be reached.

Sometimes it's the smallest improvements that make an environment feel much more polished.

For the first time, the homelab had its own landing page.

---

# Breaking the Cluster on Purpose

With Homepage out of the way, curiosity took over.

Over the last few days I had built a two-node Kubernetes cluster, but there was one question I couldn't answer honestly:

**What actually happens when one of the nodes disappears?**

I had read countless articles explaining how Kubernetes reschedules workloads when nodes fail, but reading documentation and watching it happen are two completely different experiences.

So I did something that felt slightly uncomfortable.

I shut down my worker node.

Then I sat back and watched.

Almost immediately the cluster reacted. Kubernetes marked the node as **NotReady**, and after a short delay Deployments began scheduling replacement Pods onto the control plane.

Watching Kubernetes make those decisions automatically was strangely satisfying. It wasn't magic anymore—I could finally see the control plane doing exactly what it had been designed to do.

But while stateless applications recovered gracefully, the same couldn't be said for everything else.

---

# The Moment Everything Broke

Not long after bringing the worker node back online, something was clearly wrong.

Gitea refused to work.

At first glance everything looked healthy. The Pods were running, PostgreSQL appeared fine, and Kubernetes wasn't reporting any obvious failures.

But users couldn't log in.

The logs eventually pointed me in an unexpected direction.

The problem wasn't Gitea itself.

It wasn't PostgreSQL either.

It was **Valkey**.

During the outage, the Valkey cluster had lost quorum. When Kubernetes recreated one of the Pods, the cluster still remembered the old node identity. From Kubernetes' perspective everything looked healthy, but from Valkey's perspective the cluster had split into two different realities.

That was one of those moments where distributed systems stop being abstract concepts and become very real.

---

# Learning More From Failure Than Success

The next few hours disappeared into troubleshooting.

I inspected StatefulSets.

Compared cluster node IDs.

Checked readiness probes.

Looked at cluster slot assignments.

Restarted Pods.

Read logs.

Compared old IP addresses with new ones.

Every command revealed another small piece of the puzzle.

Eventually I realised that I wasn't dealing with a broken Pod.

I was dealing with a distributed system that no longer agreed with itself.

The cleanest solution turned out to be rebuilding the Valkey cluster entirely.

A few moments later the cluster reported:

```text
cluster_state: ok
cluster_slots_ok: 16384
```

Almost instantly Gitea came back to life.

Looking back, I probably learned more from those few hours of troubleshooting than I had from several successful deployments combined.

---

# Building My Own CI Platform

Once the cluster was healthy again, I turned my attention to something I had been looking forward to for weeks.

Continuous Integration.

I deployed **Drone Server** into Kubernetes and connected it to my self-hosted Gitea instance using OAuth.

Seeing Drone redirect me to **my own Git server** instead of GitHub felt surprisingly rewarding.

For the first time, every major part of the development workflow was running on infrastructure I had built myself.

But the Drone Server is only half the story.

Without a runner, it can't actually execute pipelines.

So I deployed the **Drone Kubernetes Runner**.

This was particularly interesting because it highlighted one of the biggest differences between my previous homelab and the one I'm building now.

Previously, Drone launched Docker containers directly.

Now, Drone simply asks Kubernetes to create Pods on its behalf.

The responsibility for running builds has shifted from Docker to Kubernetes.

It was a subtle difference in architecture, but an important one.

---

# Solving the Registry Problem

With CI running, another missing piece became obvious.

Where would all the container images go?

The answer was Harbor.

Unfortunately, Harbor is a fairly large application, and my Longhorn cluster had already started warning me that storage was becoming limited.

Rather than forcing the installation and hoping for the best, I decided to understand the problem first.

After inspecting Longhorn's available capacity, I created a dedicated **single-replica StorageClass** specifically for Harbor.

It wasn't the most resilient configuration, but it fit my current hardware while still giving me a fully functional private registry.

I also disabled Trivy for now, choosing stability over additional features.

A few minutes later, Harbor Core, Registry, Portal, PostgreSQL, Redis, and Jobservice were all running successfully.

It was another reminder that good engineering is often about making sensible trade-offs rather than blindly enabling every feature available.

---

# Cleaning Up the Project

Before calling it a day, I spent some time looking beyond the cluster itself.

Over the last few weeks I had accumulated dozens of configuration files spread across my home directory.

It was time to organise them properly.

I created dedicated directories for every major service, moved Helm values files into version control, removed duplicate manifests, replaced sensitive passwords with placeholders, and started treating the repository like an actual infrastructure codebase rather than a dumping ground for YAML files.

It was one of those tasks that isn't particularly exciting, but pays dividends later.

---

# Looking Back

A week ago, I was mostly learning Kubernetes commands.

Today, those commands are only a small part of what I'm building.

Source code now lives in **Gitea**.

**Drone** watches repositories and executes pipelines.

**Harbor** stores container images.

**Argo CD** handles deployments.

**Longhorn** provides persistent storage.

**Loki** and **Alloy** collect logs.

**Prometheus** gathers metrics.

**Grafana** visualises everything.

And **Homepage** brings it all together behind a single dashboard.

For the first time since I started this project, I stopped looking at individual applications and started seeing an ecosystem.

---

# Final Thoughts

Today wasn't really about installing software.

It was about understanding how each component fits into a larger system.

More importantly, it reminded me that the most valuable lessons rarely come from deployments that work the first time. They come from unexpected failures, confusing log files, and long debugging sessions where understanding is earned one clue at a time.

The homelab is still far from finished.

There are plenty of technologies left to explore and countless improvements to make.

But today it crossed an important milestone.

It no longer feels like a collection of Kubernetes experiments.

It feels like the beginning of a platform I can continue building, breaking, improving, and learning from for a long time to come.

---
