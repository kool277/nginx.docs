---
title: Create Microsoft Azure virtual machines for NGINX Open Source and F5 NGINX Plus
description: Create Microsoft Azure virtual machines for NGINX Open Source or F5 NGINX Plus, connect to them, and install NGINX by hand or with Ansible.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "Microsoft Azure, virtual machine, Azure portal, SSH, Ansible, image, NGINX Plus"
f5-summary: >
  This guide shows how to create Microsoft Azure virtual machines for NGINX Open Source and F5 NGINX Plus, connect to them, and install NGINX.
  The names match the Azure deployment guides, but the virtual machines suit other uses too.
  It also shows how to install NGINX with Ansible and how to copy a virtual machine with an image.
f5-audience: operator
---

## Overview

This guide creates Microsoft Azure virtual machines (VMs) that run NGINX Open Source or F5 NGINX Plus. The names and settings match [Active-Active HA for NGINX Plus on Microsoft Azure Using the Azure Standard Load Balancer]({{< ref "/nginx/deployment-guides/microsoft-azure/high-availability-standard-load-balancer.md" >}}), but the VMs suit other uses too.

For NGINX Plus, a prebuilt VM from the [Azure Marketplace](https://azuremarketplace.microsoft.com/en-us/marketplace/apps?search=NGINX%20Plus) is faster. For details, see [Installing NGINX Plus on Microsoft Azure]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-microsoft-azure.md" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **An Azure account and subscription**: To create them, see [Azure free account](https://azure.microsoft.com/en-us/free/).
- **A resource group**: This guide calls it `NGINX-Plus-HA`. For details, see [Manage resource groups](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/manage-resource-groups-portal).
- **A virtual network**: For details, see [What is Azure Virtual Network?](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- **An NGINX Plus subscription, for NGINX Plus**: A paid or trial subscription, with the certificate, key, and JWT license files from the [MyF5 Customer Portal](https://account.f5.com/myf5).

---

## Create a Microsoft Azure virtual machine

1. Log in to the [Azure portal](https://portal.azure.com/), and open **Virtual machines**.

    <img src="/nginx/images/azure-portal.png" alt="The Azure portal with the Virtual machines service" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Create**, and then **Virtual machine**. The **Create a virtual machine** page opens.

1. On the **Basics** tab, enter these values:

    {{< table >}}

   | Name | Value |
   | ---- | ----- |
   | Subscription | Your subscription, for example `NGINX-Plus-HA-subscription` |
   | Resource group | `NGINX-Plus-HA` |
   | Virtual machine name | `ngx-plus-1` (see the note after this table) |
   | Region | A region near your users, for example **(US) West US 2** |
   | Availability options | **No infrastructure redundancy required** (for testing) |
   | Image | **Ubuntu Server 24.04 LTS**, or another distribution that NGINX supports |
   | Security type | **Trusted launch virtual machines**, the default. For a VM that you capture as a managed image, **Standard**; see [Create an NGINX Open Source image](#create-an-nginx-open-source-image). |
   | Size | **B1s**, or another small size for testing |
   | Authentication type | **SSH public key** |
   | Username | `nginx_azure` |
   | SSH public key source | **Generate new key pair** |
   | Key pair name | `nginx_key` |
   | Public inbound ports | **Allow selected ports** |
   | Select inbound ports | **SSH (22)** and **HTTP (80)**, and **HTTPS (443)** if you use TLS |

    {{< /table >}}

    `ngx-plus-1` is one of the six VM names of the Active-Active HA guide. In production, put a copy of each VM in another Availability Zone, instead of **No infrastructure redundancy required**.

    <img src="/nginx/images/azure-create-vm-basics.png" alt="The Basics tab of the Create a virtual machine page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. On the **Networking** tab, keep the new public IP address that Azure suggests, for example `ngx-plus-1-ip`. Azure creates public IP addresses with the **Standard** SKU, which the Active-Active HA guide needs.

    <img src="/nginx/images/azure-create-vm-networking.png" alt="The Networking tab of the Create a virtual machine page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Optional: Change other settings on the **Disks**, **Management**, **Advanced**, and **Tags** tabs. For example, for the OS disk, Azure selects **Premium SSD**, and **Standard HDD** costs less.

1. Select **Review + create**. After **Validation passed**, check the settings, and select **Create**.

    <img src="/nginx/images/azure-create-vm-validation-passed.png" alt="The Review + create tab after validation passed" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. In the **Generate new key pair** window, select **Download private key and create resource**. Keep the key file: you need it to connect to the VM.

1. Wait a few minutes for the deployment. Azure then shows a summary of the new resources.

    <img src="/nginx/images/azure-create-vm-deployment-complete.png" alt="The summary of the completed VM deployment" style="border:2px solid #666666; padding:2px; margin:2px;" />

For the Active-Active HA guide, create six VMs with these names: `ngx-plus-1`, `ngx-plus-2`, `ngx-oss-app1-1`, `ngx-oss-app1-2`, `ngx-oss-app2-1`, and `ngx-oss-app2-2`. For `ngx-plus-2`, repeat these steps, or use a prebuilt VM from the Azure Marketplace. For the `ngx-oss-*` VMs, you can create one and copy it, as described in [Create an NGINX Open Source image](#create-an-nginx-open-source-image).

---

## Connect to the virtual machine

1. On the **Virtual machines** page, select the name of the VM.

    <img src="/nginx/images/azure-create-vm-virtual-machines.png" alt="The list of virtual machines in the Azure portal" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. On the overview page of the VM, note its **Public IP address**.

    <img src="/nginx/images/azure-create-vm-ngx-plus-1.png" alt="The overview page of the ngx-plus-1 VM with its public IP address" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Connect to the VM over SSH:

    ```shell
    ssh -i <PRIVATE_KEY_FILE> <USERNAME>@<PUBLIC_IP>
    ```

    - `<PRIVATE_KEY_FILE>` is the key file that you downloaded when you created the VM.
    - `<USERNAME>` is the user name from the **Basics** tab, `nginx_azure` in this guide.
    - `<PUBLIC_IP>` is the public IP address of the VM.

---

## Install NGINX Open Source or NGINX Plus

After you connect to the VM, install NGINX as described in [Installing NGINX Open Source]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md" >}}) or [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

---

## Install NGINX Open Source or NGINX Plus with Ansible

F5 publishes an Ansible role for NGINX Open Source and NGINX Plus, `nginxinc.nginx`, on [Ansible Galaxy](https://galaxy.ansible.com/ui/standalone/roles/nginxinc/nginx/). Its source is in the [ansible-role-nginx project](https://github.com/nginx/ansible-role-nginx) on GitHub. These steps run the role on the VM itself.

1. [Connect to the VM](#connect-to-the-virtual-machine).

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
              certificate: /home/nginx_azure/license/nginx-repo.crt
              key: /home/nginx_azure/license/nginx-repo.key
              jwt: /home/nginx_azure/license/license.jwt
    ```

1. Run the playbook:

    ```shell
    ansible-playbook playbook.yml
    ```

1. Check that NGINX is installed:

    ```shell
    nginx -v
    ```

---

<span id="create-an-an-nginx-open-source-image"></span>

## Create an NGINX Open Source image

After you install NGINX Open Source on one VM, you can capture the VM as an image, and create more VMs from it. Capturing a generalized image makes the source VM unusable. So capture an extra VM, for example `ngx-oss`. If you capture one of the six VMs of the Active-Active HA guide, create that VM again afterward.

This procedure creates a managed image. Azure can't create a managed image from a VM that uses Trusted Launch, which is the default security type for Gen2 images such as Ubuntu 24.04. So create the source VM with **Security type** set to **Standard**. For a VM with Trusted Launch, capture the image to an Azure Compute Gallery instead, as described in [Create an image of a VM in the Azure portal](https://learn.microsoft.com/en-us/azure/virtual-machines/capture-image-portal).

1. Connect to the source VM, and remove its machine-specific data. Confirm the prompt, and then end the SSH session:

    ```shell
    sudo waagent -deprovision+user
    ```

    After this step, you can no longer use the VM, only capture it.

1. On the **Virtual machines** page, select the name of the VM, for example `ngx-oss`.

1. On the overview page of the VM, select **Capture**.

    <img src="/nginx/images/azure-create-image-ngx-oss.png" alt="The Capture button on the overview page of the ngx-oss VM" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Enter these values:

    - **Name**: keep the suggested name.
    - **Resource group**: `NGINX-Plus-HA`
    - **Automatically delete this virtual machine after creating the image**: **Yes**
    - **Zone resiliency**: **On**
    - **Type the virtual machine name**: the name of the VM, for example `ngx-oss`

1. Select **Create**. The portal options can differ from this example, depending on how you share the image.

    <img src="/nginx/images/azure-create-image-create-image.png" alt="The Create an image page" style="border:2px solid #666666; padding:2px; margin:2px;" />

### Create a virtual machine from an NGINX Open Source image

1. In the Azure portal, open **Images**.

    <img src="/nginx/images/azure-create-image-images.png" alt="The Images page of the Azure portal" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select the name of the image, and then select **Create VM**.

    <img src="/nginx/images/azure-create-image-create-vm.png" alt="The Create VM button on the page of an image" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Complete the **Create a virtual machine** page, as described in [Create a Microsoft Azure virtual machine](#create-a-microsoft-azure-virtual-machine). The **Image** field already shows your image.

---

## Troubleshooting

### ssh reports that the key permissions are too open

**Symptom**: `ssh` refuses the key file with the message `UNPROTECTED PRIVATE KEY FILE!` and `Permissions ... are too open`.

**Cause**: Other users can read the key file that you downloaded.

**Fix**: Make the key file readable only by you, and connect again:

```shell
chmod 600 <PRIVATE_KEY_FILE>
```

---

### Ansible installs NGINX Open Source instead of NGINX Plus

**Symptom**: After you run the playbook, `nginx -v` shows a version without `nginx-plus`.

**Cause**: The playbook doesn't set `nginx_type: plus`, so the role uses its default, `opensource`.

**Fix**: Add the variables for NGINX Plus to the playbook, as described in [Install NGINX Open Source or NGINX Plus with Ansible](#install-nginx-open-source-or-nginx-plus-with-ansible), and run it again.

---

## References

- [Quickstart: Create a Linux virtual machine in the Azure portal](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal)
- [Installing NGINX Plus on Microsoft Azure]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-microsoft-azure.md" >}})
- [The ansible-role-nginx project on GitHub](https://github.com/nginx/ansible-role-nginx)

---

## Revision history

- Version 3 (November 2026): Rewritten to the how-to template, with a supported Ubuntu image, a valid playbook, the NGINX Plus variables of the Ansible role, and the Standard SKU as the only public IP option.
- Version 2 (December 2025): Revised version
- Version 1 (September 2020): Initial version (NGINX Plus Release 22)
