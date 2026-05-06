#  Kubernetes HA Cluster Setup using kubeadm (EC2 + HAProxy)

This project demonstrates how to set up a **High Availability Kubernetes Cluster** using:

- kubeadm
- 3 Control Plane Nodes
- HAProxy Load Balancer
- Worker Nodes
- AWS EC2 (Ubuntu)

---

## Architecture

    HAProxy (Load Balancer)
            |
---------------------------
|           |            |
M1          M2           M3


---

##  Prerequisites

- AWS EC2 instances (Ubuntu 22.04)
- Minimum 5 instances:
  - 1 Load Balancer
  - 3 Masters
  - 1 Worker
- Security Groups allowing:
  - 6443 (K8s API)
  - 2379-2380 (etcd)
  - 10250
  - 30000-32767
  - Internal communication

---

##  Common Setup (All Nodes)

- Disable swap
- Enable kernel modules
- Configure sysctl

---

## Install Container Runtime

- containerd installation
- Enable systemd cgroup

---

## Install Kubernetes

- kubeadm
- kubelet
- kubectl
- -----------------------------------------------------------

---STEP 1- Main Execution — Common Setup (Run on ALL nodes)
-------------------------------------------------------------

sudo apt update && sudo apt upgrade -y

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

--------------------------------------
STEP 2 — Install Container Runtime (containerd)
-----------------------------------------

sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable systemd cgroup
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

------------------------------------------
STEP 3 — Install Kubernetes Components
------------------------------------------
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo tee /etc/apt/keyrings/kubernetes-apt-keyring.asc

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.asc] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update

sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
--------------------------------------------------
STEP 4 — Setup HAProxy (on LB server)          ------------------------------- RUNS ON THE HAPROXY EC2 ONLY ------------------------------
----------------------------------------
sudo apt install -y haproxy

---Edit config:--------

sudo nano /etc/haproxy/haproxy.cfg

-------Add at bottom:----------

frontend kubernetes
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    option tcp-check
    server m1 <MASTER1_PRIVATE_IP>:6443 check
    server m2 <MASTER2_PRIVATE_IP>:6443 check
    server m3 <MASTER3_PRIVATE_IP>:6443 check

    sudo systemctl restart haproxy  ---------------AFTER EDIT CONFIG PLEASE RESTART THE HAPROXY SERVICE

    -----------------------------------------------------------
STEP 5 — Initialize First Master
------------------------------------------
sudo kubeadm init \
--control-plane-endpoint "<LB_PRIVATE_IP>:6443" \
--upload-certs \
--pod-network-cidr=192.168.0.0/16

----------------------------------------------
STEP 6 — Setup kubeconfig
----------------------------------------------
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

------------------------------------------
STEP 7 — Install CNI (Calico)
--------------------------------------------
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

----------------
STEP 8 — Join Other Masters
--------------------------

kubeadm join <LB_IP>:6443 --token xxxx \
--discovery-token-ca-cert-hash sha256:xxxx \
--control-plane --certificate-key xxxx

---Run that on:-------

Master2
Master3

-----------------------
STEP 9 — Join Worker Nodes
-------------------------
kubeadm join <LB_IP>:6443 --token xxxx \
--discovery-token-ca-cert-hash sha256:xxxx

-------------------------------
STEP 10 — Verify Cluster
---------------------------
kubectl get nodes
    
