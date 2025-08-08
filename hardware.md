# Physical Hardware

## Proxmox node 1
- **Server**: Dell Poweredge R720 with RAID-controller in IT-mode
- **CPU**: 2 x Intel Xeon E5-2620 (6 cores, 12 threads)
- **RAM**: 72GB DDR3 PC3-10600R ECC
- **Storage**: 
    - **SAS**: 6 x 600GB Dell Enterprise ZFS-Mirrored
    - **BOOT**: 128GB SK Hynix NVME
- **Network**: 4 x 1 GBPS NICs 
- **Mian purpose**: Host various VMs, containers, Kubernetes controlplane and workers - weekly backups on everything, allowing a maximum of 2 snapshots being stored.

## Proxmox node 2
- **Server**: HPE Proliant DL360p Gen8 with RAID-controller in IT-mode
- **CPU**: 2 x Intel Xeon E5-2620 (6 cores, 12 threads)
- **RAM**: 192GB DDR3 PC3-10600R ECC
- **Storage**: 
    - **SAS**: 6 x 900GB Dell Enterprise ZFS-Mirrored
    - **BOOT**: 16GB SD-Card Verbatim Extreme
- **Network**: 6 x 1 GBPS NICs 
- **Mian purpose**: Host various VMs, containers, Kubernetes controlplane and workers - weekly backups on everything, allowing a maximum of 2 snapshots being stored.

## Proxmox node 3
- **Server**: Dell Latitude 7290 Laptop
- **CPU**: 1 x Intel Core i5-6200U (2 cores, 4 threads)
- **RAM**: 16GB DDR4 PC4-17000 non-ECC
- **Storage**: 
    - **BOOT**: 128GB SK Hynix NVME
- **Network**: 1 x 1GBPS NIC 
- **Mian purpose**: Runs a third Kubernetes controlplane to ensure quorum and to perform daily etcd-backups. Subsequently also ensures quorum for the Proxmox cluster. The built in battery also gives the clusters some time to be restored without the need to manually restore etcd from backups, in the event of an poweroutage.