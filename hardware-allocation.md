Here is a comprehensive summary of our architectural planning and technical decisions, formatted for your project documentation.

# Project Documentation: Collaboration-App Homelab

**Status:** Architecture Finalized / Infrastructure Phase

**Last Updated:** January 22, 2026

---

## 1. Hardware Overview

| Machine | Role | Specs | Key Technologies |
| --- | --- | --- | --- |
| **Mac Mini M4** | Development | 16GB RAM | Apple MLX, Local AI Testing, UI Design |
| **GMKtec K8 Plus** | Production | 96GB RAM, 2TB SSD | Proxmox, Talos K8s, Ryzen AI (NPU) |

---

## 2. Global Resource Allocation (GMKtec K8 Plus)

To balance production databases (Cassandra/Mongo) with 24/7 AI inference, we are using a **60/40 RAM split** strategy:

* **Talos K8s Cluster:** 48GB (Microservices & Databases)
* **Ubuntu AI VM:** 32GB - 40GB (LLM Inference)
* **Proxmox Host:** ~8GB (System Stability)

---

## 3. Infrastructure Architecture

### A. Talos Kubernetes Cluster (48GB RAM)

We use a **Node Specialization** strategy to ensure database stability.

| Node | Role | RAM | Primary Workloads |
| --- | --- | --- | --- |
| **talos-cp-01** | Control Plane | 4GB | K8s API, etcd |
| **talos-worker-01** | Worker (Data) | 22GB | **Cassandra** (Pinned via Node Labels) |
| **talos-worker-02** | Worker (Apps) | 22GB | **MongoDB**, Webapps, 10x Microservices |

### B. Ubuntu Inference VM (32GB RAM)

A dedicated VM for hosting LLM models that serve the Talos cluster.

* **Software Stack:** Ubuntu 24.04 LTS, Ollama, Python/MLX.
* **Hardware Passthrough:** Configured for **Radeon 780M iGPU** access.
* **Networking:** Exposed via Internal Bridge to Talos microservices as `llm-service`.

---

## 4. AI Strategy: Training vs. Inference

* **Fine-Tuning:** Performed on the **Mac Mini M4** using **MLX/LoRA**. Once the model is specialized for the "collaboration-app," the weights are exported.
* **Inference:** Specialized models are loaded onto the **GMKtec K8 Plus** Ubuntu VM.
* **Capability:** The 32GB allocation allows for **32B models (4-bit)** or high-context **8B models** (up to 128k context).

---

## 5. Technical Cheat Sheet

### Node Labeling for Database Pinning

```bash
kubectl label nodes talos-worker-01 storage-type=cassandra
kubectl label nodes talos-worker-02 storage-type=mongodb

```

### LLM Connectivity

To allow Talos microservices to reach the AI VM, update `/etc/systemd/system/ollama.service.d/override.conf`:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0"

```

---

### Next Steps

1. [ ] Provision Ubuntu 24.04 VM in Proxmox.
2. [ ] Enable IOMMU and PCI Passthrough for the Radeon 780M.
3. [ ] Configure Kubernetes `Endpoints` to map `llm-service` to the Ubuntu VM IP.

Would you like me to generate the **Kubernetes manifest (Service + Endpoints)** so your microservices can start talking to the Ubuntu VM right away?

---

[Managing Databases on Kubernetes](https://www.google.com/search?q=https://www.youtube.com/watch%3Fv%3D2vWfI2i0W3Q)
This video explains the best practices for running high-state applications like Cassandra and MongoDB in a Kubernetes environment, specifically focusing on resource isolation.