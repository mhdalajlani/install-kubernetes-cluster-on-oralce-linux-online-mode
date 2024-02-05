
# Install Kubernetes Cluster on Oralce Linux


Within this file, you will find the necessary steps to install Kubernetes on three different servers, one of which will be the Master Node and the rest will be the Worker Node. This assumes that the servers are connected to the Internet.

## Set hostname

Login to master node and set hostname using hostnamectl command.

```bash
hostnamectl set-hostname "k8smaster"
```
On the worker nodes, run the following commands.

```bash
hostnamectl set-hostname "k8sworker1"
hostnamectl set-hostname "k8sworker2"
```
## Update /etc/hosts

If you do not have a DNS server to resolve the hostname then you must update your /etc/hosts file with the hostname and IP information of all the cluster nodes on all the nodes.

## Disable Swap

It is mandatory to disable swap memory for kubelet to work properly. Follow these steps on all the cluster nodes.

```bash
swapoff -a
```

Once done, we also need to make sure swap is not re-assigned after node reboot so comment out the swap filesystem entry from /etc/fstab.

```bash
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## Disable SELinux
You must disable selinux or change it to Permissive mode on all the cluster nodes. This is required to allow containers to access the host filesystem, which is needed by pod networks. You have to do this until SELinux support is improved in the kubelet.

```bash
setenforce 0
```

To make the changes persistent across reboot execute the following command:
```bash
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

## Configure Networking
Load the following kernel modules on all the nodes.

```bash
tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
Make sure that the br_netfilter and overlay modules are loaded. This can be done by running these commands.

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

Since br_netfilter and overlay are not in loaded state, I will load this module manually.
```bash
modprobe br_netfilter
modprobe overlay
```

As a requirement for your Linux Node's iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config.

```bash
sysctl -a | grep net.bridge.bridge-nf-call-iptables
```

It is enabled by-default but to be on the safe side we will also create a sysctl configuration file and add this for both IPv4 and IPv6.

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
Activate the newly added changes.

```bash
sysctl --system
```
## Install container runtime (containerd)

You need to install a container runtime into each node in the cluster so that Pods can run there. We can use Docker, containerd or CRI-O as our runtime on all the nodes. First, we need to install some dependent packages.

```bash
dnf install -y yum-utils device-mapper-persistent-data lvm2
```

Add the docker repository to be able to install the needed packages on all the nodes.

```bash
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

Install containerd package on all the nodes.

```bash
dnf install containerd.io
```
Configure containerd so that it starts using systemd as cgroup.

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
After that the last step its configure both runtime-endpoint and image-endpoint using the below command.
```bash
crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock --set image-endpoint=unix:///run/containerd/containerd.sock
```

We are all done with the configuration, time to start (restart) our containerd daemon on all the nodes.

```bash
systemctl restart containerd
systemctl enable containerd
```
## Install Kubernetes components (kubelet, kubectl and kubeadm)
You will install these packages Kubeadm, Kubelet and Kubectl on all of your machines. 

Create the kubernetes repository file on all the nodes which will be used to download the packages.

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
Install the Kubernetes component packages using package manager on all the nodes.
```bash
dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
## Initialize control node

The control-plane node is the machine where the control plane components run, including etcd (the cluster database) and the API Server (which the kubectl command line tool communicates with).

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.26.0 --control-plane-endpoint HOSTNAME:6443
```
The initialization has completed successfully. If you notice the highlighted output from previous command, there are certain steps which you must perform if you have executed the above command as regular user.

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
But since we are using root user, we have to execute following commands.

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```
and also add this to your /etc/profile.
```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
```

## Install Pod network add-on plugin

You must deploy a Container Network Interface (CNI) based Pod network add-on so that your Pods can communicate with each other. Cluster DNS (CoreDNS) will not start up before a network is installed.
Common Pod networking add-on plugins: Flannel. Weave. Calico. 

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
You can install only one Pod network per cluster.
Once a Pod network has been installed, you can confirm that it is working by checking that the CoreDNS Pod is Running in the output of kubectl get pods --all-namespaces. We can also use the same command with -o wide which will give much more details about each namespace. Check the status again in few minutes and all the namespaces should be in Running state.
Next check the status of cluster, if you recall this was in NotReady state before installing the pod networking plugin but now the cluster is in Ready state.
