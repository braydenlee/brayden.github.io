+++
title = "Setting Up NFS Server"
description = ""
tags = [
    "nfs",
    "nfs server",
    "ubuntu",
    "centos",
]
date = "2022-01-23"
categories = [
    "memos",
    "tips",
]
image="images/kissclipart-nfs.png"
+++ 

Setting up a NFS server is quite straight forward, there're user guides from linux distributors and blogs. However, there are always some cornor cases which requires additional steps or configurations to make it work properly.

This post is focusing on the trouble shooting part with a reference to guides for the NFS server setup.

## Set up the NFS server

I would like to recomend the redhat user guide on setting up the NFS server, which has in depth and in detail explainations on the services and components, along with the step-by-step instructions.  
[RHEL8 NFS server setup](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_file_systems/exporting-nfs-shares_managing-file-systems)

Do enable the services and ports as described in the user guide if you are running the firewall.

    firewall-cmd --permanent --add-service mountd
    firewall-cmd --permanent --add-service rpc-bind
    firewall-cmd --permanent --add-service nfs
    firewall-cmd --permanent --add-port=2049/tcp
    firewall-cmd --reload

Depends on the version and configurations of the server, the nfs server service might failed to start as shown below.

### Failed to start up the NFS server
    
    $ systemctl status rpcbind
       rpcbind.service - RPC bind service
       Loaded: loaded (/usr/lib/systemd/system/rpcbind.service; indirect; vendor preset: enabled)
       Active: failed (Result: exit-code) since Tue 2017-12-26 13:30:11 CST; 14s ago
      Process: 20650 ExecStart=/sbin/rpcbind -w $RPCBIND_ARGS (code=exited, status=0/SUCCESS)
     Main PID: 20653 (code=exited, status=1/FAILURE)
    
    Dec 26 13:30:11 8d58 systemd[1]: Starting RPC bind service...
    Dec 26 13:30:11 8d58 systemd[1]: Started RPC bind service.
    Dec 26 13:30:11 8d58 rpcbind[20650]: rpcbind: fork failed: No such device
    Dec 26 13:30:11 8d58 systemd[1]: rpcbind.service: main process exited, code=exited, status=1/FAILURE
    Dec 26 13:30:11 8d58 systemd[1]: Unit rpcbind.service entered failed state.
    Dec 26 13:30:11 8d58 systemd[1]: rpcbind.service failed.

The root cause might be /dev/null is not correctly labeled if selinux is on, or turn on/off in between. Normally reboot the system will resolve the issue - the system will relabel the file system at startup upon switching on/off selinux. Or explicitely execute `fixfile relabel` and `reboot`.

### selinux is preventing /usr/bin/rpc.idmapd from write access on the file null
    *****  Plugin catchall (100. confidence) suggests   **************************
                                    
    If you believe that rpc.idmapd should be allowed write access on the null file by default.
    Then you should report this as a bug.
    You can generate a local policy module to allow this access.
    Do allow this access for now by executing:
    $ ausearch -c 'rpc.idmapd' --raw | audit2allow -M my-rpcidmapd
    $ semodule -i my-rpcidmapd.pp

## Mount the NFS share on client

Normally, the mount will automatically load the required kernel modules, like lockd, nfs, nfsv4 etc. However, the mount might fail due to missing dependencies or in-correct configurations.

### mount.nfs no such device

For some clients, the mount might fail as:

    mount 192.168.8.3:/store /mnt/nfs
    mount.nfs: No such device

Usually, it's because the lockd, or nfs/nfsv4 module is not loaded, trying to load the module

    $ modprobe nfs
    modprobe: ERROR: could not insert 'nfs': Invalid argument

From dmesg:

    [2192390.701420] FS-Cache: Loaded
    [2192390.709699] lockd: `' invalid for parameter `nlm_tcpport'

This is caused by improper configurations of lockd.
Update the `/etc/modprobe.d/lockd.conf` to add "=" between `nlm_xxxport` and `<port>`.

    options lockd nlm_tcpport=60001
    options lockd nlm_udpport=60002

Try `mount 192.168.8.3:/store /mnt/nfs` again, should be completed successfully.

### Missing dependencies

For ubuntu client, the mount might fail:

    $ sudo mount 192.168.8.3:/store /mnt
    [sudo] password for admin:
    mount: /mnt: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.

The message indicates /sbin/mount.nfs is missing, so we shall need to install the missing dependencis. 

    sudo apt install nfs-common

nfs-common as well as its dependencies will be installed. You could check now the existence of /sbin/mount.nfs by `ls /sbin/mount.nfs`.  

Now try to mount the nfs share again.