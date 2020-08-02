+++
author = "mahartma"
categories = ["kubernetes", "k8s", "ubuntu", "devops"]
tags = ["tutorial"]
date = "2020-07-31"
description = "Learn how to setup a single master kubernetes cluster on ubuntu with kubeadm"
featured = "k8s-setup-ubuntu-kubeadm.jpg"
featuredalt = "k8s-setup-ubuntu-kubeadm"
featuredpath = "date"
linktitle = ""
title = "Setup a single master kubernetes cluster on ubuntu with kubeadm"
type = "post"

+++

# Setting up a single master Kubernetes cluster on a Ubuntu server

In this Tutorial create a single control-plane cluster with kubeadm. 
Therefore we will use 3 VMs at Hetzner Cloud.
This may suit for a development environment but is not HA because we only have one master. If you do not expect much traffic and have a DDoS protection, you may consider this way for production, too.

For this tutorial I used 3 Ubuntu 18.04 virtual machines hosted on hetzner.com.

Before setting up the kubernetes cluster I additionally put them in a private network at hetzner that we may introduce a firewall/DMZ one day.

# Bootstrapping clusters with kubeadm

## Before you begin

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

These are the official requirements:

```
    One or more machines running one of:
        Ubuntu 16.04+
        Debian 9+
        CentOS 7
        Red Hat Enterprise Linux (RHEL) 7
        Fedora 25+
        HypriotOS v1.0.1+
        Container Linux (tested with 1800.6.0)
    2 GB or more of RAM per machine (any less will leave little room for your apps)
    2 CPUs or more
    Full network connectivity between all machines in the cluster (public or private network is fine)
    Unique hostname, MAC address, and product_uuid for every node. See here for more details.
    Certain ports are open on your machines. See here for more details.
    Swap disabled. You MUST disable swap in order for the kubelet to work properly.
```

## Verify that MAC address and product_uuid are unique for every node

You can get the MAC address of the network interfaces using the command `ip link` or `ifconfig -a`
The product_uuid can be checked by using the command `sudo cat /sys/class/dmi/id/product_uuid`

It is very likely that hardware devices will have unique addresses, although some virtual machines may have identical values. Kubernetes uses these values to uniquely identify the nodes in the cluster. If these values are not unique to each node, the installation process may fail

To check if any machines share MAC-addresses, you should check and compare all the machines with the `ifconfig -a` command.

# Turn off swap

For the kubelet to work properly, we need to turn off the swap, using follwing command:
```
sudo swapoff -a
```
# Check network adapters

If you have more than one network adapter, and your Kubernetes components are not reachable on the default route, we recommend you add IP route(s) so Kubernetes cluster addresses go via the appropriate adapter.

In my case this should usually work, because I put them in a private network on Hetzner which they will communicate with over.

# Letting iptables see bridged traffic

As a requirement for your Linux Node’s iptables to correctly see bridged traffic, you should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.

```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

Make sure that the br_netfilter module is loaded before this step. This can be done by running `lsmod | grep br_netfilter`. To load it explicitly call `modprobe br_netfilter`.

For more details please see the Network Plugin Requirements page.

## br_netfilter
Do following steps to check if br_netfilter is enabled and do so if not.

Check if loaded:
```
lsmod | grep br_netfilter
```

if not:
```
sudo modprobe br_netfilter
```

re-check:
```
lsmod | grep br_netfilter
```

## sysctl config
### Add a config file for k8s
Use vim or the editor of your choice to create a k8s.conf.
```
vim /etc/sysctl.d/k8s.conf
```
### and add following lines
```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

### read values from all system directories and apply
```
sudo sysctl --system
```

# Check required ports

We need to allow some ports for kubernetes to work.

The pod network plugin you use (see below) may also require certain ports to be open. Since this differs with each pod network plugin, please see the documentation for the plugins about what port(s) those need.

I've used the official kubernetes documentation as a source: [kubernetes docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports).

Therefore we use Ubuntus ufw.

If you want to use your cluster in a private network, use the first option. Otherwise, use the second.


## Control-plane (master) nodes in private network on hetzner
Precondition: as I already mentioned the kubernetes cluster is in a private network and the servers will communicate via this network. Therefore I only expose the Kubernetes API server to the internet and the other ports only to request coming from the private network.

In this example the interface for the private ip address is ens10. (You can look it up with the `ifconfig -a` command)

Kubernetes API server:
```
sudo ufw allow in on ens10 to any port 6443 proto tcp 
```
in addition, if you want to expose API server to the internet use
```
sudo ufw allow 6443/tcp
```
etcd server client API
```
sudo ufw allow in on ens10 to any port 2379:2380 proto tcp
```

Kubelet API
```
sudo ufw allow in on ens10 to any port 10250 proto tcp
```

kube-scheduler
```
sudo ufw allow in on ens10 to any port 10251 proto tcp
```

kube-controller-manager
```
sudo ufw allow in on ens10 to any port 10252 proto tcp
```
## Worker nodes in private network on hetzner

DNS
```
sudo ufw allow 53/tcp
sudo ufw allow 53/udp

sudo ufw allow in on ens10 to any port 53 proto tcp
sudo ufw allow in on ens10 to any port 53 proto udp
```

Kubelet API
```
sudo ufw allow in on ens10 to any port 10250 proto tcp
```

NodePort Services
```
sudo ufw allow in on ens10 to any port 30000:32767 proto tcp
```
## Control-plane (master) nodes in public network

Kubernetes API server:
```
sudo ufw allow in to any port 6443 proto tcp 
```

etcd server client API
```
sudo ufw allow in to any port 2379:2380 proto tcp
```

Kubelet API
```
sudo ufw allow in to any port 10250 proto tcp
```

kube-scheduler
```
sudo ufw allow in to any port 10251 proto tcp
```

kube-controller-manager
```
sudo ufw allow in to any port 10252 proto tcp
```

## Worker nodes in public network

Kubelet API
```
sudo ufw allow in to any port 10250 proto tcp
```

NodePort Services
```
sudo ufw allow in to any port 30000:32767 proto tcp
```

# Installing Pre-Requisites

# Install Docker CE

Official k8s docs: [k8s docs](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

See docker for more detailled information: [docker docs](https://docs.docker.com/install/)

## install prerequisites
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg2
```
## Add Docker’s official GPG key
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

### Add Docker apt repository.
```
sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```
or
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

## Install Docker CE.
```
sudo apt-get update
sudo apt-get install -y \
  containerd.io=1.2.13-1 \
  docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) \
  docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)
```
or
```
sudo apt-get install -y containerd.io=1.2.13-1 docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)
```
## Setup daemon (as root)
```
cat > /etc/docker/daemon.json <<EOF
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
```
sudo mkdir -p /etc/systemd/system/docker.service.d
```

## Restart docker.
```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# Installing kubeadm, kubelet and kubectl

You will install these packages on all of your machines:

- kubeadm: the command to bootstrap the cluster.
- kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
- kubectl: the command line util to talk to your cluster.

kubeadm will not install or manage kubelet or kubectl for you, so you will need to ensure they match the version of the Kubernetes control plane you want kubeadm to install for you. If you do not, there is a risk of a version skew occurring that can lead to unexpected, buggy behaviour. However, one minor version skew between the kubelet and the control plane is supported, but the kubelet version may never exceed the API server version. For example, kubelets running 1.7.0 should be fully compatible with a 1.8.0 API server, but not vice versa.

For information about installing kubectl, see Install and set up kubectl. [k8s docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

    Warning: These instructions exclude all Kubernetes packages from any system upgrades. This is because kubeadm and Kubernetes require special attention to upgrade.

For more information on version skews, see:

    Kubernetes version and version-skew policy
    Kubeadm-specific version skew policy


> This means that we will need to upgrade kubernetes manually in future

## Installation

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

> The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.


# Configure cgroup driver used by kubelet on control-plane node

When using Docker, kubeadm will automatically detect the cgroup driver for the kubelet and set it in the `/var/lib/kubelet/kubeadm-flags.env` file during runtime.


# Creating a single control-plane cluster with kubeadm

Source: [k8s docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## initialize the control-plane node (master node)

To initialize the control-plane node run: (we use calico for networking and therefore add the pod-network-cidr parameter / we use the private network 10.0.0.0 for, otherwise the public IP of the server)
```
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=10.0.0.3
```

## Installing the kubectl 
To make kubectl work for your non-root user, run these commands, which are also part of the kubeadm init output:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

oras root:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
We now have access to the kubernetes master and the API via the kubectl.

# Installing a pod network addon
For our network to work, we need to install a network addon.

# Calico
I chose to use calico (this is why i used --pod-network-cidr=192.168.0.0/16 at kubeadm init)

## Prerequisites
Source: https://docs.projectcalico.org/getting-started/kubernetes/requirements

### Allow traffic for Calico networking (BGP) in IPTables
Do this on all masters and nodes:
```
sudo ufw allow in on ens10 to any port 179 proto tcp
```

## Apply calico yaml on master(s)
After all the firewall rules were implemented, apply the official yaml file on your master nodes.

```
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

# Join nodes

Make a record of the kubeadm join command that kubeadm init outputs. You need this command to join nodes to your cluster.

The token is used for mutual authentication between the control-plane node and the joining nodes. The token included here is secret. Keep it safe, because anyone with this token can add authenticated nodes to your cluster. These tokens can be listed, created, and deleted with the kubeadm token command. See the kubeadm reference guide.

## Credit
Image:
https://unsplash.com/photos/M5tzZtFCOfs 
Kubernetes Logo:
https://en.wikipedia.org/wiki/Kubernetes#/media/File:Kubernetes_logo_without_workmark.svg 


