---
title: NGINX Developer Kit
description: Install and load the NGINX Developer Kit (NDK) dynamic module in F5 NGINX Plus, which the Encrypted-Session, Lua, and Set-Misc modules need.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "NGINX Developer Kit, NDK, ngx_devel_kit, nginx-plus-module-ndk, dynamic module"
f5-summary: >
  This guide shows how to install and load the NGINX Developer Kit (NDK) dynamic module in F5 NGINX Plus.
  The NDK helps developers write NGINX modules, and other modules depend on it.
  The Encrypted-Session, Lua, and Set-Misc modules need the NDK.
f5-audience: operator
---

## Overview

The NGINX Developer Kit (NDK) helps developers write NGINX modules. The [Encrypted-Session]({{< ref "/nginx/admin-guide/dynamic-modules/encrypted-session.md" >}}), [Lua]({{< ref "/nginx/admin-guide/dynamic-modules/lua.md" >}}), and [Set-Misc]({{< ref "/nginx/admin-guide/dynamic-modules/set-misc.md" >}}) modules need it. It's an [NGINX-certified community module](https://github.com/vision5/ngx_devel_kit) that F5 packages for NGINX Plus.

The `nginx-plus-module-ndk` package in the NGINX Plus repository contains the module. For NGINX-certified community modules, F5 supports the [installation and basic configuration]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-certified-community-dynamic-modules" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: Dynamic modules are available on the same operating systems as NGINX Plus, with a few exceptions. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

## Install the module package

Install the `nginx-plus-module-ndk` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-ndk
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-ndk
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-ndk
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-ndk
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-ndk
    ```

---

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive:

    ```nginx
    load_module modules/ndk_http_module.so;
    ```

    Load the NDK once, before the modules that need it.

---

## Configure the module

You don't set up the NDK itself. Load it, and then set up the module that needs it.

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

---

## Troubleshooting

### nginx -t reports that dlopen() failed

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ndk_http_module.so" failed (/etc/nginx/modules/ndk_http_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The module package isn't installed, or the file name in `load_module` is wrong.

**Fix**: Install the package, and check the file names in `/etc/nginx/modules/`, or in `/usr/local/etc/nginx/modules/` on FreeBSD.

---

### nginx -t reports that load_module is specified too late

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] "load_module" directive is specified too late in /etc/nginx/nginx.conf:12`.

**Cause**: The `load_module` directive comes after an `events`, `http`, `stream`, or `mail` block.

**Fix**: Move the `load_module` directive to the top of the file, before those blocks.

---

### nginx -t reports that a module is already loaded

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] module "ndk_http_module" is already loaded`.

**Cause**: Two `load_module` directives load the same file. This can happen when several modules need the same module, such as the NDK.

**Fix**: Keep only the first `load_module` directive for that file.

---

### nginx -t reports a module version mismatch

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] module "/etc/nginx/modules/ndk_http_module.so" version 1029003 instead of 1029008`.

**Cause**: The module package doesn't match the NGINX Plus version, for example after you upgrade NGINX Plus but not the module.

**Fix**: Upgrade the module package to the version of NGINX Plus, as described in [Upgrade NGINX Plus modules]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md#upgrade_modules" >}}).

---

## References

For more information, see:

- [The ngx_devel_kit project on GitHub](https://github.com/vision5/ngx_devel_kit)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
