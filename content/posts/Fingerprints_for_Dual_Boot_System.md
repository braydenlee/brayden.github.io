+++
title = "SSH: Fingerprint Mismatch for Dual-boot Servers"
description = ""
tags = [
    "ssh",
    "fingerprint",
    "dual boot"
]
date = "2021-12-22"
categories = [
    "memo",
    "tips",
]
image="images/man-in-the-middle.png"
+++

# SSH: Fingerprint Mismatch for Dual-boot Servers

## The issue

For server with multiple operations systems installed, ssh tends to fail with the fingerprints check:  

    $ ssh 10.67.126.129
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the ECDSA key sent by the remote host is
    SHA256:I4+yk7+wa9rETt+jKFZ2tEvmSecoXRsDYQc1G/f2exA.
    Please contact your system administrator.
    Add correct host key in /root/.ssh/known_hosts to get rid of this message.
    Offending ECDSA key in /root/.ssh/known_hosts:91
    ECDSA host key for 10.67.126.129 has changed and you have requested strict checking.
    Host key verification failed.

One workaround would be remove the existing key in .ssh/known_hosts and then accept the new fingerprint.  
However, once you switch the server to another operation system, you will repeat the workaround again.  
There are a few solutions or kind of workarounds are available, depends on context of the usage.

## Solution

If we are within a private development env and no strict security/access control required, then all the options listed here could be used. Otherwise the option - create dedeciated host profile at client side is recommendated.

### Bypass the local known_host file

    $ ssh -o UserKnownHostsFile=/dev/null 10.67.126.129 -i .ssh/id_rsa
    The authenticity of host '10.67.126.129 (10.67.126.129)' can't be established.
    ECDSA key fingerprint is SHA256:I4+yk7+wa9rETt+jKFZ2tEvmSecoXRsDYQc1G/f2exA.
    ECDSA key fingerprint is MD5:46:22:d8:01:02:dd:10:10:ec:17:28:21:f7:97:22:f0.
    Are you sure you want to continue connecting (yes/no)?

Note that, with this method, you have to type yes to accept the fingerprint everytime, and enter the password if ssh key is not deployed.  
Or another option if passwordless enabled (ssh key is deployed) `-o StrictHostKeyChecking=no` as:  

    $ ssh -o StrictHostKeyChecking=no 10.67.126.129
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the ECDSA key sent by the remote host is
    SHA256:I4+yk7+wa9rETt+jKFZ2tEvmSecoXRsDYQc1G/f2exA.
    Please contact your system administrator.
    Add correct host key in /root/.ssh/known_hosts to get rid of this message.
    Offending ECDSA key in /root/.ssh/known_hosts:91
    Password authentication is disabled to avoid man-in-the-middle attacks.
    Keyboard-interactive authentication is disabled to avoid man-in-the-middle attacks.
    Activate the web console with: systemctl enable --now cockpit.socket

    Last login: Sat Jan  1 22:34:40 2022 from 10.67.126.101

> Note that, if passwordless is not enabled, `-o StrictHostKeyChecking=no` doesn't work - it still complains the fingerprint and prohabits the login with password or keyboard interaction to mitigate the "man-in-the-middle" attacks.

### Fake Single Server

Another workaround would be copy one of the identity file from one OS to the other, so the two OS appears the same to fake the client to believe it is connecting the same instance. 
The identity files are under `/etc/ssh/ssh_host*`, simply copy them to the other operating system.
 
## Reference
[Dual boot - ssh fingerprints for two OSs instances](https://unix.stackexchange.com/questions/521269/ssh-accept-two-key-fingerprints-for-the-same-server-ip)