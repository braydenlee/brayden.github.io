+++
title = "Recover Ubuntu from initramfs error caused by improper shutdown"
description = ""
tags = [
    "recover",
    "ubuntu",
    "initramfs",
]
date = "2022-01-04"
categories = [
    "memos",
    "tips",
]
image="initramfs.png"
+++ 

## Issue description

Following a force shutdown of the server, the system failed to boot with following message:

> initramfs: waiting for /dev/mapper/ubuntu--vg-ubuntu--lv to appear ...  
> timeout waiting ...  
> ...

Then the init process crashed with kernel panic.

## Solution

As if it's due to initramfs damage by the improper shutdown, so the solution is to re-generate the initramfs.

### Boot from a live CD/USB

Could either burn a live iso to a USB stick or mount the ISO to the server (how to mount the ISO to a server via its BMC/IPMI tools, is out of the scope).

### Enter the CLI

Once booted from the live CD, if it's GUI, then press ALT + F2, to enter the CLI. We shall rebuild the initramfs from the CLI.

### The Magic

    sudo lvscan
    sudo mount /dev/ubuntu-vg/ubuntu-lv  /mnt

    \# prepare chroot environment
    sudo mount /dev/sda2 /mnt/boot/   # replace sda2 here is my boot partition!
    sudo mount -o rbind /dev/ /mnt/dev/
    sudo mount -t proc proc /mnt/proc/
    sudo mount -t sysfs sys /mnt/sys/

    \# make dns available in chroot
    sudo cp /etc/resolv.conf  /mnt/etc/resolv.conf 

    \# enter chroot
    sudo chroot /mnt /bin/bash

    \# re-install missing packages
    apt install initramfs-tools

    \# re-generate  (this might be done also by apt in the step before, I'm not sure)
    update-initramfs -u -k all

    \# Leave chroot environment - not sure if the following is really necessary...
    exit
    \# Write buffers to disk
    sudo sync
    \# Unmount file systems
    sudo umount /mnt/sys
    sudo umount /mnt/proc
    sudo umount /mnt/boot

### Done

Reboot and you now should be able to boot into the original system.

## Reference

[Gave up waiting for root device ubuntu-vg-root doesnt exist](https://newbedev.com/gave-up-waiting-for-root-device-ubuntu-vg-root-doesnt-exist)