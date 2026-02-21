
---

# Step 0 – Infrastructure

## Objective

Prepare freshly provisioned Linux servers for Kubernetes automation using Ansible.

Since new servers only support password-based SSH access, this step creates a dedicated automation user and enables secure, passwordless authentication for future cluster deployment tasks.

---

## What Was Implemented

### 1️⃣ Secure Initial Access

* Defined raw servers in the Ansible inventory.
* Used **Ansible Vault** to encrypt SSH and sudo passwords.
* Avoided storing any credentials in plaintext.

This allows secure first-time access to new machines.

---
![alt text](<screenshots/Screenshot 2026-02-21 011459.png>)
### 2️⃣ Created Automation Service Account

A dedicated user (`ansible-admin`) was created on all nodes:

* Added to the `sudo` group
* Configured with `/bin/bash`
* Intended only for automation tasks

This separates human access from automation access.

---

### 3️⃣ Enabled SSH Key Authentication

* The controller’s public SSH key was installed on each server.
* This allows passwordless login.
* Eliminates interactive prompts during automation.

---

### 4️⃣ Configured Passwordless Sudo

* A dedicated file was created under `/etc/sudoers.d/`
* Enabled `NOPASSWD` for the automation user
* Used `visudo` validation to prevent configuration errors

This ensures safe, non-interactive privilege escalation.

---
![alt text](<screenshots/Screenshot 2026-02-21 011747.png>)


---

# Kubernetes Cluster Deployment – Automation Design

## Overview

After completing the infrastructure bootstrapping phase (SSH + automation user), the Kubernetes cluster is deployed using a role-based Ansible architecture.

The deployment is divided into **four logical stages**:

1. System preparation
2. Kubernetes prerequisites
3. Control plane initialization
4. Worker node joining

The `site.yml` playbook orchestrates these stages sequentially.

---

# Stage 1 – System Preparation

**Hosts:** `kube_cluster`

### Role: `common`

This role prepares all nodes with baseline requirements before Kubernetes installation.

Responsibilities:

* Install required system packages
* Configure system dependencies
* Update `/etc/hosts` with cluster node mappings

Purpose:

Ensure all nodes can resolve each other and meet the base OS requirements before Kubernetes components are installed.

---

# Stage 2 – Kubernetes Node Preparation

**Hosts:** `kube_cluster` (Masters + Workers)

This stage transforms Linux servers into Kubernetes-ready nodes.

---

### Role: `swap_manage`

* Disables swap immediately
* Removes swap entries from `/etc/fstab`
* Ensures compliance with Kubernetes requirements

Why:
Kubernetes requires swap to be disabled for proper memory management.

---

### Role: `containerd`

* Installs containerd runtime
* Configures required kernel modules
* Applies correct cgroup driver configuration
* Starts and enables containerd service

Why:
Kubernetes requires a container runtime to run pods.

---

### Role: `k8s_packages`

* Adds Kubernetes GPG key
* Adds Kubernetes repository
* Installs:

  * `kubelet`
  * `kubeadm`
  * `kubectl`
* Enables kubelet service

Why:
These components are required for cluster initialization and node participation.

---

# Stage 3 – Control Plane Initialization

**Hosts:** `master-nodes`

This stage initializes the Kubernetes control plane.

---

### Role: `cluster_init`

* Executes `kubeadm init`
* Generates cluster certificates
* Creates bootstrap token for workers
* Initializes API server and control plane components

Result:
The Kubernetes control plane becomes operational.

---

### Role: `k8s_admin`

* Copies `admin.conf`
* Configures kubeconfig for cluster administration

Result:
The master node can manage the cluster using `kubectl`.

---

### Role: `cluster_network`

* Deploys Weave CNI manifest
* Enables pod-to-pod networking

Why:
Without a CNI plugin, pods cannot communicate across nodes.

---

### Role: `cluster_status`

* Verifies control plane status
* Checks node readiness
* Confirms system pods are running

Purpose:
Ensure cluster health before joining worker nodes.

---

# Stage 4 – Worker Node Join

**Hosts:** `kube_cluster`

### Role: `cluster_join_workers`

* Retrieves join command from master
* Executes `kubeadm join` on worker nodes
* Validates successful registration

Result:
All worker nodes become part of the cluster.

---
![alt text](screenshots/image.png)
# Execution Flow Summary

```
System Preparation
        ↓
Disable Swap
        ↓
Install Container Runtime
        ↓
Install Kubernetes Packages
        ↓
Initialize Control Plane
        ↓
Deploy CNI Network
        ↓
Join Worker Nodes
        ↓
Cluster Ready
```
# verfication
![alt text](<screenshots/Screenshot 2026-02-17 132639.png>) 

![alt text](<screenshots/Screenshot 2026-02-17 132710.png>) 

![alt text](<screenshots/Screenshot 2026-02-21 003344.png>)

# Stage 5 – RabbitMQ Operator Installation

## Objective
Install the RabbitMQ Cluster Operator using Helm to enable Kubernetes-native management of RabbitMQ clusters.

The operator manages:

- Stateful deployment  
- Configuration reconciliation  
- Self-healing  
- Scaling operations  

---

## Helm Installation
Helm is installed directly on the control plane node:

- Download Helm binary  
- Extract archive  
- Move binary to `/usr/local/bin`  

**Purpose:**  
Helm is required to install the RabbitMQ Operator chart from Bitnami.

---

## Add Bitnami Repository
The Bitnami Helm repository is added:

```

[https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)

```

**Why:**  
The RabbitMQ Cluster Operator chart is maintained and distributed via Bitnami.

---


**Result:**  
Kubernetes is now capable of managing RabbitmqCluster custom resources (CRDs).  
The operator continuously watches cluster definitions and enforces the desired state.

---

# Stage 6 – RabbitMQ Cluster Deployment

## Objective
Deploy a highly available RabbitMQ cluster using the installed operator.

---

## Namespace Creation
Namespace created: `rabbitmq`

**Purpose:**  
Isolate messaging infrastructure from system components.

---

## RabbitMQ Custom Resource Deployment
A `RabbitmqCluster` resource is applied.

**Key specifications:**

- Replicas: 2 nodes  
- Image: `bitnamilegacy/rabbitmq:4.1.3-debian-12-r1`  
- Service Type: NodePort  
- Persistent Storage: 10Gi  
- StorageClass: local-path  
- Resource Limits: Defined CPU and memory boundaries  

---

## Architecture Behavior
When the custom resource is applied, the operator detects the new RabbitmqCluster and creates:

- StatefulSet  
- Services  
- PersistentVolumeClaims  
- ConfigMaps  
- Secrets  
- Configures clustering between replicas  
- Exposes RabbitMQ via NodePort  

**Result:**  
RabbitMQ runs as a stateful, replicated message broker cluster inside Kubernetes.

---

## Stateful Architecture
RabbitMQ is deployed using:

- StatefulSet  
- Persistent Volumes  
- Clustered Erlang distribution  

**Each pod:**

- Has stable network identity  
- Retains persistent data  
- Automatically rejoins cluster after restart  

---

## Execution Flow Summary

```

Kubernetes Cluster Ready
↓
Install Helm
↓
Add Bitnami Repository
↓
Install RabbitMQ Operator
↓
Create Messaging Namespace
↓
Deploy RabbitmqCluster CR
↓
Operator Creates StatefulSet
↓
RabbitMQ Cluster Operational

```

---

## Operational Validation

Cluster health can be verified by checking:

- Operator pods running in `rabbitmq-system`  
- RabbitMQ pods running in `rabbitmq`  
- StatefulSet replicas ready  
- Services exposed via NodePort  
- PersistentVolumeClaims bound  

When all pods are **Running** and **Ready**, the RabbitMQ cluster is fully operational.
```

```
# verfication
![alt text](<screenshots/rabbit-pods.png>) 
![alt text](<screenshots/rabbit-svc.png>) 
![alt text](<screenshots/rabbit-verfication.png>)

 