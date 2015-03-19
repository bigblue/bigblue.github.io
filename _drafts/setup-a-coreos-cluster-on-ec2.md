---
layout: post
title:  "Setup a coreos cluster on EC2"
tags: coreos ec2 docker
author: Michael Vigor
---

If you don't care about testing your cluster across multiple data centers then you can launch all the machines in the the same region, and use their private IP addresses to network between the machines.

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
discoverytoken=$(curl --silent "https://discovery.etcd.io/new?size=${numServers}")
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

## Launching in multiple regions

Here are the AMI IDs for launching coreos instances in each of the EC2 regions:

Region         | AMI ID
-------------- | -------------
eu-west-1      | ami-5b911f2c
eu-central-1   | ami-88c1f295
us-east-1      | ami-8097d4e8
us-west-1      | ami-26b5ad63
us-west-2      | ami-f3702bc3
ap-northeast-1 | ami-ea5c46eb
ap-southeast-1 | ami-70dcf622
ap-southeast-2 | ami-4fd3a775
sa-east-1      | ami-2fe95632

The security group and key pair needs to be created on each region you are using, and then You'll need to launch each machine individually.

```bash
# use public_ip rather than private ec2 ip
userdata=$(cat <<EOF
#cloud-config

coreos:
  etcd:
    discovery: ${discoverytoken} 
    addr: \$public_ipv4:4001
    bind-addr: 0.0.0.0:4001
    peer-addr: \$public_ipv4:7001
    peer-bind-addr: 0.0.0.0:7001
    peer-election-timeout: 2500
    peer-heartbeat-interval: 500
  fleet:
      public-ip: \$public_ipv4
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
EOF
)
```

Then for each of the regions run these commands to launch an instance

```bash
function launchInstance {
  region=$1
  ami_id=$2
  coreosgroup=$(aws --region "$region" ec2 describe-security-groups --output=text | grep "coreos-testing" | cut -s -f 3)
  if [ -z "$coreosgroup" ]; then
    coreosgroup=$(aws --region "$region" ec2 create-security-group --group-name coreos-testing --description "CoreOS instances" --output text)
    ports=(22 4001 7001)
    for port in "${ports[@]}"; do aws --region "$region" ec2 authorize-security-group-ingress --group-id "$coreosgroup" --protocol tcp --port "$port" --cidr 0.0.0.0/0; done
  fi
  if [ ! -f ~/.ssh/CoreOSKey_rsa.pub ]; then 
    ssh-keygen -q -t rsa -C "Core OS cluster key" -f ~/.ssh/CoreOSKey_rsa
  fi
  public_key=$(cat ~/.ssh/CoreOSKey_rsa.pub)
  keypair=$(aws --region "$region" ec2 describe-key-pairs --output "text" | grep "CoreOSKey_rsa")
  if [ -z "$keypair" ]; then
    aws --region "$region" ec2 import-key-pair --key-name "CoreOSKey_rsa" --public-key-material "$public_key"
  fi

  aws --region "$region" ec2 run-instances --image-id "$ami_id" --count 1 --instance-type "m1.small" --security-group-ids "$coreosgroup" --key-name "CoreOSKey_rsa" --monitoring "Enabled=true" --user-data "$userdata"
}

launchInstance "eu-west-1" "ami-5b911f2c"
launchInstance "us-west-1" "ami-26b5ad63"
launchInstance "us-east-1" "ami-8097d4e8"
```

## See your running instances

```bash
function listInstancesForRegion {
  region=$1
  aws --region "$region" ec2 describe-instances --filters Name=group-name,Values=coreos-testing,Name=instance-state-name,Values=running --output text | cut -s -f 8,15 | sed '/^$/d' | sed "s/^/$region  /"
}

function listInstances {
  for region in $@; do
    listInstancesForRegion $region
  done
}

listInstances "eu-west-1" "us-west-1" "us-east-1"
```

## Terminating your cluster

Once you've finished testing, run this command to easily terminate your cluster

```bash
function terminateInstancesForRegion {
  region=$1
  aws --region "$region" ec2 describe-instances --filters Name=group-name,Values=coreos-testing,Name=instance-state-name,Values=running --output text | cut -s -f 8 | sed '/^$/d' | xargs aws --region "$region" ec2 terminate-instances --instance-ids
}

function terminateInstances {
  for region in $@; do
    terminateInstancesForRegion $region
  done
}

terminateInstances "eu-west-1" "us-west-1" "us-east-1"
```
