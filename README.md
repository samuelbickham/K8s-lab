
# K8S + Illumio Home Lab Guide

```
Author: Samuel Bickham
Version 2023.01

```

## Overview

This document is a walkthrough for setting up Kubernetes (K8s) with Docker container runtime and connecting it to an Illumio Core PCE. There are 3 main components:
1. Build a K8s cluster (the guide your reading)
2. Build an Illumio Core PCE 
3. Connect the K8s cluster to Illumio Core (Click <a href="#">here</a> for this guide)

This guide walks through the first step, building a K8s cluster. Follow <a href="https://github.com/johnwesterman/illumio_core/blob/cliffnotes_2022.08/README.md">this guide</a> by John Westerman to build your PCE. Follow <a href="#">this guide</a> to connect your K8s cluster to the Illumio Core PCE. Let's get to it. 


## Summary of Tasks:

1. Download Ubuntu 20.04  <a href="https://www.linuxvmimages.com/images/ubuntuserver-2204/">(link)</a> and install it on three virtual machines in hypervisor of your choice
2. Install K8s and Docker
3. Rollback kubelet 1.26 to 1.25 to mitigate bug and "hold" packages
4. Disable swap memory 
5. Set node hostnames
6. Enable traffic bridging
7. Change Docker CGroup driver
8. Initialize K8s cluster on Master Node
9. Export K8s control config as root and normal user
10. Disable firewall and deploy CNI (pod networking)
11. Join worker nodes to cluster
12. Deploy K8s application and make it accessible


## Step 1: Download Ubuntu 20.04 (link) and install it on three virtual machines in hypervisor of your choice

You can download it <a href="https://www.linuxvmimages.com/images/ubuntuserver-2204/">here.</a> Accept all defaults during install, do not run updates and install the OpenSSH server. 


## Step 2: Install K8s and Docker (ALL NODES)

I recommend running the entire setup as root (sudo -i) and then switching back to your normal user account once everyting is setup. 

Install http/https transport and curl
```

sudo apt-get update && sudo apt-get install -y apt-transport-https curl

```

Add K8s signing key
```

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

```

Get Updates
```

sudo apt update

```

Install K8s, tools and Docker
```
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni

sudo apt install docker.io
```

## Step 3: Rollback kubelet 1.26 to 1.25 to mitigate bug and (optionally) "hold" packages (ALL NODES)

There is a bug in Kubelet 1.26 that causes problems when you try to initialize the cluster. There are workarounds on the internet, however since this is a lab and the goal is to quickly get things up and running, I recommend just rolling back to Kubelet 1.25. You can optionally put a "hold" on the k8s packages so you can update the system without breaking your cluster. 

```
apt remove --purge kubelet

apt install -y kubeadm kubelet=1.25.5-00

sudo apt-mark hold kubelet kubeadm kubectl

```

To remove the "hold" on the packages:
```

apt-mark unhold $(apt-mark showhold)

```

## Step 4: Disable swap memory  (ALL NODES)

Current versions of K8s are NOT designed to run with swap memory enabled on the system. Tecnically, you can initialize the cluster with switches that will ignore proboems (e.g. sudo kubeadm init --ignore-preflight-errors=NumCPU,Mem,Swap...) but this is generally not advised as is can create unpredictable behaviors. 

Disable swap memory now:

```

sudo swapoff -a

```
Disable swap memory on reboot - comment out "/swapfile" with a hash(#) - if its missing you can move on.

```

sudo vi /etc/fstab

```

## Step 5: Set node hostnames (ALL NODES)

In my lab, I have 1 masternode (K8s control plane) and 2 worker nodes (K8s data plane). 

Master:

```

sudo hostnamectl set-hostname k8s-master
sudo vi /etc/hosts
init 6

```

Worker 1 & 2

```

sudo hostnamectl set-hostname k8s-worker-1
sudo vi /etc/hosts
init 6

sudo hostnamectl set-hostname k8s-worker-2
sudo vi /etc/hosts
init 6

```

## Step 6: Enable Traffic Bridging (ALL NODES)

For the master and worker nodes to correctly see bridged traffic, we need to enable that module. 

Load the traffic bridging module:

```

sudo modprobe br_netfilter

```

Enable bridging for IPtables:

```

sudo sysctl net.bridge.bridge-nf-call-iptables=1

```

## Step 7: Change the Docker CGroup driver (ALL NODES)

Kubernetes recommends running containers with the "systemd” driver set rather than the Docker default “cgroupfs.” This is technically not required, but will throw a warnning message everytime you start the cluster, and it's a simple thing to change.

Change the Docker CGroup driver (copy/paste the whole block as-is:

```

sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{ "exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts":
{ "max-size": "100m" },
"storage-driver": "overlay2"
}
EOF

```


## Step 8: Initialize K8s cluster on Master Node (MASTER NODE ONLY)

Initialize the cluster and CIDR (IP address block) to be used for container networking. If the cluster successfully initializes, you will be provided with a code snippet to join worker nodes to the cluster, as well as export commands to make the "kubectl" commands available to root and normal users.  


```

sudo kubeadm init --pod-network-cidr=10.244.0.0/16

```

If you lab is barebones / under-spec'd, you can force the system to ignore resource checks:

```

sudo kubeadm init --ignore-preflight-errors=NumCPU,Mem --pod-network-cidr=10.244.0.0/16

```

If something goes wrong, you can start over by running these commands:

```

sudo kubeadm reset
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

```



## Step 9: Export K8s control config as root and normal user (MASTER NODE ONLY)

These commands are required to access the "kubectl" commands for cluster config.
For root user:

```

export KUBECONFIG=/etc/kubernetes/admin.conf

```

Exit root and run this as your normal user:

```

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

Verify K8s kubectl command is working

```

kubectl get nodes
kubectl get pods --all-namespaces

```
 

## Step 10: Disable firewall (ALL NODES) and deploy CNI for pod networking (MASTER NODE ONLY)

In production, we would punch holes in the local firewall to make this work. For a lab, let's just disable the firewall. For CNI (container networking interface) we are deploying Flannel. If you want to deply something else, you can find the info here:
https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy


Disable the firewall:

```

sudo ufw disable

```

Deploy Flannel CNI for pod networking:

```

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

Verify Flannel Pod is deployed:

```

kubectl get pods --all-namespaces

```



## Step 11: Join worker nodes to cluster (WORKER NODES ONLY)

On your master node, you will have a config snippet to join the worker nodes right after the cluster initialization. Copy the text exactly as it is, even if there is a line break and past it into your worker nodes CLI.


It will look something like this:

```

sudo kubeadm join 127.0.0.188:6443 --token u81y02.91gqwkxx6rnhnnly 
	--discovery-token-ca-cert-hash sha256:4482ab1c66bf17992ea02c1ba580f4af9f3ad4cc37b24f189db34d6e3fe95c2d

```

After joining the cluster, exit root and create the following directory. $HOME is a variable, you can find yours by "cd ~"
```

mkdir -p $HOME/.kube

```


Go back to the MASTER NODE, switch to users home directory and copy /$HOME/.kube/config to the same directory on your worker nodes
```

# ON MASTER NODE (my lab example)

scp /home/sbickham/.kube/config sbickham@192.168.1.109:/home/sbickham/.kube/config


```

Go to worker nodes and change ownership of the newly copied file
```

sudo chown $(id -u):$(id -g) $HOME/.kube/config

```


Verify you can now run "kubectl" commands on the master AND worker nodes:
```

kubectl get nodes

```






## Step 12: Deploy K8s application and make it accessible (MASTER NODE ONLY)

At this point, we're ready to deploy a containerized app. Let's deploy an NginX container. 


Deploy NginX:

```

kubectl create deployment nginx --image=nginx

```

Verify it was created:

```

kubectl describe deployment nginx

```

Make the service accessible to the outside world:

```

kubectl create service nodeport nginx --tcp=80:80

```

Verify the service port used (it will be in the 32000+ range):

```

kubectl get svc

```

To test the worker nodes running NginX, you can curl them, or just open a web browser:

```

curl your-kubernetes-worker-ip:PORT_FROM_PREVIOUS_COMMAND

```


