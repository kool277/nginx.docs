---
title: Image-Filter
description: Install and load the Image-Filter dynamic module in F5 NGINX Plus to resize, crop, and rotate JPEG, GIF, PNG, and WebP images.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "image filter, image_filter, resize images, crop, WebP, nginx-plus-module-image-filter, dynamic module"
f5-summary: >
  This guide shows how to install and load the Image-Filter dynamic module in F5 NGINX Plus.
  The module resizes, crops, and rotates JPEG, GIF, PNG, and WebP images on the fly.
f5-audience: operator
---

## Overview

The Image-Filter dynamic module transforms images in JPEG, GIF, PNG, and WebP formats. It can crop, resize, and rotate them on the fly. It's the [`ngx_http_image_filter_module`](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html) of NGINX, built as a dynamic module.

The `nginx-plus-module-image-filter` package in the NGINX Plus repository contains the module. NGINX-authored modules have [full support]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}) from F5.

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: Dynamic modules are available on the same operating systems as NGINX Plus, with a few exceptions. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

## Install the module package

Install the `nginx-plus-module-image-filter` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-image-filter
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-image-filter
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-image-filter
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-image-filter
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-image-filter
    ```

---

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive:

    ```nginx
    load_module modules/ngx_http_image_filter_module.so;
    ```

---

## Configure the module

Set up the module as described in its [documentation](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html).

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

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_http_image_filter_module.so" failed (/etc/nginx/modules/ngx_http_image_filter_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The module package isn't installed, or the file name in `load_module` is wrong.

**Fix**: Install the package, and check the file names in `/etc/nginx/modules/`, or in `/usr/local/etc/nginx/modules/` on FreeBSD.

---

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "image_filter"`.

**Cause**: The configuration uses a directive of the module, but no `load_module` directive loads the module.

**Fix**: Add the `load_module` directive, as described in [Load the module](#load-the-module).

---

### nginx -t reports that load_module is specified too late

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] "load_module" directive is specified too late in /etc/nginx/nginx.conf:12`.

**Cause**: The `load_module` directive comes after an `events`, `http`, `stream`, or `mail` block.

**Fix**: Move the `load_module` directive to the top of the file, before those blocks.

---

### nginx -t reports that a module is already loaded

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] module "ngx_http_image_filter_module" is already loaded`.

**Cause**: Two `load_module` directives load the same file. This can happen when several modules need the same module, such as the NDK.

**Fix**: Keep only the first `load_module` directive for that file.

---

### nginx -t reports a module version mismatch

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] module "/etc/nginx/modules/ngx_http_image_filter_module.so" version 1029003 instead of 1029008`.

**Cause**: The module package doesn't match the NGINX Plus version, for example after you upgrade NGINX Plus but not the module.

**Fix**: Upgrade the module package to the version of NGINX Plus, as described in [Upgrade NGINX Plus modules]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md#upgrade_modules" >}}).

---

## References

For more information, see:

- [Module ngx_http_image_filter_module](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
