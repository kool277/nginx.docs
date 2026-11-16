---
description: Control access using HTTP Basic authentication, and optionally in combination
  with IP address-based access control.
title: Restricting Access with HTTP Basic Authentication
toc: true
weight: 300
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "basic authentication, auth_basic, htpasswd, allow, deny, satisfy, 401"
f5-summary: >
  This page explains how to protect a website or part of it with a user name and password, using HTTP Basic authentication.
  It also shows how to combine the password check with restrictions by client IP address.
  It applies to both NGINX Open Source and NGINX Plus.
f5-audience: operator
---

<span id="intro"></span>

## Introduction

With HTTP Basic authentication, NGINX Open Source and F5 NGINX Plus ask clients for a user name and password before they give access to a website or part of it. NGINX checks the credentials against a password file, which you create with a tool such as `htpasswd`.

You can combine Basic authentication with other access restrictions, for example by [IP address]({{< ref "/nginx/admin-guide/security-controls/denylisting-ip-addresses.md" >}}) or by [geographical location]({{< ref "/nginx/admin-guide/security-controls/controlling-access-by-geoip.md" >}}).

Basic authentication sends the password in every request, encoded but not encrypted. Use it only over HTTPS. Refer to [NGINX SSL Termination]({{< ref "/nginx/admin-guide/security-controls/terminating-ssl-http.md" >}}).

<span id="prereq"></span>

## Prerequisites

- NGINX Open Source or NGINX Plus.
- A tool to create the password file, such as `htpasswd` from the `apache2-utils` package (Debian, Ubuntu) or the `httpd-tools` package (RHEL, CentOS, Oracle Linux).
- Permission to edit the configuration and reload NGINX. On most systems, this means root or `sudo` access.

---

## How access checks work

NGINX checks access to a location in its access phase:

- The [allow](https://nginx.org/en/docs/http/ngx_http_access_module.html#allow) and [deny](https://nginx.org/en/docs/http/ngx_http_access_module.html#deny) directives check the client IP address.
- The [auth_basic](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic) directive checks the user name and password.

The [satisfy](https://nginx.org/en/docs/http/ngx_http_core_module.html#satisfy) directive decides how the checks combine. With `satisfy all`, the default, the request must pass every check. With `satisfy any`, one passing check is enough.

```mermaid
flowchart TD
    req([Request to a protected location]) --> mode{"satisfy"}
    mode -- "all (default)" --> ip_all{"IP address allowed?"}
    ip_all -- No --> deny([403 Forbidden])
    ip_all -- Yes --> pw_all{"Valid user name and password?"}
    pw_all -- No --> auth([401 Unauthorized, asks for credentials])
    pw_all -- Yes --> ok([Process the request])
    mode -- any --> ip_any{"IP address allowed?"}
    ip_any -- Yes --> ok
    ip_any -- No --> pw_any{"Valid user name and password?"}
    pw_any -- Yes --> ok
    pw_any -- No --> auth
```

Text description: With `satisfy all`, NGINX returns `403` if the IP address isn't allowed, and `401` if the credentials aren't valid; it processes the request only if both checks pass. With `satisfy any`, NGINX processes the request if either the IP address is allowed or the credentials are valid, and returns `401` otherwise.

---

<span id="pass"></span>

## Creating a password file

1. Make sure `htpasswd` is installed. It's in the `apache2-utils` package (Debian, Ubuntu) or the `httpd-tools` package (RHEL, CentOS, Oracle Linux).

2. Create a password file and the first user. Run `htpasswd` with the `-c` flag, which creates a new file, the path of the file, and the user name. On systems without the `/etc/apache2` directory, such as RHEL, create it first:

    ```shell
    sudo mkdir -p /etc/apache2
    sudo htpasswd -c /etc/apache2/.htpasswd user1
    ```

    Type the password for **user1** when prompted.

3. Add more users without the `-c` flag, because the file already exists. With `-c`, `htpasswd` would replace the file:

    ```shell
    sudo htpasswd /etc/apache2/.htpasswd user2
    ```

4. Check that the file contains the user names and hashed passwords:

    ```shell
    sudo cat /etc/apache2/.htpasswd
    ```

    ```none
    user1:$apr1$...
    user2:$apr1$...
    ```

5. Let the NGINX worker processes read the file. Use the group of the user that the worker processes run as, set by the `user` directive: `nginx` for NGINX Plus and the packages from nginx.org, `www-data` for Debian and Ubuntu packages. To check it, run `ps -o user= -C nginx`. For example:

    ```shell
    sudo chown root:nginx /etc/apache2/.htpasswd
    sudo chmod 640 /etc/apache2/.htpasswd
    ```

NGINX supports these password formats: the `$apr1$` format, which `htpasswd` uses by default; `{SHA}` and `{SSHA}`; and the formats of the system's `crypt()` function, such as bcrypt (`htpasswd -B`), on most current Linux systems. NGINX also accepts passwords in clear text with the `{PLAIN}` prefix, which you have to write into the file yourself; `htpasswd -p` writes clear text without this prefix, which doesn't work. NGINX checks the password on every request, so a high bcrypt cost (`htpasswd -B -C`) slows down every request; keep the default cost.

<span id="config"></span>

## Configuring NGINX and NGINX Plus for HTTP Basic Authentication

1. In the location to protect, add the [auth_basic](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic) directive with the name of the protected area, called the realm. NGINX sends it in the `WWW-Authenticate` header; some browsers show it when they ask for credentials. Use plain ASCII characters in it:

    ```nginx
    location /api {
        auth_basic "Administrator's Area";
    }
    ```

2. Add the [auth_basic_user_file](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic_user_file) directive with the path of the password file. Both directives are needed: without `auth_basic_user_file`, NGINX doesn't ask for credentials. A relative path is relative to the directory of the main configuration file, so `htpasswd` means `/etc/nginx/htpasswd` for `/etc/nginx/nginx.conf`.

    ```nginx
    location /api {
        auth_basic           "Administrator's Area";
        auth_basic_user_file /etc/apache2/.htpasswd;
    }
    ```

3. Test and reload the configuration:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

NGINX reads the password file for each request, so changes to the file take effect without a reload.

To protect a whole website but keep some areas public, use `auth_basic` at the `server` level, and `auth_basic off` in the public locations. The `off` value cancels the setting inherited from the higher level:

```nginx
server {
    listen 80;

    auth_basic           "Administrator's Area";
    auth_basic_user_file /etc/apache2/.htpasswd;

    location /public/ {
        auth_basic off;
    }
}
```

<span id="combine"></span>

## Combining Basic authentication with access restriction by IP address

You can combine Basic authentication with access restriction by IP address in two ways:

- A client must both have an allowed IP address and authenticate.
- A client must either have an allowed IP address or authenticate.

1. Allow or deny IP addresses with the [allow](https://nginx.org/en/docs/http/ngx_http_access_module.html#allow) and [deny](https://nginx.org/en/docs/http/ngx_http_access_module.html#deny) directives:

    ```nginx
    location /api {
        deny  192.168.1.2;
        allow 192.168.1.0/24;
        allow 127.0.0.1;
        deny  all;
    }
    ```

    NGINX checks the rules in order and uses the first one that matches. In this example, it allows the `192.168.1.0/24` network except `192.168.1.2`, and the address `127.0.0.1`. It denies all other addresses with code `403`. The IPv6 loopback address `::1` needs its own `allow ::1;` rule.

2. Combine the IP address rules and Basic authentication with the [satisfy](https://nginx.org/en/docs/http/ngx_http_core_module.html#satisfy) directive. With `all`, the default, a client must pass both checks. With `any`, one passing check is enough. An address passes the IP check only if an `allow` rule matches it; an address that no rule matches doesn't pass:

    ```nginx
    location /api {
        satisfy all;

        deny  192.168.1.2;
        allow 192.168.1.0/24;
        allow 127.0.0.1;
        deny  all;

        auth_basic           "Administrator's Area";
        auth_basic_user_file /etc/apache2/.htpasswd;
    }
    ```

<span id="example"></span>

## Complete example

This complete configuration protects the **/admin/** area of a website with Basic authentication, and allows access only from the local network. It's a whole **nginx.conf**, so back up your existing file first. Keep the `user` directive of your package: `nginx` for NGINX Plus and the packages from nginx.org, `www-data` for Debian and Ubuntu packages. To add only the website to a configuration that reads files from `conf.d`, put the `server` block in a file there; its `server_name` keeps the package's default server from answering the requests.

```nginx
user nginx;

events {}

http {
    server {
        listen      80;
        server_name www.example.com;
        root        /usr/share/nginx/html;

        location /admin/ {
            satisfy all;

            allow 192.168.1.0/24;
            allow 127.0.0.1;
            allow ::1;
            deny  all;

            auth_basic           "Administrator's Area";
            auth_basic_user_file /etc/apache2/.htpasswd;
        }
    }
}
```

To protect the NGINX Plus API in the same way, copy the `satisfy`, `allow`, `deny`, `auth_basic`, and `auth_basic_user_file` directives into the `location` block that has the `api` directive.

---

## Verify the configuration

1. Request the protected area without credentials:

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: www.example.com' http://127.0.0.1/admin/
    ```

    The output is `401`. A browser shows a dialog that asks for a user name and password:

    ![The "Authentication Required" prompt](/nginx/images/auth-required.png)

2. Request it with valid credentials:

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' -u user1 -H 'Host: www.example.com' http://127.0.0.1/admin/
    ```

    Type the password when prompted. The output is the status of the page, for example `200`, or `404` if **/admin/** has no index file.

3. From a host whose address isn't allowed, the same request returns `403`, and the error log shows `access forbidden by rule`.

---

## Troubleshooting

### Valid credentials are rejected with 401

**Symptom**: NGINX returns `401` even with the correct user name and password. The error log shows `user "user1": password mismatch` or `user "user1" was not found in ...`.

**Cause**: The password doesn't match the hash in the file, the user isn't in the file, or the hash format isn't supported.

**Fix**: Create the user again with `htpasswd`, and check that the file contains the user.

### Requests with credentials return 403 or 500

**Symptom**: Requests with a user name and password return `403 Forbidden` or `500 Internal Server Error`, and the error log shows `open() "/etc/apache2/.htpasswd" failed`.

**Cause**: With `403`, the password file doesn't exist at that path. With `500`, NGINX can't open it for another reason, usually because the worker processes can't read it. A `500` with `crypt_r() failed` in the error log means that the system's `crypt()` function doesn't support the password hash.

With `satisfy any`, a client whose address isn't allowed also gets `403` when the password file is missing, and the error log shows `access forbidden by rule` after the `open()` error.

**Fix**: Check the path in `auth_basic_user_file`, let the worker processes' user read the file, and create the passwords with `htpasswd`.

### Allowed clients get 403

**Symptom**: Clients that should have access get `403 Forbidden`, and the error log shows `access forbidden by rule`.

**Cause**: An earlier `allow` or `deny` rule matches the client. Behind a proxy or load balancer, NGINX sees the proxy's address, not the client's.

**Fix**: Check the order of the rules. Behind a proxy, use the [realip](https://nginx.org/en/docs/http/ngx_http_realip_module.html) module so that NGINX uses the real client address.

---

## References

For more information, see:

- [ngx_http_auth_basic_module reference](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html) on nginx.org
- [ngx_http_access_module reference](https://nginx.org/en/docs/http/ngx_http_access_module.html) on nginx.org
- [NGINX SSL Termination]({{< ref "/nginx/admin-guide/security-controls/terminating-ssl-http.md" >}})
