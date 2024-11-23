# Installation and configuration

This section covers the installation and configuration steps to initialize a kubernetes cluster using kubeadm

Following steps are necessary to initialize a cluster:

1. Initialize a control plane node
2. Install a networking plugin (cilium) 
3. Join a worker node

Official documentation: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/>

Ubuntu22.04 is used as OS for both, cp and worker nodes

## Prepare the node and install dependencies

Official documentation: <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration>

* swap configuration
* installing a container runtime
* install kubeadm, kubelet and kubectl

### Configure system settings

Following tasks are done as root user

Prepare by installing necessary packages

```bash
apt install curl apt-transport-https git wget software-properties-common lsb-release ca-certificates socat -y
```

Disable swap (disabled by default by cloud provicders)

```bash
swapoff -a
```

Load modules necessary for next steps

```bash
modprobe overlay
modprobe br_netfilter
```

Update kernel networking to allow necessary traffic

```bash
cat << EOF | tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# ensure the changes are by the current kernel
sysctl --system
```

### Install a container runtime (containerd)

Install the neccessary key and add docker repository

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install and configure containerd

```bash
apt-get update && apt-get install containerd.io -y
containerd config default | tee /etc/containerd/config.toml
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
systemctl restart containerd
```

### Install kubeadm, kubelet and kubectl (v1.30)

Install necessary key for kuernetes repositories

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add kubernetes repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install the packages

```bash
apt-get update
apt-get install -y kubeadm=1.30.1-1.1 kubelet=1.30.1-1.1 kubectl=1.30.1-1.1
```

Disable automatic ugrading

```bash
apt-mark hold kubelet kubeadm kubectl
```

## Create the cluster using kubeadm

edit the /etc/hosts file to include an entry for the control planes IP address resolving to k8scp:

```txt
<ip addr> k8scp
127.0.0.1 localhost
```

Create a kubeadm ClusterConfiguration <https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/#kubeadm-k8s-io-v1beta4-ClusterConfiguration>

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.30.1
controlPlaneEndpoint: "k8scp:6443"
networking:
  podSubnet: "10.244.0.0/24"
```

Initialize the cluster

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs --node-name=cp | tee kubeadm-init.out
```

The kubeconfig is saved in `/etc/kubernetes/admin.conf`

## Install a network plugin

In this example i am using cilium that can be installed via helm

Cilium documentation: <https://docs.cilium.io/en/stable/installation/k8s-install-helm/>

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm repo search cilium
helm template cilium cilium/cilium --version 1.16.4 -n kube-system > cilium.yaml
kubectl apply -f cilium.yaml
```

## Join a worker node

### Prepare the worker node

Do the same configuratins as on cp node:

* Configure system settings
* Install container runtime (containerd)
* Install kubelet, kubeadm and kubectl

### Create a token on the master node

Create the join command:

```bash
sudo kubeadm token create --print-join-command
```

### Join a worker node

On the worker node, create an entry in /etc/hosts file for k8scp pointing to master node's private ip address

```txt
<ip addr> k8scp
127.0.0.1 localhost
```

Execute the kubeadm join command on the worker node:

```bash
kubeadm join k8scp:6443 --token f6yiki.t04n1lkgl77zq6rp \ 
--discovery-token-ca-cert-hash sha256:d61d81345994583a72c5b180ea62737810c419e174ced35a7796eb281771f2c6 --node-name=worker
```

If we would create a new control-plane node we would pass the `--control-plane` parameter

## Finish cluster setup 

* Allow cp node to run non-infrastructure pods

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

* Inspect interfaces on nodes using `ip a`
* Update the crictl configuraton as containerd may be using an out of date notation for the runtime-endpoint

On master and worker nodes run:

```bash
sudo crictl config --set \
runtime-endpoint=unix:///run/containerd/containerd.sock \
--set image-endpoint=unix:///run/containerd/containerd.sock
```

Verify

```bash
sudo cat /etc/crictl.yaml
```
