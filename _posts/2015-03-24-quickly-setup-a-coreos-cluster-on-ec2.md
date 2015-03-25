---
layout: post
title:  "Use these bash scripts to quickly launch a CoreOS cluster on EC2"
short-title: "Quickly launch a CoreOS cluster"
tags: coreos ec2 docker
category: coreos
author: Michael Vigor
---

While learning about launching CoreOS clusters on EC2, I found the process, even while using the AWS command line client, was slower and more painful than I would have liked. This is especially true once you start deploying to multiple regions, as then all the commands to create security groups, upload key pairs and list instances have to be run once for each region.

To ease the process I ended up writing [this set of bash scripts](https://github.com/bigblue/coreos_on_ec2), which automate the repeated commands across regions, and carry out the housekeeping tasks required before your instances are launched.

## Why CoreOS?

With the recent popularity of docker and container based deployments several stripped down docker focused flavours of linux have emerged, the most mature of which is CoreOS. It uses 40% less RAM than a typical linux server, leaving more resources for your docker containers and has clustering and container orchestration built in. The CoreOS tagline is "Linux for Massive Server Deployments", but I think it is a good fit for smaller scale deployments as well and is worth looking into.

## Quick Demo

Here’s a brief screencast of the scripts in action, and then read on for more details.

<div class="asciiplayer-container" rel="18024" style="width:527px;height:445px;"></div>

## Getting the scripts

Clone the github repository: 

```bash 
git clone https://github.com/bigblue/coreos_on_ec2.git
cd coreos_on_ec2
```

These scripts also require that you have the [AWS command line client](https://github.com/aws/aws-cli) installed and setup with the access details for your account. So if you need to just run:

```bash
pip install awscli
aws configure #Will propmt you to enter your AWS access credentials
```

See the [AWS cli docs](https://github.com/aws/aws-cli) if you run into any problems.

## Launching a cluster in one EC2 region

The included `single_region_userdata.yml` file has the basic settings for each instance where the whole cluster will be in a single region. 

```
#cloud-config

coreos:
  etcd:
    discovery: ${token}
    addr: $private_ipv4:4001
    peer-addr: $private_ipv4:7001
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

etcd will publish the private ip address for the other instances to connect to, and the `${token}` placeholder will be replaced with the discovery token obtained from https://discovery.etcd.io/new

Bear in mind the optimal cluster size for an etcd cluster is an odd number between 3 and 9 members. (see [this CoreOS documentation](https://coreos.com/docs/cluster-management/scaling/etcd-optimal-cluster-size/) for more details.)

To launch a 3 instance cluster in the `us-west-1` region you would run:

```bash
./launchInstances.sh -n 3 \
                     --region "eu-west-1" \
                     --userdata "single_region_userdata.yml"
```

On first run this script will create a new security group in the region specified, (by default called `coreos-testing`) with open ports 4001 and 7001 for etcd, and 22 for ssh.

It will also create a new keypair (by default called `~/.ssh/CoreOSKey_rsa`), which you will be prompted to enter a password for, or leave blank for a password-less key. The public key will be uploaded to EC2.

If you already have a keypair that you would prefer to use rather than generating a new one then, then specify the public key path with with `--key` argument.

With those two bits of housekeeping done the new instances will be launched. The output of the `./launchInstances.sh` command will be all the instance details from your new cluster in json format.

## Launching in multiple regions

For a multi-region cluster the userdata supplied to each instance will need to be slightly different, see the included `multi_region_userdata.yml`

```
#cloud-config

coreos:
  etcd:
    discovery: ${token} 
    addr: $public_ipv4:4001
    bind-addr: 0.0.0.0:4001
    peer-addr: $public_ipv4:7001
    peer-bind-addr: 0.0.0.0:7001
    peer-election-timeout: 2500
    peer-heartbeat-interval: 500
  fleet:
      public-ip: $public_ipv4
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

This time etcd will be publishing the public ip address of the instances. Also the default etcd timeouts are setup for machines within a single data centre. Once there is the increased latency of communicating between data centres then these timeouts need to be increased, so `peer-election-timeout` is increased to 2.5 seconds and `peer-heartbeat-interval` to 0.5 seconds.

With launching to multiple regions the security groups and key pairs have to be setup for each region individually, but the launch script will take care of this for you.

So to launch a 3 instance cluster in `eu-west-1`, `us-west-1`, and `us-east-1` you would run:

```
./launchInstances.sh --region "eu-west-1,us-west-1,us-east-1" \
                     --userdata "multi_region_userdata.yml"
```

## List your running instances

A couple of minutes after you have launched you cluster you will be able to list the instances running, the regions they are ip and their public ip addresses.

If you have only launch instances in a single region you can specify this to speed up the lookup:

```bash
./listInstances -r "us-west-1"
```

or by default `./listInstances` will query every region for running instances that are part of your CoreOS security group.

The output lists the regions, instance ids and public ip addresses of each machine in the cluster:

```bash
#eu-west-1  i-56582ab1   54.73.37.175
#us-east-1  i-68d47094   54.80.101.113
#us-west-1  i-4b8d8c88   54.177.239.168
```

Then you can SSH into any of the instances with the key that was used when launching. You will also want to connect as the `core` user.

```bash
ssh -i ~/.ssh/CoreOSKey_rsa core@54.73.37.175
```

Then to check on the status of your cluster use the fleet tool (CoreOS’s built in cluster management tool) to list the machines in the cluster.

```bash
CoreOS stable (557.2.0)
core@ip-10-246-97-190 ~ $ fleetctl list-machines
MACHINE         IP              METADATA
1fe71d0e…     54.196.11.212   -
211dc8f3…     54.177.222.204  -
71f96747…     79.125.42.50    -
```

## Terminating your cluster

`./terminateInstances.sh` takes the same options as the `./listInstances.sh`
command, so you can filter this by region or group. Running this will terminate all the instances that match the filter.

The output is the json result of aws command, which should show the state of each instance as `shutting-down`.

## Feedback?

If you have any feedback on how these scripts work, or have tried them and run into any issues then please let me know in the comments below.
