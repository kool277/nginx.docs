---
description: Learn how to enable single sign-on (SSO) with Auth0
  for applications proxied by F5 NGINX Plus.
type:
- tutorial
title: Single Sign-On With Auth0 and njs
toc: false
weight: 100
f5-product: F5 NGINX Plus
f5-content-type: how-to
f5-keywords: "single sign-on, SSO, OpenID Connect, OIDC, Auth0, njs, nginx-openid-connect, NGINX Plus R33"
f5-summary: >
  This guide shows how to set up single sign-on for F5 NGINX Plus with Auth0, with the njs-based nginx-openid-connect solution.
  It's for NGINX Plus R33 and earlier. Since R34, the native OIDC module of NGINX Plus is simpler.
f5-audience: operator
---

## Overview

{{< call-out class="note" >}} This guide applies to NGINX Plus [Release 15]({{< ref "nginx/releases.md#r15" >}}) and later, based on the [`nginx-openid-connect`](https://github.com/nginxinc/nginx-openid-connect) GitHub repo. Starting with NGINX Plus [Release 34]({{< ref "nginx/releases.md#r34" >}}), use the simpler solution with the [native OpenID connect module](https://nginx.org/en/docs/http/ngx_http_oidc_module.html).

See [Single Sign-On With Auth0]({{< ref "nginx/deployment-guides/single-sign-on/auth0.md" >}}) for details.{{< /call-out >}}

You can use F5 NGINX Plus with [Auth0](https://auth0.com/) and OpenID Connect to enable single sign-on (SSO) for your proxied applications. By following the steps in this guide, you will learn how to set up SSO using OpenID Connect as the authentication mechanism, with Auth0 as the identity provider (IdP), and NGINX Plus as the relying party.

{{< call-out class="note" >}}{{<include "nginx-plus/nginx-openid-repo-note">}}{{< /call-out>}}

<span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- An Auth0 tenant with administrator privileges.
- [NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) with a valid subscription.
- The [NGINX JavaScript module](https://nginx.org/en/docs/njs/index.html) (`njs`) -- the `njs` module handles the interaction between NGINX Plus and Auth0.

## Install NGINX Plus and the njs Module {#install-nginx-plus-njs}

1. If you do not already have NGINX Plus installed, follow the steps in the [NGINX Plus Admin Guide]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}) to do so.
2. Install the NGINX JavaScript module by following the steps in the [`njs` installation guide](https://nginx.org/en/docs/njs/install.html).
3. Add the following directive to the top-level ("main") configuration context in the NGINX Plus configuration (`/etc/nginx/nginx.conf`) to load the `njs` module:

   ```nginx
   load_module modules/ngx_http_js_module.so;
   ```

## Configure Auth0 {#config-auth0}

Take the steps in this section to create a new application for NGINX Plus.

{{< call-out class="note" >}} This section contains images that reflect the state of the Auth0 web interface at the time of publication. The actual Auth0 GUI may differ from the examples shown here. Use this guide as a reference and adapt the instructions to suit the current Auth0 GUI as necessary.{{< /call-out >}}

### Create a new Auth0 Application {#create-auth0-app}

1. Log in to your Auth0 Dashboard at [manage.auth0.com](https://manage.auth0.com/).
1. Select **Applications > Applications** from the sidebar menu.
1. On the **Applications** page, select the **Create Application** button.
1. In the **Create application** window, provide the information listed below and then select **Create**.

    - **Name**: A name for the application, for example "nginx-plus-app".
    - **Application Type**: **Regular Web Applications**

    {{< img src="/img/sso/auth0/sso-auth0-create-app.png" alt="image showing the Create application window in the Auth0 dashboard" >}}

### Set up the Web Application {#web-app-setup}

In this section, you'll set up a web application that follows the Auth0 [Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow).

1. On the **Application** page in the [Auth0 dashboard](https://manage.auth0.com/), select your web application.
1. Select the **Settings** tab for your application.
1. Make note of the Client ID and Client Secret displayed in the **Basic Information** section.

    {{< img src="/img/sso/auth0/sso-auth0-app.png" alt="image showing the basic information section of the web application settings in the Auth0 dashboard" >}}

1. In the **Application URIs** section, provide  the URI of the NGINX Plus instance in the **Allowed Callback URLs** field.

    - The URL must include the port number and end in **/_codexch**. In our example, we used the URL `http://nginx-plus-app:8010/_codexch`.
    - The port is always required, even if you use the default port for HTTP (`80`) or HTTPS (`443`).
    - The use of SSL/TLS (`443`) is strongly recommended for production environments.

    {{< img src="/img/sso/auth0/sso-auth0-app-settings.png" alt="image showing the Application URIs settings in the Auth0 dashboard" >}}

1. In the **Advanced Settings** section, select the **Endpoints** tab.
1. Make note of the **OpenID Configuration** URL.

   {{< img src="/img/sso/auth0/sso-auth0-app-advanced-settings.png" alt="image showing the Advanced Application Settings section of the Auth0 dashboard" >}}

1. Select **Save Changes**.

### Set up Authentication {#authn-setup}

{{< call-out class="note" >}}For the purposes of this guide, we will add a new Auth0 user database and user account to use for testing.

You can set up authentication using any of the available [Auth0 identity providers](https://auth0.com/docs/authenticate/identity-providers). {{< /call-out >}}

To set up a new user database and add a user account to it, take the steps below.

1. Log in to the [Auth0 dashboard](https://manage.auth0.com/) and select **Authentication > Database** from the sidebar menu.
1. Select the **Create DB Connection** button.
1. Provide a **Name** for the database connection, then select **Create**.
1. On the **Database** page, select the **Applications** tab. Then, select the toggle button next to the [application you created earlier](#create-auth0-app).

    {{< img src="/img/sso/auth0/sso-auth0-db-app.png" alt="image showing the Applications settings for an OIDC Authentication database in the Auth0 dashboard" >}}

1. In the sidebar menu, select **User Management > Users**.
1. On the **Users** page, select the **Create User** button.
1. In the **Create user** window, provide the following information, then select **Create**.
    - **Email**: user's email
    - **Password**: a password for the user account
    - **Connection**: select your **database** from the list.

    {{< img src="/img/sso/auth0/sso-auth0-create-user.png" alt="image showing the Create User settings window in the Auth0 dashboard" >}}

The user should receive an email to the email address provided. Once the user verifies their account by selecting the link in the email, the account creation process is complete.

## Set up NGINX Plus {#nginx-plus-setup}

Take the steps in this section to set up NGINX Plus as the OpenID Connect relying party.

### Configure NGINX OpenID Connect {#nginx-plus-oidc-config}

1. Clone the [nginx-openid-connect](https://github.com/nginxinc/nginx-openid-connect) GitHub repository, or download the repo files.

   ```bash
   git clone https://github.com/nginxinc/nginx-openid-connect.git
   ```

1. Run the *configure.sh* script, which will update the NGINX configuration files with the values for your Auth0 application.

    For example:

    ```bash
    ./nginx-openid-connect/configure.sh \
        --auth_jwt_key request \
        --client_id Nhotzxx...IERmUi \
        --client_secret 6ZHd0j_r...UtDZ5bkdu \
        https://<AUTH0_TENANT>.us.auth0.com/.well-known/openid-configuration
    ```

    Replace `<AUTH0_TENANT>` with your Auth0 tenant name.

1. In the `frontend.conf` file, update the **my_backend** upstream with the address of the application that you want to add OIDC authorization to.

    For example:

    ```nginx
    upstream my_backend {
        zone my_backend 64k;
        server my-backend-app.com:80;
    }
    ```

1. In the *openid_connect.server_conf* file, add the [`proxy_set_header`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive to the `/_jwks_uri` and `/_token` locations to `Accept-Encoding "gzip"`, as shown below.

    ```nginx
    ...
     location = /_jwks_uri {
        ...
        proxy_set_header    Accept-Encoding "gzip"
    }
    ...
    location = /_token {
        ...
        proxy_set_header    Accept-Encoding "gzip"
   }
   ...
    ```

1. Copy the following files to the */etc/nginx/conf.d* directory on the host machine where NGINX Plus is installed:

   - `frontend.conf`
   - `openid_connect.js`
   - `openid_connect.server_conf`
   - `openid_connect_configuration.conf`

1. Reload the NGINX configuration:

    ```bash
    sudo nginx -s reload
    ```

<span id="test-the-setup"></span>

## Test your setup

1. In a browser, enter the address of your NGINX Plus instance. You should be directed to the Auth0 login page, as shown in the example below.

    {{< img src="/img/sso/auth0/sso-auth0-login-test.png" alt="image showing an example Auth0 login screen that contains username and password fields" >}}

1. You should be able to log in using the credentials of the user account that you created in the Auth0 database.

## Troubleshooting

Refer to the [Troubleshooting](https://github.com/nginxinc/nginx-openid-connect#troubleshooting) section in the `nginx-openid-connect` repository on GitHub.

## References

For more information, see:

- [The nginx-openid-connect project on GitHub](https://github.com/nginxinc/nginx-openid-connect)
- [Single Sign-On with Auth0 with the native OIDC module]({{< ref "/nginx/deployment-guides/single-sign-on/auth0.md" >}})

---

## Revision history

- Version 1 (May 2022) - Initial version
