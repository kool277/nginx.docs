---
title: Dynamic Modules
description: Add features to F5 NGINX Plus with dynamic modules from the NGINX Plus repository, and load them into the configuration at startup.
toc: true
weight: 10
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "dynamic modules, load_module, nginx-plus-module, NGINX-authored modules, NGINX-certified community modules, apt-cache search"
f5-summary: >
  This page explains which dynamic modules are available for F5 NGINX Plus and how to install and load them.
  Dynamic modules add features, such as geolocation, image resizing, and scripting with njs or Lua, without a new NGINX Plus binary.
  Each module also has its own page with the package name and the load_module lines.
f5-audience: operator
---

## Overview

F5 NGINX Plus has a modular architecture. Dynamic modules add features to it. Examples are [geolocation by IP address]({{< ref "/nginx/admin-guide/dynamic-modules/geoip2.md" >}}), [image resizing]({{< ref "/nginx/admin-guide/dynamic-modules/image-filter.md" >}}), and scripting with [njs]({{< ref "/nginx/admin-guide/dynamic-modules/nginscript.md" >}}) or [Lua]({{< ref "/nginx/admin-guide/dynamic-modules/lua.md" >}}). NGINX and third-party developers create these modules.

<img src="/nginx/images/nginx-plus-dynamic-module-plug-ins.png" alt="Dynamic modules plug into NGINX Plus to add features" width="500" height="500" style="border:2px solid #666666; padding:2px; margin:2px;" />

A dynamic module is a shared object file (`.so`). NGINX Plus loads it with the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive, when it starts or reloads its configuration. NGINX Plus supports dynamic modules since [Release 9]({{< ref "/nginx/releases.md#r9" >}}).

### NGINX-authored and NGINX-certified dynamic modules

The NGINX Plus repository has packages of [NGINX-authored]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}) and [NGINX-certified community]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-certified-community-dynamic-modules" >}}) dynamic modules. For what F5 supports for each type, see [Install dynamic modules]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_modules" >}}).

For how to install and use a module, select its package name in the table:

{{< table >}}

| Name | Description | Package name |
|---|---|---|
| [ACME](https://github.com/nginx/nginx-acme) | Adds the automatic certificate management ([ACMEv2](https://www.rfc-editor.org/rfc/rfc8555.html)) protocol. | [`nginx-plus-module-acme`]({{< ref "/nginx/admin-guide/dynamic-modules/acme.md" >}}) |
| [Brotli](https://github.com/google/ngx_brotli) | Adds Brotli compression, with modules for dynamic compression and for serving precompressed `.br` files. | [`nginx-plus-module-brotli`]({{< ref "/nginx/admin-guide/dynamic-modules/brotli.md" >}}) |
| [Encrypted-Session](https://github.com/openresty/encrypted-session-nginx-module) | Encrypts and decrypts NGINX variables with AES-256. | [`nginx-plus-module-encrypted-session`]({{< ref "/nginx/admin-guide/dynamic-modules/encrypted-session.md" >}}) |
| [FIPS Status Check](https://github.com/ogarrett/nginx-fips-check-module) | Checks whether OpenSSL runs in FIPS mode. | [`nginx-plus-module-fips-check`]({{< ref "/nginx/admin-guide/dynamic-modules/fips.md" >}}) |
| [GeoIP](https://nginx.org/en/docs/http/ngx_http_geoip_module.html) | Adds IP-based geolocation with the precompiled MaxMind databases. | [`nginx-plus-module-geoip`]({{< ref "/nginx/admin-guide/dynamic-modules/geoip.md" >}}) |
| [GeoIP2](https://github.com/leev/ngx_http_geoip2_module) | Adds geolocation with MaxMind GeoIP2 databases. | [`nginx-plus-module-geoip2`]({{< ref "/nginx/admin-guide/dynamic-modules/geoip2.md" >}}) |
| [Headers-More](https://github.com/openresty/headers-more-nginx-module) | Extends the NGINX [Headers](https://nginx.org/en/docs/http/ngx_http_headers_module.html) module to change request and response headers. | [`nginx-plus-module-headers-more`]({{< ref "/nginx/admin-guide/dynamic-modules/headers-more.md" >}}) |
| [HTTP Substitutions Filter](https://github.com/yaoweibin/ngx_http_substitutions_filter_module) | Replaces strings and regular expression matches in response bodies. | [`nginx-plus-module-subs-filter`]({{< ref "/nginx/admin-guide/dynamic-modules/http-substitutions-filter.md" >}}) |
| [Image-Filter](https://nginx.org/en/docs/http/ngx_http_image_filter_module.html) | Resizes and crops JPEG, GIF, PNG, and WebP images on the fly. | [`nginx-plus-module-image-filter`]({{< ref "/nginx/admin-guide/dynamic-modules/image-filter.md" >}}) |
| [Lua](https://github.com/openresty/lua-nginx-module) | Embeds the Lua programming language. | [`nginx-plus-module-lua`]({{< ref "/nginx/admin-guide/dynamic-modules/lua.md" >}}) |
| [NGINX Developer Kit](https://github.com/vision5/ngx_devel_kit) | Provides helper macros for module development. | [`nginx-plus-module-ndk`]({{< ref "/nginx/admin-guide/dynamic-modules/ndk.md" >}}) |
| [njs Scripting Language](https://nginx.org/en/docs/njs/) | Adds JavaScript-like scripting for advanced server-side logic in the NGINX configuration. | [`nginx-plus-module-njs`]({{< ref "/nginx/admin-guide/dynamic-modules/nginscript.md" >}}) |
| [OpenTelemetry](https://nginx.org/en/docs/ngx_otel_module.html#directives) | Adds distributed tracing with OpenTelemetry. | [`nginx-plus-module-otel`]({{< ref "/nginx/admin-guide/dynamic-modules/opentelemetry.md" >}}) |
| [Perl](https://nginx.org/en/docs/http/ngx_http_perl_module.html) | Adds Perl scripting for advanced customization. | [`nginx-plus-module-perl`]({{< ref "/nginx/admin-guide/dynamic-modules/perl.md" >}}) |
| [Phusion Passenger](https://www.phusionpassenger.com/library/install/nginx/) | An application server for Node.js, Python, and Ruby. | [`nginx-plus-module-passenger`]({{< ref "/nginx/admin-guide/dynamic-modules/passenger-open-source.md" >}}) |
| Prometheus-njs | Converts [NGINX Plus metrics](https://demo.nginx.com/swagger-ui/) into the Prometheus format. | [`nginx-plus-module-prometheus`]({{< ref "/nginx/admin-guide/dynamic-modules/prometheus-njs.md" >}}) |
| [RTMP](https://github.com/arut/nginx-rtmp-module) | Adds streaming: RTMP, HLS, and MPEG-DASH, with FFmpeg support. | [`nginx-plus-module-rtmp`]({{< ref "/nginx/admin-guide/dynamic-modules/rtmp.md" >}}) |
| [Set-Misc](https://github.com/openresty/set-misc-nginx-module) | Adds `set_*` directives for scripting, which extend the NGINX [Rewrite](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html) module. | [`nginx-plus-module-set-misc`]({{< ref "/nginx/admin-guide/dynamic-modules/set-misc.md" >}}) |
| [SPNEGO for Kerberos](https://github.com/stnoonan/spnego-http-auth-nginx-module) | Adds [GSS-API based](https://www.rfc-editor.org/rfc/rfc2743.html) SPNEGO and Kerberos authentication. | [`nginx-plus-module-auth-spnego`]({{< ref "/nginx/admin-guide/dynamic-modules/spnego.md" >}}) |
| [XSLT](https://nginx.org/en/docs/http/ngx_http_xslt_module.html) | Applies XSLT transformations to XML responses. | [`nginx-plus-module-xslt`]({{< ref "/nginx/admin-guide/dynamic-modules/xslt.md" >}}) |

{{< /table >}}

### Community dynamic modules

Community projects, such as the [Awesome NGINX GitHub project](https://github.com/agile6v/awesome-nginx#third-party-modules), list many more third-party modules. You can [compile many of them as dynamic modules]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_modules_oss" >}}) for NGINX Plus or NGINX Open Source.

### NGINX Certified Partner dynamic modules

Commercial third parties sell NGINX Certified Partner modules. Their authors distribute and support them. For the list, see the [F5 Dynamic Modules page](https://www.f5.com/go/product/nginx-modules?filter=module-author%3Anginx-certified-partner). For what F5 supports, see [Install dynamic modules]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_modules" >}}).

---

<span id="prerequisites"></span><span id="caveats"></span>

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: The [latest NGINX Plus release]({{< ref "/nginx/releases.md" >}}), installed from the NGINX Plus repository as described in [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: Some modules aren't available on every operating system because of platform limits. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.

---

<span id="getting-started-with-the-dynamic-modules-repository"></span><span id="displaying-the-list-of-available-modules"></span>

## List the available modules

You install modules from the NGINX Plus repository with your package manager, such as `apt`, `apk`, `dnf`, `pkg`, or `zypper`. To list the module packages on Debian and Ubuntu, run:

```shell
sudo apt update
apt-cache search nginx-plus-module
```

The output lists each module package and its summary. This example leaves out the debugging symbols packages:

```none
nginx-plus-module-acme - NGINX Plus ACME dynamic module
nginx-plus-module-auth-spnego - NGINX Plus 3rd-party kerberos authentication dynamic module
nginx-plus-module-brotli - NGINX Plus 3rd-party brotli compression dynamic modules
nginx-plus-module-encrypted-session - NGINX Plus 3rd-party encrypted session dynamic module
nginx-plus-module-fips-check - NGINX Plus 3rd-party FIPS status check dynamic module
nginx-plus-module-geoip - NGINX Plus GeoIP dynamic modules
nginx-plus-module-geoip2 - NGINX Plus 3rd-party GeoIP2 dynamic modules
nginx-plus-module-headers-more - NGINX Plus 3rd-party headers-more dynamic module
nginx-plus-module-image-filter - NGINX Plus image filter dynamic module
nginx-plus-module-lua - NGINX Plus 3rd-party Lua dynamic modules
nginx-plus-module-ndk - NGINX Plus 3rd-party NDK dynamic module
nginx-plus-module-njs - NGINX Plus njs dynamic modules
nginx-plus-module-otel - NGINX Plus OpenTelemetry dynamic module
nginx-plus-module-passenger - NGINX Plus 3rd-party Passenger dynamic module
nginx-plus-module-perl - NGINX Plus Perl dynamic module
nginx-plus-module-prometheus - NGINX Plus Prometheus exporter NJS module
nginx-plus-module-rtmp - NGINX Plus 3rd-party RTMP dynamic module
nginx-plus-module-set-misc - NGINX Plus 3rd-party set-misc dynamic module
nginx-plus-module-subs-filter - NGINX Plus 3rd-party substitution dynamic module
nginx-plus-module-xslt - NGINX Plus xslt dynamic module
```

The output also lists a package with debugging symbols for each module, for example `nginx-plus-module-njs-dbg`. Install these packages only for troubleshooting, in debugging or development environments. For more information, see [Debugging NGINX]({{< ref "/nginx/admin-guide/monitoring/debugging.md" >}}).

---

<span id="installing-and-loading-the-module"></span>

## Install and load a module

This example installs and loads the [njs dynamic modules]({{< ref "/nginx/admin-guide/dynamic-modules/nginscript.md" >}}) on Ubuntu. The page of each module has the commands for every operating system.

1. Install the module package:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-njs
    ```

    The package puts the `.so` files in `/usr/lib/nginx/modules/` on Debian, Ubuntu, and Alpine Linux, and in `/usr/lib64/nginx/modules/` on most other Linux systems. On FreeBSD, they go to `/usr/local/etc/nginx/modules/`.

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add a [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive for each `.so` file. A relative path starts in the NGINX Plus prefix directory, so `modules/` means `/etc/nginx/modules/` on Linux.

    ```nginx
    load_module modules/ngx_http_js_module.so;
    load_module modules/ngx_stream_js_module.so;

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

1. Test the configuration, and reload it:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

---

<span id="compiling-your-own-dynamic-modules"></span>

## Compile your own dynamic modules

To use a module that isn't in the NGINX Plus repository, compile it yourself. Use the NGINX Open Source version that your NGINX Plus release is based on. For the steps, see [Compile a community dynamic module]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#install_modules_oss" >}}).

---

## Troubleshooting

### apt-cache search lists no module packages

**Symptom**: `apt-cache search nginx-plus-module` shows nothing.

**Cause**: The system doesn't get packages from the NGINX Plus repository yet.

**Fix**: Set up the repository, as described in [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}), and run `sudo apt update`.

---

### A module doesn't load

**Symptom**: `nginx -t` fails with an error about a `load_module` directive.

**Cause**: The package is missing, or the `load_module` directive comes after the `events` block. Or the module package doesn't match the NGINX Plus version.

**Fix**: See the Troubleshooting section on the page of the module.

---

<span id="uninstalling-a-dynamic-module"></span>

## Uninstall a dynamic module

To remove a module, see [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}}).

---

<span id="see-also"></span>

## References

For more information, see:

- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [Compiling third-party dynamic modules for NGINX Plus](https://www.f5.com/company/blog/nginx/compiling-dynamic-modules-nginx-plus)
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
