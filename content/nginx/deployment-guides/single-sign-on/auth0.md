---
title: Single Sign-On with Auth0
description: Enable OpenID Connect-based single sign-on (SSO) for applications proxied by NGINX Plus, using Auth0 as the identity provider (IdP).
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "single sign-on, SSO, OpenID Connect, OIDC, Auth0, oidc_provider, auth_oidc, NGINX Plus"
f5-summary: >
  This guide shows how to set up single sign-on with OpenID Connect for applications that F5 NGINX Plus proxies, with Auth0 as the identity provider.
  NGINX Plus uses its native OIDC module, so you need only a few directives.
  It covers sign-in, RP-initiated and front-channel logout, and PKCE.
f5-audience: operator
---

## Overview

This guide explains how to enable single sign-on (SSO) for applications being proxied by F5 NGINX Plus. The solution uses OpenID Connect for authentication. [Auth0](https://auth0.com/features/single-sign-on) is the identity provider (IdP), and NGINX Plus is the relying party (RP): the OIDC client that checks the identity of users.

{{< call-out class="note" >}} This guide needs [NGINX Plus R36]({{< ref "nginx/releases.md#r36" >}}) or later. The [OpenID Connect module](https://nginx.org/en/docs/http/ngx_http_oidc_module.html) is in NGINX Plus since R34, but R34 and R35 lack some of its directives in this guide, such as `pkce`. Earlier releases use an [njs-based solution](#legacy-njs-guide), with JavaScript files and key-value stores.{{< /call-out >}}

## Before you begin

Before you begin, make sure you have:

- An [Auth0](https://auth0.com/) tenant with administrator privileges.

- An NGINX Plus [subscription](https://www.f5.com/products/nginx/nginx-plus) and NGINX Plus [Release 36]({{< ref "nginx/releases.md#r36" >}}) or later. For installation instructions, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

- A domain name pointing to your NGINX Plus instance, for example, `demo.example.com`.

## Create a new Auth0 Application {#auth0-create}

1. Log in to your Auth0 Dashboard at [manage.auth0.com](https://manage.auth0.com/).

2. Select **Applications > Applications** from the sidebar menu.

3. On the **Applications** screen, select **Create Application**.

4. On the **Create application** screen:

   - Enter the **Name** for the application, for example, **NGINX Demo App**.

   - In **Application Type**, select **Regular Web Applications**.

   - Select **Create**.

5. On the **Settings** screen of your application:

   - Copy your **Client ID** and **Client Secret** displayed in the **Basic Information** section &mdash; you will need them later when configuring NGINX Plus.

6. On the **Application URIs** section of your application:

   - Add the NGINX Plus callback URI in the **Allowed Callback URLs** field, for example:

     `https://demo.example.com/oidc_callback`.

   - Add the post logout URL in the **Allowed Logout URLs** field, for example:

     `https://demo.example.com/post_logout/`.

### Get the OpenID Connect Discovery URL

Check the OpenID Connect Discovery URL. By default, Auth0 publishes the `.well-known/openid-configuration` document at the following address:

`https://yourTenantId.us.auth0.com/.well-known/openid-configuration`.

1. Run the following `curl` command in a terminal:

   ```shell
   curl https://yourTenantId.us.auth0.com/.well-known/openid-configuration | jq
   ```
   
   Where:

   - the `yourTenantId` is your Auth0 [Tenant ID](https://auth0.com/docs/get-started/tenant-settings/find-your-tenant-name-or-tenant-id)

   - the `yourTenantId.us.auth0.com/` is your Auth0 server address

   - the `/.well-known/openid-configuration` is the default address for Auth0 for document location

   - the `jq` command (optional) is used to format the JSON output for easier reading and requires the [jq](https://jqlang.github.io/jq/) JSON processor to be installed.

   The configuration metadata is returned in the JSON format:

   ```json
   {
       ...
       "issuer": "https://{yourTenantId}.us.auth0.com/",
       "authorization_endpoint": "https://{yourTenantId}.us.auth0.com/oauth/token",
       "token_endpoint": "https://{yourTenantId}.us.auth0.com/oauth/token",
       "jwks_uri": "https://{yourTenantId}.us.auth0.com/.well-known/jwks.json",
       "userinfo_endpoint": "https://{yourTenantId}.us.auth0.com/userinfo",
       "end_session_endpoint": "https://{yourTenantId}.us.auth0.com/oidc/logout",
       ...
   }
   ```

2. Copy the **issuer** value, you will need it later when configuring NGINX Plus. Typically, the OpenID Connect Issuer for Auth0 is `https://yourTenantId.us.auth0.com/` (including the trailing slash). To verify the accuracy of the endpoints, refer to the [Auth0 official documentation](https://auth0.com/docs/get-started/applications/configure-applications-with-oidc-discovery).

{{< call-out class="note" >}} You will need the values of **Client ID**, **Client Secret**, and **Issuer** in the next steps. {{< /call-out >}}

## Set up NGINX Plus {#nginx-plus-setup}

With Auth0 configured, you can enable OIDC on NGINX Plus. NGINX Plus serves as the Relying Party (RP) application &mdash; a client service that verifies user identity.

1.  Ensure that you are using the latest version of NGINX Plus by running the `nginx -v` command in a terminal:

    ```shell
    nginx -v
    ```

    The output should match NGINX Plus Release 36 or later:

    ```none
    nginx version: nginx/1.29.3 (nginx-plus-r36)
    ```

2.  Ensure that you have the values of the **Client ID**, **Client Secret**, and **Issuer** obtained during [Create a new Auth0 application](#auth0-create).

3.  In your preferred text editor, open the NGINX configuration file (`/etc/nginx/nginx.conf` for Linux or `/usr/local/etc/nginx/nginx.conf` for FreeBSD).

4.  In the [`http {}`](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) context, set your public DNS resolver with the [`resolver`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) directive. By default, NGINX Plus re-resolves DNS records when their time-to-live (TTL) ends. The `valid` parameter overrides the TTL:

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        # ...
    }
    ```

    <span id="auth0-setup-oidc-provider"></span>
5.  In the [`http {}`](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) context, define the Auth0 provider named `auth0` by specifying the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) context:

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        oidc_provider auth0 {

            # ...

        }
        # ...
    }
    ```

6.  In the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) context, specify:

    - Your actual Auth0 **Client ID** obtained in [Auth0 Configuration](#auth0-create) with the [`client_id`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#client_id) directive

    - Your **Client Secret** obtained in [Auth0 Configuration](#auth0-create) with the [`client_secret`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#client_secret) directive

    - The **Issuer** URL obtained in [Auth0 Configuration](#auth0-create) with the [`issuer`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#issuer) directive

      The `issuer` is typically your Auth0 OIDC URL. For Auth0, a trailing slash is included, for example: `https://yourTenantId.us.auth0.com/`.
    
    - The **logout_uri** is the URI that a user visits to start an RP-initiated logout flow.

    - The **post_logout_uri** is the absolute HTTPS URL where Auth0 should redirect the user after a successful logout. This value **must also be specified** in **Allowed Logout URLs** on the Auth0 side.

    - If the **logout_token_hint** directive is set to `on`, NGINX Plus sends the user’s ID token as a *hint* to Auth0.
      This directive is **optional**, however, if you omit it, Auth0 might display an extra confirmation page asking the user to approve the logout request.

    - If the **userinfo** directive is set to `on`, NGINX Plus will fetch `/userinfo` from Auth0 and append the claims from userinfo to the `$oidc_claims_` variables.

    - NGINX Plus turns on PKCE (Proof Key for Code Exchange) when the discovery document of the IdP lists `S256` in `code_challenge_methods_supported`. The [`pkce`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#pkce) directive overrides this: `pkce off;` turns PKCE off, and `pkce on;` turns it on even if the metadata doesn't list `S256`.

    - The module automatically selects the client authentication method for the token endpoint based on the provider metadata `token_endpoint_auth_methods_supported`. When only `client_secret_post` is advertised, NGINX Plus uses the `client_secret_post` method and sends the client credentials in the POST body. When both `client_secret_basic` and `client_secret_post` are present, the module prefers HTTP Basic (`client_secret_basic`), which remains the default for Auth0.

    - {{< call-out class="important" >}} NGINX Plus talks to the IdP only over TLS, so it must trust the certificate of the IdP. By default, it checks the certificate against the CA bundle of your operating system. If that bundle doesn't include the CA of the IdP, set a trusted certificate or chain with the [`ssl_trusted_certificate`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#ssl_trusted_certificate) directive. {{< /call-out >}}

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        oidc_provider auth0 {
            issuer            https://yourTenantId.us.auth0.com/;
            client_id         <CLIENT_ID>;
            client_secret     <CLIENT_SECRET>;
            logout_uri        /logout;
            post_logout_uri   https://demo.example.com/post_logout/;
            logout_token_hint on;
            userinfo          on;

            # Optional: PKCE configuration. By default, PKCE is automatically
            # enabled when the IdP advertises the S256 code challenge method.
            # pkce on;
        }

        # ...
    }
    ```

    Replace `<CLIENT_ID>` with the client ID of your application and `<CLIENT_SECRET>` with the client secret of your application.

7.  Set up a [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) for `demo.example.com`, with a [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) that [proxies](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) requests to your application at `http://127.0.0.1:8080` (see [Step 11](#oidc_app)). OIDC protects this location:

    ```nginx
    http {

        # ...

        server {
            listen      443 ssl;
            server_name demo.example.com;

            ssl_certificate     /etc/ssl/certs/fullchain.pem;
            ssl_certificate_key /etc/ssl/private/key.pem;

            location / {

                # ...

                proxy_pass http://127.0.0.1:8080;
            }
        }
        # ...
    }
    ```

8.  Protect this [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) with the [`auth_oidc`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#auth_oidc) directive. It names the `auth0` provider from the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) block in [Step 5](#auth0-setup-oidc-provider):

    ```nginx
    # ...
    location / {
         auth_oidc auth0;

         # ...

         proxy_pass http://127.0.0.1:8080;
    }
    # ...
    ```

9.  Pass the OIDC claims as headers to the application ([Step 11](#oidc_app)) with the [`proxy_set_header`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive. These claims are extracted from the ID token returned by Auth0:

    - [`$oidc_claim_email`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) the e-mail address of the user

    - [`$oidc_claim_name`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) - the full name of the user

    - any other OIDC claim using the [`$oidc_claim_ `](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) variable

    ```nginx
    # ...
    location / {
         auth_oidc auth0;

         proxy_set_header sub   $oidc_claim_sub;
         proxy_set_header email $oidc_claim_email;
         proxy_set_header name  $oidc_claim_name;

         proxy_pass http://127.0.0.1:8080;
    }
    # ...
    ```

10. Provide endpoint for completing logout:

    ```nginx
    # ...
    location /post_logout/ {
         return 200 "You have been logged out.\n";
         default_type text/plain;
    }
    # ...
    ```

11. <span id="oidc_app"></span>Create a simple test application referenced by the `proxy_pass` directive which returns the authenticated user's full name and email upon successful authentication:

    ```nginx
    # ...
    server {
        listen 8080;

        location / {
            return 200 "Hello, $http_name!\nEmail: $http_email\nAuth0 sub: $http_sub\n";
            default_type text/plain;
        }
    }
    ```

12. Save the configuration file, test it, and reload it:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

### Complete Example

This configuration example summarizes the steps outlined above. It includes only essential settings such as specifying the DNS resolver, defining the OIDC provider, configuring SSL, and proxying requests to an internal server.

```nginx
http {
    # Use a public DNS resolver for Issuer discovery, etc.
    resolver 10.0.0.1 ipv4=on valid=300s;

    oidc_provider auth0 {
        # Issuer from your Auth0 tenant's .well-known/openid-configuration
        issuer https://yourTenantId.us.auth0.com/;

        # Replace with your actual Client ID and Secret from Auth0
        client_id <CLIENT_ID>;
        client_secret <CLIENT_SECRET>;

        # RP-initiated logout
        logout_uri /logout;
        post_logout_uri https://demo.example.com/post_logout/;
        logout_token_hint on;

        # Fetch userinfo claims
        userinfo on;

        # Optional: PKCE configuration
        # pkce on;
    }

    server {
        listen 443 ssl;
        server_name demo.example.com;

        ssl_certificate /etc/ssl/certs/fullchain.pem;
        ssl_certificate_key /etc/ssl/private/key.pem;

        location / {
            # Enforce OIDC for root path with Auth0
            auth_oidc auth0;

            # Forward OIDC claims to the upstream as headers if desired
            proxy_set_header email $oidc_claim_email;
            proxy_set_header name  $oidc_claim_name;

            proxy_pass http://127.0.0.1:8080;
        }

        location /post_logout/ {
            return 200 "You have been logged out.\n";
            default_type text/plain;
        }
    }

    server {
        # Simple test upstream server
        listen 8080;

        location / {
            return 200 "Hello, $http_name!\nEmail: $http_email\nAuth0 sub: $http_sub\n";
            default_type text/plain;
        }
    }
}
```

### Testing

1. Open `https://demo.example.com/` in a browser. You will be redirected to the Auth0 sign-in page.

2. Enter valid Auth0 credentials of a user who has access the application. Upon successful sign-in, Auth0 redirects you back to NGINX Plus, and you will see the proxied application content (for example, “Hello, Jane Doe!”).

3. Navigate to `https://demo.example.com/logout`. NGINX Plus initiates an RP-initiated logout; Auth0 ends the session and redirects back to `https://demo.example.com/post_logout/`.

4. Refresh `https://demo.example.com/` again. You should be redirected to Auth0 for a fresh sign-in, proving the session has been terminated.

## Legacy njs-based Auth0 Solution {#legacy-njs-guide}

For NGINX Plus R33 and earlier, or if you still need the njs-based solution, see the [Legacy njs-based Auth0 Guide]({{< ref "nginx/deployment-guides/single-sign-on/oidc-njs/auth0.md" >}}). The solution uses the [`nginx-openid-connect`](https://github.com/nginxinc/nginx-openid-connect) GitHub repository and NGINX JavaScript files.

## Troubleshooting

### Auth0 reports a redirect URI mismatch

**Symptom**: After you sign in, Auth0 shows an error about the redirect URI.

**Cause**: The redirect URI that NGINX Plus sends doesn't match a redirect URI registered in Auth0. By default, NGINX Plus uses the path `/oidc_callback` on the host of the request.

**Fix**: Register the exact URI, for example `https://demo.example.com/oidc_callback`, in Auth0. Or set the registered path with the [`redirect_uri`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#redirect_uri) directive in the `oidc_provider` block.

---

### Sign-in fails before NGINX Plus redirects to Auth0

**Symptom**: Requests to the protected location fail, and the error log shows errors about the issuer or its metadata.

**Cause**: NGINX Plus can't get the metadata of the issuer. The `resolver` directive is missing or wrong, a firewall blocks outbound HTTPS, or NGINX Plus doesn't trust the certificate of the IdP.

**Fix**: Check the `resolver` directive and the network access to the issuer URL. If needed, set `ssl_trusted_certificate` in the `oidc_provider` block.

---

## References

For more information, see:

- [NGINX Plus Native OIDC Module Reference documentation](https://nginx.org/en/docs/http/ngx_http_oidc_module.html)

- [Release Notes for NGINX Plus R36]({{< ref "nginx/releases.md#r36" >}})

## Revision history

- Version 4 (November 2026): Restructured to the how-to template, with shorter sentences, a troubleshooting section, and a note on the NGINX Plus releases that this guide needs.

- Version 3 (November 2025) – Updated for NGINX Plus R36; added PKCE configuration (`pkce` directive) and the `client_secret_post` token endpoint authentication method.

- Version 2 (August 2025) – Updated for NGINX Plus R35; added RP-initiated logout (`logout_uri`, `post_logout_uri`, `logout_token_hint`) and `userinfo` support.

- Version 1 (March 2025) – Initial version (NGINX Plus Release 34).
