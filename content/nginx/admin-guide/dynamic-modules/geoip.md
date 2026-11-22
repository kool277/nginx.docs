---
title: GeoIP
description: Install and load the GeoIP dynamic module in F5 NGINX Plus to set variables from the client IP address with MaxMind legacy GeoIP databases.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "geoip, MaxMind, legacy GeoIP database, geolocation, nginx-plus-module-geoip, dynamic module"
f5-summary: >
  This guide shows how to install and load the GeoIP dynamic module in F5 NGINX Plus.
  The module sets variables from the client IP address, with data from MaxMind legacy GeoIP databases.
  MaxMind no longer updates these databases, so use the GeoIP2 module instead.
f5-audience: operator
---

## Overview

The GeoIP dynamic module sets variables from the client IP address, with data from MaxMind GeoIP databases in the legacy format. It works for HTTP with the [`ngx_http_geoip_module`](https://nginx.org/en/docs/http/ngx_http_geoip_module.html), and for TCP and UDP with the [`ngx_stream_geoip_module`](https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html).

{{< call-out class="note" >}} MaxMind [discontinued](https://blog.maxmind.com/2018/01/discontinuation-of-the-geolite-legacy-databases) the GeoLite Legacy databases. Use MaxMind GeoIP2 or GeoLite2 databases with the [GeoIP2 module]({{< ref "/nginx/admin-guide/dynamic-modules/geoip2.md" >}}) instead. {{< /call-out >}}

The `nginx-plus-module-geoip` package in the NGINX Plus repository contains the modules. NGINX-authored modules have [full support]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}) from F5.

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: This module is available on Debian, Ubuntu, SLES 15, and Alpine Linux only. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

## Install the module package

Install the `nginx-plus-module-geoip` package from the NGINX Plus repository:

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-geoip
    ```

- SLES 15:

    ```shell
    sudo zypper install nginx-plus-module-geoip
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-geoip
    ```

---

## Load the module

1. Open the NGINX Plus configuration file, `/etc/nginx/nginx.conf`.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directives:

    ```nginx
    load_module modules/ngx_http_geoip_module.so;
    load_module modules/ngx_stream_geoip_module.so;
    ```

    Load the stream module only if you use the module in the `stream` context.

---

## Configure the module

Set up the module as described in its documentation for [HTTP](https://nginx.org/en/docs/http/ngx_http_geoip_module.html) and for [TCP and UDP](https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html).

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

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_stream_geoip_module.so" failed (/etc/nginx/modules/ngx_stream_geoip_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The module package isn't installed, or the file name in `load_module` is wrong.

**Fix**: Install the package, and check the file names in `/etc/nginx/modules/`.

---

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "geoip_country"`.

**Cause**: The configuration uses a directive of the module, but no `load_module` directive loads the module.

**Fix**: Add the `load_module` directive, as described in [Load the module](#load-the-module).

---

### nginx -t reports that load_module is specified too late

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] "load_module" directive is specified too late in /etc/nginx/nginx.conf:12`.

**Cause**: The `load_module` directive comes after an `events`, `http`, `stream`, or `mail` block.

**Fix**: Move the `load_module` directive to the top of the file, before those blocks.

---

### nginx -t reports that a module is already loaded

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] module "ngx_stream_geoip_module" is already loaded`.

**Cause**: Two `load_module` directives load the same file. This can happen when several modules need the same module, such as the NDK.

**Fix**: Keep only the first `load_module` directive for that file.

---

### nginx -t reports a module version mismatch

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] module "/etc/nginx/modules/ngx_stream_geoip_module.so" version 1029003 instead of 1029008`.

**Cause**: The module package doesn't match the NGINX Plus version, for example after you upgrade NGINX Plus but not the module.

**Fix**: Upgrade the module package to the version of NGINX Plus, as described in [Upgrade NGINX Plus modules]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md#upgrade_modules" >}}).

---

## References

For more information, see:

- [GeoIP2 dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/geoip2.md" >}})
- [Restricting access by geographical location]({{< ref "/nginx/admin-guide/security-controls/controlling-access-by-geoip.md" >}})
- [Module ngx_http_geoip_module](https://nginx.org/en/docs/http/ngx_http_geoip_module.html)
- [Module ngx_stream_geoip_module](https://nginx.org/en/docs/stream/ngx_stream_geoip_module.html)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
