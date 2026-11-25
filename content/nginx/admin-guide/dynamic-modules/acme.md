---
title: ACME
description: Install the ACME dynamic module in F5 NGINX Plus to get and renew TLS certificates automatically from Let's Encrypt and other ACME servers.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "ACME, Let's Encrypt, automatic certificates, acme_issuer, acme_certificate, nginx-plus-module-acme, HTTP-01, dynamic module"
f5-summary: >
  This guide shows how to install the ACME dynamic module in F5 NGINX Plus and get certificates with it.
  The module implements the ACMEv2 protocol, so NGINX Plus requests and renews its own TLS certificates.
  It works with Let's Encrypt and other certificate authorities that support ACME.
f5-audience: operator
---

## Overview

The Automatic Certificate Management Environment (ACME) protocol, [RFC 8555](https://www.rfc-editor.org/rfc/rfc8555.html), gets TLS certificates from a certificate authority (CA) without manual steps. The ACME dynamic module adds this protocol to F5 NGINX Plus. NGINX Plus requests certificates for its server names, proves to the CA that it controls them, and renews the certificates.

This is an [NGINX-authored module]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}). The `nginx-plus-module-acme` package in the NGINX Plus repository contains it, since NGINX Plus R35. Its source code is in the [nginx-acme project](https://github.com/nginx/nginx-acme) on GitHub, and its reference is the [ngx_http_acme_module documentation](https://nginx.org/en/docs/http/ngx_http_acme_module.html).

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus R35 or later**: NGINX Plus installed from the NGINX Plus repository. To check your release, run `nginx -v`. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **DNS records for your server names**: The CA checks that you control each name in the certificate, so each name must point to the NGINX Plus server.
- **Port 80 open to the internet**: For the default HTTP-01 challenge, the CA sends HTTP requests to port 80 of each name, as [RFC 8555](https://www.rfc-editor.org/rfc/rfc8555.html#section-8.3) describes.
- **A DNS server for NGINX Plus**: The module finds the ACME server through the DNS server that the `resolver` directive sets.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

<span id="installation"></span>

## Install the module package

Install the `nginx-plus-module-acme` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-acme
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-acme
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-acme
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-acme
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-acme
    ```

---

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive:

    ```nginx
    load_module modules/ngx_http_acme_module.so;
    ```

---

<span id="configuration"></span><span id="full-example"></span>

## Configure the module

These steps get a certificate from Let's Encrypt for `example.com` and `www.example.com`. Replace the names and the email address with your own.

1. In the `http` block, set a DNS server, and define the CA with an [`acme_issuer`](https://nginx.org/en/docs/http/ngx_http_acme_module.html#acme_issuer) block:

    ```nginx
    resolver 127.0.0.1:53;

    acme_issuer letsencrypt {
        uri         https://acme-v02.api.letsencrypt.org/directory;
        contact     admin@example.com;
        state_path  /var/cache/nginx/acme-letsencrypt;
        accept_terms_of_service;
    }
    ```

    Where:

    - `resolver` sets the DNS server that NGINX Plus uses. Replace `127.0.0.1:53` with a DNS server that the NGINX Plus server can reach.
    - `uri` is the directory URL of the ACME server.
    - `contact` is the email address that the CA uses for messages about your account.
    - `state_path` is the directory where the module keeps the account key, the certificates, and their private keys. NGINX Plus creates it. Keep it private.
    - `accept_terms_of_service` accepts the terms of service of the CA. Some CAs require it. Read the terms first.

    To test your setup, use the Let's Encrypt staging directory, `https://acme-staging-v02.api.letsencrypt.org/directory`. Its rate limits are higher, but browsers don't trust its certificates.

1. Optional: If you have many certificates, make the shared memory zone of the module bigger with [`acme_shared_zone`](https://nginx.org/en/docs/http/ngx_http_acme_module.html#acme_shared_zone). The default size, `256k`, holds about 50 ECDSA keys or 35 RSA keys.

    ```nginx
    acme_shared_zone zone=ngx_acme_shared:1M;
    ```

1. If no server listens on port 80, add one. The CA sends the HTTP-01 challenge requests to port 80:

    ```nginx
    server {
        listen 80;

        location / {
            return 404;
        }
    }
    ```

    The module answers the requests to `/.well-known/acme-challenge/` before the rest of the server configuration applies. This server can also redirect other requests to HTTPS.

1. In the HTTPS server, request a certificate with [`acme_certificate`](https://nginx.org/en/docs/http/ngx_http_acme_module.html#acme_certificate), and use it:

    ```nginx
    server {
        listen 443 ssl;
        server_name .example.com;

        acme_certificate letsencrypt;

        ssl_certificate       $acme_certificate;
        ssl_certificate_key   $acme_certificate_key;
        ssl_certificate_cache max=2;
    }
    ```

    Where:

    - `acme_certificate` requests a certificate from the issuer for the names in `server_name`. A name with a leading dot, such as `.example.com`, stands for `example.com` and `www.example.com`. Regular expressions and wildcard names, such as `*.example.com`, aren't supported.
    - The [`$acme_certificate`](https://nginx.org/en/docs/http/ngx_http_acme_module.html#var_acme_certificate) and [`$acme_certificate_key`](https://nginx.org/en/docs/http/ngx_http_acme_module.html#var_acme_certificate_key) variables pass the certificate and its private key to NGINX Plus.
    - [`ssl_certificate_cache`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate_cache) keeps the certificate in memory, so NGINX Plus doesn't load it for each TLS handshake.

This is the complete configuration:

```nginx
load_module modules/ngx_http_acme_module.so;

events {}

http {
    resolver 127.0.0.1:53;

    acme_issuer letsencrypt {
        uri         https://acme-v02.api.letsencrypt.org/directory;
        contact     admin@example.com;
        state_path  /var/cache/nginx/acme-letsencrypt;
        accept_terms_of_service;
    }

    server {
        listen 80;

        location / {
            return 404;
        }
    }

    server {
        listen 443 ssl;
        server_name .example.com;

        acme_certificate letsencrypt;

        ssl_certificate       $acme_certificate;
        ssl_certificate_key   $acme_certificate_key;
        ssl_certificate_cache max=2;
    }
}
```

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

1. Check the certificate that the server sends. Replace `example.com` with one of your server names:

    ```shell
    openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | openssl x509 -noout -issuer -subject -enddate
    ```

    The `issuer` line names your CA, for example Let's Encrypt. If the command shows no certificate, wait a minute, and check the error log.

---

## Troubleshooting

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] unknown directive "acme_issuer"`.

**Cause**: The configuration uses a directive of the module, but no `load_module` directive loads the module.

**Fix**: Add the `load_module` directive, as described in [Load the module](#load-the-module).

---

### nginx -t reports that no identifiers were found

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] "acme_certificate": no identifiers found in "server_name"`.

**Cause**: The `server_name` directive of the server has no name that the module can use. The module skips regular expressions, and it reports each one with a warning such as `"acme_certificate": unsupported regular expression in server_name`.

**Fix**: Add a plain name to `server_name`, or list the names after the issuer, for example `acme_certificate letsencrypt example.com www.example.com;`.

---

### nginx -t reports that the resolver isn't configured

**Symptom**: `nginx -t` fails with a message that ends with `"resolver" is not configured`.

**Cause**: The module needs a DNS server to find the ACME server, but the `http` block has no `resolver` directive.

**Fix**: Add a `resolver` directive to the `http` block, as described in [Configure the module](#configure-the-module).

---

### The CA doesn't issue the certificate

**Symptom**: The server doesn't send a certificate from your CA, and the error log shows errors from the module.

**Cause**: The CA can't reach port 80 for a name in the certificate. Or the DNS record of the name doesn't point to the server.

**Fix**: Open port 80 in your firewall, check the DNS records, and reload the configuration.

---

<span id="more-info"></span>

## References

For more information, see:

- [Module ngx_http_acme_module](https://nginx.org/en/docs/http/ngx_http_acme_module.html)
- [The nginx-acme project on GitHub](https://github.com/nginx/nginx-acme)
- [Native support for the ACME protocol](https://blog.nginx.org/blog/native-support-for-acme-protocol)
- [RFC 8555: Automatic Certificate Management Environment (ACME)](https://www.rfc-editor.org/rfc/rfc8555.html)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
