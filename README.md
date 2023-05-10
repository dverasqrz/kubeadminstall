# Installation of kubeadm
The goal is to create your cluster on-premise using kubeadm.

Kubeadm is a tool aimed at making it easier to create a standard Kubernetes cluster that meets all the requirements of a certified cluster. In other words, you will have the basics of a Kubernetes cluster validated by the Cloud Native Computing Foundation. But you will also use kubeadm for some cluster maintenance processes, such as certificate renewal and cluster updates.

The first step is to install the container runtime on ALL machines, which will execute the containers requested by kubelet. Here, the container runtime used is Containerd, but you can also use Docker and CRI-O. Before installing Containerd, some kernel modules need to be enabled and sysctl parameters configured.

## Kernel module installation:
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

## Sysctl parameter configuration:
```
# Sysctl parameter configuration, persists even after machine reboot.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl settings without rebooting the machine.
sysctl --system
```
Now we can install and configure Containerd.

## Installation:
```
apt update && sudo apt install -y containerd
```
## Default configuration of Containerd:
```
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
```
Now the container needs to be restarted:
```
systemctl restart containerd
```
# Installation of kubeadm, kubelet, and kubectl

Now that the container runtime is installed on all machines, it's time to install kubeadm, kubelet, and kubectl. So let's follow the steps and execute these steps on ALL MACHINES.

## Update the necessary packages for installation:
```
apt-get update && apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```
## Download the Google Cloud public key and install the tools:
```
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && apt-key fingerprint 0EBFCD88 && add-apt-repository "deb [arch=amd64] https://apt.kubernetes.io/ kubernetes-xenial main" && apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
```
## Now make sure they are not automatically updated:
```
apt-mark hold kubelet kubeadm kubectl
```
# Starting the Kubernetes cluster

Now that all the elements are installed, it's time to start the Kubernetes cluster, so we'll run the cluster initialization command. This command should only be run on the machine that will be the control plane.

## Initialization command:
```
kubeadm init --apiserver-cert-extra-sans XXX --apiserver-advertise-address XXX
```
**--apiserver-cert-extra-sans** ⇒ Includes the IP or domain as valid access in the kube-api certificate. If you have more than one network adapter in the cluster (one internal and one external, for example), it's important that you use it.

**--apiserver-advertise-address** ⇒ Sets the network adapter that will be responsible for communicating with the cluster.

After starting the cluster, the access configuration of the cluster needs to be copied to kubectl.

## Configuring kubectl:
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
Now the next step is to include the worker nodes in the cluster. To do this, the kubeadm join command already appears in the cluster initialization output to execute on the worker nodes, but if you lose it or need it again, just run the token create command on the control plane.

## Generating the join command and executing it on the nodes:
```
kubeadm token create --print-join-command
```
Now, when you run `kubectl get nodes`, you will see that the control plane and nodes are not ready. To solve this, it is necessary to install the Container Network Interface or CNI, and here we will use Calico.

## Installation of CNI:
```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```
