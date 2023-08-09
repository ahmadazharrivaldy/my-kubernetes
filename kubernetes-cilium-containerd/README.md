# Setup Kubernetes Cluster using Cilium & Containerd

## Environmet
| Role | Host | IP | OS | RAM | CPU |
| ------ | ------ | ------ | ------ | ------ | ------ |
| Master | rz-master | 10.0.10.10 | Ubuntu 20.04 | 4GB | 4 |
| Worker | rz-worker-1 | 10.0.10.11 | Ubuntu 20.04 | 8GB | 4 |
| Worker | rz-worker-2 | 10.0.10.12 | Ubuntu 20.04 | 8GB | 4 |

## On all host
Disable firewall
```
ufw disable
``` 
Disable swap memory
```
swapoff -a; sed -i '/swap/d' /etc/fstab
```
Enable & load kernel modules
```
{
cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
}
```
Add kernel settings
```
{
cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
}
```
Add kubernetes & containerd repository
```
{
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
}
```
Install kubernetes, containerd, helm-3 packages
```
{
sudo apt update && sudo apt install -y kubeadm kubelet kubectl containerd.io
sudo apt-mark hold kubelet kubeadm kubectl containerd.io
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | sudo bash
}
```
Configure containerd
```
{
containerd config default | sudo tee /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
cat /etc/containerd/config.toml | grep SystemdCgroup
}
```
```
{
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 30
debug: false
EOF
sudo systemctl restart containerd 
}
```
Add bash auto completion
```
{
cat <<EOF | tee -a ~/.profile
source <(kubectl completion bash)
source <(kubeadm completion bash)
source <(crictl completion bash)
source <(helm completion bash)

## kubectl alias ##
alias k=kubectl
complete -F __start_kubectl k

## change kube editor using nano ##
export KUBE_EDITOR=nano
export NOW="--force --grace-period 0"
EOF
source ~/.profile 
}
```

## On master node
Initialize kubernetes cluster
```
YOUR_IP_ADDRESS=10.0.10.10
```
```
kubeadm init \
--control-plane-endpoint=10.0.10.10:6443 \
--upload-certs \
--apiserver-advertise-address=10.0.10.10 \
--pod-network-cidr=192.168.0.0/16 \
--service-cidr=10.96.0.0/12
```
Create kubeconfig so you can operate kubectl command
```
{
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
}
```

Deploy Cilium Container Network Interface
```
{
helm repo add cilium https://helm.cilium.io/
helm install -n kube-system cilium cilium/cilium -f ~/cilium-values.yaml
}
```
Generate join command for worker nodes
```
kubeadm token create --print-join-command
```

## On worker nodes
Join worker nodes to kubernetes cluster
```
kubeadm join 10.0.10.10:6443 --token 56zizr.8469l7qv3nw68b8d --discovery-token-ca-cert-hash sha256:0ca7604168ec223824b8538978d65c58adfe38cead3507f250d544556a8bbb4f
```

## Verify
For check kubernetes component statuses are healthy
```
kubectl get componentstatuses
```
For check kubernetes nodes are ready
```
kubectl get nodes
```
For check kubernetes pods are running
```
kubectl get pods -A
```
