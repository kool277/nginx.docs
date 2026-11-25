---
title: Lua
description: Install and load the Lua dynamic modules in F5 NGINX Plus to run Lua code in the NGINX event-processing model for HTTP and TCP/UDP.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "lua, lua-nginx-module, stream-lua-nginx-module, content_by_lua_block, nginx-plus-module-lua, NDK, dynamic module"
f5-summary: >
  This guide shows how to install and load the Lua dynamic modules in F5 NGINX Plus.
  The modules run Lua coroutines in the NGINX event-processing model, for HTTP and for TCP and UDP traffic.
  They need the NGINX Developer Kit (NDK) module.
f5-audience: operator
---

## Overview

The Lua dynamic modules integrate Lua coroutines into the NGINX event-processing model. [`lua-nginx-module`](https://github.com/openresty/lua-nginx-module) works with HTTP, and [`stream-lua-nginx-module`](https://github.com/openresty/stream-lua-nginx-module) with TCP and UDP. They're NGINX-certified community modules that F5 packages for NGINX Plus.

The `nginx-plus-module-lua` package in the NGINX Plus repository contains the modules. For NGINX-certified community modules, F5 supports the [installation and basic configuration]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-certified-community-dynamic-modules" >}}).

---

<span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: Dynamic modules are available on the same operating systems as NGINX Plus, with a few exceptions. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **The NDK module**: This module needs the [NGINX Developer Kit (NDK)]({{< ref "/nginx/admin-guide/dynamic-modules/ndk.md" >}}) module, `nginx-plus-module-ndk`. If the package manager doesn't install it with this package, install it first.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

<span id="installation"></span>

## Install the module package

Install the `nginx-plus-module-lua` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-lua
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-lua
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-lua
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-lua
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-lua
    ```

---

<span id="configuration"></span>

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directives:

    ```nginx
    load_module modules/ndk_http_module.so;
    load_module modules/ngx_http_lua_module.so;
    load_module modules/ngx_stream_lua_module.so;
    ```

    Load `ndk_http_module.so` first, and only once, even if other modules need it too. Load the stream module only if you use Lua in the `stream` context.

---

## Configure the module

Set up the modules as described in the documentation of [`lua-nginx-module`](https://github.com/openresty/lua-nginx-module) and [`stream-lua-nginx-module`](https://github.com/openresty/stream-lua-nginx-module).

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

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_stream_lua_module.so" failed (/etc/nginx/modules/ngx_stream_lua_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The module package isn't installed, or the file name in `load_module` is wrong.

**Fix**: Install the package, and check the file names in `/etc/nginx/modules/`, or in `/usr/local/etc/nginx/modules/` on FreeBSD.

---

### nginx -t reports an undefined symbol

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_stream_lua_module.so" failed (/etc/nginx/modules/ngx_stream_lua_module.so: undefined symbol: ...)`.

**Cause**: The NDK module isn't loaded before this module.

**Fix**: Put `load_module modules/ndk_http_module.so;` before the `load_module` directive of this module.

---

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "content_by_lua_block"`.

**Cause**: The configuration uses a directive of the module, but no `load_module` directive loads the module.

**Fix**: Add the `load_module` directive, as described in [Load the module](#load-the-module).

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

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] module "/etc/nginx/modules/ngx_stream_lua_module.so" version 1029003 instead of 1029008`.

**Cause**: The module package doesn't match the NGINX Plus version, for example after you upgrade NGINX Plus but not the module.

**Fix**: Upgrade the module package to the version of NGINX Plus, as described in [Upgrade NGINX Plus modules]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md#upgrade_modules" >}}).

---

<span id="more-info"></span>

## References

For more information, see:

- [The lua-nginx-module project on GitHub](https://github.com/openresty/lua-nginx-module)
- [The stream-lua-nginx-module project on GitHub](https://github.com/openresty/stream-lua-nginx-module)
- [NGINX Developer Kit]({{< ref "/nginx/admin-guide/dynamic-modules/ndk.md" >}})
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
