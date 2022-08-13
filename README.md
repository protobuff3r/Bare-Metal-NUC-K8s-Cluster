# Install Kubernetes Cluster Version 1.24

## Current Setup

3x Intel NUC 11 Pro Kit NUC11TNHi5 i5-1135G7 64GB Memory SSD 1TB NVME (1x Master & 2 Worker)   
TP-Link TL-SG108 LAN Switch 8 Port Netzwerk Gigabit Switch   
Ubuntu 22.04   
Kubernetes v1.24   

## 1. Bootstrap Nodes:

### Run as root
```bash
sudo -i
apt update
apt upgrade -y
apt install -y vim locales
locale-gen en_US.UTF-8
```

### Add public key for ssh connect
```bash
vi .ssh/authorized_keys
```

### Set static IP for node
```bash
vi /etc/netplan/00-installer-config.yaml
```

Content:
```bash
dhcp4: false
addresses: [192.168.50.101/24]
gateway4: 192.168.50.1
nameservers:
  addresses: [1.1.1.1,8.8.8.8,192.168.50.1]
```

## 2. Install Kubeadm Kubelet Kubectl
```bash
apt install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

### Set requirements for Kubernetes

Disable Swap (comment out swap line and set temp swap off)
```bash
vi /etc/fstab (comment out swap line)
swapoff -a
```

```bash
modprobe overlay
modprobe br_netfilter

tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

## 3. Install CRI-O
```bash
OS=xUbuntu_22.04
VERSION=1.24

echo "deb [signed-by=/usr/share/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb [signed-by=/usr/share/keyrings/libcontainers-crio-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list

mkdir -p /usr/share/keyrings
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-archive-keyring.gpg
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/Release.key | gpg --dearmor -o /usr/share/keyrings/libcontainers-crio-archive-keyring.gpg

apt update
apt install -y cri-o cri-o-runc cri-tools
```

### Check CRI-O:

```bash
apt-cache policy cri-o
```

Update CRI-O Config for Kubernetes: https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o
```bash
vi /etc/crio/crio.conf

# need systemd is default on Kubelet
[crio.runtime]
conmon_cgroup = "pod"
cgroup_manager = "systemd"

systemctl daemon-reload
systemctl enable crio --now
crictl info
systemctl status crio
```

Update CRI-O:
https://github.com/cri-o/cri-o/blob/main/install.md#apt-based-operating-systems-1


## 4. Enable Kubeadm

```bash
lsmod | grep br_netfilter
systemctl enable kubelet
kubeadm config images pull

vi /etc/hosts
192.168.50.101 k8s.api
192.168.50.102 k8s.worker1
192.168.50.103 k8s.worker2
```

### Only Master Node - Kubeadm init without Kubeproxy and CRI-O Container Engine

```bash
kubeadm init \
  --pod-network-cidr=10.10.0.0/16 \
  --cri-socket /var/run/crio/crio.sock \
  --skip-phases=addon/kube-proxy \
  --upload-certs \
  --control-plane-endpoint=k8s.api
```

Bugfix for CGGroups: https://github.com/kubernetes/system-validators/pull/32 (ignore warning until released)


### Configure Kubectl

To start using your cluster, you need to run the following as a regular user:
```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Schedule Pods on Master Node

Remove the Taint using this command:

```bash
kubectl taint node k8smaster node-role.kubernetes.io/master:NoSchedule-
```

## 5. Install Cilium CNI without Kubeproxy

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install --helm-set=kubeProxyReplacement=strict,k8sServiceHost=192.168.50.101,k8sServicePort=6443
cilium status --wait
```

## 6. Join Worker Nodes

```bash
kubeadm join k8s.api:6443 --token xxxxx.XXXXXXXXXX \
        --discovery-token-ca-cert-hash sha256:XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

## 7. Test Cilium Connectivity

### Restart CRI-O on every Node
https://github.com/cri-o/cri-o/issues/4276

```bash
systemctl restart crio
cilium connectivity test
```

## 8. Install Longhorn (Storage System for K8s)

On Ubuntu 22.04 - OpenISCSI is installed (https://longhorn.io/docs/1.3.1/deploy/install/#installing-open-iscsi)

Install NFS-Common on every Node !!!

```bash
sudo apt install -y jq nfs-common
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.3.1/scripts/environment_check.sh | bash
```

### Release Installation over Argo-CD !!!   

## 9. Install Argo-CD (Declarative GitOps CD for Kubernetes)

https://argo-cd.readthedocs.io/en/stable/getting_started/

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```

### Change the argocd-server service type to LoadBalancer:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### For Ingress
see https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/   

### Admin Password

Username: admin

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Open Argo-CD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
The API server can then be accessed using https://localhost:8080   





helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace

helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --namespace kube-system --install metrics-server metrics-server/metrics-server \
    --set args="{--kubelet-insecure-tls,--kubelet-use-node-status-port}"



# Troubleshooting

## Reset Kubeadm Init or joined Worker Node (Drain Node before)
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-local-data
kubeadm reset cleanup-node
kubectl delete node <node-name>
```

## Kubelet Logs
```bash
journalctl -xeu kubelet
```

# Links
  
https://github.com/cri-o/cri-o/blob/main/install.md#apt-based-operating-systems   
https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-driver   

