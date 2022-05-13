+++
title = "Entering Emergence Mode"
description = ""
tags = [
    "emergence",
    "filesystem",
    "fsck",
]
date = "2022-05-13"
categories = [
    "memos",
    "tips",
]
image="images/emergence-mode.png"
+++ 

## Issue description

Sometimes, after installing a few packages, or upgrade the kernel, re-generate the initramfs, the filesystem is left dirty, especial, where /boot is mounted. On the next boot, the system will most likely warn and entering emergence mode as shown below:
![Emergence Mode](/static/images/emergence-mode.png)

## The solution

As described in previous section, the most common cause is the filesystem cruption, so the fix in most of the time is quite straighforward:

```shell
# fsck.vfat -v -a -w /dev/sda1
fsck.fat 3.0.20 (12 Jun 2013)
fsck.fat 3.0.20 (12 Jun 2013)
Checking we can access the last sector of the filesystem
0x25: Dirty bit is set. Fs was not properly unmounted and some data may be corrupt.
Automatically removing dirty bit.
Boot sector contents:
System ID "MSWIN4.1"
Media byte 0xf8 (hard disk)
512 bytes per logical sector
4096 bytes per cluster
9 reserved sectors
First FAT starts at byte 4608 (sector 9)
2 FATs, 16 bit entries
102400 bytes per FAT (= 200 sectors)
Root directory starts at byte 209408 (sector 409)
512 root directory entries
Data area starts at byte 225792 (sector 441)
51144 data clusters (209485824 bytes)
63 sectors/track, 255 heads
2048 hidden sectors
409600 sectors total
Reclaiming unconnected clusters.
Performing changes.
/dev/sda1: 18 files, 2868/51144 clusters
```

Done.

Or as an temporary workaround, disable the emergence entry for fs failure:

``` shell
vi /lib/systemd/system/local-fs.target

#OnFailure=emergency.target
#OnFailureJobMode=replace-irreversibly
```
