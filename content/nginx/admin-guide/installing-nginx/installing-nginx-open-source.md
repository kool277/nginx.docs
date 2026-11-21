---
title: Installing NGINX Open Source
description: Install NGINX Open Source on Linux from the nginx.org package repository, from your operating system's packages, or from source code.
toc: true
weight: 200
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "install nginx, nginx.org packages, apt, yum, dnf, zypper, apk, mainline, stable, compile from source, configure options, dynamic modules"
f5-summary: >
  This guide shows how to install NGINX Open Source from the nginx.org package repository, from your operating system's repository, or from source code.
  The nginx.org packages give you the latest stable or mainline release, and your package manager keeps them up to date.
  It covers RHEL and its derivatives, Debian, Ubuntu, SLES, Alpine Linux, and Amazon Linux 2023, and source builds on Unix-like systems.
f5-audience: operator
---

## Overview

You can install NGINX Open Source in three ways. Install a package from the nginx.org repository or from your operating system's repository, or build NGINX from source code. The nginx.org packages are recommended for production: you set up the repository once, and your package manager then installs every new release. When you finish this guide, NGINX runs and serves its default page on port 80.

---

## Before you begin

Before you begin, make sure you have:

- **A supported operating system**: The nginx.org packages are built for these systems, for x86_64 and aarch64:

    - RHEL 8, 9, and 10, and their derivatives
    - Debian 12 and 13
    - Ubuntu 22.04, 24.04, and 26.04
    - SLES 15 SP6 and later, for x86_64 only, and SLES 16
    - Alpine Linux 3.21 to 3.24
    - Amazon Linux 2023

    The list changes with new releases, so check [Supported distributions and versions](https://nginx.org/en/linux_packages.html#distributions). To see your system's version and architecture, run `cat /etc/os-release` and `uname -m`.
- **Root privileges**: The commands use `sudo`. Alpine Linux doesn't include `sudo` by default, so run the commands there as root.
- **Internet access**: The server must reach `nginx.org` over HTTPS to download packages and signing keys.
- **A free port 80**: The default configuration listens on port 80. Stop any other web server that uses this port.

---

## Choose a version and a method

NGINX Open Source has two release branches:

- **Mainline** has an odd minor version number, for example 1.31. It gets new features and all bug fixes first. It's recommended for production unless your organization has strict stability requirements.
- **Stable** has an even minor version number, for example 1.30. It gets only critical fixes, which are taken from mainline. A new stable branch starts from mainline about once a year.

To see the current version numbers, go to the [download page](https://nginx.org/en/download.html). The nginx.org repositories install stable by default, and each set of steps below shows how to switch to mainline.

Pick an installation method:

{{< table >}}

| Method | Version you get | How you update | Use it when |
|---|---|---|---|
| [nginx.org repository](#prebuilt) | The latest stable or mainline release | With your package manager | You run NGINX in production. |
| [Your operating system's repository](#os-default-repository) | The version that your OS release ships, often an older one | With your package manager | You want the quickest setup for learning or testing. |
| [Source code](#sources) | Any version | You build each new version yourself | You need custom build options or third-party modules. |
| Container image | The latest stable or mainline release | You pull a new image | You run NGINX in Docker or Kubernetes. See [Deploying NGINX and NGINX Plus with Docker]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-docker.md" >}}). |

{{< /table >}}

---

## Install from the nginx.org repository {#prebuilt}

You set up the nginx.org repository once. After that, your package manager installs NGINX and its updates. Follow the steps for your operating system, and then [start NGINX](#start).

### RHEL and derivatives {#rhel}

These steps apply to Red Hat Enterprise Linux and its derivatives, such as AlmaLinux, Oracle Linux, and Rocky Linux.

1. Install the prerequisites:

    ```shell
    sudo yum install yum-utils
    ```

    The `yum-utils` package provides `yum-config-manager`, which switches between the stable and mainline repositories.

2. Create the file `/etc/yum.repos.d/nginx.repo` with this content:

    ```ini
    [nginx-stable]
    name=nginx stable repo
    baseurl=https://nginx.org/packages/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true

    [nginx-mainline]
    name=nginx mainline repo
    baseurl=https://nginx.org/packages/mainline/centos/$releasever/$basearch/
    gpgcheck=1
    enabled=0
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    ```

    The stable repository is on, and the mainline repository is off. With `module_hotfixes=true`, the package manager installs these packages even when your OS provides nginx as a module stream.

3. To get mainline packages instead of stable ones, turn on the mainline repository:

    ```shell
    sudo yum-config-manager --enable nginx-mainline
    ```

4. Install NGINX:

    ```shell
    sudo yum install nginx
    ```

    The package manager asks you to accept one or more nginx signing keys. Accept a key only if its fingerprint is one of these:

    - `8540 A6F1 8833 A80E 9C16 53A4 2FD2 1310 B49F 6B46` (`signing-key-2@nginx.com`)
    - `573B FD6B 3D8F BC64 1079 A6AB ABF5 BD82 7BD9 BF62` (`signing-key@nginx.com`)
    - `9E9B E90E ACBC DE69 FE9B 204C BCDC D8A3 8D88 A2B3` (`signing-key-3@nginx.com`)

### Debian {#debian}

1. Install the prerequisites:

    ```shell
    sudo apt update
    sudo apt install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
    ```

2. Download the nginx signing key and save it in the format that `apt` reads:

    ```shell
    curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
    ```

3. Check the keys in the downloaded file:

    ```shell
    gpg --show-keys /usr/share/keyrings/nginx-archive-keyring.gpg
    ```

    The output lists three keys with these fingerprints. If the fingerprints don't match, delete the file and stop.

    ```none
    pub   rsa4096 2024-05-29 [SC]
          8540A6F18833A80E9C1653A42FD21310B49F6B46
    uid                      nginx signing key <signing-key-2@nginx.com>

    pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
          573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
    uid                      nginx signing key <signing-key@nginx.com>

    pub   rsa4096 2024-05-29 [SC]
          9E9BE90EACBCDE69FE9B204CBCDCD8A38D88A2B3
    uid                      nginx signing key <signing-key-3@nginx.com>
    ```

    If you haven't used `gpg` before, it first prints three lines about creating the `~/.gnupg` directory and its files.

4. Add the repository for stable packages:

    ```shell
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://nginx.org/packages/debian `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

    To get mainline packages, run this command instead:

    ```shell
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://nginx.org/packages/mainline/debian `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

5. Set up repository pinning, so that `apt` prefers the nginx.org packages to the ones from Debian:

    ```shell
    echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
        | sudo tee /etc/apt/preferences.d/99nginx
    ```

6. Install NGINX:

    ```shell
    sudo apt update
    sudo apt install nginx
    ```

    The package starts NGINX and sets it to start when the system starts.

### Ubuntu {#ubuntu}

1. Install the prerequisites:

    ```shell
    sudo apt update
    sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
    ```

2. Download the nginx signing key and save it in the format that `apt` reads:

    ```shell
    curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
        | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
    ```

3. Check the keys in the downloaded file:

    ```shell
    gpg --show-keys /usr/share/keyrings/nginx-archive-keyring.gpg
    ```

    The output lists the same three keys as in the [Debian steps](#debian). If the fingerprints don't match, delete the file and stop.

4. Add the repository for stable packages:

    ```shell
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

    To get mainline packages, run this command instead:

    ```shell
    echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://nginx.org/packages/mainline/ubuntu `lsb_release -cs` nginx" \
        | sudo tee /etc/apt/sources.list.d/nginx.list
    ```

5. Set up repository pinning, so that `apt` prefers the nginx.org packages to the ones from Ubuntu:

    ```shell
    echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
        | sudo tee /etc/apt/preferences.d/99nginx
    ```

6. Install NGINX:

    ```shell
    sudo apt update
    sudo apt install nginx
    ```

    The package starts NGINX and sets it to start when the system starts.

### SLES {#sles}

1. Install the prerequisites:

    ```shell
    sudo zypper install curl ca-certificates gpg2
    ```

2. Add the repository for stable packages:

    ```shell
    sudo zypper addrepo --gpgcheck --type yum --refresh --check \
        'https://nginx.org/packages/sles/$releasever_major' nginx-stable
    ```

    To get mainline packages, run this command instead:

    ```shell
    sudo zypper addrepo --gpgcheck --type yum --refresh --check \
        'https://nginx.org/packages/mainline/sles/$releasever_major' nginx-mainline
    ```

3. Download the nginx signing key:

    ```shell
    curl -o /tmp/nginx_signing.key https://nginx.org/keys/nginx_signing.key
    ```

4. Check the keys in the downloaded file:

    ```shell
    gpg --show-keys /tmp/nginx_signing.key
    ```

    The output lists the same three keys as in the [Debian steps](#debian). If the fingerprints don't match, delete the file and stop.

5. Import the key into the RPM database:

    ```shell
    sudo rpmkeys --import /tmp/nginx_signing.key
    ```

6. Install NGINX:

    ```shell
    sudo zypper install nginx
    ```

### Alpine Linux {#alpine}

1. Install the prerequisites:

    ```shell
    sudo apk add openssl curl ca-certificates
    ```

2. Add the repository for stable packages:

    ```shell
    printf "%s%s%s%s\n" \
        "@nginx " \
        "https://nginx.org/packages/alpine/v" \
        `egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release` \
        "/main" \
        | sudo tee -a /etc/apk/repositories
    ```

    The command adds a line such as `@nginx https://nginx.org/packages/alpine/v3.24/main` to the file. The `@nginx` tag marks the repository, so `apk` takes packages from it only when you ask for them.

    To get mainline packages, run this command instead:

    ```shell
    printf "%s%s%s%s\n" \
        "@nginx " \
        "https://nginx.org/packages/mainline/alpine/v" \
        `egrep -o '^[0-9]+\.[0-9]+' /etc/alpine-release` \
        "/main" \
        | sudo tee -a /etc/apk/repositories
    ```

3. Download the nginx signing key:

    ```shell
    curl -o /tmp/nginx_signing.rsa.pub https://nginx.org/keys/nginx_signing.rsa.pub
    ```

4. Check the key in the downloaded file:

    ```shell
    openssl rsa -pubin -in /tmp/nginx_signing.rsa.pub -text -noout
    ```

    The output must show this modulus. If it doesn't, delete the file and stop.

    ```none
    Public-Key: (2048 bit)
    Modulus:
        00:fe:14:f6:0a:1a:b8:86:19:fe:cd:ab:02:9f:58:
        2f:37:70:15:74:d6:06:9b:81:55:90:99:96:cc:70:
        5c:de:5b:e8:4c:b2:0c:47:5b:a8:a2:98:3d:11:b1:
        f6:7d:a0:46:df:24:23:c6:d0:24:52:67:ba:69:ab:
        9a:4a:6a:66:2c:db:e1:09:f1:0d:b2:b0:e1:47:1f:
        0a:46:ac:0d:82:f3:3c:8d:02:ce:08:43:19:d9:64:
        86:c4:4e:07:12:c0:5b:43:ba:7d:17:8a:a3:f0:3d:
        98:32:b9:75:66:f4:f0:1b:2d:94:5b:7c:1c:e6:f3:
        04:7f:dd:25:b2:82:a6:41:04:b7:50:93:94:c4:7c:
        34:7e:12:7c:bf:33:54:55:47:8c:42:94:40:8e:34:
        5f:54:04:1d:9e:8c:57:48:d4:b0:f8:e4:03:db:3f:
        68:6c:37:fa:62:14:1c:94:d6:de:f2:2b:68:29:17:
        24:6d:f7:b5:b3:18:79:fd:31:5e:7f:4c:be:c0:99:
        13:cc:e2:97:2b:dc:96:9c:9a:d0:a7:c5:77:82:67:
        c9:cb:a9:e7:68:4a:e1:c5:ba:1c:32:0e:79:40:6e:
        ef:08:d7:a3:b9:5d:1a:df:ce:1a:c7:44:91:4c:d4:
        99:c8:88:69:b3:66:2e:b3:06:f1:f4:22:d7:f2:5f:
        ab:6d
    Exponent: 65537 (0x10001)
    ```

5. Move the key to the `apk` trusted keys directory:

    ```shell
    sudo mv /tmp/nginx_signing.rsa.pub /etc/apk/keys/
    ```

6. Install NGINX. The `@nginx` tag tells `apk` to take the package from the nginx.org repository:

    ```shell
    sudo apk add nginx@nginx
    ```

### Amazon Linux 2023 {#amazon-linux}

1. Install the prerequisites:

    ```shell
    sudo yum install yum-utils
    ```

2. Create the file `/etc/yum.repos.d/nginx.repo` with this content:

    ```ini
    [nginx-stable]
    name=nginx stable repo
    baseurl=https://nginx.org/packages/amzn/2023/$basearch/
    gpgcheck=1
    enabled=1
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    priority=9

    [nginx-mainline]
    name=nginx mainline repo
    baseurl=https://nginx.org/packages/mainline/amzn/2023/$basearch/
    gpgcheck=1
    enabled=0
    gpgkey=https://nginx.org/keys/nginx_signing.key
    module_hotfixes=true
    priority=9
    ```

    The `priority=9` setting makes the package manager prefer these repositories to the Amazon Linux ones.

3. To get mainline packages instead of stable ones, turn on the mainline repository:

    ```shell
    sudo yum-config-manager --enable nginx-mainline
    ```

4. Install NGINX:

    ```shell
    sudo yum install nginx
    ```

    Accept a signing key only if its fingerprint is one of the three listed in the [RHEL steps](#rhel).

nginx.org no longer lists Amazon Linux 2 as supported. Its Amazon Linux 2 repositories stopped getting new packages in June 2026, at versions 1.30.3 (stable) and 1.31.2 (mainline). To get current releases, move to Amazon Linux 2023.

---

## Start NGINX {#start}

How you start NGINX depends on your operating system:

- **Debian and Ubuntu**: The package already started NGINX and set it to start when the system starts. You don't need to do anything.
- **RHEL and derivatives, SLES, and Amazon Linux 2023**: Start NGINX, and set it to start when the system starts:

    ```shell
    sudo systemctl enable --now nginx
    ```

- **Alpine Linux**: Set NGINX to start when the system starts, and then start it:

    ```shell
    sudo rc-update add nginx default
    sudo rc-service nginx start
    ```

- **A container or a system without an init system**: Run `nginx` as root. In a Debian or Ubuntu container, the package doesn't start NGINX, because the container image blocks services from starting.

    ```shell
    nginx
    ```

To check that NGINX runs, [verify the installation](#verify).

---

## Install dynamic module packages {#dynamic-modules}

The `nginx` package from nginx.org includes every [module built by default](#modules-built-by-default). It also includes the modules that these `configure` options add:

- **General**: `--with-compat`, `--with-file-aio`, and `--with-threads`.
- **HTTP**: `--with-http_addition_module`, `--with-http_auth_request_module`, `--with-http_dav_module`, `--with-http_flv_module`, `--with-http_gunzip_module`, `--with-http_gzip_static_module`, `--with-http_mp4_module`, `--with-http_random_index_module`, `--with-http_realip_module`, `--with-http_secure_link_module`, `--with-http_slice_module`, `--with-http_ssl_module`, `--with-http_stub_status_module`, `--with-http_sub_module`, `--with-http_v2_module`, and `--with-http_v3_module`.
- **Mail**: `--with-mail` and `--with-mail_ssl_module`.
- **Stream**: `--with-stream`, `--with-stream_realip_module`, `--with-stream_ssl_module`, and `--with-stream_ssl_preread_module`.
- **Mainline packages only**: The mainline packages, for example 1.31.9, also add `--with-control-api` and `--with-http_json_module`.

For what each option does, see [Modules not built by default](#modules-not-built-by-default).

Other modules come in separate packages, as [dynamic modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}):

{{< table >}}

| Package | Modules | Description |
|---|---|---|
| `nginx-module-acme` | [`ngx_http_acme_module`](https://nginx.org/en/docs/http/ngx_http_acme_module.html) | Gets and renews TLS certificates automatically with the ACME protocol. |
| `nginx-module-geoip` | [`ngx_http_geoip_module`](https://nginx.org/en/docs/http/ngx_http_geoip_module.html), [`ngx_stream_geoip_module`](https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html) | Sets variables from the client IP address with MaxMind GeoIP databases. |
| `nginx-module-image-filter` | [`ngx_http_image_filter_module`](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html) | Transforms images in JPEG, GIF, PNG, and WebP formats. |
| `nginx-module-njs` | [`ngx_http_js_module`](https://nginx.org/en/docs/http/ngx_http_js_module.html), [`ngx_stream_js_module`](https://nginx.org/en/docs/stream/ngx_stream_js_module.html) | Runs your own JavaScript code (njs) to process requests and responses. |
| `nginx-module-otel` | [`ngx_otel_module`](https://nginx.org/en/docs/ngx_otel_module.html) | Sends distributed traces in the OpenTelemetry format. |
| `nginx-module-perl` | [`ngx_http_perl_module`](https://nginx.org/en/docs/http/ngx_http_perl_module.html) | Runs location and variable handlers written in Perl. |
| `nginx-module-xslt` | [`ngx_http_xslt_module`](https://nginx.org/en/docs/http/ngx_http_xslt_module.html) | Transforms XML responses with XSLT stylesheets. |

{{< /table >}}

To use a module package:

1. Install the package from the same repository as NGINX. Replace `MODULE_NAME` with a name from the table, for example `njs`:

    - RHEL and derivatives, and Amazon Linux 2023:

        ```shell
        sudo yum install nginx-module-MODULE_NAME
        ```

    - Debian and Ubuntu:

        ```shell
        sudo apt install nginx-module-MODULE_NAME
        ```

    - SLES:

        ```shell
        sudo zypper install nginx-module-MODULE_NAME
        ```

    - Alpine Linux. Use the `@nginx` tag here too:

        ```shell
        sudo apk add nginx-module-MODULE_NAME@nginx
        ```

    The package prints the `load_module` lines that its modules need. It installs the module files in `/usr/lib/nginx/modules` on Debian, Ubuntu, and Alpine Linux, and in `/usr/lib64/nginx/modules` on the other systems. In each case, `/etc/nginx/modules` links to that directory.

2. Add the `load_module` lines at the top of `/etc/nginx/nginx.conf`, before the `events` block. For example, for the njs HTTP module:

    ```nginx
    load_module modules/ngx_http_js_module.so;
    ```

3. Test the configuration and reload NGINX:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

---

## Install from your operating system's repository {#os-default-repository}

Your operating system's repository is the quickest way to get NGINX for learning or testing. For production, use the [nginx.org repository](#prebuilt) instead:

- Your OS vendor builds and updates the package, not the NGINX team.
- The version is often older than the current releases. To compare versions, see the [list of changes](https://nginx.org/en/CHANGES).
- The modules, the file locations, and the default configuration can differ from the nginx.org packages. For example, Alpine Linux's own package answers with `404 Not Found` until you add a site.

Run the commands for your operating system:

- **RHEL and derivatives, and Amazon Linux 2023**:

    ```shell
    sudo dnf install nginx
    ```

    On RHEL 8, 9, and 10 and their derivatives, the nginx package is in the AppStream repository.

- **Debian and Ubuntu**:

    ```shell
    sudo apt update
    sudo apt install nginx
    ```

- **SUSE Linux Enterprise and openSUSE**. On SUSE Linux Enterprise, the nginx package is in the Server Applications Module, so make sure that this module is on:

    ```shell
    sudo zypper refresh
    sudo zypper install nginx
    ```

- **Alpine Linux**. The nginx package is in the `main` repository:

    ```shell
    sudo apk update
    sudo apk add nginx
    ```

- **FreeBSD and DragonFly BSD**:

    ```shell
    sudo pkg update
    sudo pkg install nginx
    ```

- **macOS with [Homebrew](https://brew.sh/)**. Homebrew sets NGINX to listen on port 8080, so that it can run without `sudo`:

    ```shell
    brew update
    brew install nginx
    ```

To find where your package keeps the configuration and the logs, run the `nginx -V` command in [Verify the installation](#verify).

---

## Compile and install from source {#sources}

Build NGINX from source when you need build options or third-party modules that packages don't offer, or library versions that your OS doesn't ship. You build and install every new version yourself.

### Install the build tools and libraries {#dependencies}

NGINX needs a C compiler, `make`, and three libraries: PCRE2 for regular expressions, zlib for gzip compression, and OpenSSL for TLS. You can install the libraries from your operating system's packages, or let the NGINX build compile them from source.

To use your operating system's packages:

- **Debian and Ubuntu**:

    ```shell
    sudo apt update
    sudo apt install build-essential libpcre2-dev zlib1g-dev libssl-dev
    ```

- **RHEL and derivatives, and Amazon Linux 2023**:

    ```shell
    sudo dnf install gcc make pcre2-devel zlib-devel openssl-devel
    ```

On other systems, install a C compiler, `make`, and the development packages of PCRE2, zlib, and OpenSSL.

To build the libraries from source instead, download and unpack them next to the NGINX source. Then pass their directories to `configure`, as described in [Libraries](#libraries). You don't need to run `make install` for them: the NGINX build compiles them and links them into the `nginx` binary. Building OpenSSL from source needs Perl.

```shell
curl -LO https://github.com/PCRE2Project/pcre2/releases/download/pcre2-10.49/pcre2-10.49.tar.gz
curl -LO https://github.com/madler/zlib/releases/download/v1.4/zlib-1.4.tar.gz
curl -LO https://github.com/openssl/openssl/releases/download/openssl-3.5.9/openssl-3.5.9.tar.gz
tar xzf pcre2-10.49.tar.gz
tar xzf zlib-1.4.tar.gz
tar xzf openssl-3.5.9.tar.gz
```

These are the versions that this guide was tested with. For newer versions, see the [PCRE2](https://github.com/PCRE2Project/pcre2/releases), [zlib](https://zlib.net/), and [OpenSSL](https://openssl-library.org/source/) download pages.

### Download the NGINX source {#sources_download}

Download and unpack the source code, and then go to its directory:

```shell
curl -LO https://nginx.org/download/nginx-1.30.7.tar.gz
tar xzf nginx-1.30.7.tar.gz
cd nginx-1.30.7
```

Replace `1.30.7` with the version that you want. The [download page](https://nginx.org/en/download.html) lists the current mainline and stable versions.

### Set the build options {#configure}

The `configure` script checks your system, applies your options, and writes the `Makefile`. Run it in the source directory. To list every option, run `./configure --help`.

This example uses the libraries from your operating system's packages, and adds HTTPS, HTTP/2, and the TCP and UDP proxy:

```shell
./configure \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-stream
```

At the end, `configure` prints a summary of the libraries and paths that the build uses. It starts like this:

```none
Configuration summary
  + using system PCRE2 library
  + using system OpenSSL library
  + using system zlib library

  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx modules path: "/usr/local/nginx/modules"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
```

This example builds the libraries from the sources that you downloaded, and builds the mail proxy as a dynamic module:

```shell
./configure \
    --with-pcre=../pcre2-10.49 \
    --with-pcre-jit \
    --with-zlib=../zlib-1.4 \
    --with-openssl=../openssl-3.5.9 \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-stream \
    --with-mail=dynamic
```

With `--with-mail=dynamic`, `make install` puts `ngx_mail_module.so` in the modules directory. To use the mail proxy, load the module at the top of `nginx.conf`:

```nginx
load_module modules/ngx_mail_module.so;
```

The tables below describe the most common options. For all of them, see [Building nginx from sources](https://nginx.org/en/docs/configure.html).

#### Paths and user {#configure_paths}

`PREFIX` stands for the value of `--prefix`.

{{< table >}}

| Option | What it sets | Default |
|---|---|---|
| `--prefix=PATH` | The directory for the NGINX files. Relative paths in the other options start here. | `/usr/local/nginx` |
| `--sbin-path=PATH` | The path of the `nginx` binary. | `PREFIX/sbin/nginx` |
| `--modules-path=PATH` | The directory for dynamic modules. | `PREFIX/modules` |
| `--conf-path=PATH` | The path of the main configuration file. At startup, `nginx -c FILE` overrides it. | `PREFIX/conf/nginx.conf` |
| `--error-log-path=PATH` | The path of the error log. The [`error_log`](https://nginx.org/en/docs/ngx_core_module.html#error_log) directive can change it later. | `PREFIX/logs/error.log` |
| `--pid-path=PATH` | The file that stores the process ID of the main process. The [`pid`](https://nginx.org/en/docs/ngx_core_module.html#pid) directive can change it later. | `PREFIX/logs/nginx.pid` |
| `--http-log-path=PATH` | The path of the access log. The [`access_log`](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) directive can change it later. | `PREFIX/logs/access.log` |
| `--user=NAME` | The user of the worker processes. The [`user`](https://nginx.org/en/docs/ngx_core_module.html#user) directive can change it later. | `nobody` |
| `--group=NAME` | The group of the worker processes. The [`user`](https://nginx.org/en/docs/ngx_core_module.html#user) directive can change it later. | The group with the same name as the user. For the default user `nobody`, the `nobody` group, or `nogroup` if there's no `nobody` group. |

{{< /table >}}

#### Libraries {#libraries}

{{< table >}}

| Option | What it does |
|---|---|
| `--with-pcre=DIR` | Builds the PCRE2 or PCRE library from the source in `DIR` and links it in. Without this option, `configure` uses the system library. |
| `--with-pcre-jit` | Builds PCRE2 with just-in-time compilation support, for the [`pcre_jit`](https://nginx.org/en/docs/ngx_core_module.html#pcre_jit) directive. |
| `--with-zlib=DIR` | Builds zlib from the source in `DIR` and links it in. Without this option, `configure` uses the system library. |
| `--with-openssl=DIR` | Builds OpenSSL from the source in `DIR` and links it in. Without this option, `configure` uses the system library. |

{{< /table >}}

#### Compiler options

{{< table >}}

| Option | What it does |
|---|---|
| `--with-cc-opt=OPTIONS` | Adds options to the `CFLAGS` variable. With the system PCRE library on FreeBSD, use `--with-cc-opt="-I /usr/local/include"`. To raise the number of files that `select()` supports, use an option such as `--with-cc-opt="-D FD_SETSIZE=2048"`. |
| `--with-ld-opt=OPTIONS` | Adds options for the linker. With the system PCRE library on FreeBSD, use `--with-ld-opt="-L /usr/local/lib"`. |

{{< /table >}}

#### Connection processing methods {#configure_methods}

NGINX picks the most efficient [connection processing method](https://nginx.org/en/docs/events.html) that your platform supports, such as `epoll` or `kqueue`.

{{< table >}}

| Option | What it does |
|---|---|
| `--with-select_module`, `--without-select_module` | Builds, or doesn't build, the module for the `select()` method. `configure` builds it automatically if the platform has no better method, such as `kqueue`, `epoll`, or `/dev/poll`. |
| `--with-poll_module`, `--without-poll_module` | Builds, or doesn't build, the module for the `poll()` method. `configure` builds it automatically if the platform has no better method. |

{{< /table >}}

#### Modules built by default {#modules-built-by-default}

`configure` builds these HTTP modules unless you turn one off with its `--without-` option. For example, `--without-http_empty_gif_module` leaves out the Empty GIF module. If you add the stream or mail modules, `configure` also builds their default modules, such as `stream_access_module` or `mail_smtp_module`. Run `./configure --help` to list them.

{{< table >}}

| Module | What it does |
|---|---|
| [`http_access_module`](https://nginx.org/en/docs/http/ngx_http_access_module.html) | Allows or denies requests by client address. |
| [`http_auth_basic_module`](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html) | Checks user names and passwords with HTTP Basic authentication. |
| [`http_autoindex_module`](https://nginx.org/en/docs/http/ngx_http_autoindex_module.html) | Lists the files of a directory, for requests that end with a slash. |
| [`http_browser_module`](https://nginx.org/en/docs/http/ngx_http_browser_module.html) | Sets variables from the value of the `User-Agent` request header. |
| [`http_charset_module`](https://nginx.org/en/docs/http/ngx_http_charset_module.html) | Adds a character set to the `Content-Type` response header, and converts between character sets. |
| [`http_empty_gif_module`](https://nginx.org/en/docs/http/ngx_http_empty_gif_module.html) | Returns a transparent GIF image of one pixel. |
| [`http_fastcgi_module`](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html) | Passes requests to a FastCGI server. |
| [`http_geo_module`](https://nginx.org/en/docs/http/ngx_http_geo_module.html) | Sets variables from the client IP address. |
| [`http_grpc_module`](https://nginx.org/en/docs/http/ngx_http_grpc_module.html) | Passes requests to a gRPC server. `configure` builds it only together with `--with-http_v2_module`. |
| [`http_gzip_module`](https://nginx.org/en/docs/http/ngx_http_gzip_module.html) | Compresses responses with gzip. Needs the zlib library. |
| [`http_limit_conn_module`](https://nginx.org/en/docs/http/ngx_http_limit_conn_module.html) | Limits the number of connections per key, for example per client address. |
| [`http_limit_req_module`](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html) | Limits the request rate per key, for example per client address. |
| [`http_map_module`](https://nginx.org/en/docs/http/ngx_http_map_module.html) | Sets variables from the values of other variables. |
| [`http_memcached_module`](https://nginx.org/en/docs/http/ngx_http_memcached_module.html) | Gets responses from a memcached server. |
| [`http_mirror_module`](https://nginx.org/en/docs/http/ngx_http_mirror_module.html) | Sends copies of requests to other locations. |
| [`http_proxy_module`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) | Passes requests to another HTTP server. |
| [`http_referer_module`](https://nginx.org/en/docs/http/ngx_http_referer_module.html) | Blocks requests with unwanted values in the `Referer` header. |
| [`http_rewrite_module`](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html) | Changes request URIs with regular expressions, returns redirects, and selects configurations by condition. Needs the PCRE2 library. |
| [`http_scgi_module`](https://nginx.org/en/docs/http/ngx_http_scgi_module.html) | Passes requests to an SCGI server. |
| [`http_split_clients_module`](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html) | Sets variables for A/B testing. |
| [`http_ssi_module`](https://nginx.org/en/docs/http/ngx_http_ssi_module.html) | Processes SSI (Server Side Includes) commands in responses. |
| [`http_tunnel_module`](https://nginx.org/en/docs/http/ngx_http_tunnel_module.html) | Handles HTTP/1.1 `CONNECT` requests and tunnels them to the target server. Mainline 1.31.0 and later. |
| [`http_upstream_hash_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#hash) | Adds the hash load-balancing method. |
| [`http_upstream_ip_hash_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) | Adds the IP hash load-balancing method. |
| [`http_upstream_keepalive_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive) | Keeps connections to upstream servers open for reuse. |
| [`http_upstream_least_conn_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) | Adds the least connections load-balancing method. |
| [`http_upstream_least_time_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_time) | Adds the least time load-balancing method. Mainline 1.31.0 and later. |
| [`http_upstream_random_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#random) | Adds the random load-balancing method. |
| [`http_upstream_sticky_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky) | Adds session persistence. 1.29.6 and later. |
| [`http_upstream_zone_module`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#zone) | Keeps the configuration and state of upstream groups in shared memory. |
| [`http_userid_module`](https://nginx.org/en/docs/http/ngx_http_userid_module.html) | Sets cookies that identify clients. |
| [`http_uwsgi_module`](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html) | Passes requests to a uwsgi server. |

{{< /table >}}

#### Modules not built by default {#modules-not-built-by-default}

To build these modules, add their options to `configure`. Options marked with `=dynamic` can also build the module as a [dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}) instead, for example `--with-stream=dynamic`.

{{< table >}}

| Option | What it does |
|---|---|
| `--with-compat` | Makes the binary compatible with dynamic modules built separately for the same NGINX version with `--with-compat`. |
| `--with-control-api` | Builds the control REST API. To turn it on, start NGINX with the [`-l`](https://nginx.org/en/docs/switches.html) parameter and a UNIX-domain socket or a port. Mainline 1.31.5 and later. |
| `--with-cpp_test_module` | Tests whether the header files compile as C++. |
| `--with-debug` | Turns on the [debugging log]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}}). |
| `--with-file-aio` | Turns on [asynchronous file I/O](https://nginx.org/en/docs/http/ngx_http_core_module.html#aio) on FreeBSD and Linux. |
| `--with-google_perftools_module` | Adds profiling with the [Google Performance Tools](https://github.com/gperftools/gperftools) library. |
| [`--with-http_addition_module`](https://nginx.org/en/docs/http/ngx_http_addition_module.html) | Adds text before and after a response. |
| [`--with-http_auth_request_module`](https://nginx.org/en/docs/http/ngx_http_auth_request_module.html) | Authorizes requests with the result of a subrequest. |
| [`--with-http_dav_module`](https://nginx.org/en/docs/http/ngx_http_dav_module.html) | Manages files with the WebDAV protocol. |
| `--with-http_degradation_module` | Returns an error when memory use exceeds a set size. |
| [`--with-http_flv_module`](https://nginx.org/en/docs/http/ngx_http_flv_module.html) | Adds pseudo-streaming for Flash Video (FLV) files. |
| [`--with-http_geoip_module`](https://nginx.org/en/docs/http/ngx_http_geoip_module.html) | Sets variables from the client IP address with MaxMind GeoIP databases. Needs the GeoIP library. Also `=dynamic`. |
| [`--with-http_gunzip_module`](https://nginx.org/en/docs/http/ngx_http_gunzip_module.html) | Decompresses gzip responses for clients that don't support gzip. |
| [`--with-http_gzip_static_module`](https://nginx.org/en/docs/http/ngx_http_gzip_static_module.html) | Sends precompressed files with the `.gz` extension instead of the regular files. |
| [`--with-http_image_filter_module`](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html) | Transforms images in JPEG, GIF, PNG, and WebP formats. Needs the LibGD library. Also `=dynamic`. |
| [`--with-http_json_module`](https://nginx.org/en/docs/http/ngx_http_json_module.html) | Extracts values from a JSON document in a variable into other variables. Mainline 1.31.5 and later. |
| [`--with-http_mp4_module`](https://nginx.org/en/docs/http/ngx_http_mp4_module.html) | Adds pseudo-streaming for MP4 files. |
| [`--with-http_perl_module`](https://nginx.org/en/docs/http/ngx_http_perl_module.html) | Runs location and variable handlers written in Perl. Needs Perl. Also `=dynamic`. |
| [`--with-http_random_index_module`](https://nginx.org/en/docs/http/ngx_http_random_index_module.html) | Serves a random file from the directory, for requests that end with a slash. |
| [`--with-http_realip_module`](https://nginx.org/en/docs/http/ngx_http_realip_module.html) | Changes the client address to the one sent in a request header field. |
| [`--with-http_secure_link_module`](https://nginx.org/en/docs/http/ngx_http_secure_link_module.html) | Checks that links are authentic and not expired. |
| [`--with-http_slice_module`](https://nginx.org/en/docs/http/ngx_http_slice_module.html) | Splits a request into subrequests that each return a range of the response. This helps to cache large files. |
| [`--with-http_ssl_module`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html) | Adds HTTPS. Needs the OpenSSL library. |
| [`--with-http_stub_status_module`](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) | Shows basic status information. |
| [`--with-http_sub_module`](https://nginx.org/en/docs/http/ngx_http_sub_module.html) | Replaces strings in responses. |
| [`--with-http_v2_module`](https://nginx.org/en/docs/http/ngx_http_v2_module.html) | Adds HTTP/2. |
| [`--with-http_v3_module`](https://nginx.org/en/docs/http/ngx_http_v3_module.html) | Adds HTTP/3. For the SSL libraries that it needs, see [QUIC and HTTP/3 support](https://nginx.org/en/docs/quic.html). |
| [`--with-http_xslt_module`](https://nginx.org/en/docs/http/ngx_http_xslt_module.html) | Transforms XML responses with XSLT stylesheets. Needs the libxml2 and libxslt libraries. Also `=dynamic`. |
| [`--with-mail`](https://nginx.org/en/docs/mail/ngx_mail_core_module.html) | Adds the mail proxy for IMAP, POP3, and SMTP. Also `=dynamic`. |
| [`--with-mail_ssl_module`](https://nginx.org/en/docs/mail/ngx_mail_ssl_module.html) | Adds SSL/TLS to the mail proxy. Needs the OpenSSL library. |
| [`--with-stream`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html) | Adds the TCP and UDP proxy. Also `=dynamic`. |
| [`--with-stream_geoip_module`](https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html) | Sets stream variables from the client IP address with MaxMind GeoIP databases. Needs the GeoIP library. Also `=dynamic`. |
| [`--with-stream_realip_module`](https://nginx.org/en/docs/stream/ngx_stream_realip_module.html) | Changes the client address and port to the ones sent in the PROXY protocol header. |
| [`--with-stream_ssl_module`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html) | Adds SSL/TLS to the TCP proxy. Needs the OpenSSL library. |
| [`--with-stream_ssl_preread_module`](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html) | Reads information, such as the server name, from the TLS ClientHello message. NGINX can then route the connection without decrypting it. |
| `--with-threads` | Turns on [thread pools](https://nginx.org/en/docs/ngx_core_module.html#thread_pool), for example for file reads that would block. |

{{< /table >}}

#### Third-party modules {#third-party}

To build a third-party module, point `configure` to the directory of its source code. The stability of third-party modules isn't guaranteed, so use them at your own risk.

- To link the module into the `nginx` binary:

    ```shell
    ./configure ... --add-module=/path/to/MODULE_SOURCE
    ```

- To build it as a dynamic module, which you load with the `load_module` directive:

    ```shell
    ./configure ... --add-dynamic-module=/path/to/MODULE_SOURCE
    ```

To build a dynamic module for an NGINX binary from a package, use the source code of exactly the same NGINX version, and add `--with-compat`.

### Build and install NGINX {#install}

1. Compile NGINX:

    ```shell
    make
    ```

2. Install it:

    ```shell
    sudo make install
    ```

    `make install` copies the files under the prefix, by default `/usr/local/nginx`. The binary goes to `sbin/`, the configuration to `conf/`, the default page to `html/`, and dynamic modules to `modules/`. NGINX writes its logs to `logs/`.

3. Start NGINX. The binary isn't in your `PATH`, so use its full path:

    ```shell
    sudo /usr/local/nginx/sbin/nginx
    ```

    If you set `--sbin-path`, use that path instead. A build from source doesn't add a system service. To run NGINX as one, create a service file for your init system.

---

## Verify the installation {#verify}

For a build from source, run `/usr/local/nginx/sbin/nginx` instead of `nginx` in these commands. Its configuration file is `/usr/local/nginx/conf/nginx.conf`.

1. Check the version:

    ```shell
    nginx -v
    ```

    The output shows the version that you installed, for example:

    ```none
    nginx version: nginx/1.30.7
    ```

2. Request the default page:

    ```shell
    curl -I http://127.0.0.1/
    ```

    NGINX answers with status `200` and its version. With Homebrew, use port 8080: `http://127.0.0.1:8080/`. A package from your OS can answer differently, for example with `404 Not Found` on Alpine Linux.

    ```none
    HTTP/1.1 200 OK
    Server: nginx/1.30.7
    ```

3. Test the configuration:

    ```shell
    sudo nginx -t
    ```

    For the nginx.org packages, the output is:

    ```none
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

4. List the build options, one per line, to find the files that NGINX uses:

    ```shell
    nginx -V 2>&1 | awk -F: '/configure arguments/ {print $2}' | xargs -n1
    ```

    `--conf-path` is the configuration file, `--error-log-path` is the error log, and `--http-log-path` is the access log. For the nginx.org packages, they're `/etc/nginx/nginx.conf`, `/var/log/nginx/error.log`, and `/var/log/nginx/access.log`. A build from source that didn't set these options doesn't list them: its files are under the prefix, by default `/usr/local/nginx`.

---

## Troubleshooting

### gpg stops with "directory does not exist"

**Symptom**: You check the signing key with the command from older instructions, `gpg --dry-run --quiet --no-keyring --import --import-options import-show FILE`. It prints only the first line of the output and stops with an error such as:

```none
gpg: Fatal: /root/.gnupg: directory does not exist!
```

**Cause**: With `--dry-run`, `gpg` doesn't create its home directory, and stops if the directory doesn't exist yet.

**Fix**: Check the key with `gpg --show-keys FILE`, as in the steps above. Or create the directory first with `mkdir -p -m 700 ~/.gnupg`, and then run the command again.

---

### NGINX fails to start: "Address already in use"

**Symptom**: `sudo nginx` prints this message five times, and then `still could not bind()`:

```none
nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```

On Alpine Linux, the message ends with `(98: Address in use)`.

**Cause**: Another process already listens on port 80. On Debian and Ubuntu, that's often NGINX itself, because the package started it.

**Fix**: Find the process that uses the port:

```shell
sudo ss -ltnp 'sport = :80'
```

If `ss` isn't installed, install the `iproute2` package, or `iproute2-ss` on Alpine Linux. If it's NGINX, manage it with `sudo systemctl reload nginx` or `sudo systemctl restart nginx`, instead of starting a second copy. If it's another web server, stop that server, or change the [`listen`](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) port in the NGINX configuration.

---

### configure stops because a library is missing

**Symptom**: `./configure` stops with one of these messages:

```none
./configure: error: the HTTP rewrite module requires the PCRE library.
./configure: error: the HTTP gzip module requires the zlib library.
./configure: error: SSL modules require the OpenSSL library.
```

**Cause**: The development files of that library aren't installed.

**Fix**: [Install the library packages](#dependencies). Or point `configure` to the library source with `--with-pcre=DIR`, `--with-zlib=DIR`, or `--with-openssl=DIR`. If you don't need the module, turn it off with `--without-http_rewrite_module` or `--without-http_gzip_module`.

---

### nginx: command not found

**Symptom**: After a build from source, the shell can't find the `nginx` command.

**Cause**: `make install` puts the binary in `/usr/local/nginx/sbin/`, which isn't in your `PATH`.

**Fix**: Run the binary with its full path, `/usr/local/nginx/sbin/nginx`. Or add its directory to your `PATH`, or build again with `--sbin-path=/usr/local/sbin/nginx`.

---

### A dynamic module doesn't load: "version ... instead of ..."

**Symptom**: `nginx -t` or a reload fails with a message such as:

```none
nginx: [emerg] module "/etc/nginx/modules/ngx_http_js_module.so" version 1030007 instead of 1031009 in /etc/nginx/nginx.conf:1
```

**Cause**: The module was built for another NGINX version. A dynamic module must match the version of the `nginx` binary exactly.

**Fix**: Install the module package from the same repository and in the same version as NGINX, for example with `sudo apt install nginx nginx-module-njs`. For a module that you build, use the source code of your exact NGINX version.

---

## References

For more information, see:

- [Supported distributions and versions](https://nginx.org/en/linux_packages.html#distributions) and the [installation instructions](https://nginx.org/en/linux_packages.html#instructions) on nginx.org
- [NGINX downloads](https://nginx.org/en/download.html)
- [Building nginx from sources](https://nginx.org/en/docs/configure.html)
- [Changes in NGINX](https://nginx.org/en/CHANGES)
- [Packaging sources](https://github.com/nginx/pkg-oss)
- [Dynamic modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [Deploying NGINX and NGINX Plus with Docker]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-docker.md" >}})
- [Create NGINX Plus and NGINX configuration files]({{< ref "/nginx/admin-guide/basic-functionality/managing-configuration-files.md" >}})
- [Debugging NGINX]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}})
