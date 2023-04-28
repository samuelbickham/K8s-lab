
# K8S + Illumio Home Lab Guide - Part 3

```
Author: Samuel Bickham
Version 2023.04

```

## Overview

This document is part 3 of a series walking through building a Kubernetes (K8s) cluster with a containerD runtime and connecting it to an Illumio Core PCE. There are 3 main parts:
1. Build a K8s cluster (Follow <a href="https://github.com/samuelbickham/K8s-lab/blob/main/README.md">this guide</a>)
2. Build an Illumio Core PCE (Follow <a href="https://github.com/johnwesterman/illumio_core/blob/cliffnotes_2022.08/README.md">this guide</a> by John Westerman)
3. Connect the K8s cluster to Illumio Core PCE  (the guide your reading)

This guide walks through the 3rd step, connecting the K8s cluster to the PCE. Follow <a href="https://github.com/johnwesterman/illumio_core/blob/cliffnotes_2022.08/README.md">this guide</a> by John Westerman to build your PCE. Follow <a href="https://github.com/samuelbickham/K8s-lab/blob/main/README.md">this guide</a> to build your K8s cluster home lab. Let's get to it. 


## Summary of Tasks:

1. Verify DNS resolution for PCE FQDN
2. Create a container cluster and VEN pairing profile 
3. Install Helm
4. Create Illumio-values.yaml
5. Deploy Kube-link and C-VEN via Helm
6. Further deployments and testing (gb.yaml and changing enforcement mode)



## Step 1: Verify DNS resolution for PCE FQDN

This guide assumes a PCE has been configured and the FQDN is resolvable through DNS by both K8s infrastructure (master/worker nodes) and Containers. Containers are a bit different, just because the infrastructure (i.e. master/worker node hosts) can resolve a host name, doesn't mean the container will necessarily be able to. To test DNS resolution between containers and the PCE, open a shell to one of the NginX containers created in step 1, install ping and test.  

NOTE: If your PCE is running in the cloud ping may be blocked, at least verify the hostname is resolving and IP is correct. 

Enumerate all pods
```

kubectl get pods -A

```

Open Shell to one of the NginX containers, if you're following step 1 of the guide you should have at least 1 NGINX container deployed
```

kubectl exec --namespace default --stdin --tty nginx-YOUR_UNIQUE_ID -- /bin/bash

```

Install Ping
```

apt-get update -y
apt-get install -y iputils-ping

```

Ping PCE FQDN
```

ping YOUR-PCE-FQDN.com

```



## Step 2: Create a container cluster and VEN pairing profile 

I recommend running the entire setup as root (sudo -i) and then switching back to your normal user account once everyting is setup. 

Create your container clusters (record cluster_id and cluster_token for illum-values.yaml in next step)
```

Main menu -> Infrastructure -> Container Clusters -> "Add"

```

Create your VEN pairing profile (record pairing profile key as that = clsuter_code for illumio-values.yaml in next step)
```

Main Menu -> Workloads and VENs -> Pairing Profiles -> "Add"

```




## Step 3: Install Helm (MASTER NODE ONLY)

Technically we can install this on any node capable of running "kubectl" commands, but to be tidy let's just install it on the master node.
Helm installation can be simplified via the helm-deploy.sh script, however I like to understand what commands are being run, so I am listing them out. 
```

sudo apt-get install apt-transport-https --yes
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm


```



## Step 4: Create Illumio-values.yaml (MASTER NODE ONLY)

Again, technically we can create and run the illumio-values.yaml file from any node with helm, and we can install helm on any node capable of running "kubectl" commands. But since we installed helm on the master node, let's also create and deploy from the master node. 


Create illumio-values.yaml
```

vi illumio-values.yaml

```

Press "i" for insert mode and paste your values. Then "esc" and ":wq" to save and exit
```

pce_url: your_pce_fqdn:8443
cluster_id: [you get this during container cluster creation on the PCE]
cluster_token: [you get this during container cluster creation on the PCE]
cluster_code: [you get this from the VEN pairing created on the PCE]
containerRuntime: containerd 
containerManager: kubernetes

```

If you're following the step 1 guide your container runtime should be "containerd." But if you want to double-check your container runtime:
```

kubectl get nodes -o wide

```

NOTE: If you're using a self-signed cert OR a valid cert signed through private PKI (i.e. not a trusted public 3rd party) you will need to add the following values to illumio-values.yaml AND create a config map (below). If you PCE cert is publicly signed by a trusted 3rd party, you DO NOT need to add the extra bit to illumio-values.yaml OR do the config-map. 

Add private PKi values to yaml (Spacing on this part needs to follow the exact format)
```

extraVolumeMounts:
  - name: root-ca
    mountPath: /etc/pki/tls/ilo_certs/
    readOnly: false
extraVolumes:
  - name: root-ca
    configMap:
      name: root-ca-config

```

Since we need to reference the illumio-system namespace before it exists, let's create it
```

Kubectl create namespace illumio-system

```

Create a config-map (copy PCE server.crt to node where you're performing this command)
```

kubectl -n illumio-system create configmap root-ca-config \
       --from-file=./server.crt

```

Verify config map
```

kubectl -n illumio-system get configmap

```


## Step 5: Deploy Kube-link and C-VEN via Helm (MASTER NODE ONLY)


Deploy via Helm for the first time when no "illumio-system" namespace exists

```

helm install illumio -f ./illumio-values.yaml oci://quay.io/illumio/illumio --namespace illumio-system --create-namespace 

```

Deploy via Helm for the second time or whenever the "illumio-system" namespace already exists

```

helm install illumio -f ./illumio-values.yaml oci://quay.io/illumio/illumio --namespace illumio-system 

```

Deploy a SPECFIC VERSION via helm - this is needed if your deploying on an older/non-latest version of the PCE

```

helm install illumio -f ./illumio-values.yaml oci://quay.io/illumio/illumio --namespace illumio-system --create-namespace -version 3.1.0

```


Test your deployment, or see what version of kube-link / C-VEN is going to be installed

```

helm install illumio -f ./illumio-values.yaml oci://quay.io/illumio/illumio --namespace illumio-system --create-namespace --dryrun

```

UNINSTALL via helm

```

helm uninstall illumio --namespace illumio-system 

```



## Step 6: Troubleshooting, further deployments and testing (gb.yaml and changing enforcement mode)


GET Kubelink logs

```

 kubectl -n illumio-system logs illumio-kubelink-UNIQUE_ID

```

Restart kube-link

```

kubectl rollout restart -n illumio-system deployment/illumio-kubelink

```

Reminder on Illumio PCE folder locations

```

/opt/illumio-pce (executables)
/var/lib/illumio-pce (databases, CERTS (/cert/) , config) 
/var/log/illumio-pce (logs)
/etc/illumio-pce (runtime env files)

```

