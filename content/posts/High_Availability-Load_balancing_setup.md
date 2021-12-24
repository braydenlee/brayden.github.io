---
title = "(Hu)go Template Primer"
description = ""
tags = [
    "HA",
    "HAproxy",
    "keepalived",
    "Load balancer",
]
date = "2021-12-22"
categories = [
    "memo",
    "tips",
]
menu = "main"
---
# Introduction

High availability and Load Balancing are the most important features for online services, especially for services in production.

This memo will provide a step\-by\-step guide for how\-to setup the load balancing and High availability for an online web services. The web service used here as the experimental setup, is kubernetes api server.  You could use a dummy web service hosted by nginx or httpd as well.

In this experimental setup, three servers are used
>node\-101 192.168.8.101 \# K8s master node  
>node\-102 192.168.8.102 \# K8s master node   
>node\-103 192.168.8.103 \# K8s master node & worker node

First we shall implement a load balancer to serve the incoming access request to the API server using HAproxy, then implement high availability using keepalived for the HAproxy itself  so there's no actual single point of failure in this setup. Note that, this guide implemented a active\-passive mode high availability. 

# Prerequisites

The procedure is validated on Ubuntu 22.04 daily build 20211128.

1. Make sure the apt source is configured properly, or you have the required packages and their dependencies offline.
2. Packages HAproxy and keepalived

# Detailed steps

We shall deploy the HAproxy and keepalived on the same servers \(node\-101 & node\-102\) where the k8s api server resides. Note that it's not neccessary to be on the same servers, and on heavy loaded system, would be better to have dedicated load balancer servers to host HAproxy.

## Install the required packages

Execute the following command on both node\-101 and node\-102

sudo apt install haproxy keepalived

## Configure Load Balancer

Apply the configuration on both node\-101 and node\-102

sudo vim /etc/haproxy/haproxy.cfg

Add the frontend and backend sections to the end of the cfg file as shown below:

global

        log /dev/log    local0

        log /dev/log    local1 notice

        chroot /var/lib/haproxy

        stats socket /run/haproxy/admin.sock mode 660 level admin expose\-fd listeners

        stats timeout 30s

        user haproxy

        group haproxy

        daemon

        \# Default SSL material locations

        ca\-base /etc/ssl/certs

        crt\-base /etc/ssl/private

        \# See: https://ssl\-config.mozilla.org/\#server=haproxy&server\-version=2.0.3&config=intermediate

        ssl\-default\-bind\-ciphers ECDHE\-ECDSA\-AES128\-GCM\-SHA256:ECDHE\-RSA\-AES128\-GCM\-SHA256:ECDHE\-ECDSA\-AES256\-GCM\-SHA384:ECDHE\-RSA\-AES256\-GCM\-SHA384:ECDHE\-ECDSA\-CHACHA20\-POLY1305:ECDHE\-RSA\-CHACHA20\-POLY1305:DHE\-RSA\-AES128\-GCM\-SHA256:DHE\-RSA\-AES256\-GCM\-SHA384

        ssl\-default\-bind\-ciphersuites TLS\_AES\_128\_GCM\_SHA256:TLS\_AES\_256\_GCM\_SHA384:TLS\_CHACHA20\_POLY1305\_SHA256

        ssl\-default\-bind\-options ssl\-min\-ver TLSv1.2 no\-tls\-tickets

defaults

        log     global

        mode    http

        option  httplog

        option  dontlognull

        timeout connect 5000

        timeout client  50000

        timeout server  50000

        errorfile 400 /etc/haproxy/errors/400.http

        errorfile 403 /etc/haproxy/errors/403.http

        errorfile 408 /etc/haproxy/errors/408.http

        errorfile 500 /etc/haproxy/errors/500.http

        errorfile 502 /etc/haproxy/errors/502.http

        errorfile 503 /etc/haproxy/errors/503.http

        errorfile 504 /etc/haproxy/errors/504.http

frontend k8s\-apiserver

        bind **192.168.8.100:8443** \#ssl crt /etc/ssl/certs/haproxy.pem

        mode tcp

        default\_backend kubernetes\-apiserver

        \#option forwardfor

backend kubernetes\-apiserver

        mode tcp

        balance roundrobin

        **server node\-101 192.168.8.101:6443** check verify none inter 10000

        **server node\-102 192.168.8.102:6443** check verify none inter 10000

        **server node\-103 192.168.8.103:6443** check verify none inter 10000

listen stats

        bind 192.168.8.100:80

        mode http

        stats enable

        stats uri /

Note that, 192.168.8.100 is the virtual IP which is also known as floating IP, which will be implemented by keepalived in later section. The k8s API server is provisioned and accessible via port 6443 on master nodes, and we expose the API server on **192.168.8.100:8443**

Now restart the HAproxy

sudo systemctl restart haproxy

## Configure High availability for HAproxy

Apply the configuration on both node\-101 and node 102

sudo vim /etc/keepalived/keepalived.conf

\# Global Settings for notifications

global\_defs {

    notification\_email {

        \<yourid\>@\<yourdomain\>.com     \# Email address for notifications

    }

    notification\_email\_from \<yourid\>@\<yourdomain\>.com        \# The from address for the notifications

    smtp\_server 127.0.0.1                       \# SMTP server address

    smtp\_connect\_timeout 15

}

\# Define the script used to check if haproxy is still working

vrrp\_script chk\_haproxy {

    script "/usr/bin/killall \-0 haproxy"

    interval 2

    weight 2

}

\# Configuration for Virtual Interface

vrrp\_instance LB\_VIP {

    **interface eno2**

    **state MASTER**           \# set to MASTER on primary server, node\-101

\#    state BACKUP       \# set to BACKUP on the secondary server node\-102

    **priority 101**        \# set to  101 on primary server, node\-101

\#  priority 99          \# set to 99 on secondary server, node\-102

                               \# so by default, node\-101 will be elected as the active proxy server \(load balancer\).

    virtual\_router\_id 51

    smtp\_alert          \# Enable Notifications Via Email

    authentication {

        auth\_type AH

        auth\_pass myP@ssword    \# Password for accessing vrrpd. Same on all devices

    }

    unicast\_src\_ip **192.168.8.101** \# Private IP address of this haproxy server, set to 8.102 for node\-102

    unicast\_peer {

        **192.168.8.102**           \# Private IP address of the peer haproxy proxy server, set to 8.101 for node\-102

   }

    \# The virtual ip address shared between the two loadbalancers

    virtual\_ipaddress {

        **192.168.8.100**

    }

    \# Use the Defined Script to Check whether to initiate a fail over

    track\_script {

        chk\_haproxy

    }

}

restart the keepalived service

sudo systemctl restart keepalived

# Completion

If everything works as expected, you now should be able to see the virtual IP@ on eno2 in server node\-101:

ip addr show eno2

4: eno2: \<BROADCAST,MULTICAST,UP,LOWER\_UP\> mtu 1500 qdisc mq state UP group default qlen 1000

    link/ether 00:1e:67:e6:14:ae brd ff:ff:ff:ff:ff:ff

    altname enp3s0f3

    inet 192.168.8.101/24 scope global eno2

       valid\_lft forever preferred\_lft forever

    inet 192.168.8.100/32 scope global eno2

       valid\_lft forever preferred\_lft forever

    inet6 fe80::21e:67ff:fee6:14ae/64 scope link

       valid\_lft forever preferred\_lft forever

and try to access the service:

nc \-vz 192.168.8.100 8443

Connection to 192.168.8.100 8443 port \[tcp/https\] succeeded\!

# Reference

[Adding HAProxy as load balancer to the Kubernetes cluster | Dominique St\-Amand \(domstamand.com\)](https://www.domstamand.com/adding-haproxy-as-load-balancer-to-the-kubernetes-cluster/)

[Create a Highly Available Kubernetes Cluster Using Keepalived and HAproxy | by KubeSphere | ITNEXT](https://itnext.io/create-a-highly-available-kubernetes-cluster-using-keepalived-and-haproxy-37769d0a65ba)

[Configure Highly Available HAProxy with Keepalived on Ubuntu 20.04 \-](https://kifarunix.com/configure-highly-available-haproxy-with-keepalived-on-ubuntu-20-04/) [kifarunix.com](http://kifarunix.com)

[Install and Setup HAProxy on Ubuntu 20.04 \- kifarunix.com](https://kifarunix.com/install-and-setup-haproxy-on-ubuntu-20-04/#haproxyconfigurationfile)

[Keeping IPs alive without keepalived \- retinadata](https://www.retinadata.com/blog/keeping-ips-alive-without-keepalived/)

[High Availability Guide Red Hat CloudForms 4.6 | Red Hat Customer Portal](https://access.redhat.com/documentation/en-us/red_hat_cloudforms/4.6/html/high_availability_guide/index)

[Exposing the Kubernetes API \- D2iQ Docs](https://docs.d2iq.com/mesosphere/dcos/services/kubernetes/2.4.2-1.15.3/operations/exposing-the-kubernetes-api/)
