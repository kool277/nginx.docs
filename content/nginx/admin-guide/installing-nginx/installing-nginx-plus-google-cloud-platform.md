---
title: Install NGINX Plus on the Google Cloud Platform
description: Deploy an F5 NGINX Plus virtual machine on Google Compute Engine from the NGINX Plus image in Google Cloud Marketplace.
toc: true
weight: 400
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "nginx plus google cloud, google cloud marketplace, compute engine, virtual machine image, firewall rules"
f5-summary: >
  This guide shows how to deploy F5 NGINX Plus on Google Compute Engine from the NGINX Plus image in Google Cloud Marketplace.
  The image already contains NGINX Plus, so the virtual machine serves web traffic as soon as it starts.
  The Google Cloud console changes over time, so the names of options can differ from the screenshots.
f5-audience: operator
---

## Overview

[F5 NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) is available on the Google Cloud Platform as a virtual machine (VM) image. The image contains the latest version of NGINX Plus, optimized for Google Compute Engine. When you finish this guide, a VM runs NGINX Plus and serves its default page.

---

## Before you begin

Before you begin, make sure you have:

- **A Google Cloud account**: You need permission to create projects and Compute Engine VMs.

The Google Cloud console changes over time. The options that you see can differ from the screenshots, and they also depend on whether you already have projects.

---

## Deploy the NGINX Plus VM

1. Open the [NGINX Plus page](https://console.cloud.google.com/marketplace/details/nginx-public/nginx-plus) in Google Cloud Marketplace, and select **LAUNCH ON COMPUTE ENGINE**.

    <img src="/nginx/images/gcp-nginx-plus-page-oct2018.png" alt="The NGINX Plus product page in Google Cloud Marketplace, with the button that launches it on Compute Engine" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. The **Select or create a project** window opens. Enter a project name, select a value from the **Organization** list, and then select **Create**.

    <img src="/nginx/images/gcp-create-select-project-oct2018.png" alt="The window for creating or selecting a Google Cloud project" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. In the **Configure & Deploy** window, enter or select the zone, the machine type, and the other values. In the **Firewall** section, make sure that the **Allow HTTP traffic** checkbox is selected. For more information about incoming traffic, see the [Firewall rules overview](https://docs.cloud.google.com/firewall/docs/firewalls).

    <img src="/nginx/images/gcp-configure-deploy-feb2018.png" alt="The Configure and Deploy window for a new NGINX Plus deployment" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Deploy**. The console confirms that NGINX Plus is deployed.

    <img src="/nginx/images/gcp-deploy-confirmation-feb2018.png" alt="The confirmation that NGINX Plus is deployed on the Google Cloud Platform" style="border:2px solid #666666; padding:2px; margin:2px;" />

When the project is deployed and the new VM runs, NGINX Plus starts automatically and serves a default `index.html` page.

---

## Verify the installation

1. In a web browser, open the external IP address of the new VM. The Google Cloud console shows it in the list of VM instances. The browser shows the NGINX Plus default page.

1. To check the NGINX Plus service, log in to the VM and run:

    ```shell
    sudo systemctl status nginx
    ```

    The output shows `active (running)`.

---

## Troubleshooting

### The default page doesn't load

**Symptom**: The browser can't connect to the external IP address of the VM.

**Cause**: No firewall rule allows incoming HTTP traffic to the VM.

**Fix**: In the VM settings, select **Allow HTTP traffic**, or add a firewall rule that allows TCP port 80 to the VM.

---

## Get help

For help with the NGINX Plus configuration, see the [nginx.org documentation](https://nginx.org/en/docs/) and the [NGINX Plus Admin Guide]({{< ref "/nginx/admin-guide/installing-nginx/" >}}).

If you buy an NGINX Plus VM image on the Google Cloud Platform, the F5 NGINX engineering team supports the VM. To activate support, follow F5 article [K2633](https://my.f5.com/manage/s/article/K2633).

---

## Read the open source licenses of NGINX Plus

NGINX Plus includes open source software from F5 NGINX and other contributors. Appendix B of the _NGINX Plus Reference Guide_ has the text of the open source licenses. The guide is on the NGINX Plus VM:

```shell
less /usr/share/nginx/html/nginx-modules-reference.pdf
```

The _NGINX Plus Reference Guide_ is also [available online](https://www.nginx.com/wp-content/uploads/2023/08/nginx-modules-reference.pdf).

---

## References

- [Firewall rules overview](https://docs.cloud.google.com/firewall/docs/firewalls)
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}), for installing NGINX Plus on your own VM
