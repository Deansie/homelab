# Homelab Documentation
## Overview

Welcome to my homelab setup! This environment leverages **Proxmox** for virtualization and **Kubernetes (k8s)** for container orchestration, both configured for **high availability (HA)** to ensure resilience and uptime. The homelab is designed to support a variety of workloads, from web applications to CI/CD pipelines, with a focus on automation, monitoring, and disaster recovery. As a soon-to-be system developer with a passion for DevOps, I built this homelab to showcase my DevOps skills, demonstrating my growing skills in infrastructure management, automation, and resilient system design. This README provides an overview of the repository structure and key components to help you navigate the setup.

This documentation is an ongoing process, and will be update along with changes troughout the homelab.

- **hardware.md**
  Details the physical hardware powering the homelab, including servers, storage, with specifications and roles in the infrastructure.

- **proxmox-setup.md**
  Outlines the **Proxmox VE** configuration, including motivations for HA, virtual machine and container setups, storage management with ZFS, and networking for optimal performance and redundancy.

- **jenkins-pipeline-example.md**
  Showcases a sample **Jenkins pipeline** written in Groovy, demonstrating automated deployment of an application to the Kubernetes cluster, highlighting CI/CD integration.

- **kubernetes.md**
  Describes the **Kubernetes cluster** deployment atop Proxmox VMs, featuring a multi-master HA control plane, Keepalived-managed VIP for failover, Flannel CNI for networking, and a robust monitoring stack with Prometheus and Grafana. It also covers backup strategies and workload management for resilient container orchestration.