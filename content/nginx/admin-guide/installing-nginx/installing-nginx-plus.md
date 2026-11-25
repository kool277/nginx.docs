---
title: Installing NGINX Plus
description: Install F5 NGINX Plus and its dynamic modules from the NGINX Plus repository, with or without root access or an internet connection.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "install nginx plus, nginx-plus package, subscription certificate, license.jwt, continuous release, LTS, dynamic modules, unprivileged installation, offline installation"
f5-summary: >
  This guide shows how to install F5 NGINX Plus from the NGINX Plus repository with the certificate, key, and license file of your subscription.
  It covers each supported operating system, dynamic modules, installation without root access, and installation without internet access.
  It installs Continuous Releases; to install Long-Term Support releases instead, follow the LTS guide.
f5-audience: operator
---

## Overview

You install F5 NGINX Plus from the NGINX Plus repository with your package manager. The repository accepts only clients with the certificate and key of an NGINX Plus subscription. NGINX Plus also needs the license file of that subscription. When you finish this guide, NGINX Plus runs with its license, and you can add dynamic modules.

---

<span id="prereq"></span>

## Before you begin

Before you begin, make sure you have:

- **MyF5 access**: You need credentials for the [MyF5 Customer Portal](https://account.f5.com/myf5). They're in the email from F5, Inc.
- **A subscription**: You need an active paid or trial NGINX Plus subscription. You can check it in the [MyF5 Customer Portal](https://account.f5.com/myf5).
- **A supported operating system**: Check your operating system and architecture in the [Technical Specifications]({{< ref "nginx/technical-specs.md" >}}).
- **Root privileges**: The commands use `sudo`. Without root access, follow [Install without root privileges](#unpriv_install).
- **Internet access**: The server must reach `pkgs.nginx.com` and `cs.nginx.com` over HTTPS. Without internet access, follow [Install without internet access](#offline_install).

If you already have NGINX Plus installed, back up its configuration and log files first. For details, see [Upgrading NGINX Plus]({{< ref "nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md" >}}).

---

<span id="release-types-and-versioning"></span>

## Choose a release type

Since May 13, 2026, NGINX Plus is published in two release types:

- [Long-Term Support (LTS)](#lts): focused on stability and security.
- [Continuous Release (CR)](#cr): focused on features and performance.

### Long-Term Support (LTS) {#lts}

The NGINX Plus LTS release model is meant for mission-critical production environments. During an LTS lifecycle, F5 delivers security fixes and CVE mitigations, but no new features. New features come in Continuous Releases during the same annual cycle.

- **Cadence**: One LTS release a year.
- **Patching model**: Security and CVE fixes only. An LTS line gets no feature changes.
- **Support window**: Up to three years.
- **Concurrency**: Up to three LTS versions are supported at the same time.
- **Version format**: An LTS release has `0` as the second number, for example `PLS.37.0.0.1`. LTS updates increase the third number, for example `PLS.37.0.1.1`.

### Continuous Release (CR) {#cr}

NGINX Plus Continuous Releases come out several times during an annual [LTS](#lts) cycle. Each CR has the latest features and performance improvements. The CR cycle ends when a new LTS comes out.

- **Cadence**: Every two to six months.
- **Patching model**: CRs are never patched. Fixes, including CVE fixes, come in the next CR.
- **Support window**: Only the latest CR is supported. When a new CR comes out, the previous CR reaches End of Support immediately.
- **Version format**: CRs increase the second number, for example `PLS.37.1.0.0` and `PLS.37.2.0.0`.

<span id="repository-configuration-options"></span>

### Repository options

By default, the NGINX Plus repositories give you Continuous Releases, and this guide sets them up that way. To get LTS releases, you replace the repository URL with an LTS URL. You have three options:

- **Default**: Get the Continuous Releases of the current LTS cycle. When a new LTS comes out every year, move to it, and then get its CRs. Follow the steps in this guide.
- **Pin to the current LTS only**: Get only security updates for this LTS, with no CRs and no move to the next LTS. This is supported for up to three years. See [Installing NGINX Plus LTS]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-lts.md" >}}).
- **Pin to the LTS track**: Move to the newest LTS every year, with no CRs. See [Installing NGINX Plus LTS]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-lts.md" >}}).

---

## Get the subscription files {#subscription-files}

On every operating system, first get the files of your subscription and put the certificate and key in place:

1. {{< include "licensing-and-reporting/download-jwt-crt-from-myf5.md" >}}

1. {{< include "nginx-plus/install/create-dir-for-crt-key.md" >}}

1. {{< include "nginx-plus/install/copy-crt-and-key.md" >}}

Keep the JWT file. You copy it to NGINX Plus after you install the package.

---

## Install NGINX Plus {#install}

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

1. {{< include "nginx-plus/install/install-nginx-plus-package-yum.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### RHEL 8 and derivatives {#install_rhel8}

These steps apply to RHEL 8.1 and later, Oracle Linux 8.1 and later, AlmaLinux 8, and Rocky Linux 8.

1. Install the `ca-certificates` dependency:

    ```shell
    sudo dnf install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the [plus-8.repo](https://cs.nginx.com/static/files/plus-8.repo) file to `/etc/yum.repos.d`:

    ```shell
    sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-8.repo
    ```

    {{< details summary="Pin NGINX Plus to a specific version" >}}{{< call-out class="note">}}To pin NGINX Plus to a release, for example R33, edit `/etc/yum.repos.d/plus-8.repo` and add the release after `/plus/` in the base URL: `baseurl=https://pkgs.nginx.com/plus/R33/centos/8/$basearch/`. Then refresh the repository information with `sudo dnf makecache`.{{< /call-out >}}{{< /details >}}

1. {{< include "nginx-plus/install/install-nginx-plus-package-dnf.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### RHEL 9 and derivatives {#install_rhel}

These steps apply to RHEL 9.0 and later, Oracle Linux 9, AlmaLinux 9, and Rocky Linux 9.

1. Install the `ca-certificates` dependency:

    ```shell
    sudo dnf install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the [plus-9.repo](https://cs.nginx.com/static/files/plus-9.repo) file to `/etc/yum.repos.d`:

    ```shell
    sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-9.repo
    ```

    {{< details summary="Pin NGINX Plus to a specific version" >}}{{< call-out class="note">}}To pin NGINX Plus to a release, for example R33, edit `/etc/yum.repos.d/plus-9.repo` and add the release after `/plus/` in the base URL: `baseurl=https://pkgs.nginx.com/plus/R33/centos/9/$basearch/`. Then refresh the repository information with `sudo dnf makecache`.{{< /call-out >}}{{< /details >}}

1. {{< include "nginx-plus/install/install-nginx-plus-package-dnf.md" >}}

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

### RHEL 10 and derivatives {#install_rhel10}

These steps apply to RHEL 10.0 and later, Oracle Linux 10.0 and later, AlmaLinux 10.0 and later, and Rocky Linux 10.0 and later.

1. Install the `ca-certificates` dependency:

    ```shell
    sudo dnf install ca-certificates
    ```

1. Add the NGINX Plus repository. Download the [plus-10.repo](https://cs.nginx.com/static/files/plus-10.repo) file to `/etc/yum.repos.d`:

    ```shell
    sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/plus-10.repo
    ```

    {{< details summary="Pin NGINX Plus to a specific version" >}}{{< call-out class="note">}}To pin NGINX Plus to a release, for example R37, edit `/etc/yum.repos.d/plus-10.repo` and add the release after `/plus/` in the base URL: `baseurl=https://pkgs.nginx.com/plus/R37/rhel/10/$basearch/`. Then refresh the repository information with `sudo dnf makecache`.{{< /call-out >}}{{< /details >}}

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

1. Add the NGINX Plus repository:

    ```shell
    printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://pkgs.nginx.com/plus/debian `lsb_release -cs` nginx-plus\n" \
        | sudo tee /etc/apt/sources.list.d/nginx-plus.list
    ```

1. Download the `apt` configuration for the repository to `/etc/apt/apt.conf.d`. It tells `apt` to use your certificate and key for `pkgs.nginx.com`:

    ```shell
    sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
    ```

    {{< details summary="Pin NGINX Plus to a specific version" >}}{{< call-out class="note">}}{{< include "nginx-plus/install/pin-to-version/pin-debian-ubuntu-R32.md" >}}{{< /call-out >}}{{< /details >}}

1. Install the `nginx-plus` package. It replaces any older NGINX Plus package.

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

1. Add the NGINX Plus repository:

    ```shell
    printf "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://pkgs.nginx.com/plus/ubuntu `lsb_release -cs` nginx-plus\n" \
        | sudo tee /etc/apt/sources.list.d/nginx-plus.list
    ```

1. Download the `apt` configuration for the repository to `/etc/apt/apt.conf.d`. It tells `apt` to use your certificate and key for `pkgs.nginx.com`:

    ```shell
    sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
    ```

    {{< details summary="Pin NGINX Plus to a specific version" >}}{{< call-out class="note">}}{{< include "nginx-plus/install/pin-to-version/pin-debian-ubuntu-R32.md" >}}{{< /call-out >}}{{< /details >}}

1. Install the `nginx-plus` package. It replaces any older NGINX Plus package.

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

1. Add these lines to the `/usr/local/etc/pkg.conf` file:

    ```none
    PKG_ENV: { SSL_NO_VERIFY_PEER: "1",
    SSL_CLIENT_CERT_FILE: "/etc/ssl/nginx/nginx-repo.crt",
    SSL_CLIENT_KEY_FILE: "/etc/ssl/nginx/nginx-repo.key" }
    ```

1. Install the `nginx-plus` package. It replaces any older NGINX Plus package.

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

1. Add the NGINX Plus repository.

    For SLES 15:

    ```shell
    sudo zypper addrepo -G -t yum -c \
        "https://pkgs.nginx.com/plus/sles/15?ssl_clientcert=/etc/ssl/nginx/nginx-repo-bundle.crt&ssl_verify=peer" \
        nginx-plus
    ```

    For SLES 16:

    ```shell
    sudo zypper addrepo -G -t yum -c \
        "https://pkgs.nginx.com/plus/sles/16?ssl_clientcert=/etc/ssl/nginx/nginx-repo-bundle.crt&ssl_verify=peer" \
        nginx-plus
    ```

1. Install the `nginx-plus` package. It replaces any older NGINX Plus package.

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

1. Add the NGINX Plus repository to the `/etc/apk/repositories` file:

    ```shell
    printf "https://pkgs.nginx.com/plus/alpine/v`egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release`/main\n" \
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

1. {{< include "nginx-plus/install/configure-usage-reporting.md" >}}

1. If you use [NGINX Instance Manager]({{< ref "/nim/" >}}), install NGINX Agent and turn it on. For details, see [Installation from NGINX Plus repository]({{< ref "/agent/installation-upgrade/installation-plus.md" >}}).

1. If you changed `nginx.conf`, reload NGINX Plus:

    ```shell
    sudo nginx -s reload
    ```

Then [verify the installation](#verify).

---

## Install dynamic modules {#install_modules}

Dynamic modules add features to NGINX Plus, such as advanced security, traffic shaping, telemetry, embedded scripting, and geolocation. You can add or update them independently of the `nginx-plus` package.

A dynamic module is a shared object file (`.so`) that NGINX Plus loads at startup with the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive.

### Types of dynamic modules

{{< table >}}

| Type | Description | Distribution method | F5 NGINX support |
|---|---|---|---|
| [NGINX-authored](#nginx-authored-dynamic-modules) | Developed and distributed by NGINX | Packaged binaries from the NGINX Plus repository | Full support |
| [NGINX-certified community](#nginx-certified-community-dynamic-modules) | Tested and distributed by NGINX | Packaged binaries from the NGINX Plus repository | Installation and basic configuration support |
| [NGINX Certified Partner](#nginx-certified-partner-dynamic-modules) | Built by partners and verified through [NGINX's certification](https://www.f5.com/go/partner/nginx-certified-module-program-documentation) | Provided by partners | Provided by partners |
| [Community](#community-dynamic-modules) | Developed and distributed by third-party contributors | [Compiled by you](#install_modules_oss) | No support |

{{< /table >}}

### NGINX-authored dynamic modules

The F5 NGINX team develops and maintains these modules. They come as packages for several operating systems in the [NGINX Plus repository](#install-module-packages).

{{< table >}}

| Name | Description | Package name |
|---|---|---|
| [ACME](https://github.com/nginx/nginx-acme) | Adds the automatic certificate management ([ACMEv2](https://www.rfc-editor.org/rfc/rfc8555.html)) protocol. | [`nginx-plus-module-acme`]({{< ref "nginx/admin-guide/dynamic-modules/acme.md" >}}) |
| [GeoIP](https://nginx.org/en/docs/http/ngx_http_geoip_module.html) | Adds IP-based geolocation with the precompiled MaxMind databases. | [`nginx-plus-module-geoip`]({{< ref "nginx/admin-guide/dynamic-modules/geoip.md" >}}) |
| [Image-Filter](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html) | Resizes and crops JPEG, GIF, PNG, and WebP images on the fly. | [`nginx-plus-module-image-filter`]({{< ref "nginx/admin-guide/dynamic-modules/image-filter.md" >}}) |
| [njs Scripting Language](https://nginx.org/en/docs/njs/) | Adds JavaScript-like scripting for advanced server-side logic in the NGINX configuration. | [`nginx-plus-module-njs`]({{< ref "nginx/admin-guide/dynamic-modules/nginscript.md" >}}) |
| [OpenTelemetry](https://github.com/nginxinc/nginx-otel) | Adds distributed tracing with OpenTelemetry. | [`nginx-plus-module-otel`]({{< ref "nginx/admin-guide/dynamic-modules/opentelemetry.md" >}}) |
| [Perl](https://nginx.org/en/docs/http/ngx_http_perl_module.html) | Adds Perl scripting for advanced customization. | [`nginx-plus-module-perl`]({{< ref "nginx/admin-guide/dynamic-modules/perl.md" >}}) |
| [XSLT](https://nginx.org/en/docs/http/ngx_http_xslt_module.html) | Applies XSLT transformations to XML responses. | [`nginx-plus-module-xslt`]({{< ref "nginx/admin-guide/dynamic-modules/xslt.md" >}}) |

{{< /table >}}

### NGINX-certified community dynamic modules

These are popular third-party modules that F5 NGINX distributes, with support for installation and basic configuration. They also come as packages in the [NGINX Plus repository](#install-module-packages).

{{< table >}}

| Name | Description | Package name |
|---|---|---|
| [Brotli](https://github.com/google/ngx_brotli) | Adds Brotli compression, with modules for dynamic compression and for serving precompressed `.br` files. | [`nginx-plus-module-brotli`]({{< ref "nginx/admin-guide/dynamic-modules/brotli.md" >}}) |
| [Encrypted-Session](https://github.com/openresty/encrypted-session-nginx-module) | Encrypts and decrypts NGINX variables with AES-256. | [`nginx-plus-module-encrypted-session`]({{< ref "nginx/admin-guide/dynamic-modules/encrypted-session.md" >}}) |
| [FIPS Status Check](https://github.com/ogarrett/nginx-fips-check-module) | Checks whether OpenSSL runs in FIPS mode. | [`nginx-plus-module-fips-check`]({{< ref "nginx/admin-guide/dynamic-modules/fips.md" >}}) |
| [GeoIP2](https://github.com/leev/ngx_http_geoip2_module) | Adds geolocation with MaxMind GeoIP2 databases. | [`nginx-plus-module-geoip2`]({{< ref "nginx/admin-guide/dynamic-modules/geoip2.md" >}}) |
| [Headers-More](https://github.com/openresty/headers-more-nginx-module) | Extends the NGINX [Headers](https://nginx.org/en/docs/http/ngx_http_headers_module.html) module to change request and response headers. | [`nginx-plus-module-headers-more`]({{< ref "nginx/admin-guide/dynamic-modules/headers-more.md" >}}) |
| [HTTP Substitutions Filter](https://github.com/yaoweibin/ngx_http_substitutions_filter_module) | Replaces strings and regular expression matches in response bodies. | [`nginx-plus-module-subs-filter`]({{< ref "nginx/admin-guide/dynamic-modules/http-substitutions-filter.md" >}}) |
| [Lua](https://github.com/openresty/lua-nginx-module) | Embeds the Lua programming language. | [`nginx-plus-module-lua`]({{< ref "nginx/admin-guide/dynamic-modules/lua.md" >}}) |
| [NGINX Developer Kit](https://github.com/vision5/ngx_devel_kit) | Provides helper macros for module development. | [`nginx-plus-module-ndk`]({{< ref "nginx/admin-guide/dynamic-modules/ndk.md" >}}) |
| [Phusion Passenger](https://www.phusionpassenger.com/library/install/nginx/) | An application server for Node.js, Python, and Ruby. | [`nginx-plus-module-passenger`]({{< ref "nginx/admin-guide/dynamic-modules/passenger-open-source.md" >}}) |
| [Prometheus-njs](https://github.com/nginx/nginx-prometheus-exporter) | Converts [NGINX Plus metrics](https://demo.nginx.com/swagger-ui/) into the Prometheus format. | [`nginx-plus-module-prometheus`]({{< ref "nginx/admin-guide/dynamic-modules/prometheus-njs.md" >}}) |
| [RTMP](https://github.com/arut/nginx-rtmp-module) | Adds streaming: RTMP, HLS, and MPEG-DASH, with FFmpeg support. | [`nginx-plus-module-rtmp`]({{< ref "nginx/admin-guide/dynamic-modules/rtmp.md" >}}) |
| [Set-Misc](https://github.com/openresty/set-misc-nginx-module) | Adds `set_*` directives for scripting, which extend the NGINX [Rewrite](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html) module. | [`nginx-plus-module-set-misc`]({{< ref "nginx/admin-guide/dynamic-modules/set-misc.md" >}}) |
| [SPNEGO for Kerberos](https://github.com/stnoonan/spnego-http-auth-nginx-module) | Adds [GSS-API based](https://www.rfc-editor.org/rfc/rfc2743) SPNEGO and Kerberos authentication. | [`nginx-plus-module-auth-spnego`]({{< ref "nginx/admin-guide/dynamic-modules/spnego.md" >}}) |

{{< /table >}}

<span id="dynamic-module-package-installation"></span>

### Install module packages {#install-module-packages}

You can install [NGINX-authored](#nginx-authored-dynamic-modules) and [NGINX-certified community](#nginx-certified-community-dynamic-modules) modules as packages from the NGINX Plus repository. In the commands, replace `<MODULE_NAME>` with the package name, for example `nginx-plus-module-njs`.

- **RHEL and derivatives, Amazon Linux, CentOS, and Oracle Linux**. The `.so` files go to `/usr/lib64/nginx/modules/`.

    ```shell
    sudo dnf install <MODULE_NAME>
    ```

    On Amazon Linux 2, use `yum` instead of `dnf`.

- **Debian and Ubuntu**. The `.so` files go to `/usr/lib/nginx/modules/`.

    ```shell
    sudo apt update
    sudo apt install <MODULE_NAME>
    ```

- **FreeBSD**. The `.so` files go to `/usr/local/etc/nginx/modules/`.

    ```shell
    sudo pkg update
    sudo pkg install <MODULE_NAME>
    ```

- **SUSE Linux Enterprise**. The `.so` files go to `/usr/lib64/nginx/modules/`.

    ```shell
    sudo zypper refresh
    sudo zypper install <MODULE_NAME>
    ```

- **Alpine Linux**. The `.so` files go to `/usr/lib/nginx/modules/`.

    ```shell
    sudo apk update
    sudo apk add <MODULE_NAME>
    ```

Some modules aren't available on every operating system because of platform limits. For details, see [Dynamic modules]({{< ref "nginx/technical-specs.md#dynamic-modules" >}}) in the [NGINX Plus Technical Specifications]({{< ref "nginx/technical-specs.md" >}}). For a description and the installation steps of each module, see [Dynamic Modules]({{< ref "nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}).

After you install a module, [load it](#enable_dynamic) and set it up as its documentation describes.

### Load dynamic modules {#enable_dynamic}

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events`, `http`, and `stream` blocks, add a [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive for each module. After `events`, `nginx -t` reports that `load_module` is `specified too late`. A relative path starts in the NGINX Plus prefix directory. On Linux, `modules/` means `/etc/nginx/modules/`, which leads to the module files of your operating system.

    ```nginx
    load_module modules/<MODULE_NAME_1>.so;
    load_module modules/<MODULE_NAME_2>.so;

    events {
        # ...
    }

    http {
        # ...
    }

    stream {
        # ...
    }
    ```

    Replace `<MODULE_NAME_1>` and `<MODULE_NAME_2>` with the file names of the modules, without the `.so` extension.

1. Save the file.

1. Test the configuration:

    ```shell
    sudo nginx -t
    ```

1. Reload the configuration:

    ```shell
    sudo nginx -s reload
    ```

Then set up each module in the NGINX Plus configuration, as its documentation describes.

### NGINX Certified Partner dynamic modules

These modules are extensions that partners build for NGINX Plus, for example for security, identity and access management, device detection, and application delivery. They're verified through [NGINX's certification process](https://www.f5.com/go/partner/nginx-certified-module-program-documentation). The partners provide the packages, the documentation, and the support.

{{< table >}}

| Name | Description | Commercial support |
|---|---|---|
| [CQ botDefence](https://www.cequence.ai/contact-us/) | Analyzes traffic to prevent fraud and theft from automated bot attacks on your public web, mobile, and API-based applications. | [Support](https://www.cequence.ai/demo/) provided by [Cequence](https://www.cequence.ai) |
| [Curity Identity Server](https://developer.curity.io/) | An OAuth and OpenID Connect server for user logins and for securing access to APIs, mobile apps, and microservices. | [Support](https://curity.io/support/professional-services/) and docs [[1]](https://curity.io/resources/learn/nginx-phantom-token-module/), [[2]](https://curity.io/resources/learn/nginx-oauth-proxy/) provided by [Curity](https://curity.io/support/professional-services/) |
| [DeviceAtlas](https://deviceatlas.com/deviceatlas-nginx-module) | Detects the devices that users have, such as smartphones, laptops, and wearable devices, to deliver customized experiences. | [Support](https://deviceatlas.com/resources/support) and [docs](https://docs.deviceatlas.com/apis/enterprise/c/3.1.3/README.Nginx.html) provided by [DeviceAtlas](https://deviceatlas.com/resources/support) |
| [ForgeRock Policy Agent](https://backstage.forgerock.com/downloads/browse/am/featured/web-agents) | Works with ForgeRock Access Management to authenticate access to your applications and APIs. | [Support](https://support.pingidentity.com/s/) and [docs](https://backstage.forgerock.com/docs/openam-web-policy-agents/2023.9/installation-guide/install-nginx.html) provided by [PingIdentity](https://www.pingidentity.com) |
| [HUMAN Security for F5 NGINX](https://docs.humansecurity.com/home) | Provides an enforcement layer that protects websites and apps from automated security threats. | Support provided by [HUMAN Security](https://docs.humansecurity.com/home) |
| IDFConnect SSO/Rest | Integrates all features of your web access management platform with NGINX Plus. | Support and docs provided by IDFConnect |
| [OPSWAT](https://www.f5.com/go/product/nginx-modules) | Protects your networks and applications from malware and unknown (zero-day) malicious file content. | [Support](https://www.opswat.com/support) and [docs](https://www.opswat.com/docs/mdicap/integrations/nginx-integration-module) provided by [OPSWAT](https://www.opswat.com/) |
| [Passenger Enterprise](https://www.phusionpassenger.com/features) | An application server for Meteor, Node.js, Python, and Ruby apps. | [Support](https://www.phusionpassenger.com/support) and [docs](https://www.phusionpassenger.com/docs/advanced_guides/install_and_upgrade/nginx/install_as_nginx_module.html) provided by [Phusion](https://www.phusionpassenger.com/) |
| [Ping Access](https://support.pingidentity.com/s/marketplace-integration/a7i1W0000004ICRQA2/pingaccess-agent-for-nginx-plus) | Manages access security centrally, with contextual policies for your mobile and web properties in any domain. | [Support](https://support.pingidentity.com/s/) and [docs](https://docs.pingidentity.com/pingaccess/latest/agents_and_integrations/pa_agent_for_nginx.html) provided by [PingIdentity](https://www.pingidentity.com) |
| [PingIntelligence](https://hub.pingidentity.com/datasheets/3742-pingintelligence-apis) | Secures the APIs of an organization across on-premises, public cloud, private cloud, and hybrid environments. | [Support](https://support.pingidentity.com/s/) and [docs](https://docs.pingidentity.com/pingintelligence/5.1/pingintelligence_integrations/pingintelligence_nginx_plus_integration.html) provided by [PingIdentity](https://www.pingidentity.com) |
| [Seer Box by Plurbius One](https://seerbox.it/en/) | A cloud-native web application security manager for monitoring and protection. | Support provided by [Seer Box](https://support.seerbox.it/) |
| [Signal Sciences](https://docs.fastly.com/en/ngwaf/about-the-nginx-module) | Detects and blocks malicious requests. Its fail-open design passes legitimate requests through. | [Support](https://support.fastly.com/s/) and [docs](https://docs.fastly.com/en/ngwaf/installing-the-nginx-dynamic-module) provided by [Fastly](https://www.fastly.com/) |
| [Wallarm](https://www.wallarm.com/company) | A WAF that protects against advanced Layer 7 application attacks. | [Support](https://www.wallarm.com/support) and [docs](https://docs.wallarm.com/installation/nginx-native-node-internals/#nginx-node) provided by [Wallarm](https://wallarm.com/) |
| [WURFL InFuse](https://www.scientiamobile.com/secondary-products/wurfl-infuze-module-for-nginx-plus/) | Detects devices with high accuracy and performance. | [Support](https://www.scientiamobile.com/support/) and [docs](https://docs.scientiamobile.com/documentation/infuze/infuze-nginx-plus-module-user-guide) provided by [Scientiamobile](https://www.scientiamobile.com/) |
| [51Degrees Device Detection](https://github.com/51Degrees/device-detection-nginx) | Detects devices quickly and accurately, for a better user experience and detailed analysis. | [Support](https://51degrees.com/pricing/index) and [docs](https://github.com/51Degrees/device-detection-nginx/blob/main/README.md) provided by [51Degrees](https://51degrees.com/about-us) |

{{< /table >}}

For the complete list of Certified Partner modules, see the [F5.com Dynamic Modules](https://www.f5.com/go/product/nginx-modules?filter=module-author%3Anginx-certified-partner) page.

### Community dynamic modules

Third-party contributors of the NGINX community develop and distribute these open source modules.

They aren't in the NGINX Plus repository. To use one, download its source code and [compile it with the NGINX Open Source version](#install_modules_oss) that matches your NGINX Plus version. For lists of community modules, see community resources such as the [Awesome NGINX GitHub project](https://github.com/agile6v/awesome-nginx#third-party-modules).

### Compile a community dynamic module {#install_modules_oss}

A community dynamic module works with NGINX Plus only if you compile it with the matching version of NGINX Open Source.

1. Find the NGINX Open Source version of your NGINX Plus:

    ```shell
    nginx -v
    ```

    The output shows the NGINX Open Source version first, for example:

    ```none
    nginx version: nginx/1.31.6 (nginx-plus-r37.2.0)
    ```

1. Prepare a build environment. Compile modules on a separate system, not on the production server. This keeps the risk and complexity away from production. The build environment needs:

    - The same operating system as the production environment.
    - A C compiler and `make`.
    - The development files of the PCRE2 and zlib libraries.

    To install them:

    - Debian and Ubuntu:

        ```shell
        sudo apt update
        sudo apt install gcc make libpcre2-dev zlib1g-dev
        ```

    - RHEL and derivatives:

        ```shell
        sudo dnf install gcc make pcre2-devel zlib-devel
        ```

1. Get the NGINX Open Source source code of that version. To find the version for your NGINX Plus release, see [NGINX Plus Releases]({{< ref "nginx/releases.md" >}}). For example, for 1.31.6:

    ```shell
    wget -qO - https://nginx.org/download/nginx-1.31.6.tar.gz | tar zxf -
    ```

1. Get the source code of the module, in any directory of the build environment. For example, the [NGINX "Hello World" module](https://github.com/perusio/nginx-hello-world-module.git/):

    ```shell
    git clone https://github.com/perusio/nginx-hello-world-module.git
    ```

1. Compile the module. The `--with-compat` option makes the module binary compatible with NGINX Plus, and `make modules` builds only the modules:

    ```shell
    cd nginx-1.31.6/
    ./configure --with-compat --add-dynamic-module=<MODULE_SOURCES>
    make modules
    ```

    Replace `<MODULE_SOURCES>` with the path of the module's source code directory, for example `../nginx-hello-world-module`. The build puts the `.so` file in the `objs` directory:

    ```shell
    ls objs/*.so
    ```

    ```none
    objs/ngx_http_hello_world_module.so
    ```

1. Copy the module file and add the NGINX Open Source version to the name. This helps when you keep several versions of a module on the production server.

    ```shell
    cp objs/ngx_http_hello_world_module.so ./ngx_http_hello_world_module_1.31.6.so
    ```

1. Transfer the `.so` file to the production server.

1. On the production server, copy the file to the modules directory. On Linux, `/etc/nginx/modules/` leads to it; on FreeBSD, it's `/usr/local/etc/nginx/modules/`.

    ```shell
    sudo cp ngx_http_hello_world_module_1.31.6.so /etc/nginx/modules/
    ```

Then [load the module](#enable_dynamic) in the NGINX Plus configuration.

---

## Install without root privileges {#unpriv_install}

<span id="nginx-plus-unprivileged-installation"></span>

In some environments, the root account isn't available for security reasons. Without root privileges, you can't use a package manager to install NGINX Plus on Linux.

Instead, a script can install NGINX Plus for a user without root privileges. The script:

- Downloads the NGINX Plus packages.
- Extracts the packages into a directory that you choose.
- Changes the paths in the NGINX Plus configuration to that directory.
- Backs up the configuration directory.
- Upgrades an existing unprivileged installation, if you ask it to.

An unprivileged installation has these limits:

- NGINX Plus needs root privileges to listen on ports below `1024`.
- The script doesn't replace your operating system's package manager, and it installs only NGINX Plus and its modules. Support doesn't cover changes to the script for other software.
- NGINX Plus doesn't start automatically. For each unprivileged installation on the host, add your own init script or `systemd` unit file.
- The script doesn't install the dependencies and libraries that NGINX Plus and its modules need. Check and install them yourself.

The script runs on these operating systems:

- Red Hat Enterprise Linux and CentOS
- Amazon Linux 2023
- Debian and Ubuntu
- Alpine Linux
- AlmaLinux and Rocky Linux

Before you start, make sure that you meet the [prerequisites](#before-you-begin), except for root privileges. The script also needs:

- `wget`, but not the BusyBox version that Alpine Linux has by default. On Alpine Linux, install the `wget` package.
- On RPM-based distributions, [`rpm2cpio`](https://man7.org/linux/man-pages/man8/rpm2cpio.8.html) and `cpio`.

To install NGINX Plus without root privileges:

1. {{< include "licensing-and-reporting/download-jwt-crt-from-myf5.md" >}}

1. Name the downloaded JWT license file `license.jwt`.

1. Download the script:

    ```shell
    wget https://raw.githubusercontent.com/nginxinc/nginx-plus-install-tools/main/ngxunprivinst.sh
    ```

1. Make the script executable:

    ```shell
    chmod +x ngxunprivinst.sh
    ```

1. Download the NGINX Plus package and its module packages for your operating system. `<CERT_FILE>` and `<KEY_FILE>` are the certificate and key of your subscription:

    ```shell
    ./ngxunprivinst.sh fetch -c <CERT_FILE> -k <KEY_FILE>
    ```

    To install a specific version of NGINX Plus:

    1. List the versions in the repository:

        ```shell
        ./ngxunprivinst.sh list -c <CERT_FILE> -k <KEY_FILE>
        ```

    1. Download that version with the `-v` option:

        ```shell
        ./ngxunprivinst.sh fetch -c <CERT_FILE> -k <KEY_FILE> -v <VERSION>
        ```

        Replace `<VERSION>` with a package version from the list.

1. Extract the downloaded packages to the directory `<INSTALL_PATH>`, set with the `-p` option. Set the path of your `license.jwt` file with the `-j` option. The optional `-y` option answers "yes" to all questions, for example to overwrite an existing installation.

    ```shell
    ./ngxunprivinst.sh install [-y] -p <INSTALL_PATH> -j <LICENSE_FILE> <FILE1> <FILE2>
    ```

    Replace `<FILE1>` and `<FILE2>` with the names of the downloaded package files.

1. Start NGINX Plus with the command that the script prints when it finishes. For NGINX Plus R23 and later, it's:

    ```shell
    <INSTALL_PATH>/usr/sbin/nginx -p <INSTALL_PATH>/etc/nginx -c nginx.conf -e <INSTALL_PATH>/var/log/nginx/error.log
    ```

    The `-p` option sets the directory of the NGINX Plus files. The `-c` option sets the configuration file, relative to that directory, and `-e` sets the error log. Without root privileges, NGINX Plus can listen only on ports 1024 and above. The script changes the port of the default server from 80 to 8080.

To upgrade an unprivileged installation in `<INSTALL_PATH>`, run the script with `upgrade`. For NGINX Plus R33 and later, the `-j` option with the license file is required. The optional `-y` option upgrades without asking for confirmation.

```shell
./ngxunprivinst.sh upgrade [-y] -p <INSTALL_PATH> -j <LICENSE_FILE> <FILE1> <FILE2>
```

---

## Install without internet access {#offline_install}

You can install NGINX Plus and its [dynamic modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}) on a server with limited or no internet access. You need a machine with internet access to get the NGINX Plus packages, the license file, the certificate, and the key. Then you transfer these files to the target server.

### Step 1: Get the files on a machine with internet access {#offline-obtain-files}

1. {{< include "licensing-and-reporting/download-jwt-crt-from-myf5.md" >}}

1. Download the NGINX Plus package and the module packages that you need, on a machine with the same operating system as the target server. For example, the `fetch` command of the [installation script for unprivileged installations](#unpriv_install) downloads them with your certificate and key. For Alpine Linux, also download the signing key from `https://cs.nginx.com/static/keys/nginx_signing.rsa.pub`.

1. Transfer the files to the target server.

<span id="step-2-installing-nginx-plus-on-a-server-without-internet-connectivity"></span>

### Step 2: Install NGINX Plus on the server without internet access

1. {{< include "nginx-plus/install/back-up-config-and-logs.md" >}}

1. {{< include "nginx-plus/install/create-dir-for-crt-key.md" >}}

1. {{< include "nginx-plus/install/copy-crt-and-key.md" >}}

1. Install the NGINX Plus package or a dynamic module package. This replaces any older NGINX Plus package. These commands don't install the packages that NGINX Plus or a module depends on, so download those too, and install them first.

    - **RHEL, Amazon Linux, CentOS, Oracle Linux, AlmaLinux, Rocky Linux, and SLES**:

        ```shell
        sudo rpm -Uvh <RPM_PACKAGE_NAME>
        ```

        Replace `<RPM_PACKAGE_NAME>` with the name of the RPM package file.

    - **Debian and Ubuntu**:

        ```shell
        sudo dpkg -i <DEB_PACKAGE_NAME>
        ```

        Replace `<DEB_PACKAGE_NAME>` with the name of the DEB package file.

    - **Alpine Linux**. Put the signing key in `/etc/apk/keys/` first, or `apk` stops with `UNTRUSTED signature`:

        ```shell
        sudo cp nginx_signing.rsa.pub /etc/apk/keys/
        sudo apk add ./<APK_PACKAGE_NAME>
        ```

        Replace `<APK_PACKAGE_NAME>` with the name of the APK package file.

1. {{< include "nginx-plus/install/copy-jwt-to-etc-nginx-dir.md" >}}

1. {{< include "nginx-plus/install/check-nginx-binary-version.md" >}}

1. Install NGINX Instance Manager 2.18 or later in your local network. Since R33, usage reporting is mandatory, and NGINX Instance Manager reports usage for servers without internet access. For more information, see [Disconnected environments]({{< ref "nim/disconnected/_index.md" >}}) and [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}}).

1. Set up usage reporting from NGINX Plus to NGINX Instance Manager. In the `nginx.conf` file, add:

    - The [`mgmt {}`](https://nginx.org/en/docs/ngx_mgmt_module.html#mgmt) block, which holds the licensing and usage reporting configuration of NGINX Plus.
    - The [`usage_report`](https://nginx.org/en/docs/ngx_mgmt_module.html#usage_report) directive, which sets the domain name or IP address of NGINX Instance Manager.
    - The [`enforce_initial_report`](https://nginx.org/en/docs/ngx_mgmt_module.html#enforce_initial_report) directive. Set to `off`, it turns on a 180-day grace period for the initial usage report. If the F5 licensing endpoint doesn't receive the initial report during this period, NGINX Plus stops processing traffic.

    ```nginx
    mgmt {
        usage_report endpoint=<NIM_FQDN>;
        enforce_initial_report off;
    }
    ```

1. {{< include "nginx-plus/install/nim-disconnected-report-usage.md" >}}

1. Upload the usage acknowledgement to NGINX Instance Manager. For more information, see [Report usage to F5 in a disconnected environment]({{< ref "nim/licensing-and-reporting/report-usage-disconnected-deployment.md#submit-usage-report" >}}).

1. [Start NGINX Plus](#start). If it's already running, reload the configuration with `sudo nginx -s reload`.

---

## Verify the installation {#verify}

1. {{< include "nginx-plus/install/check-nginx-binary-version.md" >}}

    The output shows the NGINX Open Source version and the NGINX Plus release, for example:

    ```none
    nginx version: nginx/1.31.6 (nginx-plus-r37.2.0)
    ```

1. Test the configuration:

    ```shell
    sudo nginx -t
    ```

    The output ends with `test is successful`.

---

## Troubleshooting

### The package manager can't get packages from pkgs.nginx.com

**Symptom**: Updating the package list shows errors for `pkgs.nginx.com`, and installing `nginx-plus` then fails. For example, `apt` says `Unable to locate package nginx-plus`, and `apk` says `nginx-plus (no such package)`.

**Cause**: The package manager can't present the certificate and key of your subscription, which the repository requires. The messages show why:

- `Could not load client certificate (... SslCert option): ... No such file or directory`: the files aren't at `/etc/ssl/nginx/nginx-repo.crt` and `/etc/ssl/nginx/nginx-repo.key`.
- `Could not load client certificate (... SslCert option): ... Permission denied`: `apt` downloads as the `_apt` user, which can't read the files.
- `400  Bad Request` from `apt`, or `HTTP 400: Bad Request` from `apk`: the package manager didn't send a certificate. On Debian and Ubuntu, `/etc/apt/apt.conf.d/90pkgs-nginx` is missing.

**Fix**: [Copy the certificate and the key](#subscription-files) to `/etc/ssl/nginx/`, and check the repository configuration that points to them:

- On Debian and Ubuntu, download `90pkgs-nginx` as in the installation steps. If `apt` can't read the files, give the `_apt` user read access. The `setfacl` command is in the `acl` package:

    ```shell
    sudo setfacl -m u:_apt:r /etc/ssl/nginx/nginx-repo.crt /etc/ssl/nginx/nginx-repo.key
    ```

- On Alpine Linux, check `/etc/apk/cert.pem` and `/etc/apk/cert.key`.
- On RHEL-based systems, check the `sslclientcert` and `sslclientkey` lines in the `.repo` file.
- On SLES, check the bundle file `/etc/ssl/nginx/nginx-repo-bundle.crt`.
- On FreeBSD, check the `PKG_ENV` lines in `/usr/local/etc/pkg.conf`.

---

### apk stops with "UNTRUSTED signature"

**Symptom**: On Alpine Linux, installing a package file fails with a message such as `ERROR: ./nginx-plus.apk: UNTRUSTED signature`.

**Cause**: The NGINX signing public key isn't in the `/etc/apk/keys/` directory.

**Fix**: Copy `nginx_signing.rsa.pub` to `/etc/apk/keys/`, and then install the package again.

---

### A dynamic module doesn't load: "version ... instead of ..."

**Symptom**: `nginx -t` or a reload fails with a message such as `module "/etc/nginx/modules/ngx_http_hello_world_module_1.31.6.so" version 1031006 instead of 1031009`.

**Cause**: The module was built for another NGINX version. A dynamic module must match the NGINX Open Source version of your NGINX Plus exactly.

**Fix**: Install the module package from the NGINX Plus repository together with NGINX Plus. For a community module, compile it again with the NGINX Open Source version that `nginx -v` shows.

---

<span id="explore-related-topics"></span><span id="install-f5-waf-for-nginx"></span>

## References

<span id="upgrade"></span>

- [Upgrading NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md" >}})
- [Installing NGINX Plus LTS]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-lts.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "nginx/technical-specs.md" >}})
- [NGINX Plus Releases]({{< ref "nginx/releases.md" >}})
- [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}})
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [F5 WAF for NGINX installation]({{< ref "/waf/install/" >}})
