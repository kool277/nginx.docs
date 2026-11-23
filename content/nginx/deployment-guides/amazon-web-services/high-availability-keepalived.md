---
title: Active-Passive HA for NGINX Plus on AWS Using Elastic IP Addresses
description: Create an active-passive F5 NGINX Plus deployment on AWS that moves an Elastic IP address to the backup instance with keepalived.
toc: true
weight: 200
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, active-passive, keepalived, VRRP, Elastic IP address, AWS, nginx-ha-notify"
f5-summary: >
  This guide shows how to make two F5 NGINX Plus instances on AWS highly available with keepalived and an Elastic IP address.
  When the primary instance fails, the backup instance takes over the Elastic IP address, so clients keep one static address.
  The scripts behind this solution need changes on current Linux distributions, which the guide describes.
f5-audience: operator
---

<span id="ha-aws_overview"></span>

## Overview

This guide creates an active-passive high availability (HA) deployment of F5 NGINX Plus on Amazon Web Services (AWS). It combines the [`keepalived` solution for on-premises HA]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}) with an AWS [Elastic IP address](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html). For an active-active deployment, see [Active-Active HA for NGINX Plus on AWS Using AWS Network Load Balancer]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-network-load-balancer.md" >}}).

The on-premises `keepalived` solution moves a virtual IP address between instances. AWS networking doesn't support that. A load balancer such as Elastic Load Balancing (ELB) in front of NGINX Plus is another option, but it adds cost, limits the protocols you can use, and gives no single static IP address. Route 53 DNS load balancing has delays, because clients and DNS servers cache records.

In this solution, the primary NGINX Plus instance holds the Elastic IP address. If it fails, the backup instance becomes the primary and moves the Elastic IP address to itself:

<img src="/nginx/images/aws-elastic-ip-address-switch.png" alt="When two NGINX Plus nodes hosted in AWS share an elastic IP address, the address switches to the backup automatically when the primary goes down, preserving high availability" style="border:2px solid #666666; padding:2px; margin:2px;" />

Keep these points in mind:

- Your NGINX Plus support contract doesn't cover this solution.
- F5 tested it on Ubuntu 16.04 and CentOS 7, which are no longer supported. On current distributions, the notify script needs the changes in [Update the notify script](#update-the-notify-script).
- Perform each step on both instances, unless the step says otherwise.
- F5 also offers a solution based on [AWS Lambda](https://aws.amazon.com/lambda/), which needs no extra software on the NGINX Plus instances. [F5 Professional Services](https://www.f5.com/services) can deploy and support it for you.

---

## Before you begin

Before you begin, make sure you have:

- **An AWS account**: You need permission to create IAM roles, EC2 instances, and Elastic IP addresses.
- **NGINX Plus**: You install it on two instances in [Step 1](#ha-aws_step1).
- **Root access**: You need `sudo` on both instances.

---

<span id="ha-aws_nginx-plus"></span><span id="ha-aws_step1"></span>

## Step 1: Create two NGINX Plus instances with an IAM role

The HA scripts use the AWS API to move the Elastic IP address, so they need credentials. AWS gives credentials to an instance through an IAM role. For how to create the role and attach it, see [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html).

1. Create an IAM role with this policy. It allows the instance to move the Elastic IP address, and to read the instances and addresses that the scripts need:

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ec2:AssociateAddress",
                    "ec2:DescribeInstances",
                    "ec2:DescribeAddresses",
                    "ec2:DisassociateAddress"
                ],
                "Resource": "*"
            }
        ]
    }
    ```

    Instead of a role, you can also give the scripts access keys, as described in [Step 5](#ha-aws_nginx-ha-notify-script).

1. Create two instances, and install NGINX Plus on each, as described in [Installing NGINX Plus AMIs on Amazon EC2]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-amazon-web-services.md" >}}). Put both instances in the same security group.

1. Attach the IAM role to both instances.

1. Allow `keepalived` traffic between the instances. `keepalived` sends Virtual Router Redundancy Protocol (VRRP) messages, which use IP protocol number 112. In the security group of the instances, add an inbound rule:

    - **Type**: Custom protocol
    - **Protocol**: 112 (VRRP)
    - **Source**: the security group itself

    If you restricted the outbound rules of the group, also allow outbound VRRP to the group, and HTTPS to the AWS API.

---

<span id="ha-aws_eip"></span>

## Step 2: Allocate an Elastic IP address

Allocate an Elastic IP address, and note its allocation ID. This step is needed only once. For the steps, see [Allocate an Elastic IP address](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-eips.html#using-instance-addressing-eips-allocating) in the AWS documentation.

---

<span id="ha-aws_keepalived-install"></span><span id="ha-aws_step3"></span>

## Step 3: Install keepalived, wget, and the AWS CLI

1. Install `keepalived` and `wget` from the repositories of your operating system:

    - Ubuntu:

        ```shell
        sudo apt install keepalived wget
        ```

    - RHEL and its derivatives:

        ```shell
        sudo dnf install keepalived wget
        ```

1. Install the AWS CLI, as described in [Installing or updating to the latest version of the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

---

<span id="ha-aws_ha-scripts"></span><span id="ha-aws_step4"></span>

## Step 4: Download the HA scripts

`keepalived` runs two scripts from the [aws-ha-elastic-ip project](https://github.com/nginxinc/aws-ha-elastic-ip) on GitHub:

- `nginx-ha-check` checks the health of NGINX Plus.
- `nginx-ha-notify` moves the Elastic IP address when the state of the instance changes, for example when the backup becomes the primary.

1. Download the scripts to a directory for `keepalived` scripts, and make them executable. On Ubuntu, this guide uses `/usr/lib/keepalived`:

    ```shell
    sudo mkdir -p /usr/lib/keepalived
    cd /usr/lib/keepalived
    sudo wget https://raw.githubusercontent.com/nginxinc/aws-ha-elastic-ip/master/nginx-ha-check
    sudo wget https://raw.githubusercontent.com/nginxinc/aws-ha-elastic-ip/master/nginx-ha-notify
    sudo chmod +x nginx-ha-check nginx-ha-notify
    ```

    On RHEL and its derivatives, use `/usr/libexec/keepalived` instead.

<span id="update-the-notify-script"></span>

### Update the notify script

The `nginx-ha-notify` script was written for older systems. It reads instance metadata with IMDSv1 requests, which fail on instances that require IMDSv2. It also parses JSON with Python 2 at `/usr/bin/python`, which current distributions don't have. Python 3 stops at its `print` statements with `SyntaxError: Missing parentheses in call to 'print'`.

In `nginx-ha-notify`, replace the two `wget` lines that request `http://169.254.169.254` with these IMDSv2 requests. Keep the lines after them, which only clean up the values:

```shell
TOKEN=`curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 300"`
LOCAL_INTERNAL_DNS_NAME=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-hostname`
INSTANCE_ID=`curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id`
```

Then replace the lines that set `OTHER_INSTANCE_ID`, `ASSOCIATION_ID`, and `EIP_INSTANCE` with AWS CLI queries, which need no Python:

```shell
OTHER_INSTANCE_ID=`aws ec2 describe-instances --filters "Name=private-dns-name,Values=$OTHER_INSTANCE_DNS_NAME" --query 'Reservations[0].Instances[0].InstanceId' --output text`
ASSOCIATION_ID=`aws ec2 describe-addresses --allocation-ids $ALLOCATION_ID --query 'Addresses[0].AssociationId' --output text`
EIP_INSTANCE=`aws ec2 describe-addresses --allocation-ids $ALLOCATION_ID --query 'Addresses[0].InstanceId' --output text`
```

---

<span id="ha-aws_keepalived-configure"></span>

## Step 5: Configure keepalived and the HA scripts

You set up two files on each instance: `keepalived.conf`, which differs a little between the instances, and the variables in `nginx-ha-notify`.

<span id="ha-aws_keepalived-conf-file"></span>

### Create keepalived.conf

1. Create the file `/etc/keepalived/keepalived.conf` with this content:

    ```none
    vrrp_script chk_nginx_service {
        script "<PATH/TO/HEALTH_CHECK_SCRIPT>"
        interval 3
        weight 50
    }

    vrrp_instance VI_1 {
        interface <INTERFACE>
        priority <PRIORITY>
        virtual_router_id 51
        advert_int 1
        unicast_src_ip <INSTANCE_IP>
        unicast_peer {
            <OTHER_INSTANCE_IP>
        }
        authentication {
            auth_type PASS
            auth_pass <PASSWORD>
        }
        track_script {
            chk_nginx_service
        }
        notify "<PATH/TO/NOTIFY_SCRIPT>"
    }
    ```

1. Replace the placeholders, including the angle brackets:

    - `<PATH/TO/HEALTH_CHECK_SCRIPT>`: the path of `nginx-ha-check`, for example `/usr/lib/keepalived/nginx-ha-check` on Ubuntu or `/usr/libexec/keepalived/nginx-ha-check` on RHEL.
    - `<INTERFACE>`: the network interface of the instance, such as `eth0` or `ens5`. The `ip -br addr` command lists the interfaces.
    - `<PRIORITY>`: `101` on the primary instance and `100` on the backup. The instance with the higher value becomes the primary.
    - `<INSTANCE_IP>`: the private IP address of this instance.
    - `<OTHER_INSTANCE_IP>`: the private IP address of the other instance.
    - `<PASSWORD>`: a password that both instances share.
    - `<PATH/TO/NOTIFY_SCRIPT>`: the path of `nginx-ha-notify`, in the same directory as `nginx-ha-check`.

<span id="ha-aws_nginx-ha-notify-script"></span>

### Set the variables in nginx-ha-notify

In the user-defined variables at the top of `nginx-ha-notify`, set these values:

```none
#export AWS_ACCESS_KEY_ID=
#export AWS_SECRET_ACCESS_KEY=
export AWS_DEFAULT_REGION=<VALUE>
HA_NODE_1=<VALUE>
HA_NODE_2=<VALUE>
ALLOCATION_ID=<VALUE>
```

- `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`: access keys for the AWS API. Uncomment and set them only if the instances have no IAM role.
- `AWS_DEFAULT_REGION`: the AWS region of your deployment.
- `HA_NODE_1` and `HA_NODE_2`: the private DNS names of the two NGINX Plus instances.
- `ALLOCATION_ID`: the allocation ID of the Elastic IP address.

---

<span id="ha-aws_testing"></span>

## Test the failover

1. Start `keepalived` on both instances, and set it to start when the system starts:

    ```shell
    sudo systemctl enable --now keepalived
    ```

    The instance with the higher priority becomes the primary. The AWS console shows the Elastic IP address on the primary instance.

1. Check the state of each instance:

    ```shell
    cat /var/run/nginx-ha-keepalived.state
    ```

    The output is `STATE=MASTER` on the primary instance and `STATE=BACKUP` on the backup.

1. To simulate a failure, stop `keepalived` on the primary instance:

    ```shell
    sudo systemctl stop keepalived
    ```

1. On the backup instance, check the state again. It's now `STATE=MASTER`, and the AWS console shows the Elastic IP address on this instance.

---

<span id="ha-aws_caveats"></span>

## Limitations

- In F5's tests, AWS took 5 to 6 seconds to move the Elastic IP address.
- Moving an Elastic IP address can cost money. See [Amazon EC2 pricing](https://aws.amazon.com/ec2/pricing/).
- The solution uses the AWS API to move the address. If the instances change state quickly or lose contact with each other (split brain), the address can end up on the wrong instance. F5 couldn't reproduce this in testing. If it happens, restart `keepalived` on both instances.

---

<span id="ha-aws_troubleshooting"></span>

## Troubleshooting

### Both instances report STATE=MASTER

**Symptom**: The state file shows `STATE=MASTER` on both instances, and the Elastic IP address moves back and forth.

**Cause**: The instances don't receive each other's VRRP messages. The security group doesn't allow IP protocol 112, or `unicast_src_ip` and `unicast_peer` have the wrong addresses.

**Fix**: Add the inbound VRRP rule from [Step 1](#ha-aws_step1), check the addresses in `keepalived.conf`, and restart `keepalived` on both instances.

---

### The Elastic IP address doesn't move

**Symptom**: An instance becomes the primary, but the Elastic IP address stays where it was.

**Cause**: The notify script fails. Common reasons are IMDSv1 requests on an instance that requires IMDSv2, Python 2 code, missing AWS credentials, or a wrong allocation ID.

**Fix**: Read the messages of `keepalived` and of the scripts, which the systemd journal records for the `keepalived` service:

```shell
sudo journalctl -u keepalived
```

Apply the changes in [Update the notify script](#update-the-notify-script). To find the failing command, run the `aws` commands of the script by hand.

---

<span id="has-aws_resources"></span>

## References

- [The aws-ha-elastic-ip project on GitHub](https://github.com/nginxinc/aws-ha-elastic-ip)
- [High availability support for NGINX Plus in on-premises deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}})
- [Configure the instance metadata options](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-existing-instances.html)
- [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)

---

## Revision history

- Version 2 (November 2026): Rewritten to the how-to template. Adds the VRRP security group rule, the download commands, and the changes that the notify script needs for IMDSv2 and Python 3.
- Version 1 (May 2017): Initial version (NGINX Plus Release 12)
