---
title: Installing NGINX Plus AMIs on Amazon EC2
description: Launch F5 NGINX Plus on Amazon EC2 from an Amazon Machine Image (AMI) in the AWS Marketplace, and check that it serves its default page.
toc: true
weight: 300
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "nginx plus ami, aws marketplace, amazon ec2, launch instance, security group"
f5-summary: >
  This guide shows how to launch F5 NGINX Plus on Amazon EC2 from an Amazon Machine Image (AMI) in the AWS Marketplace.
  The AMI already contains NGINX Plus, so the instance serves web traffic as soon as it starts.
  AMIs are available for several operating systems, including Amazon Linux, Red Hat Enterprise Linux, and Ubuntu.
f5-audience: operator
---

## Overview

F5 offers Amazon Machine Images (AMIs) with NGINX Plus in the AWS Marketplace. They're available for several operating systems, including Amazon Linux, Red Hat Enterprise Linux, and Ubuntu. An AMI contains the latest version of [NGINX Plus](https://www.f5.com/products/nginx/nginx-plus), optimized for Amazon EC2, and software for building highly available (HA) NGINX Plus setups. When you finish this guide, an EC2 instance runs NGINX Plus and serves its default page.

---

## Before you begin

Before you begin, make sure you have:

- **An AWS account**: To sign up and learn about EC2, follow [Get started with Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html).

---

## Create an NGINX Plus instance from the AMI

1. Go to the [NGINX Plus seller page](https://aws.amazon.com/marketplace/seller-profile?id=741df81b-dfdc-4d36-b8da-945ea66b522c) in the AWS Marketplace, and open the product page of the AMI for your operating system.

1. Select **Continue to Subscribe** to go to the **Launch on EC2** page.

1. Select the tab for the type of launch: {{<nb>}}**1-Click Launch**{{</nb>}}, {{<nb>}}**Manual Launch**{{</nb>}}, or {{<nb>}}**Service Catalog**{{</nb>}}. Choose the options for billing, instance size, and so on, and then select {{<nb>}}**Accept Software Terms…**{{</nb>}}.

1. In the firewall rules, allow incoming web traffic on TCP ports 80 and 443. The {{<nb>}}**1-Click Launch**{{</nb>}} tab adds these rules for you.

When the new EC2 instance starts, NGINX Plus starts automatically and serves a default `index.html` page.

---

## Verify the installation

1. In a web browser, open the public DNS name of the new instance. The browser shows the NGINX Plus default page.

1. To check the NGINX Plus service, log in to the instance and run:

    ```shell
    sudo systemctl status nginx
    ```

    The output shows `active (running)`.

---

## Troubleshooting

### The default page doesn't load

**Symptom**: The browser can't connect to the public DNS name of the instance.

**Cause**: The security group of the instance doesn't allow incoming traffic on port 80.

**Fix**: Add an inbound rule for TCP port 80 to the security group of the instance. If you use HTTPS, add one for port 443 too.

---

## Get help

For help with the NGINX Plus configuration, see the [nginx.org documentation](https://nginx.org/en/docs/) and the [NGINX Plus Admin Guide]({{< ref "/nginx/admin-guide/installing-nginx/" >}}).

If you buy an NGINX Plus AMI in the AWS Marketplace, you can get support from the F5 NGINX engineering team. To activate support, follow F5 article [K2633](https://my.f5.com/manage/s/article/K2633). When you ask for support, have your AWS account number ready, and in some cases the IDs of your EC2 instances.

---

## References

- [NGINX Plus on the AWS Cloud Quick Start](https://aws.amazon.com/about-aws/whats-new/2017/08/quick-start-update-deploy-nginx-plus-on-the-aws-cloud/)
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}), for installing NGINX Plus on your own instance
