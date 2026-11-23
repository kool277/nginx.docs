---
title: Active-Active HA for NGINX Plus on Microsoft Azure Using the Azure Standard Load Balancer
description: Create an active-active F5 NGINX Plus deployment on Microsoft Azure behind an Azure Standard Load Balancer, and spread it across regions with Traffic Manager.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, active-active, Azure Standard Load Balancer, Traffic Manager, health probe, backend pool, NGINX Plus"
f5-summary: >
  This guide shows how to put an Azure Standard Load Balancer in front of two F5 NGINX Plus load balancers on Microsoft Azure.
  The Standard Load Balancer spreads TCP connections across the NGINX Plus instances, and NGINX Plus load balances HTTP requests to the web servers.
  Azure Traffic Manager can then spread the traffic across copies of the setup in several regions.
f5-audience: operator
---

<span id="about-slb"></span><span id="about-traffic-manager"></span><span id="about-nginx"></span>

## Overview

This guide creates a highly available, active-active deployment of F5 NGINX Plus on Microsoft Azure:

- The [Azure Standard Load Balancer](https://azure.microsoft.com/en-us/products/load-balancer/) works at Layer 4. It spreads inbound connections across the NGINX Plus instances in its backend pool, with a [five-tuple hash](https://learn.microsoft.com/en-us/azure/load-balancer/distribution-mode-concepts) by default. It's similar to the AWS [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html).
- **NGINX Plus** works at Layer 7. It load balances requests to the web servers by their content and by the results of [active health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}}). It also offers [session persistence]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md#session-persistence" >}}), [caching]({{< ref "/nginx/admin-guide/content-cache/content-caching.md" >}}), and monitoring with a [dashboard]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}).
- **Azure Traffic Manager** is optional. It load balances with DNS across copies of the setup in several Azure regions. It sends each client to the best endpoint for the routing method that you choose.

<span id="overview"></span>

A Standard Load Balancer has a public front-end IP address, a backend pool, a health probe, and load balancing rules. The health probe checks whether each NGINX Plus instance can take new connections. The rule says which front-end port goes to which backend pool and port.

<img src="/nginx/images/azure-ha-load-balancer-topology.png" alt="Topology diagram for high availability deployment of NGINX Plus in Azure" style="border:2px solid #666666; padding:2px; margin:2px;" />

In this guide, the backend pool has two NGINX Plus instances. They load balance requests to two applications, each on two NGINX Open Source web servers. You can repeat the setup in other regions, and add Traffic Manager in front of the regions.

---

<span id="prereqs"></span>

## Before you begin

Before you begin, make sure you have:

- **An Azure account and subscription**: To create them, see [Azure free account](https://azure.microsoft.com/en-us/free/).
- **A resource group**: This guide calls it `NGINX-Plus-HA`. Use one for this deployment only, if you can.
- **A virtual network**: For details, see [What is Azure Virtual Network?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- **Six VMs in each region**: Two run NGINX Plus, and four run NGINX Open Source. To create and set them up, see [Set up the VMs](#appendix). Each NGINX Plus instance needs a paid or trial subscription.

---

<span id="appendix"></span><span id="create-vm-install-nginx"></span>

## Set up the VMs

The deployment uses six VMs in each region. Create them, and install the NGINX software, as described in [Create Microsoft Azure virtual machines for NGINX Open Source and F5 NGINX Plus]({{< ref "/nginx/deployment-guides/microsoft-azure/virtual-machines-for-nginx.md" >}}). Use these names:

{{< table >}}

| Role | VM names | Software |
|---|---|---|
| Web servers for App 1 | `ngx-oss-app1-1`, `ngx-oss-app1-2` | NGINX Open Source |
| Web servers for App 2 | `ngx-oss-app2-1`, `ngx-oss-app2-2` | NGINX Open Source |
| Load balancers | `ngx-plus-1`, `ngx-plus-2` | NGINX Plus |

{{< /table >}}

The two NGINX Plus VMs need public IP addresses with the same SKU as the Standard Load Balancer, **Standard**.

<span id="configure-web-servers"></span>

1. Set up the four web servers, as described in [Set up the NGINX Open Source web servers]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-oss" >}}). The `ngx-oss-app1-*` VMs serve App 1, and the `ngx-oss-app2-*` VMs serve App 2.

1. <span id="configure-load-balancers"></span>Set up `ngx-plus-1` and `ngx-plus-2`, as described in [Set up the NGINX Plus load balancer]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-plus" >}}).

---

<span id="slb-set-up"></span>

## Set up a Standard Load Balancer

Do these steps in each region of the deployment.

<span id="slb-create"></span>

### Create the Standard Load Balancer

1. Sign in to the [Azure portal](https://portal.azure.com/), and open **Load balancers**.

    <img src="/nginx/images/azure-create-lb-navigate.png" alt="Screenshot of navigating to the 'Load Balancers' page in Azure" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Create**, and then **Standard Load Balancer**.

1. On the **Basics** tab of the **Create load balancer** page, enter these values:

    - **Subscription**: your subscription, `NGINX-Plus-HA-subscription` in this guide
    - **Resource group**: `NGINX-Plus-HA`
    - **Name**: `lb`
    - **Region**: the region of the VMs, **(US) West US 2** in this guide
    - **SKU**: **Standard**
    - **Type**: **Public**
    - **Tier**: **Regional**

    <img src="/nginx/images/azure-create-lb-basics.png" alt="The Basics tab of the Create load balancer page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. On the **Frontend IP configuration** tab, select **Add a frontend IP configuration**. Enter the name `lb_frontend`, the IP version **IPv4**, and a new public IP address with the name `public_ip_lb` and the availability zone **Zone-redundant**. Then save the frontend IP configuration.

1. Select **Review + create**. You set up the backend pool, the health probe, and the rule after the load balancer exists.

1. Check the settings, and select **Create**. The portal shows a message when the deployment is complete.

    <img src="/nginx/images/azure-create-lb-validation-passed.png" alt="Screenshot of Azure 'Validation passed' page for creating a Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/azure-create-lb-deployment-complete.png" alt="The message that the load balancer deployment is complete" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="slb-configure"></span>

### Configure the Standard Load Balancer

1. On the **Load balancers** page, select `lb`.

    <img src="/nginx/images/azure-create-lb-load-balancers.png" alt="Screenshot of Azure 'Load Balancers' page" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <span id="slb-configure-lb-overview"></span>

1. The **Overview** page of the load balancer shows its **Public IP address**. Note it for the rule and the verification steps.

    <img src="/nginx/images/azure-create-lb-select-backend-pools.png" alt="Screenshot of selecting 'Backend pools' on details page for an Azure Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Backend pools**, and then **Add**. Enter these values, and select **Save**:

    - **Name**: `lb_backend_pool`
    - **IP version**: **IPv4**
    - **Virtual machines**: `ngx-plus-1` and `ngx-plus-2`

    <img src="/nginx/images/azure-create-lb-add-backend-pool.png" alt="Screenshot of Azure 'Add backend pool' page for Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

    After a few moments, the VMs appear in the new backend pool.

1. Select **Health probes**, and then **Add**. Enter these values, and select **Save**:

    - **Name**: `lb_probe`
    - **Protocol**: **HTTP**
    - **Port**: `80`
    - **Path**: `/`
    - **Interval**: `5`
    - **Probe threshold**: `2`

    <img src="/nginx/images/azure-create-lb-add-health-probe.png" alt="Screenshot of Azure 'Add health probe' page for Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

    This probe requests the NGINX Plus landing page every five seconds to check that NGINX Plus runs.

1. Select **Load balancing rules**, and then **Add**. Enter these values, and select **Save**:

    - **Name**: `lb_rule`
    - **IP version**: **IPv4**
    - **Frontend IP address**: `lb_frontend`, the frontend IP configuration of the load balancer
    - **Protocol**: **TCP**
    - **Port**: `80`
    - **Backend port**: `80`
    - **Backend pool**: `lb_backend_pool`
    - **Health probe**: `lb_probe (HTTP:80)`
    - **Session persistence**: **None**
    - **Idle timeout (minutes)**: `4`
    - **Enable TCP reset**: cleared
    - **Enable Floating IP**: cleared

    <img src="/nginx/images/azure-create-lb-add-load-balancing-rule.png" alt="Screenshot of Azure 'Add load balancing rule' page for Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

    After a few moments, the new rule appears on the **Load balancing rules** page.

<span id="slb-verify-operation"></span>

### Verify the Standard Load Balancer

1. In a web browser, open the public IP address of the load balancer. The **Welcome to nginx!** page of NGINX Plus opens. It means that the load balancer sent the connection to an NGINX Plus instance.

    <img src="/nginx/images/azure-create-lb-welcome-to-nginx.png" alt="Screenshot of 'Welcome to nginx!' page that verifies correct configuration of an Azure Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Add `/application1/` to the address, and then `/application2/`. The demo pages of App 1 and App 2 open. They mean that NGINX Plus load balances requests to the web servers.

    <img src="/nginx/images/aws-nlb-app1.png" alt="Screenshot of standard NGINX web server demo page from App 1" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/aws-nlb-app2-v2.png" alt="Screenshot of standard NGINX web server demo page from App 2" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="traffic-manager"></span>

## Set up Azure Traffic Manager

After the Standard Load Balancer works, you can deploy the whole setup in more regions for more resilience. Azure Traffic Manager then spreads traffic across the regions with DNS. For this, give the public IP address of each Standard Load Balancer a DNS name, and add it to Traffic Manager as an endpoint.

1. Open **Public IP addresses**, and select the address of the load balancer, `public_ip_lb`.

    <img src="/nginx/images/azure-create-lb-public-ip-addresses.png" alt="Screenshot of Azure 'Public IP addresses' page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Configuration**. In **DNS name label**, enter a DNS name for the load balancer, for example `public-ip-dns`, and save the change.

    <img src="/nginx/images/azure-create-lb-public-ip-lb.png" alt="Screenshot of Azure page for public IP address of a Standard Load Balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Open **Traffic Manager profiles**, and select **Create**.

1. Enter these values, and select **Create**:

    - **Name**: `ngx`
    - **Routing method**: **Performance**
    - **Subscription**: `NGINX-Plus-HA-subscription` in this guide
    - **Resource group**: `NGINX-Plus-HA`

    <img src="/nginx/images/azure-create-trafficmgr-create-tm-profile.png" alt="Screenshot of Azure 'Create Traffic Manager profile' page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. When the profile appears on the **Traffic Manager profiles** page, select `ngx`.

1. Select **Endpoints**, and then **Add**. Enter these values, and select **Add**:

    - **Type**: **Azure endpoint**
    - **Name**: an endpoint name, for example `ep-lb-west-us`
    - **Target resource type**: **Public IP address**
    - **Public IP address**: `public_ip_lb`

    <img src="/nginx/images/azure-create-trafficmgr-add-endpoint.png" alt="Screenshot of Azure 'Add endpoint' page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. For each other region, repeat [Set up a Standard Load Balancer](#slb-set-up), and add its public IP address as another endpoint.

---

## Troubleshooting

### The public IP address of the load balancer doesn't answer

**Symptom**: The browser can't connect to the public IP address of the Standard Load Balancer.

**Cause**: The health probe marks both NGINX Plus VMs as down, or a network security group blocks port 80. If the probe fails, the load balancer has no VM to send connections to.

**Fix**: Check that NGINX Plus runs on both VMs and answers on port 80 with the path `/`. Check that the network security groups of the VMs allow inbound TCP port 80.

---

### The application pages return 404

**Symptom**: `/application1/` or `/application2/` returns `404 Not Found`.

**Cause**: The NGINX Plus configuration doesn't have the locations of the demo environment.

**Fix**: Create `lb.conf` on both NGINX Plus VMs, as described in [Set up the NGINX Plus load balancer]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md#nginx-plus" >}}).

---

## References

- [What is Azure Load Balancer?](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
- [Azure Load Balancer health probes](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview)
- [What is Traffic Manager?](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
- [Set up an NGINX demo environment]({{< ref "/nginx/deployment-guides/setting-up-nginx-demo-environment.md" >}})

---

## Revision history

- Version 2 (November 2026): Rewritten to the how-to template, with consistent names and the five-tuple hash distribution of the Standard Load Balancer.
- Version 1 (September 2020): Initial version (NGINX Plus Release 22)
