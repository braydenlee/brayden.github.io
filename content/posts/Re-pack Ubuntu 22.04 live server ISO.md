+++
title = "Re-pack Ubuntu 22.04 live server ISO"
description = ""
tags = [
    "ISO",
    "ubuntu",
]
date = "2022-03-06"
categories = [
    "memos",
    "tips",
]
image="images/iso_disc.png"
+++ 

## Issue description

Previously, bootable ISO image usually has the efi.img and ISOLINUX embeded which are required to boot the ISO and facilitate the installer.  
However, Ubuntu 22.04 replaced ISOLINUX package with EI-Torito and the efi.img with the efi boot partition in the live filesystem on the ISO. So previous methods or commands that relies on the isolinux/isolinux.bin, and efi.img no longer work.  
My initial approach was to bring back the efi.img and add the isolinux package, that's what I'm fimiliar with, and faster to get the bootable ISO created. Although the principle is quite straightforward, the detailed steps to accomplish is bit complex and in-elegent.  
This post will present another approach, which is used to create the Ubuntu 22.04 live ISO. 

## Solution

### The recipe to reproduce the ISO

xorriso has an option which could show how the ISO is cooked and so you could use the same recipe to reproduce it:

``` bash
$ xorriso -indev ../jammy-live-server-amd64.iso -report_el_torito as_mkisofs  
xorriso 1.5.4 : RockRidge filesystem manipulator, libburnia project.

xorriso : NOTE : Loading ISO image tree from LBA 0
xorriso : UPDATE :     810 nodes read in 1 seconds
libisofs: NOTE : Found hidden El-Torito image for EFI.
libisofs: NOTE : EFI image start and size: 685590 * 2048 , 8504 * 512
xorriso : NOTE : Detected El-Torito boot information which currently is set to be discarded
Drive current: -indev '../jammy-live-server-amd64.iso'
Media current: stdio file, overwriteable
Media status : is written , is appendable
Boot record  : El Torito , MBR protective-msdos-label grub2-mbr cyl-align-off GPT
Media summary: 1 session, 687882 data blocks, 1344m data,  802g free
Volume id    : 'Ubuntu-Server 22.04 LTS amd64'
-V 'Ubuntu-Server 22.04 LTS amd64'
--modification-date='2022021411383300'
--grub2-mbr --interval:local_fs:0s-15s:zero_mbrpt,zero_gpt:'../jammy-live-server-amd64.iso'
--protective-msdos-label
-partition_cyl_align off
-partition_offset 16
--mbr-force-bootable
-append_partition 2 28732ac11ff8d211ba4b00a0c93ec93b --interval:local_fs:2742360d-2750863d::'../jammy-live-server-amd64.iso'
-appended_part_as_gpt
-iso_mbr_part_type a2a0d0ebe5b9334487c068b6b72699c7
-c '/boot.catalog'
-b '/boot/grub/i386-pc/eltorito.img'
-no-emul-boot
-boot-load-size 4
-boot-info-table
--grub2-boot-info
-eltorito-alt-boot
-e '--interval:appended_partition_2_start_685590s_size_8504d:all::'
-no-emul-boot
-boot-load-size 8504
```

You now have all the secret recipes that is used to create the original live bootable ISO, simply copy and reuse it.

### An elegant method to apply your customizations to the ISO

Most of the existing methods, requires copy the whole content from the ISO to another folder, add the write permission and then apply the changes.  
It works well - except for some ISOs, you need to pay attention to some of the link files which may results in recursive copying the content. (refer to rsync use in [Ubuntu install CD Customization](https://help.ubuntu.com/community/InstallCDCustomization))  
However, if the original ISO is frequently updated, for example, you are using an early daily build of a new release, you will need to copy and update the files from time to time. Kind of boring and waste of time.  
Inspired by the livefs-edit project, I'm now using the overlay fs technology, which isolate the original iso and the customization content into separate layers, and could be updated separately.

1. mount the original iso as lower layer
1. mount the customized content as the upper layer;
1. generate the new iso using the overlay;

With this approach, you could simply mount the new input iso as the lower layer, re-use the upper layer to generate the new customized iso. No need to copy the content and modify the content.  
Super efficient and elegant.  
Here's an step by step example:

``` shell
$ mkdir content  
$ mkdir content/base  
$ mkdir content/upper  
$ mkdir content/work  
$ mkdir content/ol  
$ mount ubuntu.iso content/base  
\# copy the new content to content/upper
\# for example a new boot/grub/grub.cfg, the autoinstall files user_data, metadata etc.
$ mount -t overlay -o lowerdir=content/base,upperdir=content/upper,workdir=content/work non content/ol

$ xorriso -as mkisofs <the options copied from original recipe> -o <new image>.iso ./content/ol
```

## Acknoledgement

Special thanks to the [livefs_edit project](https://github.com/mwhudson/livefs-editor), where I first tried the livefs_edit, and then learned howto use xorriso to retrieve the recipes and reproduce the ISO.

## Reference

[Repack Bootable ISO](https://wiki.debian.org/RepackBootableISO)  
[Ubuntu 20.04 bootable ISO for EFI](https://utcc.utoronto.ca/~cks/space/blog/linux/Ubuntu2004ISOWithUEFI-2)  
[xorriso man page](https://www.gnu.org/software/xorriso/man_1_xorrisofs.html)  
[Ubuntu install CD Customization](https://help.ubuntu.com/community/InstallCDCustomization)  
[Ubuntu Live CD Customization](https://help.ubuntu.com/community/LiveCDCustomization)  