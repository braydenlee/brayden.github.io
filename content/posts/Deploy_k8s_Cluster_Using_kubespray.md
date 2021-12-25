+++
title = "Deploy k8s Cluster Using kubespray"
description = ""
tags = [
    "kubespray",
    "k8s",
    "installation",
]
date = "2021-12-25"
categories = [
    "memo",
    "tips",
]
menu = "main"
+++ 

## Introduction

Nowadays, there're couples of way to deploy a k8s cluster in various of form factors, from experimental cluster with a single node to production deployment with hundreds of servers. 

This memo is a note for deploy a 'product' k8s cluster using kubespray \- limited by the resources for the experiment, no dedicated storage nodes provisioned, and the networking is not covered as well.

|Node     |Role               |Etcd|External IP|Internal IP  |           |
|---------|-------------------|----|-----------|-------------|-----------|
|node\-101|Controller         |yes |           |192.168.8.101|ubuntu22.04|
|node\-102|Controller         |yes |           |192.168.8.102|ubuntu22.04|
|node\-103|Controller & worker|yes |           |192.168.8.103|ubuntu22.04|

## Prerequisites

1. 3 servers, with reasonable CPU, memory and disk space
2. At least 2 Ethernet interfaces per server
    1. One interface for Internet access as some packages are downloaded on the fly;
    2. The other interface used for the cluster management, \(as well as pods network in this setup\);
3. Assume ubuntu 22.04 installed;

## Detailed Steps

### Disable swap 

This is required on every node

    $ swapoff \-a

### Enable passwordless login via ssh

This is to enable the passwordless login to the target nodes, so this is required to run on the deployment server \(where we run the kubespray\). 

Generate ssh keys by running ssh\-keygen, simply press "enter" on asking the passphrase.

    $ ssh-keygen

Copy the ssh key id to target server, enter the password following the prompt.

```shell
$ ssh\-copy\-id brayden@192.168.8.101
$ ssh\-copy\-id brayden@192.168.8.102
$ ssh\-copy\-id brayden@192.168.8.103  
```

### Enable passwordless sudo 

Add the line in **BOLD**  as shown below in the %sudo section, for the user used to provision the k8s cluster. This is required on every node.
```
$ sudo vim /etc/sudoers

\# Allow members of group sudo to execute any command
%sudo   ALL=\(ALL:ALL\) ALL
brayden ALL=\(ALL:ALL\) NOPASSWD:ALL
```

### Dependencies

```Bash
$ sudo apt install python3\-pip
$ sudo pip3 install \-\-upgrade pip
$ pip \-\-version
pip 21.3.1 from /home/brayden/.local/lib/python3.9/site\-packages/pip \(python 3.9\)
```

### Download kubespray

```bash
$ git clone https://github.com/kubernetes\-sigs/kubespray.git
$ cd kubespray
$ sudo pip install \-r requirements.txt
```

### Configure the cluster setup

Copy the required configuration files, scripts to a dedicated folder to this cluster, in this example, the new folder is named as k8s\-100\-cluster, as the controller api server will be available at 192.168.8.100.

    $ cp \-rfp inventory/sampe inventory/k8s\-100\-cluster

The major configuration about the cluster is done by the inventory.ini, which consists of four major sections

* **all**
* **kube\_control\_plane**
* **etcd**
* **kube\_node**
```
$ vim inventory/k8s\-100\-cluster/inventory.ini

\# \#\# Configure 'ip' variable to bind kubernetes services on a   
\# \#\# different ip than the default iface 
\# \#\# We should set etcd\_member\_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value. 
\[all\] 
**node\-101 ansible\_host=192.168.8.101   ip=192.168.8.101 etcd\_member\_name=etcd1  
node\-102 ansible\_host=192.168.8.102   ip=192.168.8.102 etcd\_member\_name=etcd2   
node\-103 ansible\_host=192.168.8.103   ip=192.168.8.103 etcd\_member\_name=etcd3**

\# \#\# configure a bastion host if your nodes are not directly reachable 
\# \[bastion\] 
\# bastion ansible\_host=x.x.x.x ansible\_user=some\_user 

\[kube\_control\_plane\]
**node\-101 
node\-102 
node\-103**
\[etcd\]
**node\-101 
node\-102 
node\-103**

\[kube\_node\] 
**node\-103**
```

Another important configuration file we shall touch is `inventory\k8s-100-cluster\group_vars\all\all.yml`.

Here we disable the internal nginx based proxy, and use external load balanced implemented with haproxy.
```
**apiserver_loadbalancer_domain_name: "lb.npg.intel"**  
**loadbalancer\_apiserver:**  
**addres: 192.168.8.100**  
**port: 8443**  
**loadbalancer\_apiserver\_localhost: false**  
**\#loadbalancer\_apiserver\_port: 6443**  
**\#loadbalancer\_apiserver\_healthcheck\_port: 8081**  
**upstream\_dns\_servers:**  
**\- 8.8.8.8**  
**\- 8.8.4.4**  
\# Set the proxy, will be populated to apt source and container runtime proxy conf
**http\_proxy: "http://child\-prc.intel.com:913"**
**https\_proxy: "http://child\-prc.intel.com:913"**
```
inventory/k8s-100-cluster/group_vars/k8s_cluster/k8s-cluster.yml
```
**kube\_service\_addresses: 192.168.0.0/18**  
**kube\_pods\_subnet: 192.168.64.0/18**  
**container\_manager: docker \#containerd**  
**kubelet\_deployment\_type: host**
```

### Required Workaround 

> Only required if docker is selected as the container runtime.


At the time this memo is being written, the ubuntu 22.04 is not yet GA, so some of the URLs or versions for the docker packages are not valid, hardcode required to fix the issue. 
```
diff \-\-git a/roles/container\-engine/docker/vars/ubuntu.yml b/roles/container\-engine/docker/vars/ubuntu.yml
index 253dbf17..c0077ebf 100644

\-\-\- a/roles/container\-engine/docker/vars/ubuntu.yml
\+\+\+ b/roles/container\-engine/docker/vars/ubuntu.yml
@@ \-17,16 \+17,16 @@ docker\_versioned\_pkg:
  'latest': docker\-ce
  '18.09': docker\-ce=5:18.09.9~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
  '19.03': docker\-ce=5:19.03.15~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
- '20.10': docker\-ce=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
- 'stable': docker\-ce=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
+ '20.10': docker\-ce=5:20.10.12~3\-0~ubuntu\-focal
+ 'stable': docker\-ce=5:20.10.12~3\-0~ubuntu\-focal
  'edge': docker\-ce=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}

docker\_cli\_versioned\_pkg:

  'latest': docker\-ce\-cli
  '18.09': docker\-ce\-cli=5:18.09.9~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
  '19.03': docker\-ce\-cli=5:19.03.15~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}

- '20.10': docker\-ce\-cli=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
- 'stable': docker\-ce\-cli=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}
+ '20.10': docker\-ce\-cli=5:20.10.12~3\-0~ubuntu\-focal
+ 'stable': docker\-ce\-cli=5:20.10.12~3\-0~ubuntu\-focal
  'edge': docker\-ce\-cli=5:20.10.11~3\-0~ubuntu\-{{ ansible\_distribution\_release|lower }}

docker\_package\_info:

@@ \-44,5 \+44,8 @@ docker\_repo\_info:

repos:
  - >
    deb \[arch={{ host\_architecture }}\] {{ docker\_ubuntu\_repo\_base\_url }}
-     {{ ansible\_distribution\_release|lower }}
+     focal
      stable

diff \-\-git a/roles/kubernetes/node/tasks/main.yml b/roles/kubernetes/node/tasks/main.yml
index a342d940..9118a3e6 100644
--- a/roles/kubernetes/node/tasks/main.yml
+++ b/roles/kubernetes/node/tasks/main.yml
@@ \-107,7 \+107,7 @@
- name: Modprobe nf_conntrack_ipv4
  modprobe:
-   name: nf_conntrack_ipv4
+   name: nf_conntrack

  state: present
  register: modprobe_nf_conntrack_ipv4
  ignore_errors: true \# noqa ignore\-errors
```

#### Update the hosts file

you may want to add one line for the virtual ip:
```
$ sudo vim /etc/hosts
192.168.8.100 lb.npg.intel
```

## Provision the Cluster

    $ ansible\-playbook \-i inventory/k8s\-100\-cluster/inventory.ini \-\-become \-\-user=brayden \-\-become\-user=root cluster.yml

If error occured during the provistion, it's better to reset the cluster, fix the issue, and relaunch the provision. 

    $ ansible\-playbook \-i inventory/k8s\-100\-cluster/inventory.ini \-\-become \-\-user=brayden \-\-become\-user=root reset.yml
    $ ansible\-playbook \-i inventory/k8s\-100\-cluster/inventory.ini \-\-become \-\-user=brayden \-\-become\-user=root cluster.yml

### Kubectl conf settings
```
$ cd ~
$ mkdir .kube
$ cp /etc/kubernetes/admin.conf .kube/
$ echo "export KUBECONFIG=/home/brayden/.kube/admin.conf"

$ kubectl get pods \-n kube\-system

NAME                                         READY    STATUS    RESTARTS    AGE
calico\-kube\-controllers\-5788f6558\-kkbf4    1/1    Running    0          38h
calico\-node\-hcdh4                            1/1    Running    0          38h
calico\-node\-xfbcq                            1/1    Running    0          38h
calico\-node\-z7qpt                            1/1    Running    0          38h
coredns\-8474476ff8\-7l69z                     1/1    Running    0          38h
coredns\-8474476ff8\-88                        1/1    Running    0          38h
dns\-autoscaler\-5ffdc7f89d\-jqq5m             1/1    Running    0          38h
kube\-apiserver\-node\-101                     1/1    Running    1 (38h ago) 39h
kube\-apiserver\-node\-102                     1/1    Running    1 (38h ago) 39h
kube\-apiserver\-node\-103                     1/1    Running    1 (38h ago) 39h
kube\-controller\-manager\-node\-101           1/1    Running    1          39h
kube\-controller\-manager\-node\-102           1/1    Running    2 (38h ago) 39h
kube\-controller\-manager\-node\-103           1/1    Running    1          39h
kube\-proxy\-fzlvr                             1/1    Running    0          39h
kube\-proxy\-k9z6g                             1/1    Running    0          39h
kube\-proxy\-wrhd7                             1/1    Running    0          39h
kube\-scheduler\-node\-101                     1/1    Running    2 (38h ago) 39h
kube\-scheduler\-node\-102                     1/1    Running    1          39h
kube\-scheduler\-node\-103                     1/1    Running    1          39h
nodelocaldns\-285cc                            1/1    Running    0          38h
nodelocaldns\-2d477                            1/1    Running    0          38h
nodelocaldns\-cpckx                            1/1    Running    0          38h
```

## Troubleshootings

### Failed to download container images

Initially, the container runtime is default to containerd, and kubespray uses nerdctl which is docker compatible tools to pull the container images.

kubespray populated the proxy setting for apt source and containerd service according to the all.yml. However, nerdctl is not able to pull the container image, and no luck with \-\-extra\-vars to the ansiable\-playbook.

    ansible\-playbook \-i inventory/k8s\-100\-cluster/inventory.ini \-\-become \-\-user=brayden cluster.yml \-\-extra\-vars "https\_proxy=http://child\-prc.intel.com:913,http\_proxy=http://child\-prc.intel.com:913"

While manually run the nerdctl from command line could pull the images successfully.

So this memo switch the container runtime to docker, which could pull the images with the proxy settings.

### Node\-102 & Node\-103 failed to join controller node\-101

One of the error message reads:

    error execution phase preflight: couldn't validate the identity of the API Server: configmaps "cluster\-info" is forbidden: User "system:anonymous" cannot get resource "configmaps" in API group "" in the namespace "kube\-public"

No dedicated effort spent to root cause this issue, a few changes made and the node could join the cluster successfully.

* Do reset before relaunch the provision procedure. Refer to section "Provision the Cluster".
* Clean up the ip routing, name server setting.
    * Configure the Ethernet and nameserver explicitely in /etc/netplan/00\-installer\-config.yaml
```
# This is the network config written by 'subiquity'
network:
  ethernets:
    eno1:
      dhcp4: true
      dhcp6: false
      match:
        macaddress: 00:1e:67:e6:14:ad
      nameservers:
        addresses:
        - 10.248.2.5
        - 10.239.27.228
    eno2:
      dhcp4: false
      addresses:
      - 192.168.8.101/24
      match:
        macaddress: 00:1e:67:e6:14:ae

    * Cleanup the /etc/resolve.conf, remove the local addresses.
    * Make sure "sudo apt update" could be executed successfully.
```

### coredns service crashedbackoff

Server's console will continuous show: 

    [19713.675335] IPVS: rr: UDP 192.168.0.3:53 - no destination available

coredns is in crashloopBackoff status:  

    kube-system    coredns\-576cbf47c7\-8phwt    0/1    CrashLoopBackOff     8  
And the coredns container's log reads  

    plugin/loop: **Loop** \(127.0.0.1:55953 \-\> :53\) **detected for zone "."**, see https://coredns.io/plugins/loop\#troubleshooting. Query: "HINFO 4547991504243258144.3688648895315093531."

There are also some logs reads 

    "... 192.168.0.1 connection refused ..."

This was due to miss-configuration of nameserver, after correct the settings in all.yml, and clean up the /etc/resolv.conf, the issue is fixed.

## Reference

[Install Kubernetes Cluster on Debian 10 with Kubespray | ComputingForGeeks](https://computingforgeeks.com/deploy-kubernetes-cluster-debian-with-kubespray/)  
[Deploying kubernetes using Kubespray \- YouTube](https://www.youtube.com/watch?v=CJ5G4GpqDy0)  
[loop \(coredns.io\)](https://coredns.io/plugins/loop/#troubleshooting)  
[Setup Highly Available Kubernetes Cluster with HAProxy 🇬🇧 \- DEV Community](https://dev.to/mrturkmen/setup-highly-available-kubernetes-cluster-with-haproxy-2dm8)  
[Install Docker Engine on Ubuntu | Docker Documentation](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)  
[k8s join User “system:anonymous“ cannot get resource “configmaps“ in API group ““ in the namespace \- stdworkflow](https://stdworkflow.com/1247/k8s-join-user-system-anonymous-cannot-get-resource-configmaps-in-api-group-in-the-namespace)
