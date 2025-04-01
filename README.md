
# Ansible Proxmox Kubernetes Cluster Deployment

This Ansible project automates the deployment of a 4-node Kubernetes cluster on Proxmox. The process involves creating a Proxmox VM template, configuring VM settings, and provisioning the cluster.

## 📌 Prerequisites

Before running the playbooks, ensure you have:

1. A running Proxmox VE server.
2. Ansible installed on your control machine.
3. SSH key-based authentication set up between the control machine and Proxmox.
4. A VM template created in Proxmox with the Ansible control machine’s SSH key copied into it (e.g., `ssh-copy-id root@template_ip`).

## 🔧 Setup Instructions

### 1️⃣ Prepare the Proxmox Template
1. Create an Ubuntu VM template on Proxmox.
2. Copy your SSH public key to the Proxmox template to allow Ansible to access it:
    ```
    ssh-copy-id root@PROXMOX_IP
    ```

### 2️⃣ Configure Variables
Edit `vm_vars_config.yml` inside the playbooks directory with your Proxmox API details and VM configurations:
```yaml
PROXMOX_API_URL: "https://PROXMOX_IP:8006/api2/json"
PROXMOX_USER: "your-proxmox-user"
PROXMOX_PASSWORD: "your-api-token"

# Change based on your template ID and naming convention
TEMPLATE_VMID: 500
DEFAULT_IP: "192.168.1.3"  # Default VM template IP

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
Edit `kube_inventory` to define your Proxmox server:
```ini
[Proxmox-server]
PROXMOX_HOSTNAME ansible_host=PROXMOX_IP
```

### 5️⃣ Run the Ansible Playbook
Execute the Ansible playbook to deploy the VMs:
```
ansible-playbook -i inventory.ini main.yml
```

## 🚀 Playbook Overview

This Ansible project automates the provisioning of a 4-node Kubernetes cluster on Proxmox. The playbook follows these steps:

### 1️⃣ Load Environment Variables
The playbook begins by loading configuration settings from `vm_vars_config.yml`. These variables include Proxmox API details, VM template ID, and node-specific configurations.

### 2️⃣ Clone VMs from the Template
The playbook clones new VMs from the existing Proxmox cloud-init template. VM names, VMIDs, and IP addresses are assigned dynamically from `vm_vars_config.yml`.

### 3️⃣ Start and Configure VMs
Each cloned VM is started and configured one by one. The `configure_vm.yml` playbook is included for per-VM configuration.

### 4️⃣ Set Up Network and Host Configuration
The playbook waits for SSH to be available on the default IP (192.168.1.3). It configures static IPs, hostnames, and updates DNS settings.

### 5️⃣ Apply Changes and Reboot VMs
The hostname is set, and the system is rebooted to apply new settings.

### 6️⃣ Install Prerequisites (Kubelet, Docker, Containerd, Kubeadm)
The following tools are installed on each node:
- **Docker**: A container runtime for Kubernetes.
- **Containerd**: Another container runtime.
- **Kubelet**: The Kubernetes node agent.
- **Kubeadm**: A tool for bootstrapping Kubernetes clusters.
  
This installation is automated through Ansible tasks in `install_tools.yml`.

```yaml
- name: Install Docker, Containerd, Kubelet, and Kubeadm
  hosts: all
  become: yes
  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present
    - name: Install Containerd
      apt:
        name: containerd
        state: present
    - name: Install Kubelet, Kubeadm, Kubectl
      apt:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
```

### 7️⃣ Initialize Kubernetes Cluster (Master Node)
On the master node, the Kubernetes cluster is initialized using `kubeadm`. The `kubeadm_init.yml` playbook handles this step:
```yaml
- name: Initialize Kubernetes Cluster on Master
  hosts: master
  become: yes
  tasks:
    - name: Initialize Kubernetes Cluster with Kubeadm
      command: kubeadm init --pod-network-cidr=192.168.0.0/16
      register: kubeadm_init_output
    - name: Set up kubeconfig
      copy:
        dest: /root/.kube/config
        content: "{{ kubeadm_init_output.stdout }}"
```

### 8️⃣ Join Worker Nodes to the Cluster
Once the master node is initialized, worker nodes join the cluster using the token generated during `kubeadm init`. This is handled by `join_worker_nodes.yml`:
```yaml
- name: Join Worker Nodes to the Cluster
  hosts: worker
  become: yes
  tasks:
    - name: Join Worker to Kubernetes Cluster
      command: kubeadm join --token {{ token }} --discovery-token-ca-cert-hash sha256:{{ hash }}
```

### 9️⃣ Deploy Nginx on Kubernetes
The playbook deploys an Nginx web server as a test application to verify the Kubernetes cluster is functioning correctly:
```yaml
- name: Deploy Nginx on Kubernetes
  hosts: master
  become: yes
  tasks:
    - name: Apply Nginx Deployment
      kubectl:
        definition: 
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: nginx
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: nginx
            template:
              metadata:
                labels:
                  app: nginx
              spec:
                containers:
                - name: nginx
                  image: nginx
    - name: Expose Nginx Deployment
      kubectl:
        definition:
          apiVersion: v1
          kind: Service
          metadata:
            name: nginx-service
          spec:
            ports:
            - port: 80
              targetPort: 80
            selector:
              app: nginx
```

### 🔟 Deploy Kubernetes Dashboard
The Kubernetes dashboard is deployed for a web-based UI to manage the cluster:
```yaml
- name: Deploy Kubernetes Dashboard
  hosts: master
  become: yes
  tasks:
    - name: Apply Dashboard YAML
      kubectl:
        definition: 
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: kubernetes-dashboard
            namespace: kubernetes-dashboard
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: kubernetes-dashboard
            template:
              metadata:
                labels:
                  app: kubernetes-dashboard
              spec:
                containers:
                - name: kubernetes-dashboard
                  image: kubernetesui/dashboard:v2.0.0
```

