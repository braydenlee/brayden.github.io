layout: page
title: "Set up HA service with keepalived & HAproxy"
permalink: /k8s-ha/

The conf example of keepalived
# Global Settings for notifications
global_defs {
    notification_email {
        baoqian.li@intel.com     # Email address for notifications
    }
    notification_email_from baoqian.li@intel.com        # The from address for the notifications
    smtp_server 127.0.0.1                       # SMTP server address
    smtp_connect_timeout 15
}

# Define the script used to check if haproxy is still working
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2
    weight 2
}

# Configuration for Virtual Interface
vrrp_instance LB_VIP {
    interface eno2
    state MASTER        # set to BACKUP on the peer machine
    priority 101        # set to  99 on the peer machine
    virtual_router_id 51

    smtp_alert          # Enable Notifications Via Email

    authentication {
        auth_type AH
        auth_pass myP@ssword    # Password for accessing vrrpd. Same on all devices
    }
    unicast_src_ip 192.168.8.101 # Private IP address of master
    unicast_peer {
        192.168.8.102           # Private IP address of the backup haproxy
   }

    # The virtual ip address shared between the two loadbalancers
    virtual_ipaddress {
        192.168.8.100
    }

    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}

The conf example of HAproxy
frontend k8s-apiserver
        bind 192.168.8.100:443 #ssl crt /etc/ssl/certs/haproxy.pem
        default_backend kubernetes-apiserver
        option forwardfor

backend kubernetes-apiserver
        balance roundrobin
        server node-101 192.168.8.101:6443 check check-ssl verify none inter 10000
        server node-102 192.168.8.102:6443 check check-ssl verify none inter 10000
        server node-103 192.168.8.103:6443 check check-ssl verify none inter 10000
