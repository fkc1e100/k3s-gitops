# K3s GitOps Homelab

Welcome to the GitOps control repository for Frank's hybrid-architecture K3s cluster. This repository serves as the single source of truth for all cluster workloads, platform configurations, and deployment specifications, managed autonomously with safe **Human-in-the-Loop (HITL)** approvals.

---

## 🚀 Repository Intent

This repository uses **ArgoCD** to declare, track, and reconcile the physical cluster state against our version-controlled manifests. 

*   **Continuous Reconciliation:** ArgoCD automatically polls this repository and applies changes to the cluster.
*   **GitOps Lifecycle:** All infrastructure modifications, scale adjustments, and workload upgrades are proposed as branches, tested via automated linter actions, and submitted as Pull Requests on GitHub for manual review and merge.

---

## 🖥️ Cluster Topology & Hardware

The cluster is a hybrid x86_64 and ARM64 architecture designed for high availability, storage offloading, and resource efficiency.

### Nodes and Architecture

| Node Name | IP Address | Architecture | Role | Operating System | Container Runtime |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ubuntu-dev** | `192.168.100.100` | `x86_64` | Control Plane, Master | Ubuntu 25.10 | `containerd://2.1.5` |
| **i7-ubu-59d1a7** | `192.168.100.200` | `x86_64` | Control Plane, Master, NFS | Ubuntu 25.10 | `containerd://2.1.5` |
| **rpi-5b-794ad8** | `192.168.100.2` | `ARM64` | Control Plane, Master | Debian 13 (Trixie) | `containerd://2.1.5` |
| **rpi-5b-8acfe0** | `192.168.100.1` | `ARM64` | Worker | Debian 13 (Trixie) | `containerd://2.1.5` |
| **rpi-3b-0d4ab0** | `192.168.100.3` | `ARM64` | Worker (Diskless PXE) | Debian 12 (Bookworm) | `containerd://2.1.5` |
| **rpi-3bp-e9b3e3** | `192.168.100.4` | `ARM64` | Worker (Diskless PXE) | Debian 12 (Bookworm) | `containerd://2.1.5` |
| **rpi-3bp-3dc54f** | `192.168.100.5` | `ARM64` | Worker (Diskless PXE) | Debian 12 (Bookworm) | `containerd://2.1.5` |

---

## 💾 Booting & Storage Architecture

### Diskless PXE Network Booting
The Raspberry Pi 3/3+ nodes boot dynamically over the network using PXE:
1.  **Network Master:** `i7-ubu-59d1a7` (`192.168.100.200`) hosts the central **DHCP**, **TFTP**, and **NFS** servers.
2.  **TFTP Booting:** Nodes pull initial boot files from `/srv/tftpboot/<serial-id>/`.
3.  **NFS RootFS:** Each Raspberry Pi mounts its root filesystem (`/`) over NFS from isolated paths under `/srv/nfs/rpi-client-*` to avoid write collisions.

### Performance Optimizations (Raspberry Pi 3 Nodes)
To prevent network and I/O bottlenecks inherent in running low-resource (1GB RAM) nodes over NFS:
*   **Offloaded Container Runtime:** Container layers (`/var/lib/rancher/k3s/agent/containerd`) are symlinked and offloaded to local SD card partitions formatted as `ext4` and mounted at `/mnt/sdcard/`.
*   **Local Swap:** Swap is enabled using local SD cards (`/mnt/sdcard/swap`) to absorb memory spikes and prevent kernel lockups. NFS-backed swap is completely disabled.
*   **Inotify Tuning:** Kernel watch limits are tuned high (`fs.inotify.max_user_watches=524288`, `fs.inotify.max_user_instances=8192`) on all low-resource nodes to prevent pod crash loops during heavy churn.

---

## 📦 Active Workloads & Projects

Our cluster hosts several custom services, monitoring solutions, and development workspaces:

### 1. Developer Services
*   **Hermes Agent Gateway (`hermes-system`):** Containerized deployment of the autonomous Hermes Agent Gateway, complete with dedicated RBAC rules for cluster management.
*   **Coder (`coder`):** On-demand development environment provisioning and workspace management, backed by a PostgreSQL database.
*   **Gitea (`gitea`):** On-premise Git hosting and CI/CD Runner system (backed by Valkey cache).

### 2. IoT & Hardware Integrations
*   **LED Screen Panel (`led-screen`):** Custom hardware controllers containing `usb-feeder`, `k8s-proxy`, a visual `display-emulator`, and cluster-wide `network-prober` DaemonSets.
*   **Radio Scanner SDR (`radio-scanner`):** Custom SDR (Software Defined Radio) receiver service interfacing with hardware scanners to decode local radio signals.

### 3. Monitoring & Analytics
*   **VictoriaMetrics Stack (`monitoring`):** High-performance time-series logging cluster containing `vmsingle`, a configured `vmagent` (resource-optimized), and node exporters tracking historical loads. 
    *   **SRE Grafana Dashboard:** [grafana.192.168.4.247.sslip.io](http://grafana.192.168.4.247.sslip.io)
*   **Eero Monitoring (`eero-monitoring`):** Exporters tracking home Eero network statuses, Speedtest (Ookla) metrics, and custom Grafana dashboards for local topology visualization.

### 4. Storage & Platform Infrastructure
*   **Longhorn (`longhorn-system`):** Distributed, block storage engine providing high-availability storage paths for StatefulSets and PVCs.
*   **Rancher (`cattle-system`):** Master management control plane, featuring Cluster API (CAPI) turtles and fleet provisioning.
*   **Cert-Manager (`cert-manager`):** Automated internal root certificate generation for Android-locked device compatibility and secure internal HTTPS routing.

---

## 🛠️ Development & Contribution

### 1. Proposing Changes
Always create a branch from `main` to propose modifications:
```bash
git checkout -b feature/your-change-name
```

### 2. Standard Branch Naming Conventions
*   `feature/` — New applications, dashboard dashboards, or service additions.
*   `fix/` — Adjustments to resource limits, secrets, taints, or service configurations.
*   `chore/` — Refactoring, dependency updates, or documentation.

### 3. Merging
Once you push your branch, open a Pull Request on GitHub. The automated linter will execute, and you can merge it to immediately let ArgoCD synchronize the change to the cluster.
