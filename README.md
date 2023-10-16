# Step 1: Pre-requisites

1.a.. Check the OS, Hardware Configurations & Network connectivity
1.b.. Turn off the swap & firewall
```bash
sudo swapoff -a
sudo systemctl stop firewalld
sudo systemctl disable firewalld

```

# Step 2. Configure the local IP tables to see the Bridged Traffic

2.a.. Enable the bridged traffic
```bash
lsmod | grep br_netfilter
sudo modprobe br_netfilter
lsmod | grep br_netfilter
```

2.b.. Copy the below contents in this file.. /etc/modules-load.d/k8s.conf
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```

2.c.. Copy the below contents in this file.. /etc/sysctl.d/k8s.conf
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```
```bash
sudo sysctl --system
```

# Step 3. Install Docker as a Container RUNTIME

3.a.. Uninstall any Older versions
```bash
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

3.b.. Install Yum Utilities | Config Manager
```bash
sudo yum install -y yum-utils
```

3.c.. Setup the Docker Repository
```bash
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3.d.. Install Docker Engine, Docker CLI, Docker RUNTIME
```bash
sudo yum install -y docker-ce docker-ce-cli containerd.io
```

# Step 4. Configure Docker Daemon for cgroups management & Start Docker

4.a.. Create directory 
```bash
sudo mkdir /etc/docker
```

4.b.. Copy the below contents in this file.. /etc/docker/daemon.json
```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```
```bash
sudo systemctl status docker
```
# Step 5. Install kubeadm, kubectl, kubelet

5.a.. Copy the below contents in this file.. /etc/yum.repos.d/kubernetes.repo
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```
5.b.. Set SELinux in permissive mode (effectively disabling it)
```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
```bash
sudo systemctl enable --now kubelet
```

# Step 6. Configuring a cgroup driver

Ignore if docker is used as a CRI

# Step 7. Deploy a  kubernetes cluster using kubeadm

# Run only in Master node
```bash
kubeadm init --pod-network-cidr=10.10.0.0/16 --apiserver-advertise-address=master_nodeIP
```
# fix [ERROR CRI]: container runtime is not running:
https://stackoverflow.com/questions/72504257/i-encountered-when-executing-kubeadm-init-error-issue


#To start using your cluster, you need to run the following as a regular user:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.10.100:6443 --token lvlg85.ogciz6go2ru6iibw \
        --discovery-token-ca-cert-hash sha256:f55bcbb5524abdbbb476ed60e71733d1ca668f824abcca3ba490cca7379cc7cb

# Step 8. Install CNI for POD Networking

# Run only in Master node

Weave Networks
---
Weave Networks:
( 1 trong 2 )
```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s-1.11.yaml
```
# Step 9. Join the worker nodes to the master

# Run in Worker Nodes as "Root"
```bash
kubeadm join 172.16.10.100:6443 --token lvlg85.ogciz6go2ru6iibw \
        --discovery-token-ca-cert-hash sha256:f55bcbb5524abdbbb476ed60e71733d1ca668f824abcca3ba490cca7379cc7cb
```
# Make sure to replace your tokens and IP's in the above command accordingly