# kubeadm-microservice-deployment

# Deploying a Microservice using Kubeadm

## 📌 Introduction
This guide walks through the process of setting up a Kubernetes cluster using `kubeadm`, configuring a master and worker node, and deploying an Nginx microservice with a NodePort service.

## 🖥️ Setup Overview
- **Master Node**: Handles cluster management (API Server, Controller Manager, etc.)
- **Worker Node**: Runs workloads (pods and containers)
- **Microservice**: Nginx exposed via NodePort on the worker node

## 🏗️ Prerequisites
Ensure you have:
- Two Virtual Machines (VMs) with Linux (Ubuntu/CentOS)
- Minimum 2 CPUs, 2GB RAM per VM
- Root/sudo access
- `kubeadm`, `kubelet`, and `kubectl` installed
- Container runtime (`containerd`) configured

## 🚀 Step 1: Prepare the Nodes
### Disable Swap on Both Nodes
```bash
sudo swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```
### Enable Kernel Modules for Kubernetes
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Configure sysctl Parameters
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system
```

## 🚀 Step 2: Install Kubernetes Components
### Add Kubernetes Repository
```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
### Install kubeadm, kubelet, and kubectl
```bash
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## 🚀 Step 3: Initialize the Kubernetes Master Node
On the **master node**, run:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```
Set up `kubectl` for the current user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 🚀 Step 4: Join the Worker Node to the Cluster
Copy the `kubeadm join` command from the master node and execute it on the worker node:
```bash
sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
Verify nodes on the master:
```bash
kubectl get nodes
```

## 🚀 Step 5: Deploy a CNI (Container Network Interface)
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

## 🚀 Step 6: Deploy Nginx Microservice
### Create a Deployment YAML
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:latest
        ports:
        - containerPort: 80
```
Apply the deployment:
```bash
kubectl apply -f nginx-deployment.yaml
```

## 🚀 Step 7: Expose Nginx Using NodePort
Create a Service YAML:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30007
  type: NodePort
```
Apply the service:
```bash
kubectl apply -f nginx-service.yaml
```

## 🚀 Step 8: Access the Nginx Microservice
Find the worker node IP and access it via:
```bash
http://<worker-node-ip>:30007
```

## 🎯 Conclusion
successfully deployed a Kubernetes cluster using `kubeadm` and run an Nginx microservice exposed via NodePort.
