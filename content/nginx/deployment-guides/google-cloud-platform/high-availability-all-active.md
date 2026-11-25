---
title: All-Active HA for NGINX Plus on the Google Cloud Platform
description: Create an all-active F5 NGINX Plus deployment on Google Compute Engine, with instance groups, health checks, and a Google Cloud network load balancer.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, all-active, Google Compute Engine, instance group, instance template, health check, network load balancer, NGINX Plus"
f5-summary: >
  This guide shows how to deploy F5 NGINX Plus in an all-active, highly available setup on Google Compute Engine.
  A Google Cloud network load balancer spreads connections across NGINX Plus load balancers, which load balance requests to two PHP applications.
  Instance groups with health checks replace instances that fail.
f5-audience: operator
---

<span id="design"></span><span id="design-and-topology"></span>

## Overview

This guide deploys F5 NGINX Plus in a highly available setup on Google Compute Engine (GCE), the virtual machine service of the Google Cloud Platform (GCP). Several NGINX Plus instances work together as load balancers, and all of them are active.

The deployment combines:

- **NGINX Plus** load balancers, which load balance HTTP requests across the instances of two applications. They also keep each client on one application instance with a session cookie.
- **PHP-FPM**, which runs the two sample applications behind NGINX Plus.
- A **Google Cloud network load balancer**, which spreads TCP connections from clients across the NGINX Plus load balancers. It keeps each client on one NGINX Plus instance, by client IP address.
- **Instance groups** with **health checks**, which manage the instances of each role and replace instances that fail.

<img src="/nginx/images/gce-all-active-load-balancing-topology.png" alt="A Google Cloud network load balancer sends connections to a group of NGINX Plus load balancers, which load balance requests to two groups of application instances" style="border:2px solid #666666; padding:2px; margin:2px;" />

The two application groups, `app-1` and `app-2`, have the same content. To route different content to each group, create an `upstream` block for each, and route requests to them by URI. For details, see [Module ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html).

{{< call-out class="caution" >}} In this setup, any public IP address can reach the NGINX Plus instances. That's fine for testing, but not for production. In production, block external HTTP and HTTPS access to the application instances, and to the external IP addresses of the NGINX Plus load balancers. {{< /call-out >}}

The Google Cloud console changes over time, so the names and places of its options can differ from this guide and its screenshots. All names, such as those of projects and instances, are examples.

---

<span id="prereqs"></span><span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- **A Google Cloud account**: You can [create one](https://accounts.google.com/SignUp), and use [free trial credit](https://cloud.google.com/free) or a payment account.
- **Google Cloud knowledge**: You can create instances and manage IAM policies in the Google Cloud console.
- **An NGINX Plus subscription**: A paid subscription or a [free trial](https://www.f5.com/trials).
- **NGINX Plus knowledge**: You can install NGINX Plus and read its configuration.

---

<span id="project-firewall"></span><span id="task-1-creating-a-project-and-firewall-rules"></span><span id="creating-firewall-rules"></span>

## Task 1: Create a project and firewall rules

1. Log in to the [Google Cloud console](https://console.cloud.google.com).

1. Create a project for the deployment. If you have no projects, select **Create a project**. Otherwise, open the project list in the header, and select **New project**. This guide names the project `NGINX Plus All-Active-LB`.

    <img src="/nginx/images/gce-dashboard-no-project.png" alt="The Google Cloud dashboard without projects, with the button to create a project" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-new-project-popup.png" alt="The New Project window of the Google Cloud console" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Open **VPC network** > **Firewall**, and select **Create firewall rule**.

    <img src="/nginx/images/gce-firewall-rules-tab.png" alt="The firewall rules page of the Google Cloud console" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Enter these values, and select **Create**:

    - **Name**: `nginx-plus-http-fw-rule`
    - **Description**: `Allow access to ports 80, 8080, and 443 on all NGINX Plus instances`
    - **Targets**: specified target tags, with the tag `nginx-plus-http-fw-rule`
    - **Source IPv4 ranges**: `0.0.0.0/0`, or the range of your network to restrict access
    - **Protocols and ports**: TCP ports `80`, `8080`, and `443`

    <img src="/nginx/images/gce-create-firewall-rule.png" alt="The page for creating a firewall rule in the Google Cloud console" style="border:2px solid #666666; padding:2px; margin:2px;" />

    If you restrict the source range, also allow the IP ranges of the Google Cloud health checks. Otherwise, the health checks fail, and Google Cloud replaces healthy instances. For the ranges, see [Probe IP ranges and firewall rules](https://cloud.google.com/load-balancing/docs/health-check-concepts#ip-ranges).

---

<span id="source"></span><span id="source-vm"></span><span id="task-2-creating-source-instances"></span><span id="creating-source-instances-from-vm-images"></span>

## Task 2: Create source instances

Create three source instances: one NGINX Plus load balancer and two NGINX Plus PHP application servers. Later tasks turn them into images and templates for the instance groups.

<span id="source-vm-app-1"></span><span id="creating-the-first-application-instance-from-a-vm-image"></span>

### Create the first application instance

1. Open **Compute Engine** > **VM instances**, and select **Create instance**.

1. Enter these values:

    - **Name**: `nginx-plus-app-1`
    - **Zone**: a zone near you, for example `us-west1-a`
    - **Machine type**: a size for your traffic. A small type, such as `e2-micro`, is enough for testing.
    - **Boot disk**: select **Change**, select **Ubuntu 24.04 LTS** or another [operating system that NGINX Plus supports]({{< ref "/nginx/technical-specs.md" >}}), and select **Select**.

    <img src="/nginx/images/gce-ubuntu-instance-boot-disk.png" alt="The Boot disk page, where you select the operating system image" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-ubuntu-instance-create.png" alt="The Create an instance page with the values for nginx-plus-app-1" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. In the advanced options, enter these values:

    - **Description**: `NGINX Plus app-1 Image`
    - **Network tags**: `nginx-plus-http-fw-rule`. This tag applies the firewall rule from Task 1. Leave the **Allow HTTP traffic** and **Allow HTTPS traffic** checkboxes clear.
    - **Boot disk deletion**: clear **Delete boot disk when instance is deleted**. Task 3 creates an image from this disk.
    - **External IPv4 address**: **Ephemeral**, and IP forwarding off (the defaults).
    - **SSH keys**: optionally, add your own public key.

    <img src="/nginx/images/gce-ubuntu-instance-management.png" alt="The management settings of the new instance" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-ubuntu-instance-disks.png" alt="The disk settings, with the checkbox to delete the boot disk cleared" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-ubuntu-instance-networking.png" alt="The networking settings of the new instance" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-ubuntu-instance-ssh-keys.png" alt="The SSH keys settings of the new instance" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Create**. Wait until the **VM instances** page shows the instance as running. It can take several minutes.

    <img src="/nginx/images/gce-ubuntu-instance-summary.png" alt="The VM instances page with the new instance" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="source-vm-app-2"></span><span id="source-vm-lb"></span><span id="creating-the-second-application-instance-from-a-vm-image"></span><span id="creating-the-load-balancing-instance-from-a-vm-image"></span>

### Create the second application instance and the load balancing instance

Repeat [Create the first application instance](#source-vm-app-1) twice, with these values:

- Second application instance: name `nginx-plus-app-2`, description `NGINX Plus app-2 Image`
- Load balancing instance: name `nginx-plus-lb`, description `NGINX Plus Load Balancing Image`

<span id="source-vm-php"></span><span id="configuring-php-and-fastcgi-on-the-vm-based-instances"></span>

### Set up PHP-FPM on the application instances

Do these steps on `nginx-plus-app-1` and `nginx-plus-app-2`.

1. Connect to the instance over SSH. On the **VM instances** page, select **SSH** in the row of the instance.

    <img src="/nginx/images/gce-ubuntu-instance-ssh.png" alt="The SSH options in the row of an instance on the VM instances page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Install PHP-FPM:

    ```shell
    sudo apt update
    sudo apt install php-fpm
    ```

1. Make PHP-FPM listen on a local TCP port instead of a UNIX socket, because the NGINX configuration of the application connects to `127.0.0.1:9000`. In `/etc/php/8.3/fpm/pool.d/www.conf`, replace the line `listen = /run/php/php8.3-fpm.sock` with these lines:

    ```none
    listen = 127.0.0.1:9000
    listen.allowed_clients = 127.0.0.1
    ```

    PHP 8.3 is the version of Ubuntu 24.04. For another version, `php -v` shows it, and the path and service name change accordingly.

1. Restart PHP-FPM:

    ```shell
    sudo systemctl restart php8.3-fpm
    ```

<span id="source-vm-nginx-plus"></span><span id="installing-and-configuring-nginx-plus-on-the-vm-based-instances"></span>

### Install and set up NGINX Plus on the source instances

The configuration and content files of this deployment are in the [NGINX-Demos repository](https://github.com/nginxinc/NGINX-Demos/tree/master/gce-nginx-plus-deployment-guide-files) on GitHub. Do these steps on all three instances.

1. Install NGINX Plus, as described in [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

1. Clone the repository, and copy its content files to `/usr/share/nginx`:

    ```shell
    git clone https://github.com/nginxinc/NGINX-Demos.git
    sudo cp -r NGINX-Demos/gce-nginx-plus-deployment-guide-files/usr_share_nginx/* /usr/share/nginx/
    ```

1. Copy the configuration file for the role of the instance to `/etc/nginx/conf.d`:

    - On `nginx-plus-app-1` and `nginx-plus-app-2`:

        ```shell
        sudo cp NGINX-Demos/gce-nginx-plus-deployment-guide-files/etc_nginx_conf.d/gce-all-active-app.conf /etc/nginx/conf.d/
        ```

    - On `nginx-plus-lb`:

        ```shell
        sudo cp NGINX-Demos/gce-nginx-plus-deployment-guide-files/etc_nginx_conf.d/gce-all-active-lb.conf /etc/nginx/conf.d/
        ```

    To download one file without cloning the repository, use its raw URL, for example `https://raw.githubusercontent.com/nginxinc/NGINX-Demos/master/gce-nginx-plus-deployment-guide-files/etc_nginx_conf.d/gce-all-active-app.conf`. The `github.com/.../blob/...` address of a file returns a web page, not the file.

1. On `nginx-plus-lb`, edit `/etc/nginx/conf.d/gce-all-active-lb.conf`. In the `upstream` block, set the `server` directives to the internal IP addresses of `nginx-plus-app-1` and `nginx-plus-app-2`. The **Internal IP** column of the **VM instances** page shows them.

    ```nginx
    upstream upstream_app_pool {
        server <NGINX_PLUS_APP_1_IP>;
        server <NGINX_PLUS_APP_2_IP>;
        zone upstream-apps 64k;
        sticky cookie GCPPersist expires=300;
    }
    ```

    The [`sticky cookie`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky) directive keeps each client on one application instance.

1. Rename the default configuration file, so NGINX Plus doesn't load it:

    ```shell
    sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
    ```

1. Turn on the NGINX Plus API and dashboard on port 8080. Create the file `/etc/nginx/conf.d/dashboard.conf` with this content. Replace `<ADMIN_NETWORK>` with the address range of the computers that open the dashboard, such as `203.0.113.0/24`.

    ```nginx
    server {
        listen 8080;
        status_zone status-page;

        location /api {
            api;
            allow <ADMIN_NETWORK>;
            deny  all;
        }

        location = /dashboard.html {
            root /usr/share/nginx/html;
        }

        location = / {
            return 301 /dashboard.html;
        }
    }
    ```

    The `status.conf` file of the repository uses the Extended Status module, which NGINX Plus doesn't include since [R16]({{< ref "/nginx/releases.md#r16" >}}). Don't copy it. For more about the API and the dashboard, see [Live activity monitoring]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}).

1. Test the configuration, and reload it:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

1. Check the instance. Replace `<EXTERNAL_IP>` with the external IP address of the instance, which the **VM instances** page shows:

    ```shell
    curl http://<EXTERNAL_IP>/
    ```

    The dashboard is at `http://<EXTERNAL_IP>:8080/dashboard.html`.

<span id="source-prebuilt"></span><span id="source-prebuilt-app-1"></span><span id="source-prebuilt-app-2"></span><span id="source-prebuilt-lb"></span><span id="source-prebuilt-php"></span><span id="source-prebuilt-nginx-plus"></span><span id="creating-source-instances-from-prebuilt-nginxplus-images"></span><span id="creating-the-first-application-instance-from-a-prebuilt-image"></span><span id="creating-the-second-application-instance-from-a-prebuilt-image"></span><span id="creating-the-load-balancing-instance-from-a-prebuilt-image"></span><span id="configuring-php-and-fastcgi-on-the-prebuilt-based-instances"></span><span id="configuring-nginx-plus-on-the-prebuilt-based-instances"></span>

### Use the NGINX Plus image from Google Cloud Marketplace instead

Instead of installing NGINX Plus, you can create the three source instances from the NGINX Plus image in Google Cloud Marketplace, as described in [Install NGINX Plus on the Google Cloud Platform]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-google-cloud-platform.md" >}}). Give them the names, descriptions, network tag, and disk setting of this task. Then set up PHP-FPM, and do the steps of the previous section except installing NGINX Plus. Use the PHP version of the image's operating system in the paths and commands.

<img src="/nginx/images/nginx-plus-in-gcp-marketplace.png" alt="The NGINX Plus image in Google Cloud Marketplace" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="gold"></span><span id="task-3-creating-gold-images"></span>

## Task 3: Create gold images

Gold images are the base images that Google Cloud copies when it creates instances for the instance groups. Create them from the disks of the source instances.

1. Open **Compute Engine** > **VM instances**, and select the three source instances.

1. Select **Stop**, and then **Delete**. If the confirmation says that it deletes a boot disk, cancel. Edit that instance, clear **Delete boot disk when instance is deleted**, save, and delete the instance again.

    <img src="/nginx/images/gce-vm-instances-toolbar.png" alt="The toolbar of the VM instances page, with the Stop and Delete buttons" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Open **Compute Engine** > **Images**, and select **Create image**.

1. Enter these values, and select **Create**:

    - **Name**: `nginx-plus-app-1-image`
    - **Description**: `NGINX Plus Application 1 Gold Image`
    - **Source**: **Disk**
    - **Source disk**: `nginx-plus-app-1`

1. Create two more images the same way:

    - `nginx-plus-app-2-image`, with the description `NGINX Plus Application 2 Gold Image` and the source disk `nginx-plus-app-2`
    - `nginx-plus-lb-image`, with the description `NGINX Plus LB Gold Image` and the source disk `nginx-plus-lb`

1. Check that the three images appear on the **Images** page.

---

<span id="templates"></span><span id="task-4-creating-instance-templates"></span>

## Task 4: Create instance templates

The instance groups create their instances from instance templates.

<span id="templates-app-1"></span><span id="creating-the-first-application-instance-template"></span>

### Create the first application instance template

1. Open **Compute Engine** > **Instance templates**, and select **Create instance template**.

1. Enter these values:

    - **Name**: `nginx-plus-app-1-instance-template`
    - **Machine type**: the same type as for the source instances
    - **Boot disk**: select **Change**. On the **Custom images** tab, select the project `NGINX Plus All-Active-LB` and the image `nginx-plus-app-1-image`, and select **Select**.

    <img src="/nginx/images/gce-instance-template-boot-disk.png" alt="The Boot disk page with the custom image of the template" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-create-instance-template.png" alt="The Create an instance template page" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. In the advanced options, enter these values:

    - **Description**: `NGINX Plus app-1 Instance Template`
    - **Network tags**: `nginx-plus-http-fw-rule`
    - **Boot disk deletion**: keep **Delete boot disk when instance is deleted** selected. Instances from the template are temporary copies of the gold image, so their disks don't need to stay.
    - **External IPv4 address**: **Ephemeral** (the default).
    - **SSH keys**: optionally, add your own public key.

    <img src="/nginx/images/gce-instance-template-management.png" alt="The management settings of the instance template" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-instance-template-disks.png" alt="The disk settings of the instance template" style="border:2px solid #666666; padding:2px; margin:2px;" />

    <img src="/nginx/images/gce-instance-template-networking.png" alt="The networking settings of the instance template" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Create**.

<span id="templates-app-2"></span><span id="templates-lb"></span><span id="creating-the-second-application-instance-template"></span><span id="creating-the-load-balancing-instance-template"></span>

### Create the second application template and the load balancing template

Repeat [Create the first application instance template](#templates-app-1) twice, with these values:

- `nginx-plus-app-2-instance-template`, with the image `nginx-plus-app-2-image` and the description `NGINX Plus app-2 Instance Template`
- `nginx-plus-lb-instance-template`, with the image `nginx-plus-lb-image` and the description `NGINX Plus Load-Balancing Instance Template`

---

<span id="health-checks"></span><span id="task-5-creating-image-health-checks"></span>

## Task 5: Create a health check

The instance groups use this HTTP health check to find instances that don't work, and to replace them.

1. Open **Compute Engine** > **Health checks**, and select **Create health check**.

1. Enter these values, and select **Create**:

    - **Name**: `nginx-plus-http-health-check`
    - **Description**: `Basic HTTP health check to monitor NGINX Plus instances`
    - **Protocol**: **HTTP**
    - **Port**: `80`
    - **Request path**: `/status-old.html`. The content files of the repository include this page for the application and load balancing instances.
    - **Check interval**: 10 seconds
    - **Timeout**: 10 seconds
    - **Healthy threshold**: 2 consecutive successes
    - **Unhealthy threshold**: 10 consecutive failures

    <img src="/nginx/images/gce-create-health-check.png" alt="The page for creating a health check" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="groups"></span><span id="task-6-creating-instance-groups"></span>

## Task 6: Create instance groups

<span id="groups-app-1"></span><span id="creating-the-first-application-instance-group"></span>

### Create the first application instance group

1. Open **Compute Engine** > **Instance groups**, and select **Create instance group**.

1. Enter these values, and select **Create**:

    - **Name**: `nginx-plus-app-1-instance-group`
    - **Description**: `Instance group to host NGINX Plus app-1 instances`
    - **Instance template**: `nginx-plus-app-1-instance-template`
    - **Location**: **Single zone**, in the zone of the source instances
    - **Autoscaling**: off, with **2** instances
    - **Autohealing**: the health check `nginx-plus-http-health-check`, with an initial delay of 300 seconds

    <img src="/nginx/images/gce-create-instance-group.png" alt="The page for creating an instance group" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="groups-app-2"></span><span id="groups-lb"></span><span id="creating-the-second-application-instance-group"></span><span id="creating-the-load-balancing-instance-group"></span>

### Create the second application group and the load balancing group

Repeat [Create the first application instance group](#groups-app-1) twice, with these values:

- `nginx-plus-app-2-instance-group`, with the template `nginx-plus-app-2-instance-template` and the description `Instance group to host NGINX Plus app-2 instances`
- `nginx-plus-lb-instance-group`, with the template `nginx-plus-lb-instance-template` and the description `Instance group to host NGINX Plus load balancing instances`

<span id="update-test"></span><span id="updating-and-testing-the-nginxplus-configuration"></span>

### Update and test the NGINX Plus configuration

The load balancing instances must list the four application instances. Do these steps on both instances of `nginx-plus-lb-instance-group`.

1. Connect to the instance over SSH.

1. In `/etc/nginx/conf.d/gce-all-active-lb.conf`, set the `server` directives of the `upstream` block to the internal IP addresses of the four application instances, from both application groups:

    ```nginx
    upstream upstream_app_pool {
        zone upstream-apps 64k;
        server 10.10.10.1;
        server 10.10.10.2;
        server 10.10.10.3;
        server 10.10.10.4;
        sticky cookie GCPPersist expires=300;
    }
    ```

1. Test the configuration, and reload it:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

1. Open the dashboard of the instance, at `http://<LB_EXTERNAL_IP>:8080/dashboard.html`. The four application instances appear as healthy upstream servers.

1. On another computer, request the page repeatedly. Replace `<LB_EXTERNAL_IP>` with the external IP address of the load balancing instance:

    ```shell
    while true; do curl -s http://<LB_EXTERNAL_IP>/ | grep Server: ; done
    ```

    The `Server` field of each application instance appears in turn. `curl` sends no cookie, so `sticky cookie` doesn't keep it on one instance.

    {{< call-out class="caution" >}} The gold image of the load balancing instances still lists the addresses of the source application instances, which you deleted in Task 3. If the group replaces a load balancing instance, the new instance starts from that image, and returns `502 Bad Gateway` errors. After you update the configuration, create a new gold image and instance template from an updated load balancing instance. Or list the application servers by DNS name, with the `resolve` parameter of the `server` directive, so that NGINX Plus finds their current addresses. {{< /call-out >}}

---

<span id="static-ip"></span><span id="task-7-configuring-gce-network-load-balancer"></span>

## Task 7: Set up the Google Cloud network load balancer

The network load balancer spreads incoming client connections across the NGINX Plus load balancing instances. This task creates a backend service-based external passthrough Network Load Balancer. The older target pool-based load balancers can't be created in the Google Cloud console, and need legacy health checks. For details, see [Backend service-based external passthrough Network Load Balancer overview](https://cloud.google.com/load-balancing/docs/network/networklb-backend-service).

1. Open **VPC network** > **IP addresses**, and select **Reserve external static IP address**. Enter these values, and select **Reserve**:

    - **Name**: `nginx-plus-network-lb-static-ip`
    - **Description**: `Static IP address for Network LB frontend to NGINX Plus LB instances`
    - **Type**: **Regional**, in the region of the source instances

    <img src="/nginx/images/gce-reserve-static-address.png" alt="The page for reserving a static external IP address" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Open **Network services** > **Load balancing**, and select **Create load balancer**. Select **Network Load Balancer (TCP/UDP/SSL)**, **Passthrough load balancer**, and **Public facing (external)**, and then select **Configure**.

1. Enter the name `nginx-plus-network-lb-frontend`, and the region of the static IP address, for example `us-west1`.

1. In **Backend configuration**:

    - **Health check**: select **Create a health check**, and create a regional HTTP health check on port `80` with the request path `/status-old.html`. Network Load Balancers of this type use regional health checks, so they can't use the health check from Task 5.
    - **Backends**: the instance group `nginx-plus-lb-instance-group`.
    - **Session affinity**: **Client IP**.

    <img src="/nginx/images/gce-backend-configuration.png" alt="The backend configuration of the network load balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. In **Frontend configuration**, select the IP version **IPv4** and the static IP address `nginx-plus-network-lb-static-ip`. For **Ports**, enter `80`, `8080`, and `443`.

    <img src="/nginx/images/gce-frontend-configuration.png" alt="The frontend configuration of the network load balancer" style="border:2px solid #666666; padding:2px; margin:2px;" />

1. Select **Create**.

For the details of each step, see [Set up a backend service-based external passthrough Network Load Balancer](https://cloud.google.com/load-balancing/docs/network/setting-up-network-backend-service).

---

<span id="testing"></span><span id="task-8-testing-the-all-active-load-balancing-deployment"></span>

## Task 8: Test the all-active deployment

1. On another computer, request the page repeatedly through the network load balancer. Replace `<GCE_LB_EXTERNAL_IP>` with its static IP address:

    ```shell
    while true; do curl -s http://<GCE_LB_EXTERNAL_IP>/ | grep Server: ; done
    ```

    The `Server` field of each application instance appears in turn. You can also open `http://<GCE_LB_EXTERNAL_IP>/` in a browser.

1. To check the high availability, take one load balancing instance offline. Connect to it over SSH, and block port 80:

    ```shell
    sudo iptables -A INPUT -p tcp --destination-port 80 -j DROP
    ```

1. Check that the requests still succeed through the other load balancing instance. The network load balancer notices the failed instance only after its health check fails often enough. That takes the check interval multiplied by the unhealthy threshold, for example 100 seconds with 10-second checks and 10 failures.

1. Remove the rule, to bring the instance back:

    ```shell
    sudo iptables -D INPUT -p tcp --destination-port 80 -j DROP
    ```

    If the instance stays offline longer than the health check allows, the instance group replaces it with a new instance.

---

## Troubleshooting

### nginx -t reports an unknown directive "status"

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] unknown directive "status"`.

**Cause**: `/etc/nginx/conf.d` contains the `status.conf` file of the repository. It uses the Extended Status module, which NGINX Plus doesn't include since R16.

**Fix**: Delete `status.conf`, and use `dashboard.conf`, as described in [Install and set up NGINX Plus on the source instances](#source-vm-nginx-plus).

---

### nginx -t reports an unknown directive "<!DOCTYPE"

**Symptom**: After you download a configuration file, `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "<!DOCTYPE" in /etc/nginx/conf.d/gce-all-active-app.conf:28`.

**Cause**: You downloaded the `github.com/.../blob/...` address of the file, which returns an HTML page.

**Fix**: Download the file from its `raw.githubusercontent.com` address, or copy it from a clone of the repository.

---

### The instance groups keep replacing instances

**Symptom**: Instances in the groups are deleted and created again, although NGINX Plus works on them.

**Cause**: The health check fails, because the firewall rule doesn't allow the IP ranges of the Google Cloud health checks, or `/status-old.html` is missing.

**Fix**: Allow the health check ranges in the firewall rule, and check that `http://<INSTANCE_IP>/status-old.html` answers with `200`.

---

## References

- [Live activity monitoring]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}})
- [HTTP load balancing]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md" >}})
- [The gce-nginx-plus-deployment-guide-files in the NGINX-Demos repository](https://github.com/nginxinc/NGINX-Demos/tree/master/gce-nginx-plus-deployment-guide-files)
- [Health checks overview](https://cloud.google.com/load-balancing/docs/health-check-concepts)

---

## Revision history

- Version 4 (November 2026): Rewritten to the how-to template. Uses raw file downloads, the NGINX Plus API dashboard instead of the removed Extended Status module, current PHP-FPM paths, and an `iptables` rule that is deleted on its own.
- Version 3 (July 2018): Updates for Google Cloud Platform Marketplace
- Version 2 (April 2018): Standardized information about root privilege and links to directive documentation
- Version 1 (November 2016): Initial version (NGINX Plus R11)
