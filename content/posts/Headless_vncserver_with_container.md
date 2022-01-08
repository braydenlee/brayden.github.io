+++
title = "Headless vncserver with Container"
description = ""
tags = [
    "vnc",
    "vncserver",
    "container"
]
date = "2022-01-08"
categories = [
    "memo",
    "tips",
]
image="images/real-vnc.png"
+++

## Prerequisites 

- Docker intalled, up and running

## Start the vncserver

It's quite straightforward to run the vncserver with container. Here we use the image from accetto.

    docker run -d -p 5920:5901 -p 6901:6901 accetto/ubuntu-vnc-xfce-firefox-g3

Docker will pull the image if not found in the localhost.

## Connect to the vncserver via vnc viewer

Here I use realvnc viwer on windows, enter `<IP@>:5920` to connect, and enter the password, "headless".

## Connect to the vncserver via browser (HTML5 support required)

Enter the url - `<IP@>:6901/vnc.html` to connect. You will be prompted to enter the password.

## MISC

You could change the vnc window resolution by `Application --> Settings --> Display`. Select the resolution fits your need.

Enjoy!

## Reference

[Headless VNCSERVER with Container](https://opensourcelibs.com/lib/ubuntu-vnc-xfce)