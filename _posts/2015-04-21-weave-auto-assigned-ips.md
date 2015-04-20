---
layout: post
title:  "Auto assigning IPs with Weave"
short-title: "Auto assigning IPs with Weave"
tags: docker weave
category: weave
author: Michael Vigor
---

One of the minor annoyances with launching new containers onto a weave network is that currently you have to manually specify the IP address that each container will have. (This is something that the Weave team are working on and is currently [in the design phase](https://github.com/weaveworks/weave/wiki/IP-allocation-design)).

In the mean time one simple solution to this is to use the etcd cluster already running across your coreos machines to store the central record of which IPs have been assigned to which containers across the network. I stumbled across [this bash script](https://github.com/gonkulator/weave-clanmgmt) on github by [gonkulator](https://github.com/gonkulator), and it does exactly that.

I’ve made a couple of minor changes to the script and now you can specify the name of your container you are launching and get it assigned to a free IP address. This has the benefit that you can kill a container and relaunch it using the same IP address as before.

## Getting the management script

Update your install-weave unit so that it installs the management script as well as the weave binary:

```yaml
  - name: install-weave.service
    command: start
    enable: true
    content: |
      [Unit]
      After=network-online.target
      After=docker.service
      Description=Install Weave
      Documentation=http://zettio.github.io/weave/
      Requires=network-online.target
      Requires=docker.service

      [Service]
      Type=oneshot
      RemainAfterExit=yes
      ExecStartPre=/usr/bin/wget -N -P /opt/bin \
          https://raw.github.com/zettio/weave/master/weave
      ExecStartPre=/usr/bin/chmod +x /opt/bin/weave
      ExecStartPre=/usr/bin/wget -N -P /opt/bin \
          https://raw.githubusercontent.com/bigblue/weave-clanmgmt/master/weave-clanmgmt
      ExecStartPre=/usr/bin/chmod +x /opt/bin/weave-clanmgmt
      ExecStartPre=/usr/bin/docker pull zettio/weave:latest
      ExecStart=/bin/echo Weave Installed
```

## Getting IPs for new containers

You then create a new c-lan to assign the IP addresses for a particular subnet.

```bash
weave-clanmgmt create clan myclan
weave-clanmgmt create network 10.0.2.0 myclan
```

After that you can assign the free IPs on that subnet when launching new containers:

```bash
CONTAINER_IP=$(weave-clanmgmt assign myclan bb1)
weave run $CONTAINER_IP/24 -t -i --name bb1 -h bb1 busybox
```

## Releasing the IPs again

If you’re removing a container from the cluster and want to free up the IP address again, then you can run this command. Ideally you would have a script listening for docker end events and free up the IPs automatically, but that’s something for another day’s hacking.

```bash
weave-clanmgmt release myclan bb1
```

## Feedback?

If you have any feedback on how this works, or have run into any issues then please let me know in the comments below.
