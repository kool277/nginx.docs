---
title: Uninstall a dynamic module
description: Remove a dynamic module from F5 NGINX Plus. Take it out of the configuration first, and then uninstall its package.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "uninstall dynamic module, remove module, load_module, nginx-plus-module, dnf remove, apt remove"
f5-summary: >
  This guide shows how to remove a dynamic module from F5 NGINX Plus.
  You take the module out of the configuration first, so NGINX Plus never loads a module file that's gone.
  Then you uninstall the module package with your package manager.
f5-audience: operator
---

## Overview

You might uninstall a dynamic module when:

- A replacement exists. For example, the [OpenTelemetry]({{< ref "/nginx/admin-guide/dynamic-modules/opentelemetry.md" >}}) module replaces the [OpenTracing]({{< ref "/nginx/admin-guide/dynamic-modules/opentracing.md" >}}) module, and the `proxy_cookie_flags` directive replaces the [Cookie-Flag]({{< ref "/nginx/admin-guide/dynamic-modules/cookie-flag.md" >}}) module.
- F5 no longer supports the module, such as the [NGINX ModSecurity WAF]({{< ref "/nginx/admin-guide/dynamic-modules/nginx-waf.md" >}}) module.
- The module doesn't support your operating system. For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- You no longer need the module, it causes errors, or your security policy doesn't allow it.

Remove the module from the configuration before you uninstall its package. If the configuration loads a module file that's gone, NGINX Plus can't start, and a reload keeps the old configuration.

---

## Before you begin

Before you begin, make sure you have:

- **The package name of the module**: Each module's page lists its package, for example `nginx-plus-module-njs`. For the list, see [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}).
- **Root access**: You need `sudo` to change the configuration and to remove packages.

---

<span id="configuration"></span>

## Remove the module from the configuration

1. Find the `load_module` directives in the configuration:

    ```shell
    sudo grep -rn load_module /etc/nginx/
    ```

    On FreeBSD, search `/usr/local/etc/nginx/` instead. The output shows the file and line number of each directive.

1. Delete the `load_module` directives of the module. The module's page lists them.

1. Delete or replace the directives of the module. The module's documentation lists them.

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

<span id="instructions"></span>

## Uninstall the module package

Run the command for your operating system. Replace `<MODULE_PACKAGE>` with the package name, for example `nginx-plus-module-njs`.

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf remove <MODULE_PACKAGE>
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt remove <MODULE_PACKAGE>
    ```

- SLES:

    ```shell
    sudo zypper remove <MODULE_PACKAGE>
    ```

- Alpine Linux:

    ```shell
    sudo apk del <MODULE_PACKAGE>
    ```

- FreeBSD:

    ```shell
    sudo pkg delete <MODULE_PACKAGE>
    ```

Other module packages can depend on the one you remove. For example, the Lua package needs the NDK package. In that case, the package manager might also remove them, or refuse to remove the package. Read its output before you confirm.

---

## Troubleshooting

### nginx -t reports that dlopen() failed

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] dlopen() "/etc/nginx/modules/ngx_http_js_module.so" failed (/etc/nginx/modules/ngx_http_js_module.so: cannot open shared object file: No such file or directory)`.

**Cause**: The package is gone, but a `load_module` directive still loads its module.

**Fix**: Delete the `load_module` directive, as described in [Remove the module from the configuration](#remove-the-module-from-the-configuration).

---

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "js_import"`.

**Cause**: The configuration still uses a directive of the module that you removed.

**Fix**: Delete or replace the directive, and test the configuration again.

---

<span id="more-info"></span>

## References

For more information, see:

- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [NGINX Plus Technical Specifications]({{< ref "/nginx/technical-specs.md" >}})
- [The `load_module` directive](https://nginx.org/en/docs/ngx_core_module.html#load_module)
