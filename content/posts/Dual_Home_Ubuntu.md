+++
author = "Brayden"
title = "Route setting in Multihomed Server"
date = "2022-01-01"
description = "For dual homed servers which connects to Internet and internal networks, used to have routing issues. This guide provides a approach to correct the routing settings."
tags = [
    "network",
    "ubuntu",
    "netplan",
    "routing",
]
categories = [
    "tips",
	"memos"
]
image = "images/Networking.jpg"
+++

# Route setting in Multihomed servers

## The Issue

For servers with multiples interfaces connecting to differenct networks, tend to have routing issues if multiple interfaces have default route settings.
For example, I have a server with two NICs, one connect to external network in subnet 10.67.126.x/23, and the other connects to internal network in subnet 192.168.8.x/24.
As the DHCP servers for the two subnets both provide default route, the routing table in my server will results in multiple default routes:
	default via 192.168.8.8 dev eno2 proto dhcp src 192.168.8.42 metric 100
	default via 10.67.126.1 dev eno1 proto dhcp src 10.67.126.153 metric 100
	10.67.126.0/23 dev eno1 proto kernel scope link src 10.67.126.153
	10.67.126.1 dev eno1 proto dhcp scope link src 10.67.126.153 metric 100
	192.168.8.0/24 dev eno2 proto kernel scope link src 192.168.8.42
	192.168.8.8 dev eno2 proto dhcp scope link src 192.168.8.42 metric 100
As a result, most of the services resides on the external network are un-reachable as the traffic would be directed to the wrong network.

## The solution

As my internal network is quite simply, all the other nodes are in the same subnet, and don't really need a gateway, so a simply fix is to disable the default route for the internal internface. 
Following netplan conf is an example:
	$ cat /etc/netplan/00-installer-config.yaml
	# This is the network config written by 'subiquity'
	network:
	  ethernets:
		ens260f0:
		  critical: true
		  dhcp-identifier: mac
		  dhcp4: true
		ens260f1:
		  critical: true
		  dhcp-identifier: mac
		  dhcp4: true
		  dhcp4-overrides:
			use-routes: false
	  version: 2

Then apply the settings
	$ sudo netplan apply
Now check the routing table
	$ sudo ip route
	
	default via 10.67.126.1 dev eno1 proto dhcp src 10.67.126.153 metric 100
	10.67.126.0/23 dev eno1 proto kernel scope link src 10.67.126.153 metric 100
	10.67.126.1 dev eno1 proto dhcp scope link src 10.67.126.153 metric 100
	10.239.27.228 via 10.67.126.1 dev eno1 proto dhcp src 10.67.126.153 metric 100
	10.248.2.5 via 10.67.126.1 dev eno1 proto dhcp src 10.67.126.153 metric 100
	172.17.6.9 via 10.67.126.1 dev eno1 proto dhcp src 10.67.126.153 metric 100

## References

[Set default route with netplan on Ubuntu](https://askubuntu.com/questions/1042582/how-to-set-default-route-with-netplan-ubuntu-18-04-server-2-nic)