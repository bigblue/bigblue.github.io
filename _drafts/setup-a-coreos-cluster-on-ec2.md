---
layout: post
title:  "Setup a coreos cluster on EC2"
categories: coreos
---


## Prerequisites

Easier if you have the [aws command line client](https://github.com/aws/aws-cli) installed

```bash
pip install awscli
aws configure #Setup client with your AWS access details
  AWS Access Key ID [None]: your access key
  AWS Secret Access Key [None]: your secret key
  Default region name [None]: eu-west-1
  Default output format [None]: json
```

Also [jq](http://stedolan.github.io/jq/) is used to extract and save the private key from the aws json response, but you could do this manually if you don't have jq installed.

```bash
    brew install jq
```

## EC2 Preparation

Setup new security group for the CoreOS instances

```bash
    coreosgroup=$(aws ec2 create-security-group --group-name coreos-testing --description "CoreOS instances" --output text)
    ports=(22 4001 7001)
    for port in "${ports[@]}"; do aws ec2 authorize-security-group-ingress --group-id "$coreosgroup" --protocol tcp --port "$port" --cidr 0.0.0.0/0; done
```

Create a new key pair for the cluster

```bash
    aws ec2 create-key-pair --key-name CoreOSKey | jq -r '.KeyMaterial' > ~/.ssh/CoreOSKey.pem
    chmod 0600 ~/.ssh/CoreOSKey.pem
```

Decide on the number of servers you want to launch in this cluster (in this case 3), and get a discovery token to configure coreos on launch.

```bash
    numServers=3
    discoverytoken=$(curl "https://discovery.etcd.io/new?size=${numServers}")
```

## Launching a cluster in one EC2 region

If you don't care about testing your cluster across multiple data centers then you can launch all the machines in the the same region, and use their private IP addresses to network between the machines.

```bash
    ami="ami-5b911f2c"
    userdata=$(cat <<EOF
    #cloud-config
    
    coreos:
      etcd:
        discovery: ${discoverytoken}
        addr: \$private_ipv4:4001
        peer-addr: \$private_ipv4:7001
      units:
        - name: etcd.service
          command: start
        - name: fleet.service
          command: start
    EOF
    )
    json_request="{\"Monitoring\":{\"Enabled\":true},\"InstanceType\":\"m1.small\",\"SecurityGroupIds\":[\"${coreosgroup}\"],\"KeyName\":\"CoreOSKey\",\"MaxCount\":${numServers},\"MinCount\":${numServers},\"ImageId\":\"${ami}\"}"
    aws ec2 run-instances --cli-input-json "$json_request" --count "$numServers" --user-data "$userdata"
```

## See your running instances

```bash
    aws ec2 describe-instances --filters Name=key-name,Values=CoreOSKey,Name=instance-state-name,Values=running --output text | cut -s -f 8,16 | sed '/^$/d'
```

## Terminating your cluster

Once you've finished testing, run this command to easily terminate your cluster

```bash
    aws ec2 describe-instances --filters Name=key-name,Values=CoreOSKey,Name=instance-state-name,Values=running --output text | cut -s -f 8 | sed '/^$/d' | xargs aws ec2 terminate-instances --instance-ids
```
