# Set up a Highly Available Kubernetes Cluster using kubeadm
## LOAD BALANCER

### 1 Install haproxy
```
apt update && apt install -y haproxy
```

### 2 haproxy conf
```
frontend kubernetes-frontend
    bind 192.168.56.31:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server kmaster1 192.168.56.11:6443 check fall 3 rise 2
    server kmaster2 192.168.56.12:6443 check fall 3 rise 2
```

## 3 Restart haproxy    
```
systemctl restart haproxy
```

## General Install

### 1 Disable and turn off SWAP
```
sed -i '/swap/d' /etc/fstab
swapoff -a
```

### 2 Stop and Disable firewall
```
systemctl disable --now ufw >/dev/null 2>&1
```

### 3 Enable and Load Kernel modules
```
cat >>/etc/modules-load.d/containerd.conf<<EOF
overlay
br_netfilter
EOF
```
```
modprobe overlay
```
```
modprobe br_netfilter
```

### 4 Add Kernel settings
```
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF
```
```
sysctl --system
```

### 5 Install containerd runtime
```
apt update
```
```
apt install -qq -y ca-certificates curl gnupg lsb-release
```
```
mkdir -p /etc/apt/keyrings
```
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
```
```
apt update
```
```
apt install -y containerd.io
```
```
containerd config default > /etc/containerd/config.toml
```
```
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
```
systemctl restart containerd
```
```
systemctl enable containerd
```

### 6 Add apt repo for kubernetes
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
```

### 7 Install Kubernetes components (kubeadm, kubelet and kubectl)
```
apt install -y kubeadm=1.26.0-00 kubelet=1.26.0-00 kubectl=1.26.0-00
```


## Kube Master

### 1 Pull required containers
```
kubeadm config images pull
```

### 2 Initialize Kubernetes Cluster
```
kubeadm init --control-plane-endpoint="192.168.56.31:6443" --upload-certs --apiserver-advertise-address=192.168.56.11 --pod-network-cidr=192.168.0.0/16
```

### 3 Deploy Calico network
```
kubectl --kubeconfig=/etc/kubernetes/admin.conf create -f https://docs.projectcalico.org/v3.18/manifests/calico.yaml
```
