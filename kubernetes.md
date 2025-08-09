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

***

## Networking, Storage, Monitoring

### Networking

- **CNI Overlay:** Flannel (VXLAN/host-gw), ensures pod-to-pod communication cluster-wide
- **Service Types:**
    - **ClusterIP:** Internal service endpoints
    - **NodePort:** Exposes fixed ports per node, accessible via keepalived VIP on control-plane nodes
- **Ingress:** Deployed via ingress-nginx controller, exposed externally with NodePort and VIP


### Storage

- **Persistent Volumes:** Configured to ZFS-pool (tank)
- **Volume Provisioning:** Managed by PVCs


### Monitoring \& Observability

- **Prometheus:** Collection of metrics from all nodes/pods 
- **Grafana:** Cluster/project dashboards for visualization
- **Logs:** Ingress, system, and workload logs accessible via kubectl
- **Node Exporter:** Hardware/system metrics for each node

***

## Key Workloads and Services

Applications scheduled on worker nodes:

- **deansie-wordle-backend/frontend:**  Web application
- **kvikkjokk-nextjs:** Next.js server-side application
- **MongoDB:** Database as deployment with persistent volumes
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
- **Hybrid HA Design:** Integration of Linux-native keepalived (VRRP) with Kubernetes multi-master
- **Operational Visibility:** Prometheus, Grafana, node exporters, and centralized logging
- **Resilience Focus:** Designed for both hardware and node/cluster service failure scenarios, with ongoing improvements to backup storage for enhanced disaster recovery.

***

