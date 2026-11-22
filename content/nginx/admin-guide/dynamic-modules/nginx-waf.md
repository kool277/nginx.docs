---
title: NGINX ModSecurity WAF
description: Remove the NGINX ModSecurity WAF dynamic module from F5 NGINX Plus. The module reached its end of life on March 31, 2024.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "modsecurity, NGINX ModSecurity WAF, nginx-plus-module-modsecurity, end of life, F5 WAF for NGINX"
f5-summary: >
  This guide shows how to remove the NGINX ModSecurity WAF dynamic module from F5 NGINX Plus.
  The module reached its end of life on March 31, 2024, and its package is no longer in the NGINX Plus repository.
  F5 WAF for NGINX is the web application firewall that F5 offers for NGINX Plus.
f5-audience: operator
---

## Overview

The NGINX ModSecurity WAF dynamic module protected applications against Layer 7 attacks, such as SQL injection and cross-site scripting. F5 stopped selling it on April 1, 2022. [NGINX Plus R29]({{< ref "/nginx/releases.md#r29" >}}) no longer supports it, and it reached its end of life on March 31, 2024. Since [NGINX Plus R32]({{< ref "/nginx/releases.md#r32" >}}), the `nginx-plus-module-modsecurity` package isn't in the NGINX Plus repository. For details, see the [end-of-life announcement](https://www.f5.com/company/blog/nginx/f5-nginx-modsecurity-waf-transitioning-to-eol).

To keep a web application firewall in front of your applications, use [F5 WAF for NGINX]({{< ref "/waf/_index.md" >}}), formerly NGINX App Protect WAF.

---

## Before you begin

Before you begin, make sure you have:

- **A replacement plan**: When you remove the module, NGINX Plus no longer filters attacks. Set up your replacement first, or accept that gap.
- **Root access**: You need `sudo` to change the configuration.

---

## Remove the module

1. Find the module's directives in the configuration. All of them start with `modsecurity`:

    ```shell
    sudo grep -rn modsecurity /etc/nginx/
    ```

    The output shows the file and line number of each directive, including its `load_module` directive.

1. Delete the `modsecurity` directives, such as `modsecurity on;` and `modsecurity_rules_file`.

1. Delete the `load_module` directive of the module: `load_module modules/ngx_http_modsecurity_module.so;`

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

1. If the `nginx-plus-module-modsecurity` package is still installed, remove it as described in [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}}).

---

## Troubleshooting

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "modsecurity"`.

**Cause**: NGINX Plus no longer loads the module, but the configuration still uses its directives.

**Fix**: Delete the directives, as described in [Remove the module](#remove-the-module).

---

## References

For more information, see:

- [F5 NGINX ModSecurity WAF Is Transitioning to End-of-Life](https://www.f5.com/company/blog/nginx/f5-nginx-modsecurity-waf-transitioning-to-eol)
- [F5 WAF for NGINX]({{< ref "/waf/_index.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
