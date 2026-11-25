---
title: Single Sign-On with Ping Identity
description: Enable OpenID Connect-based single sign-on (SSO) for applications proxied by NGINX Plus, using Ping Identity as the identity provider (IdP).
toc: true
weight: 800
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "single sign-on, SSO, OpenID Connect, OIDC, Ping Identity, oidc_provider, auth_oidc, NGINX Plus"
f5-summary: >
  This guide shows how to set up single sign-on with OpenID Connect for applications that F5 NGINX Plus proxies, with Ping Identity as the identity provider.
  NGINX Plus uses its native OIDC module, so you need only a few directives.
  It covers sign-in, RP-initiated and front-channel logout, and PKCE.
f5-audience: operator
---

## Overview

This guide explains how to enable single sign-on (SSO) for applications being proxied by F5 NGINX Plus. The solution uses OpenID Connect for authentication. [Ping Identity](https://www.pingidentity.com/en.html), with PingFederate or PingOne, is the identity provider (IdP), and NGINX Plus is the relying party (RP).

{{< call-out class="note" >}} This guide needs [NGINX Plus R36]({{< ref "nginx/releases.md#r36" >}}) or later. The [OpenID Connect module](https://nginx.org/en/docs/http/ngx_http_oidc_module.html) is in NGINX Plus since R34, but R34 and R35 lack some of its directives in this guide, such as `pkce`. Earlier releases use an [njs-based solution](#legacy-njs-guide), with JavaScript files and key-value stores.{{< /call-out >}}

<span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- [PingFederate](https://docs.pingidentity.com/pingfederate/latest/pf_pf_landing_page.html) Enterprise Federation Server or [PingOne](https://docs.pingidentity.com/pingone/p1_cloud__platform_main_landing_page.html) Cloud deployment with a Ping Identity account.

- An NGINX Plus [subscription](https://www.f5.com/products/nginx/nginx-plus) and NGINX Plus [Release 36]({{< ref "nginx/releases.md#r36" >}}) or later. For installation instructions, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

- A domain name pointing to your NGINX Plus instance, for example, `demo.example.com`.

## Configure PingFederate or PingOne for Enterprise {#ping-create}

{{< call-out class="note" >}} These steps outline an example with the cloud offering of **PingOne**. If you are using the on-premises PingFederate, the user interface might slightly differ. {{< /call-out >}}

Create a new application for NGINX Plus:

1. Log in to your Ping Identity admin console.

2. Go to **Applications** > **Applications**.

3. Select the **+** (plus) symbol to create a new OIDC Application.

4. On the New Application screen:

   - Enter the Name of your application, for example, `nginx-demo-app`.

   - Select the Application Type **OIDC Web App**.

   - Select **Save**.

5. In your OIDC application, Select the **Overview** tab:

   - in the **General** section, copy your **Client ID** and **Client Secret** values. You will need then later when configuring NGINX Plus.

   - In the **Connection Details** section, copy your **Issuer ID**. You will need it later when configuring NGINX Plus.

     For PingOne Cloud, the Issuer ID generally structured as `https://auth.pingone.com/<ENVIRONMENT_ID>/as`.

     For PingFederate, the Issuer ID generally structured as `https://pingfederate.example.com:9031` appended with the realm path of your environment.

6. On the **Configuration** tab of your OIDC application:

   - In the **Redirect URIs** field, add the NGINX Plus callback URI, for example:

    `https://demo.example.com/oidc_callback`.

   - In the **Post Logout Redirect URIs** field, add the post logout redirect URI, for example:

    `https://demo.example.com/post_logout/`.

   - Select **Save**.

7. Assign the application to the appropriate **Groups** or **Users** who will be allowed to log in.

### Enable Front-Channel Logout (optional) {#ping-frontchannel-logout}

Front-channel logout allows Ping Identity to notify NGINX Plus when a user signs out of other Single Logout (SLO)-participating applications or from their Ping portal. Ping sends a front-channel HTTP request (typically loaded in a hidden iframe) to a logout URL that you configure in the application settings.

This feature is optional and requires NGINX Plus Release 36 or later, together with the `frontchannel_logout_uri` directive in the NGINX OIDC provider configuration.

1. In the Ping Identity admin console, open the OIDC application that you created earlier (for example, `nginx-demo-app`).
2. Locate the **Front-channel logout** or **Single Logout** settings for the application. In PingOne, these are available under the **Configuration** or **Policies** sections; in PingFederate they are available in the OAuth / OIDC application connection settings. The exact location and labels can vary between product versions.
3. Configure the front-channel logout URL to point to the NGINX Plus front-channel logout endpoint, for example:

   ```text
   https://demo.example.com/front_logout
   ```

4. Ensure that the front-channel logout request includes both the issuer (`iss`) and the session identifier (`sid`) as query parameters, in accordance with the OpenID Connect front-channel logout specification. If your Ping Identity admin console exposes an option to **include session details** or **send session ID and issuer** in logout requests, enable it.
5. Save the application configuration.

### Get the OpenID Connect Discovery URL

Check the OpenID Connect Discovery URL. By default, Ping Identity publishes the `.well-known/openid-configuration` document at the following address:

For PingOne: `https://auth.pingone.com/<ENVIRONMENT_ID>/as/.well-known/openid-configuration`

For PingFederate: `https://pingfederate.example.com:9031/<REALM_PATH>/.well-known/openid-configuration`

1. Run the following `curl` command in a terminal:

   ```shell
   curl https://auth.pingone.com/{environment_id}/as/.well-known/openid-configuration | jq
   ```

   Where:

   - the `auth.pingone.com` is your PingOne server address (or your PingFederate server for on-premises)

   - the `{environment_id}` is your PingOne environment ID

   - the `/as` is the authorization server path

   - the `/.well-known/openid-configuration` is the default address for Ping Identity for document location

   - the `jq` command (optional) is used to format the JSON output for easier reading and requires the [jq](https://jqlang.github.io/jq/) JSON processor to be installed.

   The configuration metadata is returned in the JSON format:

   ```json
   {
       ...
       "issuer": "https://auth.pingone.com/<ENVIRONMENT_ID>/as",
       "authorization_endpoint": "https://auth.pingone.com/<ENVIRONMENT_ID>/as/authorize",
       "token_endpoint": "https://auth.pingone.com/<ENVIRONMENT_ID>/as/token",
       "jwks_uri": "https://auth.pingone.com/<ENVIRONMENT_ID>/as/jwks",
       "userinfo_endpoint": "https://auth.pingone.com/<ENVIRONMENT_ID>/as/userinfo",
       "end_session_endpoint": "https://auth.pingone.com/<ENVIRONMENT_ID>/as/signoff",
       ...
   }
   ```

2. Copy the **issuer** value, you will need it later when configuring NGINX Plus. Typically, the OpenID Connect Issuer for PingOne is `https://auth.pingone.com/<ENVIRONMENT_ID>/as`.

{{< call-out class="note" >}} You will need the values of **Client ID**, **Client Secret**, and **Issuer** in the next steps. {{< /call-out >}}

## Set up NGINX Plus {#nginx-plus-setup}

With PingOne or PingFederate configured, you can enable OIDC on NGINX Plus. NGINX Plus serves as the Relying Party (RP) application &mdash; a client service that verifies user identity.

1.  Ensure that you are using the latest version of NGINX Plus by running the `nginx -v` command in a terminal:

    ```shell
    nginx -v
    ```

    The output should match NGINX Plus Release 36 or later:

    ```none
    nginx version: nginx/1.29.3 (nginx-plus-r36)
    ```

2.  Ensure that you have the values of the **Client ID**, **Client Secret**, and **Issuer** obtained during [PingOne or PingFederate Configuration](#ping-create).

3.  In your preferred text editor, open the NGINX configuration file (`/etc/nginx/nginx.conf` for Linux or `/usr/local/etc/nginx/nginx.conf` for FreeBSD).

4.  In the [`http {}`](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) context, set your public DNS resolver with the [`resolver`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) directive. By default, NGINX Plus re-resolves DNS records when their time-to-live (TTL) ends. The `valid` parameter overrides the TTL:

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        # ...
    }
    ```

    <span id="ping-setup-oidc-provider"></span>
5.  In the [`http {}`](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) context, define the PingOne or PingFederate provider named `ping` by specifying the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) context:

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        oidc_provider ping {

            # ...
        }
        # ...
    }
    ```

6.  In the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) context, specify:

    - your actual Ping **Client ID** obtained in [Ping Configuration](#ping-create) with the [`client_id`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#client_id) directive

    - your **Client Secret** obtained in [Ping Configuration](#ping-create) with the [`client_secret`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#client_secret) directive

    - the **Issuer** URL obtained in [Ping Configuration](#ping-create) with the [`issuer`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#issuer) directive

        The `issuer` is typically your Ping Identity OIDC URL.

        For PingOne Cloud, the URL is `https://auth.pingone.com/<ENVIRONMENT_ID>/as`.

        For PingFederate, the URL is `https://pingfederate.example.com:9031` followed by your environment's realm path.

        By default, NGINX Plus creates the OpenID metadata URL by appending the `/.well-known/openid-configuration` part to the Issuer URL. If your metadata URL is different, you can explicitly specify the metadata document with the [`config_url`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#config_url) directive.

    - The **logout_uri** is the URI that a user visits to start an RP-initiated logout flow.

    - The **post_logout_uri** is the absolute HTTPS URL where Ping Identity should redirect the user after a successful logout. This value **must also be configured** in the Ping Identity application's Post Logout Redirect URIs.

    - If the **logout_token_hint** directive is set to `on`, NGINX Plus sends the user's ID token as a *hint* to Ping Identity.
      This directive is **required** by PingOne.

    - The **frontchannel_logout_uri** directive defines the URI that receives OpenID Connect front-channel logout requests from Ping Identity. This URI must be an HTTPS path and must match the front-channel logout URL configured for the application in Ping Identity. Ping sends the front-channel logout GET request to this URI, usually from a hidden iframe, with the `iss` and `sid` query parameters. The OIDC module then clears the user session on NGINX Plus.

    - If the **userinfo** directive is set to `on`, NGINX Plus will fetch userinfo from Ping Identity and append the claims from userinfo to the `$oidc_claims_` variables.

    - NGINX Plus turns on PKCE (Proof Key for Code Exchange) when the discovery document of the IdP lists `S256` in `code_challenge_methods_supported`. The [`pkce`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#pkce) directive overrides this: `pkce off;` turns PKCE off, and `pkce on;` turns it on even if the metadata doesn't list `S256`.

    - The OIDC module automatically selects the client authentication method for the token endpoint based on the provider metadata `token_endpoint_auth_methods_supported`. When only `client_secret_post` is advertised, NGINX Plus uses the `client_secret_post` method and sends the client credentials in the POST body. When both `client_secret_basic` and `client_secret_post` are present, the module prefers HTTP Basic (`client_secret_basic`), which remains the default for Ping Identity.

    - {{< call-out class="important" >}} NGINX Plus talks to the IdP only over TLS, so it must trust the certificate of the IdP. By default, it checks the certificate against the CA bundle of your operating system. If that bundle doesn't include the CA of the IdP, set a trusted certificate or chain with the [`ssl_trusted_certificate`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#ssl_trusted_certificate) directive. {{< /call-out >}}

    ```nginx
    http {
        resolver 10.0.0.1 ipv4=on valid=300s;

        oidc_provider ping {
            issuer                  https://auth.pingone.com/<ENVIRONMENT_ID>/as;
            client_id               <CLIENT_ID>;
            client_secret           <CLIENT_SECRET>;
            logout_uri              /logout;
            logout_token_hint       on;
            post_logout_uri         https://demo.example.com/post_logout/;
            frontchannel_logout_uri /front_logout;
            userinfo                on;

            # Optional: PKCE configuration. By default, PKCE is automatically
            # enabled when the IdP advertises the S256 code challenge method.
            # pkce                  on;
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

8.  Protect this [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) with the [`auth_oidc`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#auth_oidc) directive. It names the `ping` provider from the [`oidc_provider {}`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#oidc_provider) block in [Step 5](#ping-setup-oidc-provider):

    ```nginx
    # ...
    location / {
         auth_oidc ping;

         # ...

         proxy_pass http://127.0.0.1:8080;
    }
    # ...
    ```

9.  Pass the OIDC claims as headers to the application ([Step 11](#oidc_app)) with the [`proxy_set_header`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive. These claims are extracted from the ID token returned by Ping:

    - [`$oidc_claim_sub`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) - a unique `Subject` identifier assigned for each user by Ping Identity

    - [`$oidc_claim_email`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) - the e-mail address of the user

    - [`$oidc_claim_name`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) - the full name of the user

    - any other OIDC claim using the [`$oidc_claim_ `](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#var_oidc_claim_) variable

    {{< call-out class="note" >}} Ensure the `openid`, `profile`, `email` Scopes are enabled in Ping Identity.{{< /call-out >}}

    ```nginx
    # ...
    location / {
         auth_oidc ping;

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
            return 200 "Hello, $http_name!\nEmail: $http_email\nPing Identity sub: $http_sub\n";
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

    oidc_provider ping {
        # The issuer is typically something like:
        # https://auth.pingone.com/<ENVIRONMENT_ID>/as
        issuer https://auth.pingone.com/<ENVIRONMENT_ID>/as;

        # Your Ping Identity Client ID and Secret
        client_id <CLIENT_ID>;
        client_secret <CLIENT_SECRET>;

        # RP-initiated logout
        logout_uri /logout;
        post_logout_uri https://demo.example.com/post_logout/;
        logout_token_hint on;

        # Front-channel logout (OP-initiated single sign-out)
        frontchannel_logout_uri /front_logout;

        # Fetch userinfo claims
        userinfo on;

        # Optional: PKCE configuration (enabled automatically when supported by the IdP)
        # pkce on;
    }

    server {
        listen 443 ssl;
        server_name demo.example.com;

        ssl_certificate /etc/ssl/certs/fullchain.pem;
        ssl_certificate_key /etc/ssl/private/key.pem;

        location / {
            # Enforce OIDC with Ping Identity
            auth_oidc ping;

            # Forward OIDC claims as headers if desired
            proxy_set_header sub $oidc_claim_sub;
            proxy_set_header email $oidc_claim_email;
            proxy_set_header name $oidc_claim_name;

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
            return 200 "Hello, $http_name!\nEmail: $http_email\nPing Identity sub: $http_sub\n";
            default_type text/plain;
        }
    }
}
```

### Testing

1. Open `https://demo.example.com/` in a browser. You will be automatically redirected to the PingOne sign-in page.

2. Enter valid Ping Identity credentials of a user who has access the application. Upon successful sign-in, PingOne redirects you back to NGINX Plus, and you will see the proxied application content (for example, "Hello, Jane Doe!").

3. Navigate to `https://demo.example.com/logout`. NGINX Plus initiates an RP-initiated logout; Ping Identity ends the session and redirects back to `https://demo.example.com/post_logout/`.

4. Refresh `https://demo.example.com/` again. You should be redirected to Ping Identity for a fresh sign-in, proving the session has been terminated.

## Legacy njs-based Ping Identity Solution {#legacy-njs-guide}

For NGINX Plus R33 and earlier, or if you still need the njs-based solution, see the [Legacy njs-based Ping Identity Guide]({{< ref "nginx/deployment-guides/single-sign-on/oidc-njs/ping-identity.md" >}}). The solution uses the [`nginx-openid-connect`](https://github.com/nginxinc/nginx-openid-connect) GitHub repository and NGINX JavaScript files.

## Troubleshooting

### Ping Identity reports a redirect URI mismatch

**Symptom**: After you sign in, Ping Identity shows an error about the redirect URI.

**Cause**: The redirect URI that NGINX Plus sends doesn't match a redirect URI registered in Ping Identity. By default, NGINX Plus uses the path `/oidc_callback` on the host of the request.

**Fix**: Register the exact URI, for example `https://demo.example.com/oidc_callback`, in Ping Identity. Or set the registered path with the [`redirect_uri`](https://nginx.org/en/docs/http/ngx_http_oidc_module.html#redirect_uri) directive in the `oidc_provider` block.

---

### Sign-in fails before NGINX Plus redirects to Ping Identity

**Symptom**: Requests to the protected location fail, and the error log shows errors about the issuer or its metadata.

**Cause**: NGINX Plus can't get the metadata of the issuer. The `resolver` directive is missing or wrong, a firewall blocks outbound HTTPS, or NGINX Plus doesn't trust the certificate of the IdP.

**Fix**: Check the `resolver` directive and the network access to the issuer URL. If needed, set `ssl_trusted_certificate` in the `oidc_provider` block.

---

<span id="see-also"></span>

## References

For more information, see:

- [NGINX Plus Native OIDC Module Reference documentation](https://nginx.org/en/docs/http/ngx_http_oidc_module.html)

- [Release Notes for NGINX Plus R36]({{< ref "nginx/releases.md#r36" >}})

## Revision history

- Version 4 (November 2026): Restructured to the how-to template, with shorter sentences, a troubleshooting section, and a note on the NGINX Plus releases that this guide needs.

- Version 3 (November 2025) – Updated for NGINX Plus R36; added front-channel logout support (`frontchannel_logout_uri`), PKCE configuration (`pkce` directive), and the `client_secret_post` token endpoint authentication method.

- Version 2 (August 2025) – Updated for NGINX Plus R35; added RP-initiated logout (`logout_uri`, `post_logout_uri`, `logout_token_hint`) and `userinfo` support.

- Version 1 (March 2025) – Initial version for NGINX Plus Release 34.
