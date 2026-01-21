```markdown
# 🚀 Home Lab: Talos Kubernetes Production Cluster

This repository contains the configuration and operational procedures for the high-performance Talos Linux cluster running on Proxmox.

### 🏠 Hardware & Environment
* **Production Node:** Home Server (96GB RAM / 2TB SSD)
* **Management Device:** Local Machine (Control Machine)
* **Virtualization:** Proxmox VE

### 🌐 Network Mapping (Example)
| Node Role | Hostname | Static IP |
| :--- | :--- | :--- |
| **Control Plane** | `talos-cp-01` | `192.168.1.10` |
| **Worker 1** | `talos-worker-01` | `192.168.1.11` |
| **Worker 2** | `talos-worker-02` | `192.168.1.12` |

---

## 1. Initial Setup & Configuration
Generate secrets and point your local machine's CLI to the cluster.

```bash
# 1. Generate base config
talosctl gen config my-cluster https://<CONTROL_PLANE_IP>:6443

# 2. Configure local talosconfig endpoints
talosctl config endpoint <CONTROL_PLANE_IP>
talosctl config node <CONTROL_PLANE_IP>

# 3. Retrieve Kubernetes credentials (after initialization)
talosctl kubeconfig ~/.kube/config

```

---

## 2. Cluster Deployment & Node Recovery

Use these commands to push configurations. If nodes show TLS/Certificate errors, use **Maintenance Mode** or **Reset** via the Proxmox console.

### Initial Apply

```bash
# Control Plane
talosctl apply-config -n <CONTROL_PLANE_IP> --file controlplane.yaml --insecure

# Workers (Apply while in Maintenance Mode if locked)
talosctl apply-config -n <WORKER_IP> --file worker.yaml --insecure

```

### Emergency Recovery ("The Industrial Reset")

If `tls: certificate required` errors block access:

1. **Proxmox:** Force Reset the VM.
2. **Talos Boot Menu:** Select **Reset Talos Installation**.
3. **Local Machine:** Re-run `apply-config` with the `--insecure` flag.

---

## 3. Comprehensive Health Checks

Monitor the health of your 96GB RAM pool and internal services.

| Check | Command |
| --- | --- |
| **Cluster Members** | `talosctl get members -n <CONTROL_PLANE_IP>` |
| **Node Dashboard** | `talosctl dashboard -n <NODE_IP>` |
| **K8s Node Status** | `kubectl get nodes -o wide` |
| **System Pods** | `kubectl get pods -n kube-system` |
| **RAM Capacity** | `kubectl describe nodes |
| **Kubelet Logs** | `talosctl logs -n <node-ip> kubelet` |

---

## 4. Maintenance & Hardware Management

### Identifying Missing IPs

If a worker IP isn't showing in Proxmox (missing Guest Agent):

* **ARP Scan:** `arp -a | grep "<MAC_PREFIX>"` - Check your network's ARP table.
* **Router DHCP:** Check your router's DHCP leases for assigned IPs.

### Safe Shutdown & Reboot

Ensure 24/7 AI agents are scaled down before performing physical maintenance on the home server.

```bash
# Graceful Software Reboot
talosctl reboot -n <NODE_IP>

# Full Cluster Shutdown for Physical Maintenance
talosctl shutdown -n <CONTROL_PLANE_IP>,<WORKER1_IP>,<WORKER2_IP>

```

---

### 🛡️ Best Practices for "No-Mess" Operations

1. **Strict Static IPs:** Define `addresses` under `machine.network.interfaces` in your YAML to prevent IP shifting.
2. **Dry Runs:** Use `--mode=try` when applying network changes; Talos will auto-revert if you lose connectivity.
3. **Encrypted Backups:** Store your `talosconfig` and `.yaml` files in a private, secure Git repo.

```
