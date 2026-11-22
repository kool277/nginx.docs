---
title: Installing NGINX Plus LTS
description: Install an F5 NGINX Plus Long-Term Support (LTS) release by pinning the NGINX Plus repository to the current LTS or to the LTS track.
toc: true
weight: 120
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "nginx plus lts, long-term support, pin repository, lts track, nginx-plus package, license.jwt"
f5-summary: >
  This guide shows how to install an F5 NGINX Plus Long-Term Support (LTS) release, which gets only security fixes for up to three years.
  You point the NGINX Plus repository to the current LTS, or to the LTS track that moves to each new LTS, instead of the default Continuous Releases.
  It covers each supported operating system.
f5-audience: operator
---

## Overview

An NGINX Plus Long-Term Support (LTS) release gets security fixes and CVE mitigations for up to three years, but no new features. To install one, you point the NGINX Plus repository to an LTS URL instead of the default URL, which gives you Continuous Releases (CR). When you finish this guide, an NGINX Plus LTS release runs with its license.

---

<span id="prereq"></span>

## Before you begin

Before you begin, make sure you have:

- **MyF5 access**: You need credentials for the [MyF5 Customer Portal](https://account.f5.com/myf5). They're in the email from F5, Inc.
- **A subscription**: You need an active NGINX Plus subscription. You can check it in the [MyF5 Customer Portal](https://account.f5.com/myf5).
- **A supported operating system**: Check your operating system and architecture in the [Technical Specifications]({{< ref "nginx/technical-specs.md" >}}).
- **Root privileges**: The commands use `sudo`. Without root access, see [Install without root privileges]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#unpriv_install" >}}). Its script installs Continuous Releases by default. To use the LTS repository, set `REPOPREFIX=https://pkgs.nginx.com/plus/LTS` before you run the script.
- **Internet access**: The server must reach `pkgs.nginx.com` and `cs.nginx.com` over HTTPS. Without internet access, see [Install without internet access]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#offline_install" >}}).

If you already have NGINX Plus installed, back up its configuration and log files first. For details, see [Upgrading NGINX Plus]({{< ref "nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md" >}}).

---

## Choose an LTS option {#repo-options}

Since May 13, 2026, NGINX Plus is published in two release types: Long-Term Support (LTS) and Continuous Release (CR). An LTS release:

- **Cadence**: Comes out once a year.
- **Patching model**: Gets security and CVE fixes only, with no feature changes.
- **Support window**: Is supported for up to three years.
- **Concurrency**: Is one of up to three LTS versions that are supported at the same time.
- **Version format**: Has `0` as the second number, for example `PLS.37.0.0.1`. LTS updates increase the third number, for example `PLS.37.0.1.1`.

CRs come out several times during an annual LTS cycle, with the latest features and performance improvements. The CR cycle ends when a new LTS comes out.

By default, the NGINX Plus repositories give you [Continuous Releases]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}). For LTS, you replace the repository URL with one of these, in the steps for your operating system:

- **Pin to the current LTS only**: Get only security updates for this LTS, with no CRs and no move to the next LTS. This is supported for up to three years. The URL contains the release, for example `R37.0`.
- **Pin to the LTS track**: Move to the newest LTS every year, with no CRs. The URL contains `LTS`.

To get CRs instead, follow [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

---

## Get the subscription files {#common-steps}

On every operating system, first get the files of your subscription and put the certificate and key in place:

1. {{< include "licensing-and-reporting/download-jwt-crt-from-myf5.md" >}}

1. {{< include "nginx-plus/install/create-dir-for-crt-key.md" >}}

1. {{< include "nginx-plus/install/copy-crt-and-key.md" >}}

Keep the JWT file. You copy it to NGINX Plus after you install the package.

---

## Install NGINX Plus LTS {#install}

Follow the steps for your operating system, and then [start NGINX Plus](#start).

### Amazon Linux 2023 {#install_amazon2023}

1. Install the `ca-certificates` dependency:

    ```shell
    sudo dnf install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the [plus-amazonlinux2023.repo](https://cs.nginx.com/static/files/plus-amazonlinux2023.repo) file to `/etc/yum.repos.d`:

    ```shell
    sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-amazonlinux2023.repo
    ```

1. Edit `/etc/yum.repos.d/plus-amazonlinux2023.repo`, and change the `baseurl` line for your [LTS option](#repo-options):

    - To pin to the current LTS:

        ```none
        baseurl=https://pkgs.nginx.com/plus/R37.0/amzn/2023/$basearch
        ```

    - To pin to the LTS track:

        ```none
        baseurl=https://pkgs.nginx.com/plus/LTS/amzn/2023/$basearch
        ```

1. {{< include "nginx-plus/install/install-nginx-plus-package-dnf.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### Amazon Linux 2 {#install_amazon2}

1. Install the `ca-certificates` dependency:

    ```shell
    sudo yum install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the [nginx-plus-amazon2.repo](https://cs.nginx.com/static/files/nginx-plus-amazon2.repo) file to `/etc/yum.repos.d`:

    ```shell
    sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/nginx-plus-amazon2.repo
    ```

1. Edit `/etc/yum.repos.d/nginx-plus-amazon2.repo`, and change the `baseurl` line for your [LTS option](#repo-options):

    - To pin to the current LTS:

        ```none
        baseurl=https://pkgs.nginx.com/plus/R37.0/amzn2/$releasever/$basearch
        ```

    - To pin to the LTS track:

        ```none
        baseurl=https://pkgs.nginx.com/plus/LTS/amzn2/$releasever/$basearch
        ```

1. {{< include "nginx-plus/install/install-nginx-plus-package-yum.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### RHEL and derivatives {#install_rhel}

These steps apply to Red Hat Enterprise Linux, Oracle Linux, AlmaLinux, and Rocky Linux, versions 8.1 and later, 9.7 and later, and 10 and later.

1. Install the `ca-certificates` dependency:

    ```shell
    sudo dnf install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the repository file for the major version of your operating system to `/etc/yum.repos.d`:

    - For version 8, the [plus-8.repo](https://cs.nginx.com/static/files/plus-8.repo) file:

        ```shell
        sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-8.repo
        ```

    - For version 9, the [plus-9.repo](https://cs.nginx.com/static/files/plus-9.repo) file:

        ```shell
        sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-9.repo
        ```

    - For version 10, the [plus-10.repo](https://cs.nginx.com/static/files/plus-10.repo) file:

        ```shell
        sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-10.repo
        ```

1. Edit the repository file that you downloaded, and change the `baseurl` line for your [LTS option](#repo-options):

    {{< table >}}

   | Version | Pin to the current LTS | Pin to the LTS track |
   |---|---|---|
   | 8 | `baseurl=https://pkgs.nginx.com/plus/R37.0/centos/8/$basearch/` | `baseurl=https://pkgs.nginx.com/plus/LTS/centos/8/$basearch/` |
   | 9 | `baseurl=https://pkgs.nginx.com/plus/R37.0/centos/9/$basearch/` | `baseurl=https://pkgs.nginx.com/plus/LTS/centos/9/$basearch/` |
   | 10 | `baseurl=https://pkgs.nginx.com/plus/R37.0/rhel/10/$basearch/` | `baseurl=https://pkgs.nginx.com/plus/LTS/rhel/10/$basearch/` |

    {{< /table >}}

1. Refresh the repository information:

    ```shell
    sudo dnf makecache
    ```

1. {{< include "nginx-plus/install/install-nginx-plus-package-dnf.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### Debian {#install_debian}

1. Install the prerequisites:

    ```shell
    sudo apt update
    sudo apt install apt-transport-https lsb-release ca-certificates wget gnupg2 debian-archive-keyring
    ```

1. Download the NGINX signing key and save it in the format that `apt` reads:

    ```shell
    wget -qO - https://cs.nginx.com/static/keys/nginx_signing.key \
        | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
    ```

1. Check the keys in the downloaded file:

    ```shell
    gpg --show-keys /usr/share/keyrings/nginx-archive-keyring.gpg
    ```

    The output lists three keys. Their fingerprints must be `8540A6F18833A80E9C1653A42FD21310B49F6B46`, `573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62`, and `9E9BE90EACBCDE69FE9B204CBCDCD8A38D88A2B3`. If they don't match, delete the file and stop.

1. Add the NGINX Plus repository for your [LTS option](#repo-options).

    - To pin to the current LTS:

        ```shell
        printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
        https://pkgs.nginx.com/plus/R37.0/debian `lsb_release -cs` nginx-plus\n" \
            | sudo tee /etc/apt/sources.list.d/nginx-plus.list
        ```

    - To pin to the LTS track:

        ```shell
        printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
        https://pkgs.nginx.com/plus/LTS/debian `lsb_release -cs` nginx-plus\n" \
            | sudo tee /etc/apt/sources.list.d/nginx-plus.list
        ```

1. Download the `apt` configuration for the repository to `/etc/apt/apt.conf.d`. It tells `apt` to use your certificate and key for `pkgs.nginx.com`:

    ```shell
    sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
    ```

1. Install the `nginx-plus` package. It upgrades an older NGINX Plus package, but it keeps a newer Continuous Release. To replace one, see [Switch from a Continuous Release](#switch-from-cr).

    ```shell
    sudo apt update
    sudo apt install -y nginx-plus
    ```

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### Ubuntu {#install_debian_ubuntu}

1. Install the prerequisites:

    ```shell
    sudo apt update
    sudo apt install apt-transport-https lsb-release ca-certificates wget gnupg2 ubuntu-keyring
    ```

1. Download the NGINX signing key and save it in the format that `apt` reads:

    ```shell
    wget -qO - https://cs.nginx.com/static/keys/nginx_signing.key \
        | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
    ```

1. Check the keys in the downloaded file:

    ```shell
    gpg --show-keys /usr/share/keyrings/nginx-archive-keyring.gpg
    ```

    The output lists three keys. Their fingerprints must be `8540A6F18833A80E9C1653A42FD21310B49F6B46`, `573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62`, and `9E9BE90EACBCDE69FE9B204CBCDCD8A38D88A2B3`. If they don't match, delete the file and stop.

1. Add the NGINX Plus repository for your [LTS option](#repo-options).

    - To pin to the current LTS:

        ```shell
        printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
        https://pkgs.nginx.com/plus/R37.0/ubuntu `lsb_release -cs` nginx-plus\n" \
            | sudo tee /etc/apt/sources.list.d/nginx-plus.list
        ```

    - To pin to the LTS track:

        ```shell
        printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
        https://pkgs.nginx.com/plus/LTS/ubuntu `lsb_release -cs` nginx-plus\n" \
            | sudo tee /etc/apt/sources.list.d/nginx-plus.list
        ```

1. Download the `apt` configuration for the repository to `/etc/apt/apt.conf.d`. It tells `apt` to use your certificate and key for `pkgs.nginx.com`:

    ```shell
    sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
    ```

1. Install the `nginx-plus` package. It upgrades an older NGINX Plus package, but it keeps a newer Continuous Release. To replace one, see [Switch from a Continuous Release](#switch-from-cr).

    ```shell
    sudo apt update
    sudo apt install -y nginx-plus
    ```

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### FreeBSD {#install_freebsd}

1. Install the `ca_root_nss` package:

    ```shell
    sudo pkg update
    sudo pkg install ca_root_nss
    ```

1. Download the [nginx-plus.conf](https://cs.nginx.com/static/files/nginx-plus.conf) repository file to `/etc/pkg/`:

    ```shell
    sudo fetch -o /etc/pkg/nginx-plus.conf https://cs.nginx.com/static/files/nginx-plus.conf
    ```

1. Edit `/etc/pkg/nginx-plus.conf`, and change the `URL` line for your [LTS option](#repo-options):

    - To pin to the current LTS:

        ```none
        URL: pkg+https://pkgs.nginx.com/plus/R37.0/freebsd/${ABI}/latest
        ```

    - To pin to the LTS track:

        ```none
        URL: pkg+https://pkgs.nginx.com/plus/LTS/freebsd/${ABI}/latest
        ```

1. Add these lines to the `/usr/local/etc/pkg.conf` file:

    ```none
    PKG_ENV: { SSL_NO_VERIFY_PEER: "1",
    SSL_CLIENT_CERT_FILE: "/etc/ssl/nginx/nginx-repo.crt",
    SSL_CLIENT_KEY_FILE: "/etc/ssl/nginx/nginx-repo.key" }
    ```

1. Install the `nginx-plus` package. It upgrades an older NGINX Plus package, but it keeps a newer Continuous Release. To replace one, see [Switch from a Continuous Release](#switch-from-cr).

    ```shell
    sudo pkg install nginx-plus
    ```

1. Copy the downloaded JWT file to the `/usr/local/etc/nginx` directory, and name it `license.jwt`:

    ```shell
    sudo cp <DOWNLOADED_FILE_NAME>.jwt /usr/local/etc/nginx/license.jwt
    ```

    Replace `<DOWNLOADED_FILE_NAME>` with the name of the downloaded file without the `.jwt` extension.

On FreeBSD, the configuration file is `/usr/local/etc/nginx/nginx.conf`.

### SLES {#install_suse}

1. Combine the certificate and the key into one file:

    ```shell
    sudo sh -c 'cat /etc/ssl/nginx/nginx-repo.crt /etc/ssl/nginx/nginx-repo.key > /etc/ssl/nginx/nginx-repo-bundle.crt'
    ```

1. Install the `ca-certificates` dependency:

    ```shell
    sudo zypper refresh
    sudo zypper install ca-certificates
    ```

1. Add the NGINX Plus repository for your SLES version and [LTS option](#repo-options). Replace `15` with `16` for SLES 16.

    - To pin to the current LTS:

        ```shell
        sudo zypper addrepo -G -t yum -c \
            "https://pkgs.nginx.com/plus/R37.0/sles/15?ssl_clientcert=/etc/ssl/nginx/nginx-repo-bundle.crt&ssl_verify=peer" \
            nginx-plus
        ```

    - To pin to the LTS track:

        ```shell
        sudo zypper addrepo -G -t yum -c \
            "https://pkgs.nginx.com/plus/LTS/sles/15?ssl_clientcert=/etc/ssl/nginx/nginx-repo-bundle.crt&ssl_verify=peer" \
            nginx-plus
        ```

1. Install the `nginx-plus` package. It upgrades an older NGINX Plus package, but it keeps a newer Continuous Release. To replace one, see [Switch from a Continuous Release](#switch-from-cr).

    ```shell
    sudo zypper install nginx-plus
    ```

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### Alpine Linux {#install_alpine}

1. Copy the certificate and the key to the locations that `apk` uses. These files must contain only this key and certificate, because Alpine Linux doesn't support separate client certificates for different repositories.

    ```shell
    sudo cp /etc/ssl/nginx/nginx-repo.key /etc/apk/cert.key
    sudo cp /etc/ssl/nginx/nginx-repo.crt /etc/apk/cert.pem
    ```

1. Download the NGINX signing public key to the `/etc/apk/keys` directory:

    ```shell
    sudo wget -O /etc/apk/keys/nginx_signing.rsa.pub https://cs.nginx.com/static/keys/nginx_signing.rsa.pub
    ```

    To check the key, install OpenSSL with `sudo apk add openssl`, and run `openssl rsa -pubin -in /etc/apk/keys/nginx_signing.rsa.pub -text -noout`. The modulus must start with `00:fe:14:f6:0a:1a:b8:86:19` and end with `f4:22:d7:f2:5f:ab:6d`. If it doesn't, delete the file and stop.

1. Add the NGINX Plus repository for your [LTS option](#repo-options) to the `/etc/apk/repositories` file.

    - To pin to the current LTS:

        ```shell
        printf "https://pkgs.nginx.com/plus/R37.0/alpine/v`egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release`/main\n" \
            | sudo tee -a /etc/apk/repositories
        ```

    - To pin to the LTS track:

        ```shell
        printf "https://pkgs.nginx.com/plus/LTS/alpine/v`egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release`/main\n" \
            | sudo tee -a /etc/apk/repositories
        ```

1. Remove all community-supported NGINX packages. This also removes all their modules.

    ```shell
    sudo apk del -r nginx
    ```

1. Install the NGINX Plus package:

    ```shell
    sudo apk add nginx-plus
    ```

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

---

## Switch from a Continuous Release {#switch-from-cr}

If a Continuous Release is installed, the package manager keeps it, because it's newer than the LTS release. To switch, change the repository as described in [Install NGINX Plus LTS](#install), and then downgrade the packages. If you installed module packages, add them to the same command.

- Amazon Linux 2023, and RHEL and its derivatives:

    ```shell
    sudo dnf downgrade nginx-plus
    ```

- Amazon Linux 2:

    ```shell
    sudo yum downgrade nginx-plus
    ```

- Debian and Ubuntu: List the available versions, and then install the LTS version. Replace `<VERSION>` with a version from the list.

    ```shell
    apt list -a nginx-plus
    sudo apt install --allow-downgrades nginx-plus=<VERSION>
    ```

- SLES: List the available versions, and then install the LTS version. Replace `<VERSION>` with a version from the list.

    ```shell
    zypper search -s nginx-plus
    sudo zypper install --oldpackage nginx-plus=<VERSION>
    ```

- Alpine Linux: This command replaces each package with the version in your repositories. It also removes the version constraints in `/etc/apk/world`.

    ```shell
    sudo apk upgrade --available
    ```

- FreeBSD: Before you confirm, check that the list of changes shows the downgrade of `nginx-plus`.

    ```shell
    sudo pkg upgrade nginx-plus
    ```

Check the result with `nginx -v`, and then restart NGINX Plus as described in [Start NGINX Plus](#start).

---

## Start NGINX Plus {#start}

Start NGINX Plus, or restart it if it's already running, and set it to start when the system starts:

- **Linux with systemd**, which includes Amazon Linux, RHEL and its derivatives, Debian, Ubuntu, and SLES:

    ```shell
    sudo systemctl restart nginx
    sudo systemctl enable nginx
    ```

- **Alpine Linux and FreeBSD**: Use your system's service manager. To start NGINX Plus once from the command line, run `sudo nginx`.

Then [set up usage reporting](#usage-reporting).

---

## Set up usage reporting {#usage-reporting}

1. {{< include "nginx-plus/install/configure-usage-reporting.md" >}} On FreeBSD, the configuration file is `/usr/local/etc/nginx/nginx.conf`.

1. If you use [NGINX Instance Manager]({{< ref "/nim/" >}}), install NGINX Agent and turn it on. For details, see [Installation from NGINX Plus repository]({{< ref "/agent/installation-upgrade/installation-plus.md" >}}).

1. If you changed `nginx.conf`, reload NGINX Plus:

    ```shell
    sudo nginx -s reload
    ```

---

## Verify the installation {#verify}

1. Check the version:

    ```shell
    nginx -v
    ```

    For an LTS release, the second number of the NGINX Plus release is `0`, for example:

    ```none
    nginx version: nginx/1.29.8 (nginx-plus-r37.0.6)
    ```

1. Test the configuration:

    ```shell
    sudo nginx -t
    ```

    The output ends with `test is successful`.

---

## Troubleshooting

### nginx -v shows a Continuous Release

**Symptom**: The second number of the NGINX Plus release isn't `0`, for example `nginx-plus-r37.2.0`.

**Cause**: The repository still points to the default URL, which gives you Continuous Releases. Or a Continuous Release was installed before you changed the repository, and the package manager kept it.

**Fix**: Check that the repository URL contains `R37.0` or `LTS`, as in the steps for your operating system. Then downgrade the packages, as described in [Switch from a Continuous Release](#switch-from-cr).

---

### The package manager can't get packages from pkgs.nginx.com

**Symptom**: Updating the package list shows errors for `pkgs.nginx.com`, and installing `nginx-plus` then fails. For example, `apt` says `Unable to locate package nginx-plus`.

**Cause**: The package manager can't present the certificate and key of your subscription. For the meaning of the error messages, see [Troubleshooting]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#the-package-manager-cant-get-packages-from-pkgsnginxcom" >}}) in the installation guide for Continuous Releases.

**Fix**: Check that `/etc/ssl/nginx/nginx-repo.crt` and `/etc/ssl/nginx/nginx-repo.key` exist and that the package manager can read them. On Debian and Ubuntu, `apt` reads them as the `_apt` user, and needs `/etc/apt/apt.conf.d/90pkgs-nginx`. On RHEL-based systems, check the `sslclientcert` and `sslclientkey` lines in the `.repo` file.

---

## References {#upgrade}

- [Upgrading NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md" >}})
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}), for Continuous Releases, dynamic modules, and installations without root privileges or internet access
- [NGINX Plus Technical Specifications]({{< ref "nginx/technical-specs.md" >}})
- [NGINX Plus Releases]({{< ref "nginx/releases.md" >}})
- [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}})
