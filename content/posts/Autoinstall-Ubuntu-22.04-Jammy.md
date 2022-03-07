+++
title = "Create an autoinstall iso for ubuntu 22.04"
description = ""
tags = [
    "ISO",
    "ubuntu",
    "autoinstall",
    "kickstart",
]
date = "2022-03-07"
categories = [
    "memos",
    "tips",
]
image="images/automation.jpg"
+++ 

## Issue description  

Automation of OS installation and configuration not only saves tremendous efforts and unifies the installation and configuration which otherwise might be different as various decisions made by the user during the process, but also help you out where you could not perform a normal manual installation. I once installed a board with no video support (only serial console available) while the installer requires GUI for the interactive sections.  
Auto install is nothing new, while we single out ubuntu 22.04, because it obsoleted the legacy preseed approach, replaced with its new installer subiquity.  
There are documents from the community on the new autoinstall. However, for newbees, the doc is simply insufficient. And once you get the autoinstall up and running, you will find all info required already included in the doc.   
I realized that, what the newbees need, is a complete, working example.
In this guide, we will customize the ubuntu server live ISO, modify the grub menu to add autoinstall parameters, and add autoinstall configuration files to the ISO. 
For repack the ISO, please refer to my previous post [Re-pack Ubuntu 22.04 live ISO](https://braydenlee.gitee.io/posts/re-pack-ubuntu-22.04-live-server-iso/).

## The detailed steps

It's quite straightforward to customize the ISO with autoinstall:
- Update the GRUB menu  
``` bash
$ vim boot/grub/grub.cfg
# add auto=true autoinstall priority=critical locale=en_US ds='nocloud;s=/cdrom/nocloud/' fsck.mode=skip
# _autoinstall_ is required if you want to suppress the prompt for confirming the installation. The purpose of asking confirmation is to avoid accidentally reformat a machine with the autoinstall enabled USB drive/ISO image. So if we definitely want to have a fully automated installation, we could add the _autoinstall_.
# _locale=en_US_ to suppress the language window -- as if the locale section comes prior to the autoinstall conf, so for a fully automated installation, we add it to the command line.
# _ds=..._ specify where the autoinstall conf files could be found. Note the ending '/' is a must.
# _fsck.mode=skip_ to skip the installation media check. In most of the cases, we haven't see problem with the installation media, so skip the check to accelerate the installation process.
# you could keep the original menuentry and create a new one below the original entry, so your ISO supports both manual and automated installation.
menuentry "Ubuntu Server - *** will wipe your first disk ***" {
        set gfxpayload=keep
        linux   /casper/vmlinuz auto=true autoinstall priority=critical locale=en_US ds='nocloud;s=/cdrom/nocloud/' fsck.mode=skip quiet ---
        initrd  /casper/initrd
}
```  

- Add the autoinstall conf files

In previous section, we specified /cdrom/nocloud as the source for the autoinstall conf. So we shall create a folder with the name _nocloud_ within the root directory in the ISO.

``` bash
$ mkdir nocloud
$ cd nocloud
$ touch meta-data
$ vim user-data

#cloud-config
autoinstall:
  version: 1
  locale: en_US
  # passwd: *** you should replace this part with your own encrypted password***
  # Example (root is the password): 
  # $ printf 'root' | openssl passwd -6 -salt 'FhcddHFVZ7ABA4Gi' -stdin
  identity: {hostname: ubuntu2204, username: admin, password: '$6$2/pERGoDzVMr3JHf$2je/NWRdDEDQW5/tAEXt5hziH3eTmqj5cMirlPkWuvuXVp0o3KG9eh1aqxjk3DRM/fZr2DrwEaKKDoZ/V9HYU.'}
  ssh:
    install-server: yes
    allow-pw: yes
  network:
    version: 2
    ethernets:
      eno1:
        match:
          name: "eno*"
        dhcp6: no
        dhcp4: no
      enp3s0f3:
        critical: true
        dhcp-identifier: mac
        dhcp4: true
  late-commands:
    # in my setup, I have sometime special realtime packages and scripts
    - cp -r /media/cdrom/rt-kits /target/opt/
    - curtin in-target --target=/target -- bash -c 'apt-get install -f -y /opt/rt-kits/linux-image-*.deb /opt/rt-kits/linux-modules-*.deb'
    - curtin in-target --target=/target -- bash -c 'source /opt/rt-kits/update_kernel_cmdline.sh'
    - 
```

That's all, now it's time to repack the ISO. Refer my previous post [Re-pack Ubuntu 22.04 live ISO](https://braydenlee.gitee.io/posts/re-pack-ubuntu-22.04-live-server-iso/)

## References

[Ubuntu autoinstall](https://ubuntu.com/server/docs/install/autoinstall)  
[The obsolete method - preseed](https://help.ubuntu.com/lts/installation-guide/amd64/apb.html)