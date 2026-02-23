# Kubernetes Cluster Bootstrap with Ansible

Automated deployment of a Kubernetes cluster (1 Master + N Workers) with Ansible — **100% idempotent, modular, and production-ready**.

---

## 🚀 What Gets Installed

| Component | Version | Description |
|-----------|---------|-------------|
| **Docker** | latest via `docker.io` | Container runtime with systemd cgroupdriver |
| **cri-dockerd** | `cri_dockerd_version` (default: 0.3.15) | CRI shim for Docker with `--network-plugin=cni` |
| **CNI plugins** | `cni_plugins_version` (default: 1.5.0) | Container network plugins |
| **kubeadm / kubelet / kubectl** | `kubernetes_version` (default: 1.32.3) | Kubernetes tooling |
| **Calico** | v3.28.0 | Pod network CNI (CIDR: `10.244.0.0/16`) |
| **Local Path Provisioner** | v0.0.28 | Default storage class |
| **Docker Compose** | `docker_compose_version` (default: 2.27.0) | Docker Compose v2 CLI plugin (auto-installed) |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│               Control Machine                    │
│           (your workstation / CI)                │
│        ansible-playbook site.yml                 │
└───────────────┬─────────────────────────────────┘
                │ SSH
    ┌───────────┼───────────────────┐
    ▼           ▼                   ▼
┌─────────┐ ┌─────────┐       ┌─────────┐
│ master1 │ │ worker1 │  ...  │ worker2 │
│ control │ │         │       │         │
│  plane  │ │  node   │       │  node   │
└─────────┘ └─────────┘       └─────────┘
192.168.1.10  192.168.1.20    192.168.1.30
              (example IPs — replace with yours)
```

> ✅ **CIDR**: The VMs sit on `192.168.1.x`. Pod network is `10.244.0.0/16` (standard Calico/Flannel safe CIDR) which does **not** overlap with host networks.

---

## 📋 Prerequisites

- **Ubuntu 22.04** (or 20.04) machines — 1 master + N workers
- **SSH access** to all nodes with an ed25519 key (`~/.ssh/id_ed25519`)
- **Ansible** ≥ 2.9 installed on the control machine
- **Internet access** from each VM during deployment

---

## 🔧 Step-by-Step Configuration

### Step 1 — Configure static IPs on your VMs

On each VM, configure a static IP with Netplan (`/etc/netplan/00-installer-config.yaml`).

### Step 2 — Configure SSH

```bash
# Generate an SSH key
ssh-keygen -t ed25519 -C "k8s-cluster"

# Copy to all VMs (replace IPs with yours)
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.20
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@192.168.1.30
```

### Step 3 — Configure passwordless sudo

On each VM:
```bash
sudo visudo
# Add: ubuntu ALL=(ALL) NOPASSWD:ALL
```

### Step 4 — ⚠️ Update IPs (REQUIRED)

> ⚠️ **Two files must be updated** with your actual VM IP addresses:

**`inventory.ini`**:
```ini
[masters]
master1 ansible_host=192.168.1.10   # ← replace with your IP

[workers]
worker1 ansible_host=192.168.1.20   # ← replace with your IP
worker2 ansible_host=192.168.1.30   # ← replace with your IP

[all:vars]
ansible_user=ubuntu
ansible_become=true
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

**`ssh_config`**:
```
Host master1
    HostName 192.168.1.10   # ← replace with your IP
...
```

### Step 5 — Customize variables (optional)

Edit `group_vars/all.yml` to change versions or CIDR.

---

## 🚀 Deployment

```bash
# 1. Install required Ansible collections
ansible-galaxy collection install -r requirements.yml

# 2. Test connectivity
ansible all -m ping

# 3. Check syntax
ansible-playbook site.yml --syntax-check

# 4. Deploy the cluster
ansible-playbook site.yml
```

---

## ✅ Post-deployment Verification

```bash
# Connect to the master
ssh ubuntu@192.168.1.10

# Check nodes (all must be Ready)
kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# master1   Ready    control-plane   5m    v1.32.3
# worker1   Ready    <none>          4m    v1.32.3
# worker2   Ready    <none>          4m    v1.32.3

# Check all system pods
kubectl get pods -n kube-system

# Check default storage class
kubectl get storageclass

# Check Kubernetes version
kubectl version

# Check cluster info
kubectl cluster-info
```

---

## 🐳 Docker Compose

Docker Compose v2 is **automatically installed** by the `docker` role on all nodes. It does not interfere with Kubernetes (kubelet, cri-dockerd, Calico).

```bash
# Verify installation
docker compose version
# Docker Compose version v2.27.0
```

---

## ♻️ Idempotency

The `site.yml` playbook is **fully idempotent**: it can be run multiple times on an already-deployed cluster without errors or unintended changes. The following checks ensure idempotency:

- **cri-dockerd**: checked via `stat` before download
- **Docker Compose**: checked via `stat` before download
- **kubeadm init**: checked via presence of `/etc/kubernetes/admin.conf`
- **Calico / storage provisioner**: applied only on first cluster initialization
- **kubeadm config**: `/tmp/kubeadm-config.yaml` removed immediately after `kubeadm init`
- **apt-mark hold**: managed via `dpkg_selections` (natively idempotent)

---

## 🗑️ Uninstall

```bash
# Completely remove the cluster and all components
ansible-playbook -i inventory.ini uninstall.yml
```

The `uninstall.yml` playbook removes:
- kubelet, docker, cri-docker services
- Packages (docker.io, kubelet, kubeadm, kubectl)
- Kubernetes, Docker, CNI directories
- Kubernetes APT keyrings and sources
- Kube configs (`/root/.kube`, `/home/<user>/.kube`)
- Temporary files (`/tmp/cri-dockerd*`, `/tmp/cni.tgz`, etc.)
- Docker daemon configuration (`/etc/docker/daemon.json`)

---

## 🔍 Troubleshooting

### Nodes in `NotReady` state

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl logs -n kube-system -l k8s-app=calico-node
```

Verify that the pod CIDR (`10.244.0.0/16`) does not overlap with your VM network.

### `kubeadm init` fails with CRI error

```bash
sudo systemctl status cri-docker.service
sudo systemctl status cri-docker.socket
sudo systemctl restart cri-docker.socket cri-docker.service
```

### Workers cannot join the cluster

```bash
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 50
```

### Ansible SSH connection error

```bash
ansible all -m ping
# Verify IPs in inventory.ini AND ssh_config match actual VM IPs
ssh -i ~/.ssh/id_ed25519 ubuntu@<VM_IP>
```

### Manually reset a node

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /root/.kube
```

### Docker Compose not found

```bash
ls -la /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

---

## 🗂️ Project Structure

```
.
├── ansible.cfg                          # Ansible configuration
├── inventory.ini                        # ⚠️ Update with your IPs
├── requirements.yml                     # Ansible Galaxy collection requirements
├── site.yml                             # Main playbook
├── uninstall.yml                        # Uninstall playbook
├── ssh_config                           # ⚠️ Update with your IPs
├── group_vars/
│   └── all.yml                          # Global variables
└── roles/
    ├── common/
    │   └── tasks/main.yml               # Swap, kernel modules, sysctl
    ├── docker/
    │   ├── handlers/main.yml            # Docker restart handler
    │   └── tasks/main.yml               # Docker + cri-dockerd + CNI + Compose
    ├── kubernetes/
    │   ├── handlers/main.yml            # kubelet restart handler
    │   └── tasks/main.yml               # kubeadm/kubelet/kubectl
    ├── master/
    │   ├── tasks/main.yml               # Cluster init, Calico, storage
    │   └── templates/kubeadm-config.yaml.j2
    └── worker/
        └── tasks/main.yml               # Join the cluster
```

---

## ⚙️ Available Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `kubernetes_version` | `"1.32.3"` | Kubernetes version (full semver required by kubeadm) |
| `pod_network_cidr` | `"10.244.0.0/16"` | Pod network CIDR for Calico (no overlap with `192.168.x.x`) |
| `cri_socket` | `"unix:///var/run/cri-dockerd.sock"` | CRI Docker socket path |
| `cri_dockerd_version` | `"0.3.15"` | cri-dockerd version |
| `cni_plugins_version` | `"1.5.0"` | CNI plugins version |
| `calico_manifest_url` | Calico v3.28.0 URL | Calico manifest URL |
| `docker_compose_version` | `"2.27.0"` | Docker Compose v2 version |
| `cluster_user` | `"{{ ansible_user }}"` | Cluster user |
| `ansible_become` | `true` | Automatic privilege escalation |


## 🔒 Robustesse du déploiement Ansible

- Le playbook attend automatiquement que le gestionnaire de paquets APT/Dpkg soit disponible avant chaque installation (`roles/common/tasks/ensure-apt-not-locked.yml`) : adieu les erreurs de lock.
- Le service \`unattended-upgrades\` (mises à jour automatiques Ubuntu) est désactivé temporairement au début du déploiement, pour ne jamais gêner les installations de paquets.
- Toutes les tâches d’installation apt importantes incluent une tolérance au lock (attente et retry automatique).
- **Ne jamais lancer apt/apt-get manuellement sur les nœuds durant un déploiement.**
- Fonctionne sur Ubuntu 20.04 et 22.04.

## ⚠️ Pré-requis MANUELS : Mettre à jour vos systèmes

Avant de lancer ce playbook, **VEUILLEZ TOUJOURS vous assurer que tous vos nœuds cible sont complètement à jour**.

Sur chaque nœud cible, exécutez :
\`\`\`bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
sudo reboot
\`\`\`

*Lancez le playbook SEULEMENT APRÈS vous être assuré qu'il n'y a aucune mise à jour restante sur vos VM et que tous les reboots nécessaires ont été faits.*

Cela évite les blocages, conflits APT/dpkg, ou échecs inattendus lors de l'installation du cluster Kubernetes via Ansible.
