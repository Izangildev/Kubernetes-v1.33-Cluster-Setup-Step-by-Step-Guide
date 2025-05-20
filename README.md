# Kubernetes-v1.33-Cluster-Setup-Step-by-Step-Guide
This repository contains a detailed, step-by-step guide for creating a Kubernetes v1.33 cluster from scratch. It includes all the necessary configurations, commands, and explanations to help you understand and build your own K8s cluster — perfect for homelabs, testing environments, or learning purposes.

This guide is based on the official documentation for Kubernetes version 1.33 and is intended for x86_64 architecture devices. Any other version or architecture may require changes to the commands (such as package versions).

---
# 📚 Sources:
- Configure systemd: https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- Install containerd Option 1: https://github.com/containerd/containerd
- Install Kubelet, Kubeadm and Kubectl v1.33: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- Create a cluster: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- Configure the cdriver (default systemd): https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet-cgroup-driver
- Setting the CNI: https://vincent0426.medium.com/setting-up-a-kubernetes-cluster-with-calico-cni-and-applying-network-policies-c196b4f25687
---

# Step 1: Install containerd as container runtime:
Enable IP forwarding:
```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
On the Containerd GitHub page, look for the latest version and update the download link according to the version and architecture, then execute following commands:
```bash
wget https://github.com/containerd/containerd/releases/download/v2.1.0/containerd-2.1.0-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local/ containerd-2.1.0-linux-amd64.tar.gz
```  
We need to install the containerd.service to run the Containerd service using the systemd driver. Before that, we'll create the directory where it will be installed.
```bash
sudo mkdir -p /usr/local/lib/systemd/system
sudo curl -Lo /usr/local/lib/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```
Installing runc is required to run containers with Containerd. To install it, follow these commands:
```bash
sudo wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
The next step is to install the CNI plugins for network management:
```bash
sudo wget https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz
```
On Linux, control groups (cgroups) manage resource allocation for processes. Both kubelet and the container runtime must use the same cgroup driver to enforce CPU and memory limits consistently. Proper configuration of this driver on both components is essential.
We will use the systemd driver. To configure it, follow these steps, first of all, we will create a default config.toml file:
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
In that file we will locate the line [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options] and add the following line below:
```bash
SystemdCgroup = true
```
Finally, to verify if all the steps worked, restart and check the containerd.service:
```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```

# Step 2: Install Kubeadm, Kubelet and Kubectl.
First, update the package repositories and install the necessary packages to manage HTTPS repositories and certificates.
```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```
Download and import the public signing key used by Kubernetes package repositories. The same key is used across all repositories, so the version in the URL can be ignored. If the /etc/apt/keyrings directory does not exist, it should be created before importing the key.
```bash
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
Add the official Kubernetes repository to the system and install the essential packages: kubectl, kubeadm, and kubelet. These packages are marked to hold to prevent automatic updates, avoiding potential compatibility issues.
```bash
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
sudo echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt-get install -y kubectl kubeadm kubelet
sudo apt-mark hold kubelet kubeadm kubectl
```
Finally, enable the kubelet service to start automatically on system boot.
```bash
sudo systemctl enable --now kubelet
```

> It is necessary to run the previous steps on every single node to ensure Kubernetes runs correctly on all nodes.
>

# Step 3: Create the Cluster and Install CNI (Calico):
Initialize the Kubernetes control plane with the specified Pod network CIDR.
```bash
sudo kubeadm init --pod-network-cidr=172.16.0.0/16
```
If successful, you will see an output similar to this:
```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
Execute the following commands to configure access to the cluster:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Download and extract the Calico network plugin:
```bash
mkdir calico
sudo wget https://github.com/projectcalico/calico/releases/download/v3.30.0/release-v3.30.0.tgz
sudo tar -C calico/ -xzvf release-v3.30.0.tgz
```
To apply Calico, run the manifests located in the /calico/release-v[version]/manifests directory. After applying the custom-resources.yaml manifest, remember to modify the Pod CIDR specified in the kubeadm init command accordingly.
```bash
kubectl create -f tigera-operator.yaml
kubectl create -f custom-resources.yaml
# Check status
watch kubectl get tigerastatus
```
# Adding a worker node to the cluster:
After initializing the cluster with kubeadm init, a kubeadm join command is output. Use this exact command on each worker node to join them to the cluster. This command includes the token and discovery token CA certificate hash needed for secure connection.
If you lost the command, regenerate it with:
```bash
kubeadm token create --print-join-command
```
---
# ✅ Final notes
- Use kubectl get nodes to verify node status.
- Tail logs for debugging: journalctl -u kubelet -f
