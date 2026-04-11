# Kubernetes HA Homelab on UTM (Apple Silicon)

Production-grade, highly available Kubernetes cluster built from scratch on macOS using UTM (QEMU) virtualization. Fully automated with Ansible — no kubeadm, no managed services, just raw binaries and certificates.

## Highlights

- **Kubernetes v1.32.0** — the hard way, installed from official binaries (ARM64)
- **11 Ubuntu 24.04 VMs** on UTM with cloud-init provisioning
- **HashiCorp Vault PKI** — 3-tier CA hierarchy for all TLS certificates
- **3-node etcd cluster** with mutual TLS
- **HA control plane** — 2 masters behind HAProxy load balancer
- **Jump/bastion server** — Mac connects only to the jump host; all cluster management happens from there
- **Calico CNI** for pod networking
- **Single-command deployment** — one Ansible playbook or one shell script provisions VMs, installs everything, and outputs a working cluster

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Mac Host (Apple Silicon)                        │
│                      192.168.64.1 gateway                          │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                       UTM Shared Network
                        192.168.64.0/24
                               │
       ┌───────────┬───────────┼───────────┬───────────┐
       │           │           │           │           │
  ┌────┴────┐ ┌────┴────┐ ┌────┴────┐ ┌────┴────┐     │
  │ HAProxy │ │  Vault  │ │  Jump   │ │  etcd   │     │
  │   .10   │ │   .11   │ │   .12   │ │ .21-.23 │     │
  │   (LB)  │ │  (PKI)  │ │(bastion)│ │(cluster)│     │
  └────┬────┘ └─────────┘ └────┬────┘ └────┬────┘     │
       │                       │            │          │
       │    ┌──────────────────┴────────────┘          │
       │    │                                          │
  ┌────┴────┴──────────────────────────────────────────┴──┐
  │                   Kubernetes Cluster                   │
  │                                                        │
  │   Control Plane             Workers                    │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐            │
  │   │ master-1 │  │ worker-1 │  │ worker-2 │            │
  │   │   .31    │  │   .41    │  │   .42    │            │
  │   ├──────────┤  ├──────────┤  ├──────────┤            │
  │   │ master-2 │  │ worker-3 │  │          │            │
  │   │   .32    │  │   .43    │  │          │            │
  │   └──────────┘  └──────────┘  └──────────┘            │
  └────────────────────────────────────────────────────────┘
```

## VM Specifications

| VM | Role | IP | vCPU | RAM | Disk |
|----|------|----|------|-----|------|
| haproxy | API Server Load Balancer | 192.168.64.10 | 2 | 2 GB | 20 GB |
| vault | PKI & Secrets (HashiCorp Vault 1.15.4) | 192.168.64.11 | 2 | 4 GB | 20 GB |
| jump | Bastion / Ansible Controller | 192.168.64.12 | 2 | 4 GB | 20 GB |
| etcd-1 | etcd cluster member | 192.168.64.21 | 2 | 2 GB | 20 GB |
| etcd-2 | etcd cluster member | 192.168.64.22 | 2 | 2 GB | 20 GB |
| etcd-3 | etcd cluster member | 192.168.64.23 | 2 | 2 GB | 20 GB |
| master-1 | K8s control plane | 192.168.64.31 | 2 | 4 GB | 30 GB |
| master-2 | K8s control plane | 192.168.64.32 | 2 | 4 GB | 30 GB |
| worker-1 | K8s worker node | 192.168.64.41 | 2 | 6 GB | 40 GB |
| worker-2 | K8s worker node | 192.168.64.42 | 2 | 6 GB | 40 GB |
| worker-3 | K8s worker node | 192.168.64.43 | 2 | 6 GB | 40 GB |
| **Total** | **11 VMs** | | **22** | **38 GB** | **300 GB** |

## Component Versions

| Component | Version |
|-----------|---------|
| Kubernetes | 1.32.0 |
| etcd | 3.5.12 |
| containerd | 1.7.24 |
| runc | 1.2.4 |
| Calico CNI | 3.28.0 |
| HashiCorp Vault | 1.15.4 |
| HAProxy | 2.8 |
| Ubuntu | 24.04 LTS (ARM64 cloud image) |

## Prerequisites

- **macOS** on Apple Silicon (Tested on M4)
- **UTM** installed from [utm.app](https://mac.getutm.app/)
- **~38 GB free RAM** (all 11 VMs running simultaneously)

```bash
# Install required tools
brew install ansible

# Link utmctl for CLI-based VM control
sudo ln -sf /Applications/UTM.app/Contents/MacOS/utmctl /usr/local/bin/utmctl

# Install required Ansible collection
cd ansible && ansible-galaxy collection install -r requirements.yml
```

## Quick Start — Single Command Deployment

Two fully automated options to deploy the entire cluster end-to-end:

### Option A: Ansible Playbook (from Mac)

```bash
cd ~/k8s-utm-ha-homelab/ansible
ansible-playbook -i inventory/localhost.yml playbooks/k8s-utm-ha-homelab.yml --ask-become-pass
```

### Option B: Shell Script (from Mac)

```bash
cd ~/k8s-utm-ha-homelab
./scripts/k8s-utm-ha-homelab.sh
```

Both options perform the same 17-step end-to-end deployment:

1. **Creates 11 UTM VMs** with Ubuntu cloud images and cloud-init
2. **Configures Mac** — `/etc/hosts`, SSH config for jump host
3. **Downloads all binaries** in parallel (K8s, etcd, containerd, runc, Calico)
4. **Sets up the jump server** — copies SSH keys, Ansible project, binaries
5. **Bootstraps Vault** — install, initialize, unseal
6. **Configures Vault PKI** — 3-tier CA hierarchy with all certificate roles
7. **Issues & deploys certificates** to all nodes via Vault
8. **Deploys etcd cluster** (3 nodes with mutual TLS) + **HAProxy** (in parallel)
9. **Deploys control plane** — kube-apiserver, controller-manager, scheduler on 2 masters
10. **Deploys worker nodes** — containerd, kubelet, kube-proxy on 3 workers
11. **Installs Calico CNI** and verifies the cluster

### Access the Cluster

```bash
# SSH to the jump server (only host accessible from Mac)
ssh jump

# From jump, access any VM
ssh master-1
ssh worker-1

# Use kubectl (pre-configured on jump)
kubectl get nodes
kubectl get pods -A
```

## Step-by-Step Deployment

For more control, run each phase individually from the jump server:

```bash
# Phase 1: Bootstrap Vault (install, init, unseal, PKI setup)
ansible-playbook playbooks/vault-full-setup.yml

# Phase 2: Issue and deploy certificates to all nodes
ansible-playbook playbooks/k8s-certs.yml

# Phase 3: Deploy etcd cluster
ansible-playbook playbooks/etcd-cluster.yml

# Phase 4: Deploy HAProxy load balancer
ansible-playbook playbooks/haproxy.yml

# Phase 5: Deploy control plane (2 masters)
ansible-playbook playbooks/control-plane.yml

# Phase 6: Deploy worker nodes (3 workers)
ansible-playbook playbooks/worker.yml
```

## Deployment Flow

```
┌─ Phase 1: Mac Localhost ───────────────────────────────────────────┐
│  Create VMs ──┐                                                    │
│               ├── Mac setup + download binaries (while VMs boot)   │
│               └── Wait for jump SSH                                │
├─ Phase 2: Jump Server Setup ──────────────────────────────────────┤
│  Configure jump ── Copy binaries ── Wait for Ansible               │
├─ Phase 3: Cluster Deployment (from jump) ─────────────────────────┤
│  Vault full setup (bootstrap + PKI)                                │
│  ↓                                                                 │
│  Issue & deploy K8s certificates (all nodes, forks=12)             │
│  ↓                                                                 │
│  etcd cluster ──┬── (parallel)                                     │
│  HAProxy ───────┘                                                  │
│  ↓                                                                 │
│  Control plane (2 masters)                                         │
│  ↓                                                                 │
│  Worker nodes (3 workers)                                          │
│  ↓                                                                 │
│  Calico CNI                                                        │
└────────────────────────────────────────────────────────────────────┘
```

## PKI Architecture

All TLS certificates are issued by HashiCorp Vault using a 3-tier CA hierarchy:

```
Root CA (365 days, pathlen:2)
└── Intermediate CA (180 days, pathlen:1)
    ├── Kubernetes CA (90 days, pathlen:0)
    │   ├── kube-apiserver (server + kubelet-client)
    │   ├── kube-controller-manager
    │   ├── kube-scheduler
    │   ├── admin (cluster-admin)
    │   ├── service-account signing key
    │   ├── kube-proxy
    │   └── kubelet (server + client per node)
    ├── etcd CA (90 days, pathlen:0)
    │   ├── etcd-server (per node)
    │   ├── etcd-peer (inter-node)
    │   ├── etcd-client (apiserver → etcd)
    │   └── etcd-healthcheck-client
    └── Front Proxy CA (90 days, pathlen:0)
        └── front-proxy-client (API aggregation)
```

Separate CAs for Kubernetes, etcd, and front-proxy provide security isolation — a compromised certificate in one domain cannot be used in another.

## Project Structure

```
k8s-utm-ha-homelab/
├── README.md
├── execution_flow.txt               # Deployment flow reference
│
├── scripts/
│   ├── k8s-utm-ha-homelab.sh        # Shell script for VM creation (alternative)
│   ├── start_vms.sh                 # Start all 11 VMs
│   ├── stop_vms.sh                  # Stop all VMs (frees RAM)
│   └── destroy_utm_vms.sh           # Delete all VMs (destructive)
│
├── ansible/
│   ├── ansible.cfg                  # Ansible config (forks=12, pipelining)
│   ├── requirements.yml             # community.hashi_vault collection
│   │
│   ├── inventory/
│   │   ├── homelab.yml              # All 11 hosts grouped by role
│   │   └── localhost.yml            # Mac localhost inventory
│   │
│   ├── playbooks/
│   │   ├── k8s-utm-ha-homelab.yml   # Full end-to-end deployment
│   │   ├── vault-bootstrap.yml      # Install & initialize Vault
│   │   ├── vault-pki.yml            # Configure PKI hierarchy
│   │   ├── vault-full-setup.yml     # Bootstrap + PKI combined
│   │   ├── vault-issue-certs.yml    # Issue certs to Vault KV
│   │   ├── k8s-certs.yml           # Issue & deploy certs to nodes
│   │   ├── etcd-cluster.yml         # Deploy 3-node etcd cluster
│   │   ├── haproxy.yml              # Configure API server LB
│   │   ├── control-plane.yml        # Deploy K8s masters
│   │   ├── worker.yml               # Deploy K8s workers + kubeconfig
│   │   └── ping.yml                 # Connectivity test
│   │
│   └── roles/
│       ├── vm-provision/            # Create UTM VMs with cloud-init
│       ├── mac-setup/               # Configure Mac /etc/hosts + SSH
│       ├── download-binaries/       # Parallel download of all binaries
│       ├── jump-setup/              # Configure bastion server
│       ├── vault-bootstrap/         # Install, init, unseal Vault
│       ├── vault-pki/               # 3-tier CA hierarchy in Vault
│       ├── k8s-certs/               # Issue & deploy certs to nodes
│       ├── etcd/                    # etcd cluster with mTLS
│       ├── haproxy/                 # HAProxy for API server HA
│       ├── control-plane/           # apiserver, controller-manager, scheduler
│       └── worker/                  # containerd, kubelet, kube-proxy
│
├── docs/
│   └── 01-vm-setup-explained.md     # Deep dive on VM provisioning
│
├── images/                          # Ubuntu cloud images (generated)
├── iso/                             # Cloud-init ISOs (generated)
└── k8s-binaries/                    # Downloaded binaries (cached)
```

## Networking

| Network | CIDR | Purpose |
|---------|------|---------|
| VM Network | 192.168.64.0/24 | UTM shared network for all VMs |
| Service CIDR | 10.96.0.0/12 | Kubernetes ClusterIP services |
| Pod CIDR | 10.244.0.0/16 | Calico pod network |

**HAProxy** load balances the Kubernetes API server (port 6443) across both masters using TCP mode with health checks.

**Jump server** acts as the bastion host — the Mac only needs SSH access to jump (192.168.64.12). All Ansible playbooks for cluster setup run from the jump server.

## VM Management

```bash
# Start all VMs
./scripts/start_vms.sh

# Stop all VMs (frees ~38 GB RAM)
./scripts/stop_vms.sh

# Destroy all VMs (permanent deletion)
./scripts/destroy_utm_vms.sh
```

> **Note:** After stopping VMs, Vault will need to be unsealed on next start. A `vault-unseal` helper is configured in the jump server's `.bashrc`.

## Ansible Usage

```bash
cd ~/k8s-utm-ha-homelab/ansible

# Test connectivity to all hosts
ansible all -m ping

# Target specific groups
ansible k8s_masters -m shell -a "hostname"
ansible etcd_servers -m shell -a "systemctl is-active etcd"
ansible k8s_workers -m shell -a "systemctl is-active kubelet"
```

## Troubleshooting

### VM doesn't get the correct IP

1. Check cloud-init logs: `sudo cat /var/log/cloud-init-output.log`
2. Verify the cloud-init ISO is attached as a CD-ROM drive
3. Check the network interface name: `ip link show` (could be `enp0s1`, `ens3`, etc.)

### Can't SSH to a VM

1. Verify the VM is running: `utmctl list`
2. Check bridge interface: `ifconfig bridge100`
3. SSH to jump first, then hop: `ssh jump`, then `ssh master-1`

### Vault is sealed after VM restart

```bash
ssh jump
vault-unseal   # helper function in .bashrc
```

### etcd cluster unhealthy

```bash
ssh etcd-1
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/healthcheck-client.crt \
  --key=/etc/etcd/pki/healthcheck-client.key
```

### API server not responding

```bash
ssh master-1
sudo systemctl status kube-apiserver
sudo journalctl -u kube-apiserver --no-pager -l
```

## Documentation

- [VM Setup Deep Dive](docs/01-vm-setup-explained.md) — how cloud-init, QEMU backend, and UTM provisioning work
- [Vault PKI Role](ansible/roles/vault-pki/README.md) — detailed PKI hierarchy and certificate roles

## License

MIT
