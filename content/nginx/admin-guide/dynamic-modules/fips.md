---
title: FIPS Status Check
description: Install the FIPS Status Check dynamic module in F5 NGINX Plus, which logs at startup whether OpenSSL runs in FIPS mode.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "FIPS, FIPS 140-2, OpenSSL FIPS mode, fips-check, nginx-plus-module-fips-check, dynamic module"
f5-summary: >
  This guide shows how to install and load the FIPS Status Check dynamic module in F5 NGINX Plus.
  At startup, the module writes to the error log whether the OpenSSL library runs in FIPS mode.
  This helps you check compliance with the FIPS 140-2 security requirements for cryptographic modules.
f5-audience: operator
---

## Overview

For F5 NGINX Plus, the cryptographic boundary includes all functionality that the [`http_ssl`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html), [`http_v2`](https://nginx.org/en/docs/http/ngx_http_v2_module.html), [`stream_ssl`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html), and [`mail_ssl`](https://nginx.org/en/docs/mail/ngx_mail_ssl_module.html) modules implement. These modules handle SSL and TLS for inbound and outbound connections over HTTP, HTTP/2, TCP, and mail protocols.

The FIPS Status Check module checks the OpenSSL library that NGINX Plus uses. At startup, it writes to the error log whether the library runs in FIPS mode. For the FIPS requirements, see [FIPS compliance for NGINX Plus]({{< ref "/nginx/fips-compliance-nginx-plus.md" >}}).

The `nginx-plus-module-fips-check` package in the NGINX Plus repository contains the module. For NGINX-certified community modules, F5 supports the [installation and basic configuration]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-certified-community-dynamic-modules" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: Dynamic modules are available on the same operating systems as NGINX Plus, with a few exceptions. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

## Install the module package

Install the `nginx-plus-module-fips-check` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-fips-check
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-fips-check
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-fips-check
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-fips-check
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-fips-check
    ```

---

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive:

    ```nginx
    load_module modules/ngx_fips_check_module.so;
    ```

---

## Configure the module

The module has no directives. After you load it, it reports the FIPS mode of OpenSSL each time NGINX Plus starts.

---

## Verify the installation

1. Test the configuration:

    ```shell
    sudo nginx -t
    ```

    The output ends with a successful test:

    ```none
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

1. Reload the configuration:

    ```shell
    sudo nginx -s reload
    ```

1. Check the error log for the report of the module:

    ```shell
    sudo grep "FIPS Mode" /var/log/nginx/error.log
    ```

    The output shows whether OpenSSL runs in FIPS mode, for example:

    ```none
    2026/11/22 10:00:00 [notice] 1234#1234: OpenSSL FIPS Mode is enabled
    ```

    The module logs this line at the `notice` level, once when NGINX Plus starts or reloads with the module. If the `error_log` directive sets a higher level, such as `warn`, the log doesn't show it.

---

## Troubleshooting

### nginx -t reports that dlopen() failed

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_fips_check_module.so" failed (/etc/nginx/modules/ngx_fips_check_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The module package isn't installed, or the file name in `load_module` is wrong.

**Fix**: Install the package, and check the file names in `/etc/nginx/modules/`, or in `/usr/local/etc/nginx/modules/` on FreeBSD.

---

### nginx -t reports that load_module is specified too late

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] "load_module" directive is specified too late in /etc/nginx/nginx.conf:12`.

**Cause**: The `load_module` directive comes after an `events`, `http`, `stream`, or `mail` block.

**Fix**: Move the `load_module` directive to the top of the file, before those blocks.

---

### nginx -t reports that a module is already loaded

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] module "ngx_fips_check_module" is already loaded`.

**Cause**: Two `load_module` directives load the same file. This can happen when several modules need the same module, such as the NDK.

**Fix**: Keep only the first `load_module` directive for that file.

---

### nginx -t reports a module version mismatch

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] module "/etc/nginx/modules/ngx_fips_check_module.so" version 1029003 instead of 1029008`.

**Cause**: The module package doesn't match the NGINX Plus version, for example after you upgrade NGINX Plus but not the module.

**Fix**: Upgrade the module package to the version of NGINX Plus, as described in [Upgrade NGINX Plus modules]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md#upgrade_modules" >}}).

---

## References

For more information, see:

- [FIPS compliance for NGINX Plus]({{< ref "/nginx/fips-compliance-nginx-plus.md" >}})
- [The nginx-fips-check-module project on GitHub](https://github.com/ogarrett/nginx-fips-check-module)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
