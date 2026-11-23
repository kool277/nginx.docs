---
title: Active-Active HA for NGINX Plus on AWS Using AWS Network Load Balancer
description: Create an active-active F5 NGINX Plus deployment on AWS, with an AWS Network Load Balancer in front of two or more NGINX Plus instances.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, active-active, AWS Network Load Balancer, NLB, target group, Elastic IP address, NGINX Plus"
f5-summary: >
  This guide shows how to put an AWS Network Load Balancer (NLB) in front of two F5 NGINX Plus load balancers on AWS.
  The NLB spreads TCP connections across the NGINX Plus instances, and NGINX Plus load balances HTTP requests to the web servers.
  If an NGINX Plus instance fails its health checks, the NLB stops sending connections to it.
f5-audience: operator
---

<span id="about-nlb"></span><span id="about-nginx"></span>

## Overview

This guide creates a highly available, active-active deployment of F5 NGINX Plus on Amazon Web Services (AWS):

- The **AWS Network Load Balancer (NLB)** works at Layer 4. It sends each new connection to one of the NGINX Plus instances in its target group, with a [flow hash algorithm](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html#routing-algorithm). It handles millions of requests per second at low latency.
- **NGINX Plus** works at Layer 7. It load balances requests to the web servers by their content and by the results of [active health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}}). It also offers [session persistence]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md#session-persistence" >}}), [caching]({{< ref "/nginx/admin-guide/content-cache/content-caching.md" >}}), and monitoring with a [dashboard]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}).

<span id="overview"></span>

The deployment has an NLB with a target group of two NGINX Plus instances, and four NGINX Open Source web servers that serve two applications:

<img src="/nginx/images/nginx-plus-ha-aws-nlb.png" alt="An AWS NLB sends connections to two NGINX Plus instances, which load balance requests to two pairs of NGINX Open Source web servers" width="1024" height="577" style="border:2px solid #666666; padding:2px; margin:2px;" />

If you don't give the NLB your own Elastic IP addresses, AWS assigns one IP address for each of its Availability Zones. With your own Elastic IP address, the NLB keeps a known address, even if you create it again. You can add more NGINX Plus instances to the target group, or manage them with an [AWS Auto Scaling group](https://aws.amazon.com/autoscaling/).

For an active-passive deployment without an NLB, see [Active-Passive HA for NGINX Plus on AWS Using Elastic IP Addresses]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-keepalived.md" >}}).

---

<span id="prereqs"></span>

## Before you begin

Before you begin, make sure you have:

- **An AWS account**: To create one, see [Create an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html).
- **NGINX Plus subscriptions**: A paid or trial subscription for each NGINX Plus instance.
- **Knowledge of NGINX configuration**: You edit configuration files on each instance.

---

<span id="appendix"></span><span id="create-instance-install-nginx"></span>

## Set up the EC2 instances

The deployment uses six EC2 instances. Create them, and install the NGINX software, as described in [Create Amazon EC2 Instances for NGINX Open Source and NGINX Plus]({{< ref "/nginx/deployment-guides/amazon-web-services/ec2-instances-for-nginx.md" >}}). Use these names:

{{< table >}}

| Role | Instance names | Software |
|---|---|---|
| Web servers for App 1 | `ngx-oss-app1-1`, `ngx-oss-app1-2` | NGINX Open Source |
| Web servers for App 2 | `ngx-oss-app2-1`, `ngx-oss-app2-2` | NGINX Open Source |
| Load balancers | `ngx-plus-1`, `ngx-plus-2` | NGINX Plus |

{{< /table >}}

<img src="/nginx/images/aws-nlb-instances-summary.png" alt="The EC2 Instances page with the six instances of the deployment" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="configure-web-servers"></span>

1. Set up the four web servers, as described in [Set up the NGINX Open Source web servers]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-oss" >}}). The `ngx-oss-app1-*` instances serve App 1, and the `ngx-oss-app2-*` instances serve App 2.

1. <span id="configure-load-balancers"></span>Set up `ngx-plus-1` and `ngx-plus-2`, as described in [Set up the NGINX Plus load balancer]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-plus" >}}).

An older demo created this environment with scripts. It no longer runs, as described in [Packer and Terraform demo](#create-instances-automated).

---

<span id="nlb-configure"></span>

## Create the AWS NLB

<span id="nlb-eip"></span>

### Allocate an Elastic IP address

The Elastic IP address becomes the IP address of the NLB in an Availability Zone. It's optional. Without it, AWS assigns an address, which you lose if you delete the NLB.

1. Open the [Amazon EC2 console](https://console.aws.amazon.com/ec2/).

1. In the navigation pane, select **Elastic IPs**, and then select **Allocate Elastic IP address**.

1. Select **Allocate**. The new address appears on the **Elastic IPs** page. Note it for a later step.

<span id="nlb-routing-options"></span><span id="nlb-register-instances"></span>

### Create the target group

The target group lists the NGINX Plus instances that the NLB sends connections to.

1. In the navigation pane, select **Target Groups**, and then select **Create target group**.

1. Enter these values:

    - **Choose a target type**: **Instances**
    - **Target group name**: `aws-nlb-tg`
    - **Protocol**: **TCP**
    - **Port**: `80`
    - **VPC**: the VPC of the NGINX Plus instances

1. In **Health checks**, keep **TCP** to check that NGINX Plus accepts connections on port 80. To check HTTP responses instead, select **HTTP**, and enter the path `/`.

1. Select **Next**. Select `ngx-plus-1` and `ngx-plus-2`, and then select **Include as pending below**.

1. Select **Create target group**.

<span id="nlb-create"></span><span id="nlb-launch"></span>

### Create the load balancer

1. In the navigation pane, select **Load Balancers**, and then select **Create load balancer**. Under **Network Load Balancer**, select **Create**.

1. Enter these values:

    - **Load balancer name**: `aws-nlb-lb`
    - **Scheme**: **Internet-facing**
    - **Network mapping**: the VPC of the NGINX Plus instances, and the Availability Zone and subnet of each instance. The NLB sends traffic only to targets in the Availability Zones that you select. For the IPv4 address of a subnet, select the Elastic IP address that you allocated.
    - **Security groups**: a security group that allows inbound TCP port 80 from your clients. The console preselects the default security group of the VPC, which allows traffic only from its own members. If you create the NLB without security groups, you can't add them later.
    - **Listeners and routing**: protocol **TCP**, port `80`, and the target group `aws-nlb-tg` as the default action.

1. Select **Create load balancer**.

1. On the **Load Balancers** page, wait until the **State** of the load balancer is **Active**. It can take a few minutes.

For all options, see [Create a Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-network-load-balancer.html) in the AWS documentation.

---

## Verify the deployment

1. On the **Load Balancers** page, select the load balancer, and copy its **DNS name**.

1. In a web browser, open the DNS name or the Elastic IP address. The landing page of NGINX Plus opens. It means that the NLB sent the connection to an NGINX Plus instance.

    <img src="/nginx/images/aws-nlb-welcome-to-nginx.png" alt="The Welcome to nginx page that NGINX Plus returns through the AWS NLB" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Add `/application1/` to the address. The demo page of App 1 opens, with the name of the web server that answered.

    <img src="/nginx/images/aws-nlb-app1.png" alt="The demo page of App 1" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Add `/application2/` to the address. The demo page of App 2 opens.

    <img src="/nginx/images/aws-nlb-app2-v2.png" alt="The demo page of App 2" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="create-instances-automated"></span>

## Packer and Terraform demo

The [aws-nlb-ha-asg demo](https://github.com/nginxinc/NGINX-Demos/tree/master/aws-nlb-ha-asg) in the NGINX-Demos repository created this environment with Packer and Terraform: the AMIs, a VPC, security groups, Auto Scaling groups, and the NLB.

{{< call-out class="caution" >}} The demo isn't maintained, and it no longer runs as published. Its Terraform files use launch configurations, which AWS accounts created since October 1, 2024 can't create. Its Packer template needs plugins that current Packer releases don't bundle. It passes no JWT license file, which NGINX Plus R33 and later needs, and its NGINX Plus configuration serves `/app_one` and `/app_two` instead of the paths in this guide. Use it only as a reference. {{< /call-out >}}

---

## Troubleshooting

### The targets are unhealthy

**Symptom**: The target group shows `ngx-plus-1` and `ngx-plus-2` as **Unhealthy**, and the DNS name of the NLB doesn't answer.

**Cause**: NGINX Plus doesn't accept connections on port 80, or the security group of the instances blocks the health checks. The health checks come from IP addresses in the VPC.

**Fix**: Check that NGINX Plus runs on both instances. In the security group of the instances, allow inbound TCP port 80 from the VPC CIDR range and from the clients.

---

### The application pages return 404

**Symptom**: `/application1/` or `/application2/` returns `404 Not Found`.

**Cause**: The NGINX Plus configuration doesn't have the locations of the demo environment.

**Fix**: Create `lb.conf` on both NGINX Plus instances, as described in [Set up the NGINX Plus load balancer]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-plus" >}}).

---

## References

- [Create a Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-network-load-balancer.html)
- [Create a target group for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/create-target-group.html)
- [Set up an NGINX demo environment]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md" >}})
- [HTTP load balancing]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md" >}})
