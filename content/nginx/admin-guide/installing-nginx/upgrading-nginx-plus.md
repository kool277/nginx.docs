---
title: Upgrading NGINX Plus
description: Upgrade F5 NGINX Plus and its dynamic modules to a newer release with your package manager, including the license file that R33 and later need.
toc: true
weight: 150
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "upgrade nginx plus, nginx-plus package, license.jwt, R33, usage reporting, dynamic modules upgrade"
f5-summary: >
  This page shows how to upgrade F5 NGINX Plus and its dynamic modules to a newer release from the NGINX Plus repository.
  Upgrades bring new features, bug fixes, and security fixes.
  It also covers the extra steps for upgrades from R32 and earlier, which need a license file, and from R24 and earlier, which need new repositories.
f5-audience: operator
---

## Overview {#upgrade}

You upgrade F5 NGINX Plus with the package manager of your operating system, from the same NGINX Plus repository that you installed it from. Upgrades bring new features, bug fixes, and security fixes. Which releases get fixes depends on the release type. Only the latest Continuous Release (CR) is supported. A Long-Term Support (LTS) release gets security fixes for up to three years. For details, see [Choose a release type]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#choose-a-release-type" >}}).

---

<span id="prerequisites"></span>

## Before you begin

Before you upgrade, make sure that:

- **The repository is set up**: Your server gets packages from the NGINX Plus repository. The certificate `/etc/ssl/nginx/nginx-repo.crt`, the key `/etc/ssl/nginx/nginx-repo.key`, and the repository file are in place. If they aren't, follow the installation steps for your operating system: [Amazon Linux 2023]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_amazon2023" >}}), [Amazon Linux 2]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_amazon2" >}}), [RHEL 8]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_rhel8" >}}), [RHEL 9]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_rhel" >}}), [RHEL 10]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_rhel10" >}}), [Debian]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_debian" >}}), [Ubuntu]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_debian_ubuntu" >}}), [FreeBSD]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_freebsd" >}}), [SLES]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_suse" >}}), or [Alpine Linux]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_alpine" >}}).
- **Your subscription is active**: You can check it in the [MyF5 Customer Portal](https://account.f5.com/myf5).
- **License reporting is set up**: NGINX Plus R33 and later need a license file and [license reporting]({{< ref "/solutions/about-subscription-licenses/getting-started.md" >}}). If you upgrade from R32 or earlier, add the license file before you upgrade, as described in [Upgrade from R32 or earlier](#nginx-plus-r32-and-earlier).
- **Upgrades from R24 or earlier**: Set up the new repositories first, as described in [Upgrade from R24 or earlier](#nginx-plus-r24-and-earlier).

---

<span id="upgrade-steps"></span>

## Upgrade NGINX Plus

1. Back up the configuration and the log files. The commands add the date to the names of the backup directories:

    - On Linux:

        ```shell
        sudo cp -a /etc/nginx /etc/nginx-plus-backup-$(date +%F)
        sudo cp -a /var/log/nginx /var/log/nginx-plus-backup-$(date +%F)
        ```

    - On FreeBSD:

        ```shell
        sudo cp -a /usr/local/etc/nginx /usr/local/etc/nginx-plus-backup-$(date +%F)
        sudo cp -a /var/log/nginx /var/log/nginx-plus-backup-$(date +%F)
        ```

1. Upgrade the `nginx-plus` package to the newest version in your repository:

    - RHEL and its derivatives, and Amazon Linux:

        ```shell
        sudo yum upgrade nginx-plus
        ```

    - Debian and Ubuntu:

        ```shell
        sudo apt update
        sudo apt install nginx-plus
        ```

    - SLES:

        ```shell
        sudo zypper refresh
        sudo zypper update nginx-plus
        ```

    - Alpine Linux:

        ```shell
        sudo apk update
        sudo apk upgrade nginx-plus
        ```

    - FreeBSD:

        ```shell
        sudo pkg upgrade nginx-plus
        ```

1. [Verify the upgrade](#verify).

---

## Upgrade notes

### NGINX Plus R24 and earlier {#nginx-plus-r24-and-earlier}

Since [Release 24]({{< ref "nginx/releases.md#r24" >}}) (R24), NGINX Plus has separate repositories for each operating system and subscription. Before you upgrade from R24 or earlier, set up the repository for your operating system again. Follow the installation steps for [Amazon Linux 2023]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_amazon2023" >}}), [Amazon Linux 2]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_amazon2" >}}), [RHEL 8]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_rhel8" >}}), [RHEL 9]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_rhel" >}}), [Debian]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_debian" >}}), [Ubuntu]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_debian_ubuntu" >}}), [FreeBSD]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_freebsd" >}}), or [SLES]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_suse" >}}).

### NGINX Plus R32 and earlier {#nginx-plus-r32-and-earlier}

Since [NGINX Plus Release 33]({{< ref "nginx/releases.md#r33" >}}), every NGINX Plus instance needs a JWT license file. For more information, see [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}}).

Before you upgrade from R32 or earlier:

1. Get the JWT file of your NGINX Plus subscription from the [MyF5 Customer Portal](https://account.f5.com/myf5):

    {{< include "licensing-and-reporting/download-jwt-from-myf5.md" >}}

1. Create the directory for the license file: `/etc/nginx/` on Linux, or `/usr/local/etc/nginx/` on FreeBSD.

    - On Linux:

        ```shell
        sudo mkdir -p /etc/nginx
        ```

    - On FreeBSD:

        ```shell
        sudo mkdir -p /usr/local/etc/nginx
        ```

    {{< call-out class="note" title="Using custom paths" >}}{{< include "licensing-and-reporting/custom-paths-jwt.md" >}}{{< /call-out >}}

1. Copy the JWT file to that directory, and name it `license.jwt`. Replace `<DOWNLOADED_FILE_NAME>` with the name of the downloaded file without the `.jwt` extension.

    - On Linux:

        ```shell
        sudo cp <DOWNLOADED_FILE_NAME>.jwt /etc/nginx/license.jwt
        ```

    - On FreeBSD:

        ```shell
        sudo cp <DOWNLOADED_FILE_NAME>.jwt /usr/local/etc/nginx/license.jwt
        ```

1. [Upgrade NGINX Plus](#upgrade-nginx-plus).

1. Set up usage reporting, if your installation needs it. By default, NGINX Plus needs no extra configuration. You need it if NGINX Plus has no internet access, or if the license file isn't in the default directory.

    - **Without internet access**, NGINX Plus reports usage to NGINX Instance Manager 2.18 or later. In the `nginx.conf` file, add the [`mgmt`](https://nginx.org/en/docs/ngx_mgmt_module.html#mgmt) block, which holds the licensing and usage reporting configuration. In it, the [`usage_report`](https://nginx.org/en/docs/ngx_mgmt_module.html#usage_report) directive sets the domain name or IP address of NGINX Instance Manager. Set to `off`, the [`enforce_initial_report`](https://nginx.org/en/docs/ngx_mgmt_module.html#enforce_initial_report) directive turns on a 180-day grace period for the initial usage report. If the F5 licensing endpoint doesn't receive the initial report during this period, NGINX Plus stops processing traffic.

        ```nginx
        mgmt {
            usage_report endpoint=<NIM_FQDN>;
            enforce_initial_report off;
        }
        ```

        {{< include "nginx-plus/install/nim-disconnected-report-usage.md" >}}

    - **With the license file in another directory** than `/etc/nginx/` on Linux or `/usr/local/etc/nginx/` on FreeBSD, set its path with the [`license_token`](https://nginx.org/en/docs/ngx_mgmt_module.html#license_token) directive:

        ```nginx
        mgmt {
            license_token custom/file/path/license.jwt;
        }
        ```

---

## Upgrade NGINX Plus modules {#upgrade_modules}

How you upgrade a module depends on where it came from:

- [NGINX-authored]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}) and [NGINX-certified community]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-certified-community-dynamic-modules" >}}) modules are upgraded automatically together with NGINX Plus.

    {{< call-out class="note" >}} On FreeBSD, upgrade each NGINX-authored and NGINX-certified module separately with the FreeBSD package manager. {{< /call-out >}}

- [Community]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#community-dynamic-modules" >}}) modules must be compiled again with the NGINX Open Source version of the new NGINX Plus release. See [Compile a community dynamic module]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_modules_oss" >}}).

---

## Verify the upgrade {#verify}

1. Check the NGINX Plus version:

    ```shell
    nginx -v
    ```

    The output shows the new release, for example:

    ```none
    nginx version: nginx/1.29.8 (nginx-plus-r37.0.0)
    ```

1. Test the configuration:

    ```shell
    sudo nginx -t
    ```

    The output ends with `test is successful`.

1. Check the end of the error log for problems:

    ```shell
    sudo tail /var/log/nginx/error.log
    ```

---

## References

- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}})
- [Installing NGINX Plus LTS]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-lts.md" >}})
- [NGINX Plus Releases]({{< ref "nginx/releases.md" >}})
- [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}})
