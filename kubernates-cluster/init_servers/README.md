# 🔧 Init Servers with Ansible

This module automates the initial setup of remote servers to prepare them for Ansible control. It simplifies the process of creating a privileged `ansible-admin` user and securely storing credentials using Ansible Vault.

## 📌 Purpose

When managing multiple new servers, manually configuring Ansible users and privileges is time-consuming. This playbook helps you:

- Create a new `ansible-admin` user
- Set up passwordless sudo
- Copy SSH keys
- Disable strict host checking
- Store all sensitive data using Ansible Vault

## 📁 Inventory Example

```ini
[new_servers]
server1 ansible_host=server-ip ansible_user=your-user ansible_ssh_password="{{ vault_server1_ssh_pass }}" ansible_become_pass="{{ vault_server1_sudo_pass }}"
server2 ansible_host=server-ip ansible_user=your-user ansible_ssh_password="{{ vault_server2_ssh_pass }}" ansible_become_pass="{{ vault_server2_sudo_pass }}"

[new_servers:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
new_ansible_user=ansible-admin
````

## 🔐 Ansible Vault

To keep your passwords secure, store them in a `vault.yml` file like this:

```yaml
vault_server1_ssh_pass: your_password
vault_server1_sudo_pass: your_password
```

Encrypt it using:

```bash
ansible-vault encrypt vault.yml
```

## ▶️ Usage

Run the playbook to initialize the servers:

```bash
ansible-playbook -i hosts init.yml --ask-vault-pass
```

## 📦 Features

* SSH key-based access setup
* `ansible-admin` user with sudo privileges
* No manual editing per server — works from a centralized hosts file
* Secure credential handling via Vault
* Easy onboarding of new machines

## 📂 Folder Structure

```
init-servers/
├── init.yml
├── hosts
├── vault.yml
└── README.md
```

---

> After this setup, you can manage these servers using the new `ansible-admin` user.


