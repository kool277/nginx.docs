---
title: Create Amazon EC2 Instances for NGINX Open Source and NGINX Plus
description: Create Amazon EC2 instances for NGINX Open Source or F5 NGINX Plus, connect to them, and install NGINX by hand or with Ansible.
toc: true
weight: 600
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "Amazon EC2, AWS, EC2 instance, security group, AMI, Ansible, nginxinc.nginx"
f5-summary: >
  This guide shows how to create Amazon EC2 instances for NGINX Open Source and F5 NGINX Plus, connect to them, and install NGINX.
  The NGINX deployment guides for AWS use these instances.
  It also shows how to install NGINX with Ansible and how to create an AMI from an instance.
f5-audience: operator
---

## Overview

This guide creates instances in Amazon Elastic Compute Cloud (EC2) that run NGINX Open Source or F5 NGINX Plus.

For NGINX Plus, a prebuilt Amazon Machine Image (AMI) from the AWS Marketplace is faster. AMIs are available for several operating systems, such as Amazon Linux, Red Hat Enterprise Linux, and Ubuntu. For the steps, see [Installing NGINX Plus AMIs on Amazon EC2]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-amazon-web-services.md" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **An AWS account**: To create one, see [Create an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html).
- **An NGINX Plus subscription, for NGINX Plus**: You need the certificate, key, and JWT license files from the [MyF5 Customer Portal](https://account.f5.com/myf5). A trial subscription also works.
- **Linux command-line skills**: You install software, manage files, and run commands over SSH.

---

## Create an EC2 instance {#create-ec2-instances}

1. Log in to the EC2 dashboard of the AWS Management Console: [https://console.aws.amazon.com/ec2](https://console.aws.amazon.com/ec2).

1. In the navigation pane, select **Instances**, and then select **Launch instances**.

1. In the **Name and tags** section, enter a name for the instance. The name appears in the **Name** column of the **Instances** page.

1. In the **Application and OS Images (Amazon Machine Image)** section, select the image of a Linux distribution that NGINX supports.

    {{< img src="/img/aws/aws-nlb-instance-choose-ami.png" alt="Screenshot of AMI section on the EC2 Launch Instance page">}}

1. In the **Instance type** section, select an instance type. The **t2.micro** type is enough for a demo.

    {{< call-out class="note" >}}The [AWS Free Tier](https://aws.amazon.com/free/) can cover small instance types. Its terms change over time, and several instances that run all day use free hours quickly. Check the current terms before you create many instances.{{< /call-out >}}

    {{< img src="/img/aws/aws-nlb-instance-choose-type.png" alt="Screenshot of Instance Type on the EC2 Launch Instance page">}}

1. In the **Key pair (login)** section, select a key pair. To create one, select **Create new key pair**, and download the key file.

    {{< call-out class="note" >}}In production, create a separate key for each instance. If a key leaks, only that instance is at risk.{{< /call-out >}}

1. In the **Network settings** section, keep the default VPC and subnet. Under **Firewall (security groups)**:

    - For the first instance, keep **Create security group**, and select **Allow HTTP traffic from the internet**. If you use HTTPS, also select **Allow HTTPS traffic from the internet**.
    - For SSH, select **My IP** instead of **Anywhere** if you can, so only your network can connect.

    The new security group gets these inbound rules:

    {{< table >}}

   | Type | Protocol | Port range | Source |
   |---|---|---|---|
   | SSH | TCP | 22 | `0.0.0.0/0`, or your IP address |
   | HTTP | TCP | 80 | `0.0.0.0/0` |
   | HTTPS (optional) | TCP | 443 | `0.0.0.0/0` |

    {{< /table >}}

    {{< img src="/img/aws/aws-generic-instance-details.png" alt="Screenshot of Network Settings on the EC2 Launch Instance page">}}

    For more instances, use the same security group. Select **Select existing security group**, and select the group from the list.

    {{< img src="/img/aws/aws-instance-select-sg.png" alt="Screenshot of Network Settings on the EC2 Launch Instance page with the option of selecting an existing security group">}}

1. In the **Configure storage** section, keep the defaults.

1. In the **Summary** panel, check the settings, and select **Launch instance**.

    {{< img src="/img/aws/aws-instance-summary.png" alt="Screenshot of the Summary panel on the EC2 Launch Instance page with button to launch instance">}}

1. On the confirmation page, select the instance ID. The **Instances** page opens and lists the new instance.

    {{< img src="/img/aws/aws-instance-launch-success.png" alt="Screenshot of the confirmation page after the instance launch">}}

    {{< img src="/img/aws/aws-generic-instance-display-first.png" alt="Screenshot of the EC2 Instances page with a single instance">}}

---

<span id="connect-instance"></span>

## Connect to an EC2 instance

To install and set up NGINX on an instance, connect to it over SSH:

1. On the **Instances** page of the EC2 dashboard, select the instance.

1. Select **Connect**. The **Connect to instance** page opens on the **SSH client** tab.

1. Follow the steps on that page. They include an `ssh` command with the name of your key file and the host name of the instance.

    {{< img src="/img/aws/aws-nlb-instance-connect.png" alt="Screenshot of the EC2 Instance Connect with SSH page">}}

---

## Install NGINX software

After you connect to the instance, install NGINX as described in [Installing NGINX Open Source]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md#prebuilt" >}}) or [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

<span id="automate-installation-with-a-configuration-manager"></span>

### Install NGINX with Ansible {#automate-ansible}

F5 publishes an Ansible role for NGINX Open Source and NGINX Plus, `nginxinc.nginx`, on [Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/nginxinc/nginx/). Its source is in the [ansible-role-nginx project](https://github.com/nginx/ansible-role-nginx) on GitHub. These steps run the role on the instance itself.

1. [Connect to the EC2 instance](#connect-to-an-ec2-instance).

1. Install Ansible, as described in [Installing Ansible on specific operating systems](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html).

1. Install the role, and the Ansible collections that it needs. The `community.crypto` collection is needed only for NGINX Plus.

    ```shell
    ansible-galaxy role install nginxinc.nginx
    ansible-galaxy collection install ansible.posix community.general community.crypto
    ```

1. Create a file called `playbook.yml`. For NGINX Open Source, use this content:

    ```yaml
    ---
    - hosts: localhost
      become: true
      roles:
        - role: nginxinc.nginx
    ```

    For NGINX Plus, also set `nginx_type` to `plus`, and set the paths of your certificate, key, and JWT license files in `nginx_license`:

    ```yaml
    ---
    - hosts: localhost
      become: true
      roles:
        - role: nginxinc.nginx
          vars:
            nginx_type: plus
            nginx_license:
              certificate: /home/<USER>/license/nginx-repo.crt
              key: /home/<USER>/license/nginx-repo.key
              jwt: /home/<USER>/license/license.jwt
    ```

    Replace `/home/<USER>/license/` with the directory of your license files.

1. Run the playbook:

    ```shell
    ansible-playbook playbook.yml
    ```

1. Check that NGINX is installed:

    ```shell
    nginx -v
    ```

---

## Restrict outbound traffic

By default, a security group allows all outbound traffic. You can restrict it after you install NGINX.

Security groups are [stateful](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html): responses to inbound connections, such as client requests and health checks, need no outbound rule. Outbound rules apply only to connections that the instance opens. Examples are NGINX Plus connecting to the upstream servers, and package managers downloading updates over HTTP and HTTPS.

1. In the navigation pane of the EC2 dashboard, select **Security Groups**, and select the security group of the instances.

1. On the **Inbound rules** tab, check that the rules from [Create an EC2 instance](#create-ec2-instances) are there.

1. On the **Outbound rules** tab, select **Edit outbound rules**. Replace the default rule with a rule for each connection that the instances open. For example, if NGINX Plus proxies requests to NGINX Open Source instances in the same security group on port 80, add a rule for port 80. In **Destination**, enter the name or ID of the security group, such as **sg-0dd4d3c5284052f99**.

    {{< img src="/img/aws/aws-generic-instance-security-outbound.png" alt="Screenshot of the EC2 Security Group outbound rules page">}}

    Keep outbound HTTPS (TCP port 443) to the internet in any case. The package manager needs it for updates, and NGINX Plus R33 and later sends its usage reports over it. If the initial usage report doesn't reach F5, NGINX Plus stops processing traffic, as the [R33 release notes]({{< ref "/nginx/releases.md#r33" >}}) describe.

---

<span id="optional-create-an-nginx-open-source-ami"></span>

## Create an NGINX Open Source AMI

To install NGINX Open Source on several instances faster, create an AMI from an instance that runs NGINX. Then create the other instances from that AMI.

1. Create an instance and install NGINX, as described in [Create an EC2 instance](#create-ec2-instances) and [Install NGINX software](#install-nginx-software).

1. On the **Instances** page of the EC2 dashboard, select the instance.

1. Select **Actions** > **Image and templates** > **Create image**. By default, AWS restarts the instance to create the image. To avoid that, clear **Reboot instance**, but then the file system of the image might not be consistent.

    {{< img src="/img/aws/aws-generic-create-image-menu.png" alt="Screenshot of the EC2 Create Image button menu">}}

1. On the **Create image** page, enter an **Image name** and, optionally, an **Image description**. Select **Create image**. The **Instances** page shows a message that AWS is creating the image.

1. To check the image, select **AMIs** in the navigation pane, and find the image by its name.

---

## Troubleshooting

### The package manager can't download packages

**Symptom**: Installing or updating packages on the instance stops with timeouts.

**Cause**: The outbound rules of the security group don't allow HTTP and HTTPS to the internet.

**Fix**: Add outbound rules for TCP ports 80 and 443 to `0.0.0.0/0`, or restore the default rule that allows all outbound traffic.

---

### Ansible installs NGINX Open Source instead of NGINX Plus

**Symptom**: After you run the playbook, `nginx -v` shows a version without `nginx-plus`.

**Cause**: The playbook doesn't set `nginx_type: plus`, so the role uses its default, `opensource`.

**Fix**: Add the variables for NGINX Plus to the playbook, as described in [Install NGINX with Ansible](#automate-ansible), and run it again.

---

## References

- [Installing NGINX Plus AMIs on Amazon EC2]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-amazon-web-services.md" >}})
- [Installing NGINX Open Source]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md" >}})
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}})
- [Security groups for your VPC](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [The ansible-role-nginx project on GitHub](https://github.com/nginx/ansible-role-nginx)

---

## Revision history

- Version 4 (November 2026): Rewritten to the how-to template. Outbound rules move after the installation, and the Ansible steps add the collections and NGINX Plus variables that the role needs.
- Version 3 (February 2025): Updated the guide for the new AWS instance creation flow, with new screenshots and links.
- Version 2 (July 2018): Substituted links to the NGINX Plus Admin Guide for sample installation instructions.
- Version 1 (April 2018): Initial version (NGINX Plus Release 14)
