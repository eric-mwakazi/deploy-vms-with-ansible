# Ansible Proxmox Kubernetes Cluster Deployment
This Ansible project automates the deployment of a 4-node Kubernetes cluster on Proxmox. The process involves creating a Proxmox VM template, configuring VM settings, and provisioning the cluster.

## 📌 Prerequisites
Before running the playbooks, ensure you have:

1. A running Proxmox VE server.
2. Ansible installed on your control machine.
3. SSH key-based authentication set up between the control machine and Proxmox.
4. A vm template created in Proxmox with ansible control machine ssh-key copied in it eg ssh-copy-id root@template_ip.

## 🔧 Setup Instructions
### 1️⃣ Prepare the Proxmox Template
1. Create a ubuntuVM template on Proxmox.

2. Copy your SSH public key to the Proxmox template to allow Ansible to access it.
```
ssh-copy-id root@PROXMOX_IP
```
### 2️⃣ Configure Variables
Edit vm_vars_config.yml inside playbooks directory with your Proxmox API details and VM configurations:
```
PROXMOX_API_URL: "https://PROXMOX_IP:8006/api2/json"
PROXMOX_USER: "your-proxmox-user"
PROXMOX_PASSWORD: "your-api-token"

# Change based on your template id and name convention
TEMPLATE_VMID: 500
DEFAULT_IP: "192.168.1.3" # Should be your default vm template ip

VM_1_VMID: 805
VM_1_NAME: "proxy-node1"
VM_1_IP: "192.168.1.80"

VM_2_VMID: 806
VM_2_NAME: "master-node1"
VM_2_IP: "192.168.1.81"

VM_3_VMID: 807
VM_3_NAME: "worker-node1"
VM_3_IP: "192.168.1.82"

VM_4_VMID: 808
VM_4_NAME: "worker-node2"
VM_4_IP: "192.168.1.83"
```
Modify the file if you need more or fewer nodes.

### 3️⃣ Configure SSH Access to Proxmox
Ensure your SSH public key is copied to Proxmox:
```
ssh-copy-id root@PROXMOX_IP
```
### 4️⃣ Update kube_inventory File
Edit kube_inventory to define your Proxmox server:
```
[Proxmox-server]
PROXMOX_HOSTNAME ansible_host=PROXMOX_IP
```

### 5️⃣ Run the Ansible Playbook
Execute the Ansible playbook to deploy the VMs:

```
ansible-playbook -i kube_inventory deploy_k8s.yml
```
## 🚀 Playbook Overview

This Ansible project automates the provisioning of a 4-node Kubernetes cluster on Proxmox. The playbook follows these steps:

### 1️⃣ Load Environment Variables
The playbook begins by loading configuration settings from vm_vars_config.yml.

These variables include Proxmox API details, VM template ID, and node-specific configurations.

### 2️⃣ Clone VMs from the Template
The playbook clones new VMs from the existing Proxmox cloud-init template.

VM names, VMIDs, and IP addresses are assigned dynamically from the vm_vars_config.yml.
### 3️⃣ Start and Configure VMs
Each cloned VM is started and configured one by one.

The configure_vm.yml playbook is included for per-VM configuration.

### 4️⃣ Set Up Network and Host Configuration
The playbook waits for SSH to be available on the default IP (192.168.1.3).

It configures static IPs, hostnames, and updates DNS settings.

### 5️⃣ Apply Changes and Reboot VMs
The hostname is set, and the system is rebooted to apply new settings.
