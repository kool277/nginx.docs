---
title: Installing NGINX Plus on Microsoft Azure
description: Create an F5 NGINX Plus virtual machine from its image in the Microsoft Azure Marketplace, and allow web traffic to reach it.
toc: true
weight: 500
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "nginx plus azure, azure marketplace, virtual machine image, availability set, network security group"
f5-summary: >
  This guide shows how to create F5 NGINX Plus virtual machines from the NGINX Plus image in the Microsoft Azure Marketplace.
  The image already contains NGINX Plus, so a virtual machine serves web traffic as soon as it starts.
  It also covers redundancy with an availability set and the inbound rules for web traffic.
f5-audience: operator
---

## Overview

[F5 NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) is available in the Microsoft Azure Marketplace as a virtual machine (VM) image. The image contains the latest version of NGINX Plus, optimized for Azure. When you finish this guide, NGINX Plus runs on two or more virtual machines and serves its default page.

---

## Before you begin

Before you begin, make sure you have:

- **An Azure account**: To sign up and learn about Azure virtual machines, follow [Create a Linux virtual machine in the Azure portal](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal).

---

## Create the NGINX Plus virtual machines

1. Create an Azure availability set. If Azure maintenance, planned or not, affects one virtual machine in the set, at least one other stays available. For more information, see [Availability options for Azure Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/availability).

1. Search for "NGINX Plus" in the [Azure Marketplace](https://marketplace.microsoft.com/en-us/search/products?search=nginx%2C%20inc&page=1), open the VM image, and follow its installation instructions. Create two or more virtual machines, and add each one to the availability set while you create it. You can't add an existing virtual machine to an availability set.

1. Allow clients outside the virtual network to reach NGINX Plus. In the network security group of the virtual machines, add inbound rules for port 80 (HTTP) and port 443 (HTTPS). For more information, see [Network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview).

When a new virtual machine starts, NGINX Plus starts automatically and serves a default `index.html` page.

---

## Verify the installation

1. In a web browser, open the public IP address of a new virtual machine. The Azure portal shows it on the overview page of the virtual machine. The browser shows the NGINX Plus default page.

1. To check the NGINX Plus service, log in to the virtual machine and run:

    ```shell
    sudo systemctl status nginx
    ```

    The output shows `active (running)`.

---

## Troubleshooting

### The default page doesn't load

**Symptom**: The browser can't connect to the public IP address of the virtual machine.

**Cause**: The network security group doesn't allow incoming traffic on port 80.

**Fix**: Add an inbound rule for port 80, and for port 443 if you use HTTPS, to the network security group of the virtual machine.

---

## Get help

For help with the NGINX Plus configuration, see the [nginx.org documentation](https://nginx.org/en/docs/) and the [NGINX Plus Admin Guide]({{< ref "/nginx/admin-guide/installing-nginx/" >}}).

If you buy an NGINX Plus VM image in the Azure Marketplace, the F5 NGINX engineering team supports the virtual machine. To activate support, follow F5 article [K2633](https://my.f5.com/manage/s/article/K2633). When you ask for support, have your Azure subscription ID ready, and in some cases the deployment IDs of your virtual machines.

---

## References

- [Availability options for Azure Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/availability)
- [Network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}), for installing NGINX Plus on your own virtual machine
