## Kubernetes (k8s) Cluster Setup

Kubernetes is deployed atop **Proxmox virtual machines**, enabling resilient infrastructure for containerized workloads. High Availability (HA) is achieved via multi-master control plane nodes and Keepalived-managed VRRP, providing automated VIP failover.

***

### Control Plane

- **Nodes:** 3 (`k8s`, `k8s-cp2`, `k8s-cp3`)
- **Redundancy:** Multi-master (HA), runs etcd cluster for distributed state
- **Taints:** `node-role.kubernetes.io/control-plane:NoSchedule`
- **Hardware Separation:** VMs mapped to physical servers for hardware diversity


### Worker Nodes

- **Total:** 10 (`worker-149` ... `worker-170`)
- **Physical Distribution:** Evenly split over two physical servers for failure tolerance (`server=server1`, `server=server2`)
- **Available for:** Scheduling of all general workloads and pods


### High Availability Configuration

- **Control Plane:**
    - Multi-master, no single point of failure
    - etcd operates in quorum (cluster recovers with >50% alive)
- **Keepalived (VRRP):**
    - Manages cluster-wide Virtual IP (`192.168.0.201`)
    - Failover: VIP floats between control-plane VMs on failure via VRRP
- **External Access:**
    - NGINX reverse proxy points at VIP and NodePort (e.g., `proxy_pass http://192.168.0.201:30080`)
- **No MetalLB:** VIP is external to Kubernetes; NodePort is used for service exposure


### Namespaces

- `default`: Core system and workload pods
- `ingress-nginx`: Ingress controller management
- `kube-system`: K8s infrastructure (coredns, kube-proxy, flannel, etc.)
- `monitoring`: Prometheus, Grafana, node exporters
- `openebs`: OpenEBS components for dynamic storage provisioning (e.g., localpv-provisioner, ndm-operator)
- `kafka`: Apache Kafka cluster (KRaft mode) and management tools

***

## Networking, Storage, Monitoring

### Networking

- **CNI Overlay:** Flannel (VXLAN/host-gw), ensures pod-to-pod communication cluster-wide
- **Service Types:**
    - **ClusterIP:** Internal service endpoints
    - **NodePort:** Exposes fixed ports per node, accessible via keepalived VIP on control-plane nodes
- **Ingress:** Deployed via ingress-nginx controller, exposed externally with NodePort and VIP


### Storage

- **Persistent Volumes:** Configured to ZFS-pool (tank), with dynamic provisioning via OpenEBS
- **Volume Provisioning:** Managed by PVCs, with automatic dynamic allocation using `openebs-localpv-provisioner` for local persistent volumes and `openebs-ndm-operator` for node device management and discovery


### Monitoring \& Observability

- **Prometheus:** Collection of metrics from all nodes/pods 
- **Grafana:** Cluster/project dashboards for visualization
- **Logs:** Ingress, system, and workload logs accessible via kubectl
- **Node Exporter:** Hardware/system metrics for each node
- **k9s:** Terminal-based UI installed on client machines for interactive cluster access, management, and enhanced observability (e.g., resource viewing, log inspection, and troubleshooting)

***

## Key Workloads and Services

Applications scheduled on worker nodes:

- **deansie-wordle-backend/frontend:**  Web application
- **kvikkjokk-nextjs:** Next.js server-side application
- **jakartaee-pet-adoption:** Jakarta-EE web application
- **spring-pet-adoption:** Spring Boot web application
- **MongoDB:** Database as deployment with persistent volumes
- **MySQL (InnoDB HA Cluster):** High-availability MySQL deployment using InnoDB storage engine, configured for replication and failover across multiple pods with persistent volumes
- **Apache Kafka (KRaft Mode):** Event streaming platform managed by Strimzi operator, with 3 mixed broker-controller nodes for HA quorum, low-resource configuration for small workloads, using OpenEBS for persistent logs
- **Kafka UI:** Web-based interface for Kafka management (topic creation, message production/consumption, monitoring), secured with basic HTTP auth via Ingress
- **ingress-nginx:** Central HTTP routing for all exposed services

***

## Cluster Operations and HA Resilience

### Traffic Flow

```
[ External Client ]
       ↓
[ NGINX Reverse Proxy ]
       ↓
http://192.168.0.201:30080 (keepalived VIP)
       ↓
[ k8s NodePort Service ]
       ↓
[ Kubernetes Pod ]
```

- **VIP migration:** On node failure, keepalived triggers VRRP, migrating VIP to backup node
- **Simplified Ingress:** NodePort traffic routed by external NGINX proxy


### Reliability

- **Control Plane:** HA, automatic failover with multi-master/etcd quorum and keepalived VIP
- **Worker Nodes:** Hardware resilience, automatic scheduling/recovery
- **Network:** Flannel overlay, keeps internal traffic reliable and isolated
- **Storage:** Enhanced with OpenEBS for dynamic PV provisioning, improving resilience for stateful workloads like databases and Kafka logs
- **Messaging:** Kafka KRaft provides quorum-based HA (3 nodes, replication factor 3), with affinity rules spreading brokers across physical hosts (preference for battery-backed host3 for power outage resilience); survives node failures with automatic leader election and log recovery
- **External Exposure:** NodePort is robust but does not provide native TCP/HTTP L4/L7 load balancing, which is sufficient for current workload scale but may be augmented with MetalLB for future growth.
- **Backup:** Daily etcd and manifest backups run directly on `k8s-cp3` via a root-owned scheduled cron job. The script performs:
    - Secure encrypted snapshot of etcd (`etcdctl snapshot save`) with TLS certificates across all cluster endpoints.
    - Export of all resource manifests with `kubectl get all --all-namespaces -o yaml`.
    - Storage in a root-only, permission-restricted `/backup/etcd` directory on `k8s-cp3`, with plans to replicate backups to a distributed ZFS pool or external storage for added resilience..
    - Automatic retention policy to keep the last 5 backup days (`find ... -mtime +5 -delete`).
    - Permission and ownership checks to ensure backup integrity.
- **Best Practice:** Scheduled backup/retention follows Kubernetes disaster recovery guidelines; fully automated, root-secured, and tested via file listing and error handling for every backup operation.

***

## DevOps Skill Highlights

- **Infrastructure as Code:** Use of logical node labels, overlay networking, monitoring stack, and HA primitives
- **Hybrid HA Design:** Integration of Linux-native keepalived (VRRP) with Kubernetes multi-master, extended to stateful services like MySQL HA cluster
- **Operational Visibility:** Prometheus, Grafana, node exporters, centralized logging, and k9s for interactive management and observability. Kafka UI for messaging-specific insights.
- **Resilience Focus:** Designed for both hardware and node/cluster service failure scenarios, with dynamic storage provisioning via OpenEBS and ongoing improvements to backup storage for enhanced disaster recovery.

***

