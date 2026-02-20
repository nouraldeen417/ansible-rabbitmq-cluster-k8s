# ⚙️ Kubernetes Cluster Management with Ansible (Build, Upgrade, Remove)

This project automates the provisioning and lifecycle management of a Kubernetes cluster using **Ansible** and `kubeadm`. It supports:
- 🧱 Automatically preparing `/etc/hosts` on each node for name resolution
- ✅ Building a new HA Kubernetes cluster with container runtime and networking
- 🔁 Performing rolling, zero-downtime upgrades (via `serial`)
- ❌ Safely removing the cluster from all nodes


---

## 📌 Use Case

Deploy a consistent and production-ready Kubernetes cluster across multiple servers with minimal manual effort, and maintain it easily over time.

---

## 📂 Project Structure

```

.
├── init-servers/           # Init Ansible SSH user and privilege access
│   ├── init.yml
│   ├── vault.yml
│   └── README.md
├── k8s-cluster/            # Kubernetes build/upgrade/remove logic
│   ├── site.yml
│   ├── hosts
│   ├── roles/
│   └── README.md
├── cluster\_hosts\_template  # IP-to-FQDN entries for /etc/hosts
└── README.md               # (You're here)

````

---

## 🔧 How to Use This Repository

### 1️⃣ Initialize Your Servers (Enable Ansible Access)

First, make sure your target machines are accessible using SSH and have an `ansible-admin` user created.

Use the `init-servers` module:

```bash
cd init-servers
ansible-playbook -i hosts site.yml --ask-vault-pass
````

> This sets up the Ansible user, SSH key, sudo privileges, and disables strict host key checking.

📖 See [`init-servers/README.md`](./init-servers/README.md) for details.

---

### 2️⃣ Prepare /etc/hosts on All Nodes

To allow all cluster nodes to resolve each other's FQDNs, use the provided template:

**Example (`cluster_hosts_template`):**

```text
192.168.57.3  master-01.k8s.local master-01
192.168.57.4  master-02.k8s.local master-02
192.168.57.5  worker-01.k8s.local worker-01
192.168.57.10 lb-api-01.k8s.local lb-01
```

add your data in file   **cluster_component file**

---

### 3️⃣ Build the Kubernetes Cluster

Navigate to the cluster directory and run:

```bash
cd k8s-cluster
ansible-playbook -i hosts site.yml --tags build
```

This configures:

* HAProxy load balancer
* Containerd and Kubernetes packages
* `kubeadm init` and networking (Calico/Flannel)
* All masters and workers join the cluster
* Admin `kubeconfig` for all masters

---

### 🔁 Upgrade the Cluster (Zero Downtime)

Use the `upgrade` tag:

```bash
ansible-playbook -i hosts site.yml --tags upgrade
```

* Nodes are upgraded **one at a time** using `serial: 1`
* Keeps applications running throughout the process
* Reads updated version from group vars (`Updated_version`, `kubeadm_version`, etc.)

---

### ❌ Remove the Cluster

Use the `remove` tag:

```bash
ansible-playbook -i hosts site.yml --tags remove
```

> Removes all Kubernetes components, packages, and configuration from all nodes.


---

### 🧠 Inventory Design Notes

* **`[super-master]` group is required**:
  This must contain the initial master node where `kubeadm init` will run. This node **bootstraps the cluster**, generates the join commands, and configures the networking.

* **Why not just `master-nodes`?**
  The `super-master` is separated to allow fine-grained control for critical steps like cluster initialization and join sequencing. This also supports future upgrades or disaster recovery.

* **Inventory Variables Explained:**

  * `ansible_user`: The non-root SSH user used to connect to nodes (should have passwordless sudo).
  * `node_IP`: Internal IP address of the Kubernetes node, used in `kubeadm` configuration.
  * `HOME_USER`: Same as `ansible_user`, used to determine the user home directory for placing `kubeconfig`.
  * `LOAD_BALANCER_DNS`: FQDN used in `kubeadm` as the control plane endpoint.
  * `kubernetes_version`, `kubeadm_version`, `package_version`: Define which Kubernetes version to install or upgrade to.

---


## 🧠 Notes

* Requires Python and Ansible installed locally
* Works on any Linux-based server (tested on Ubuntu/Debian)
* Ensure proper DNS or `/etc/hosts` setup for all nodes

---
