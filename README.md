# Enterprise DevOps Homelab 🚀

[![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-blue?logo=kubernetes)]()
[![Proxmox](https://img.shields.io/badge/Virtualization-Proxmox-orange)]()
[![Longhorn](https://img.shields.io/badge/Storage-Longhorn-blue)]()
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-red)]()
[![Gitea](https://img.shields.io/badge/Git-Gitea-green)]()
[![Grafana](https://img.shields.io/badge/Monitoring-Grafana-orange)]()
[![Prometheus](https://img.shields.io/badge/Metrics-Prometheus-red)]()
[![Loki](https://img.shields.io/badge/Logging-Loki-blue)]()

---

# Overview

This repository documents my journey toward becoming a DevOps Engineer by designing, building, operating, troubleshooting, and continuously improving an enterprise-style homelab.

Instead of learning tools in isolation, I chose to build a production-inspired infrastructure that allows me to gain practical experience with virtualization, Linux administration, Kubernetes, distributed storage, GitOps, observability, networking, and automation.

Every service is deployed, tested, intentionally broken, repaired, and documented to understand **how modern infrastructure actually works** rather than simply following tutorials.

---

# Current Infrastructure

## Physical Infrastructure

```
                    Internet
                        │
             Android USB Tethering
                        │
              Ubuntu Router (NAT)
                 10.0.0.1/24
                        │
                Gigabit Ethernet
                        │
                 Proxmox VE Host
                        │
        ┌───────────────┴───────────────┐
        │                               │
   k3s-control                    k3s-worker1
     10.0.0.50                     10.0.0.51
```

---

# Kubernetes Platform

Current Kubernetes services include:

- Kubernetes (K3s)
- Longhorn Distributed Storage
- NGINX Ingress Controller
- Argo CD
- Gitea
- Prometheus
- Grafana
- Loki
- Grafana Alloy

---

# Architecture

```
                    Developer
                         │
                         ▼
                     Gitea Git
                         │
                         ▼
                  (Drone CI - Planned)
                         │
                         ▼
                Harbor Registry (Planned)
                         │
                         ▼
                    Argo CD GitOps
                         │
                         ▼
                    Kubernetes (K3s)
                         │
      ┌──────────────────┼──────────────────┐
      │                  │                  │
  Longhorn          Applications       Monitoring
      │                  │                  │
      │             NGINX Ingress      Prometheus
      │                                   │
      │                                Grafana
      │                                   │
      │                                  Loki
      │                                   │
      └────────────── Persistent Storage ─┘
```

---

# Technology Stack

## Virtualization

- Proxmox VE

## Operating Systems

- Ubuntu Server 26.04 LTS
- Ubuntu Desktop

## Container Platform

- Kubernetes (K3s)
- containerd

## Storage

- Longhorn

## GitOps

- Argo CD

## Source Control

- Git
- GitHub
- Gitea

## Monitoring & Observability

- Prometheus
- Grafana
- Loki
- Grafana Alloy
- Alertmanager

## Networking

- NGINX Ingress Controller
- Linux Routing
- NAT
- iptables
- DNS
- WireGuard VPN (Cloud ↔ Homelab)

---

# What I've Built

## Infrastructure

- Virtualized Kubernetes environment using Proxmox
- Multi-node K3s cluster
- Dedicated private network
- Ubuntu router with IP forwarding
- Persistent NAT configuration

## Kubernetes

- Control Plane
- Worker Node
- Ingress Controller
- Persistent Storage
- GitOps Platform

## Storage

- Longhorn Distributed Storage
- Multiple StorageClasses
- Persistent Volumes
- Persistent Volume Claims

## Monitoring

- Prometheus
- Grafana Dashboards
- Loki Log Aggregation
- Grafana Alloy Log Collection
- Alertmanager

## Developer Platform

- Gitea Git Server
- Argo CD
- GitOps-ready architecture

---

# Problems Solved

One of the main objectives of this project is learning through troubleshooting rather than simply deploying software.

Some of the engineering challenges solved include:

- Kubernetes cluster rebuild after deployment failure
- Worker node registration
- Kubernetes networking
- Longhorn volume scheduling failures
- Replica configuration issues
- Persistent storage design
- StorageClass customization
- Docker Hub rate limits and image pull failures
- DNS resolution failures
- Linux routing and packet forwarding
- NAT persistence
- Dynamic USB interface naming
- Kubernetes recovery after reboot
- Stateful application recovery
- Argo CD ingress configuration
- Gitea deployment troubleshooting
- Prometheus monitoring deployment
- Loki log aggregation deployment
- Grafana integration

Every issue is documented together with the troubleshooting process and lessons learned.

---

# Repository Structure

```
.
├── docs/
│   ├── day-01.md
│   ├── day-02.md
│   ├── day-03.md
│   ├── day-04.md
│   ├── troubleshooting.md
│   └── images/
│
├── kubernetes/
│   ├── argocd/
│   ├── ingress/
│   ├── longhorn/
│   ├── monitoring/
│   ├── storage/
│   └── namespaces/
│
├── scripts/
├── diagrams/
└── README.md
```

---

# Learning Philosophy

This repository is more than a collection of YAML files.

It documents the engineering decisions, failures, troubleshooting process, and lessons learned while building a production-inspired platform from scratch.

Every deployment is accompanied by:

- Objectives
- Architecture
- Configuration
- Troubleshooting
- Lessons Learned
- Future Improvements

Understanding *why* a system works is just as important as successfully deploying it.

---

# Roadmap

## CI/CD

- Drone CI
- Harbor Container Registry

## Infrastructure

- Terraform
- Ansible

## Kubernetes

- External DNS
- cert-manager
- Automatic TLS
- Horizontal Pod Autoscaling

## Security

- RBAC hardening
- Network Policies
- Secret Management

## Reliability

- Longhorn Backups
- Disaster Recovery
- Automated Snapshots
- High Availability

## Cloud

- AWS
- Azure
- Oracle Cloud
- Hybrid Cloud Architecture

---

# Current Status

**Infrastructure:** 🟢 Operational

| Component | Status |
|-----------|--------|
| Kubernetes | ✅ |
| Longhorn | ✅ |
| NGINX Ingress | ✅ |
| Argo CD | ✅ |
| Gitea | ✅ |
| Prometheus | ✅ |
| Grafana | ✅ |
| Loki | ✅ |
| Alloy | ✅ |

---

# About Me

I'm building this homelab to gain practical experience in:

- Linux System Administration
- Kubernetes
- Cloud Infrastructure
- Networking
- GitOps
- Infrastructure Automation
- Observability
- Site Reliability Engineering (SRE)

Every service deployed here is intended to deepen my understanding of modern infrastructure through hands-on experimentation rather than theoretical study.

---

> **"Build. Break. Fix. Document. Repeat."**

This homelab is continuously evolving as I learn new technologies and improve the platform.
