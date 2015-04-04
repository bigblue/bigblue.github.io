---
layout: post
title:  "CoreOS next steps: Setup networking with Weave"
short-title: "Setup networking with Weave"
tags: coreos docker weave
category: coreos
author: Michael Vigor
---

Once you have launched your CoreOS cluster, the next step is configuring the network. You can use [Weave](https://github.com/zettio/weave) to setup a virtual private network between the machines. This will connect machines across data centres to a single virtual network. This allows cross data centre networking without leaving ports open to the world. If you provide a password when launching Weave it will also [encrypt traffic between them](https://zettio.github.io/weave/features.html#security).

If you have used the Virtual Private Cloud on EC2 then this may seem redundant.  But by using Weave instead you won't be tied to the EC2 platform. You can use the same configuration on EC2 or other providers.

## Installation

Weave uses UDP and TCP on port 6783 so this will need to be open on EC2. With the [launch script from my previous post](http://blog.bigbluedev.com/coreos/quickly-setup-a-coreos-cluster-on-ec2.html) you can add the option "--weave-rules" to set this up.

Building on the userdata configuration from the [previous post](http://blog.bigbluedev.com/coreos/quickly-setup-a-coreos-cluster-on-ec2.html), Weave can be installed as a service. This configuration was taken from [Luke Bond's guide](https://github.com/lukebond/coreos-vagrant-weave/blob/master/user-data) to running Weave. (only the newly added Unit is shown below).

```yaml
coreos:
  units:
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
      ExecStartPre=/usr/bin/docker pull zettio/weave:latest
      ExecStart=/bin/echo Weave Installed
```

With this Weave will be downloaded and ready to run from /opt/bin/weave.

## Launching the Weave network

To launch, you start the Weave router on one machine. Then pass the IP of the router when starting Weave on the rest of the machines.

The guides\[[1](http://weaveblog.com/2014/10/28/running-a-weave-network-on-coreos/),[2](http://lukebond.ghost.io/running-weave-on-a-coreos-vagrant-cluster/)\] I read on setting up Weave called for you to include the IPs in the userdata. Using EC2 you won't know this before launch, so it needed a different solution.

Once etcd is running across the cluster, `fleet list-machines` can provide every IP address. You can then launch the router on the etcd leader machine. The other machines can launch Weave providing the leader ip address.

[This startup script](https://gist.githubusercontent.com/bigblue/f472de424cc37dabcba0/raw/1907912f5e5b73573c69a07e8b5259eb71d45d8a/weave_startup.sh) will wait for all the machines in the cluster to be connected. Then it will launch Weave providing the correct IPs. (Depending on whether the current machine is the leader or not). Just make sure you provide the cluster size so it knows how many machines to wait for.

```bash
#!/bin/bash
#
# Waits for all machines to have joined cluster
# then starts Weave router on leader, and passes leader
# ip address to all other Weave launches.
#
# Usage: ./weave_startup.sh [clusterSize]
#
clusterSize=$1

while true; do 
  numInCluster=$(fleetctl list-machines --no-legend | wc -l | tr -d '[[:space:]]')
  if [ $numInCluster = $clusterSize ]; then
    leaderId=$(etcdctl get /_coreos.com/fleet/lease/engine-leader | \
               pcregrep -o1 '\"MachineID\":\"(.*)\",')
    leaderIp=$(fleetctl list-machines --full | grep "$leaderId" | cut -f 2)
    selfId=$(curl -Ls http://127.0.0.1:4001/v2/stats/self | \
             pcregrep -o1 '\"name\":\"([^\"]*)\",')
    echo "Leader: $leaderId, Self: $selfId"
    if [ "$selfId" = "$leaderId" ]; then
      echo "Starting weave router"
      /opt/bin/weave launch
    else
      echo "Connecting to weave peer $leaderIp"
      /opt/bin/weave launch $leaderIp
    fi
    break
  else
    echo "$numInCluster of $clusterSize machines connected. Waiting..."
  fi;
  sleep 5;
done
```

The last additional unit to add to the userdata config is the weave service script. This will download the startup script and use it to launch weave.

```yaml
  - name: weave.service
    command: start
    enable: true
    content: |
      [Unit]
      After=install-weave.service
      Description=Weave Network
      Documentation=http://zettio.github.io/weave/
      Requires=install-weave.service

      [Service]
      Environment=WEAVE_PASSWORD='YOURSECUREWEAVEPASSWORD'
      Environment=CLUSTER_SIZE=3
      ExecStartPre=/usr/bin/wget -N -P /opt/bin \
        https://gist.githubusercontent.com/bigblue/f472de424cc37dabcba0/raw/1907912f5e5b73573c69a07e8b5259eb71d45d8a/weave_startup.sh
      ExecStartPre=/usr/bin/chmod +x /opt/bin/weave_startup.sh
      ExecStartPre=/opt/bin/weave_startup.sh $CLUSTER_SIZE
      ExecStart=/usr/bin/docker logs -f weave
      SuccessExitStatus=2
      ExecStop=/opt/bin/weave stop
```

The userdata config for this example can be viewed in its entirety [here](https://github.com/bigblue/coreos_on_ec2/blob/master/install_weave_userdata.yml).

Using the [launch script from my previous post](http://blog.bigbluedev.com/coreos/quickly-setup-a-coreos-cluster-on-ec2.html), you can launch a 3 node cluster with a weave network with this command:

```bash
./launchInstances.sh --region "us-east-1,us-west-1,eu-west-1" --userdata "./install_weave_userdata.yml" --weave-rules
```

## Checking Weave is up and running

When you launch your cluster with those changes to the userdata it should be all you need to get the weave network up and running. To check the startup process you can take a look at the logs with `journalctl -u weave`.

You should see something like this:

```
systemd[1]: Starting Weave Network…
wget[712]: —2015-04-04 09:31:56—  https://gist.githubusercontent.com/bigblue/f472de424cc37dabcba
wget[712]: Resolving gist.githubusercontent.com… 199.27.75.133
wget[712]: Connecting to gist.githubusercontent.com|199.27.75.133|:443… connected.
wget[712]: HTTP request sent, awaiting response… 200 OK
wget[712]: Length: 1009 [text/plain]
wget[712]: Saving to: ‘/opt/bin/weave_startup.sh’
wget[712]: 0K                                                       100% 25.0M=0s
wget[712]: Last-modified header missing — time-stamps turned off.
wget[712]: 2015-04-04 09:31:56 (25.0 MB/s) - ‘/opt/bin/weave_startup.sh’ saved [1009/1009]
weave_startup.sh[716]: 2 of 3 machines connected. Waiting…
weave_startup.sh[716]: 2 of 3 machines connected. Waiting…
weave_startup.sh[716]: Leader: 144949a3fba1439fa19ba44b122f4385, Self: 617fb2c5b5de46948de75ff92b4
weave_startup.sh[716]: Connecting to weave peer 54.176.74.14
weave_startup.sh[716]: Unable to find image ‘zettio/weaveexec:latest’ locally
weave_startup.sh[716]: Pulling repository zettio/weaveexec
weave_startup.sh[716]: fa482a8e857e: Pulling image (latest) from zettio/weaveexec
weave_startup.sh[716]: fa482a8e857e: Pulling image (latest) from zettio/weaveexec, endpoint: https
weave_startup.sh[716]: fa482a8e857e: Pulling dependent layers
weave_startup.sh[716]: fa482a8e857e: Download complete
weave_startup.sh[716]: Status: Downloaded newer image for zettio/weaveexec:latest
weave_startup.sh[716]: 0d9e4d9b8b8898b01dd766017f8032b010ee7882037a37bc00c1b79e50cde635
systemd[1]: Started Weave Network.
```

You can check the status of the network by running `/opt/bin/weave status`. This will show you the number of peers connected to the network:

```
weave router git-b00be096f78a
Encryption on
Our name is 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal)
Sniffing traffic on &{9 65535 ethwe 86:d0:f4:b9:ef:8c up|broadcast|multicast}
MACs:
86:d0:f4:b9:ef:8c -> 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) (2015-04-04 09:32:57.077567899 +0000 UTC)
7a:e9:c3:b7:7a:cf -> 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) (2015-04-04 09:32:57.195209795 +0000 UTC)
ba:19:e4:54:90:40 -> 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) (2015-04-04 09:32:58.057657844 +0000 UTC)
e6:1f:97:5f:33:39 -> 7a:fb:d9:0d:aa:60(ip-10-223-1-126.us-west-1.compute.internal) (2015-04-04 09:33:10.298542565 +0000 UTC)
7a:fb:d9:0d:aa:60 -> 7a:fb:d9:0d:aa:60(ip-10-223-1-126.us-west-1.compute.internal) (2015-04-04 09:33:10.782554905 +0000 UTC)
Peers:
Peer 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) (v59) (UID 2838965584384013476)
   -> 7a:fb:d9:0d:aa:60(ip-10-223-1-126.us-west-1.compute.internal) [54.176.74.14:6783]
   -> 7a:40:b1:15:b6:b2(ip-10-144-80-237.ec2.internal) [23.20.244.25:54394]
Peer 7a:fb:d9:0d:aa:60(ip-10-223-1-126.us-west-1.compute.internal) (v4) (UID 14639705089552752779)
   -> 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) [54.78.82.14:40123]
   -> 7a:40:b1:15:b6:b2(ip-10-144-80-237.ec2.internal) [23.20.244.25:52429]
Peer 7a:40:b1:15:b6:b2(ip-10-144-80-237.ec2.internal) (v62) (UID 8930241348000939739)
   -> 7a:e9:c3:b7:7a:cf(ip-10-39-54-67.eu-west-1.compute.internal) [54.78.82.14:6783]
   -> 7a:fb:d9:0d:aa:60(ip-10-223-1-126.us-west-1.compute.internal) [54.176.74.14:6783]
Routes:
unicast:
7a:e9:c3:b7:7a:cf -> 00:00:00:00:00:00
7a:40:b1:15:b6:b2 -> 7a:40:b1:15:b6:b2
7a:fb:d9:0d:aa:60 -> 7a:fb:d9:0d:aa:60
7a:40:b1:15:b6:b2 -> []
7a:e9:c3:b7:7a:cf -> [7a:fb:d9:0d:aa:60 7a:40:b1:15:b6:b2]
7a:fb:d9:0d:aa:60 -> []
```

## Launching containers on the Weave network

Once the weave network is up and running across the cluster, you can launch your containers. At this point you specify what IP address they should be assigned on the network. Then that container will be accessible across the cluster from that IP.

A simple pinging example to demonstrate (from the [weave readme](https://github.com/zettio/weave)):

```bash
host1# container=$(/opt/bin/weave run 10.0.0.1/24 -t -i busybox)

#On a separate machine in the cluster
host2# container=$(/opt/bin/weave run 10.0.0.2/24 -t -i busybox)

host1# docker attach $container
/ # ping -c 1 -q 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 77.252/77.252/77.252 ms

host2# docker attach $container
/ # ping -c 1 -q 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes

--- 10.0.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 77.468/77.468/77.468 ms
```

## Feedback?

If you have any feedback on this network setup, or have run into any issues then please let me know in the comments below.

