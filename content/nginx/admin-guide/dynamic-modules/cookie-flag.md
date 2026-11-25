---
title: Cookie-Flag
description: Replace the removed Cookie-Flag dynamic module in F5 NGINX Plus with the built-in proxy_cookie_flags directive.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "cookie-flag, set_cookie_flag, proxy_cookie_flags, HttpOnly, Secure, SameSite, nginx-plus-module-cookie-flag"
f5-summary: >
  This guide shows how to replace the Cookie-Flag dynamic module with the built-in proxy_cookie_flags directive.
  F5 removed the module package from the NGINX Plus repository in NGINX Plus R26.
  The directive sets the HttpOnly, Secure, and SameSite flags on cookies from a proxied server.
f5-audience: operator
---

## Overview

The Cookie-Flag dynamic module set the `HttpOnly`, `Secure`, and `SameSite` flags on cookies in the `Set-Cookie` headers of proxied responses. F5 deprecated it in [NGINX Plus R23]({{< ref "/nginx/releases.md#r23" >}}) and removed the `nginx-plus-module-cookie-flag` package in [NGINX Plus R26]({{< ref "/nginx/releases.md#r26" >}}).

The [`proxy_cookie_flags`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_flags) directive does the same job without a module. It's available since NGINX Plus R23 and NGINX Open Source 1.19.3. When you finish this guide, the directive sets the cookie flags, and the module is gone.

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus R23 or later**: For how to upgrade, see [Upgrade NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/upgrading-nginx-plus.md" >}}).
- **Root access**: You need `sudo` to change the configuration.

---

## Replace the set_cookie_flag directives

1. Find the `set_cookie_flag` directives in the configuration:

    ```shell
    sudo grep -rn set_cookie_flag /etc/nginx/
    ```

    The output shows the file and line number of each directive.

1. Replace each `set_cookie_flag` directive with a `proxy_cookie_flags` directive in the same block. Translate the parameters with this table:

    {{< table >}}

   | `set_cookie_flag` parameter | `proxy_cookie_flags` parameter |
   |---|---|
   | A cookie name, such as `Secret` | The same cookie name |
   | `*`, which matches every cookie | `~`, a regular expression that matches every cookie name |
   | `HttpOnly` | `httponly` |
   | `secure` | `secure` |
   | `SameSite=Lax` | `samesite=lax` |
   | `SameSite=Strict` | `samesite=strict` |
   | `SameSite` without a value | No equivalent. Choose `samesite=lax`, `samesite=strict`, or `samesite=none`. |

    {{< /table >}}

    For example, these directives set three flags on the `Secret` cookie and the `HttpOnly` flag on every other cookie:

    ```nginx
    location / {
        proxy_cookie_flags Secret httponly secure samesite=strict;
        proxy_cookie_flags ~ httponly;
        proxy_pass http://backend.example.com;
    }
    ```

    NGINX Plus applies only the first `proxy_cookie_flags` directive whose cookie name matches. Put the directives for named cookies before the one with `~`.

1. Delete the `load_module` directive of the module: `load_module modules/ngx_http_cookie_flag_filter_module.so;`

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

1. If the `nginx-plus-module-cookie-flag` package is still installed, remove it as described in [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}}).

---

## Verify the cookie flags

Send a request to a location that sets cookies, and show the `Set-Cookie` headers. Replace `localhost` with your server name:

```shell
curl -si http://localhost/ | grep -i '^set-cookie'
```

With the example configuration, the output shows the flags:

```none
Set-Cookie: Secret=abc; Path=/; Secure; HttpOnly; SameSite=Strict
Set-Cookie: theme=dark; HttpOnly
```

---

## Troubleshooting

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] unknown directive "set_cookie_flag"`.

**Cause**: NGINX Plus no longer loads the Cookie-Flag module, but the configuration still uses its directive.

**Fix**: Replace the directive with `proxy_cookie_flags`, as described in [Replace the set_cookie_flag directives](#replace-the-set_cookie_flag-directives).

---

### A cookie doesn't get the expected flags

**Symptom**: The `Set-Cookie` header doesn't have the flags that you set for the cookie.

**Cause**: An earlier `proxy_cookie_flags` directive in the same block also matches the cookie, so NGINX Plus applies that one. Or the cookie doesn't come from a proxied server: the directive doesn't change cookies from `fastcgi_pass`, `uwsgi_pass`, or other modules.

**Fix**: Move the directive for the cookie above the directives with regular expressions, such as `~`. For a cookie from another module, set the flags in the application that sets the cookie.

---

<span id="more-info"></span>

## References

For more information, see:

- [The `proxy_cookie_flags` directive](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cookie_flags)
- [Announcing NGINX Plus R23](https://www.f5.com/company/blog/nginx/nginx-plus-r23-released)
- [The nginx_cookie_flag_module project on GitHub](https://github.com/AirisX/nginx_cookie_flag_module)
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
