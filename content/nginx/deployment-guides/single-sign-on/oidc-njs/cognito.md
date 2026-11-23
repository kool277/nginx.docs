---
description: Enable OpenID Connect-based single sign-on (SSO) for applications proxied by NGINX
  Plus, using Amazon Cognito as the identity provider (IdP).
f5-content-type: how-to
title: Single Sign-On with Amazon Cognito and njs
toc: false
weight: 100
f5-product: F5 NGINX Plus
f5-keywords: "single sign-on, SSO, OpenID Connect, OIDC, Amazon Cognito, njs, nginx-openid-connect, NGINX Plus R33"
f5-summary: >
  This guide shows how to set up single sign-on for F5 NGINX Plus with Amazon Cognito, with the njs-based nginx-openid-connect solution.
  It's for NGINX Plus R33 and earlier. Since R34, the native OIDC module of NGINX Plus is simpler.
f5-audience: operator
---

## Overview

{{< call-out class="note" >}} This guide applies to NGINX Plus [Release 15]({{< ref "nginx/releases.md#r15" >}}) and later, based on the [`nginx-openid-connect`](https://github.com/nginxinc/nginx-openid-connect) GitHub repo. Starting with NGINX Plus [Release 34]({{< ref "nginx/releases.md#r34" >}}), use the simpler solution with the [native OpenID connect module](https://nginx.org/en/docs/http/ngx_http_oidc_module.html).

See [Single Sign-On With Amazon Cognito]({{< ref "nginx/deployment-guides/single-sign-on/cognito.md" >}}) for details.{{< /call-out >}}

This guide explains how to enable single sign-on (SSO) for applications being proxied by F5 NGINX Plus. The solution uses OpenID Connect as the authentication mechanism, with [Amazon Cognito](https://aws.amazon.com/cognito/) as the identity provider (IdP), and NGINX Plus as the relying party.

{{< call-out class="note" >}}{{<include "nginx-plus/nginx-openid-repo-note">}}{{< /call-out>}}

<span id="prereqs"></span>
<span id="prerequisites"></span>

## Before you begin

The instructions assume you have the following:

- An [AWS account](https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/).
- An NGINX Plus subscription and NGINX Plus R15 or later. For installation instructions, see the [NGINX Plus Admin Guide]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- The [NGINX JavaScript module](https://nginx.org/en/docs/njs/index.html) (njs), required for handling the interaction between NGINX Plus and the IdP. After installing NGINX Plus, install the module with the command for your operating system.

   For Debian and Ubuntu:

   ```none
   sudo apt install nginx-plus-module-njs
   ```

   For CentOS, RHEL, and Oracle Linux:

   ```shell
   sudo yum install nginx-plus-module-njs
   ```

- The following directive included in the top-level ("main") configuration context in **/etc/nginx/nginx.conf**, to load the NGINX JavaScript module:

   ```nginx
   load_module modules/ngx_http_js_module.so;
   ```

<span id="cognito"></span>
<span id="configuring-amazon-cognito"></span>

## Set up Amazon Cognito

**Note:** The following procedure reflects the Cognito GUI at the time of publication, but the GUI is subject to change. Use this guide as a reference and adapt to the current Cognito GUI as necessary.

Create a new application for NGINX Plus in the Cognito GUI:

1. Log in to your AWS account, open the AWS Management Console ([console.aws.amazon.com](https://console.aws.amazon.com)), and navigate to the Cognito dashboard (you can, for example, select **Cognito** in the **Security, Identity, & Compliance** section of the **Services** drop-down menu).

2. On the Cognito dashboard, select **Manage User Pools** to open the **Your User Pools** window. Select the **Create a user pool** button or the highlighted phrase.

   <img src="/nginx/images/cognito-sso-your-user-pools.png" alt="" width="1024" height="212" class="aligncenter size-full wp-image-63792" />

3. In the **Create a user pool** window that opens, type a value in the **Pool name** field (in this guide, it's **nginx-plus-pool**), then select the **Review defaults** button.

   <img src="/nginx/images/cognito-sso-create-name-tab.png" alt="" width="1024" height="808" class="aligncenter size-full wp-image-63953" />

   <span id="cognito-review-tab"></span>
4. On the **Review** tab which opens, select **Add app client...** in the **App clients** field near the bottom.

   <img src="/nginx/images/cognito-sso-create-review-tab.png" alt="" width="1024" height="677" class="aligncenter size-full wp-image-63789" />

5. On the **App clients** tab which opens, select **Add an app client**.

6. On the **Which app clients will have access to this user pool?** window which opens, enter a value (in this guide, **nginx-plus-app**) in the **App client name** field. Make sure the **Generate client secret** box is checked, then select the **Create app client** button.

   <img src="/nginx/images/cognito-sso-create-app-clients-tab.png" alt="" width="1024" height="942" class="aligncenter size-full wp-image-63788" />

7. On the confirmation page which opens, select **Return to pool details** to return to the **Review** tab. On that tab select the **Create pool** button at the bottom. (The screenshot in [Step 4](#cognito-review-tab) shows the button.)

   <span id="cognito-pool-id"></span>
8. On the details page which opens to confirm the new user pool was successfully created, make note of the value in the **Pool Id** field; you will add it to the NGINX Plus configuration in [step 3 of _Set up NGINX Plus_](#nginx-plus-variables).

   <img src="/nginx/images/cognito-sso-config-general-settings-tab.png" alt="'General settings' tab in Amazon Cognito GUI" width="1024" height="435" class="aligncenter size-full wp-image-63787" />

   <span id="cognito-users"></span>
9. Select **Users and groups** in the left navigation column. In the interface that opens, designate the users (or group of users, on the **Groups** tab) who will be able to use SSO for the app being proxied by NGINX Plus. For instructions, see the Cognito documentation about [creating users](https://docs.aws.amazon.com/cognito/latest/developerguide/how-to-create-user-accounts.html), [importing users](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-using-import-tool.html), or [adding a group](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html).

   <img src="/nginx/images/cognito-sso-config-users-tab.png" alt="'Users and groups' tab in Amazon Cognito GUI" width="1023" height="466" class="aligncenter size-full wp-image-63786" />

10. Select **App clients** in the left navigation bar. On the tab that opens, select the **Show Details** button in the box labeled with the app client name (in this guide, **nginx-plus-app**).

    <img src="/nginx/images/cognito-sso-config-app-clients-tab.png" alt="'App clients' tab in Amazon Cognito GUI" width="1024" height="497" class="aligncenter size-full wp-image-63785" />

    <span id="cognito-app-client-id-secret"></span>
11. On the details page that opens, make note of the values in the **App client id** and **App client secret** fields. You will add them to the NGINX Plus configuration in [step 3 of _Set up NGINX Plus_](#nginx-plus-variables).

    <img src="/nginx/images/cognito-sso-config-app-clients-details.png" alt="" width="1024" height="672" class="aligncenter size-full wp-image-63784" />

12. Select **App client settings** in the left navigation column. In the tab that opens, perform the following steps:

    1. In the **Enabled Identity Providers** section, select the **Cognito User Pool** checkbox (the **Select all** box gets checked automatically).
    2. In the **Callback URL(s)** field of the **Sign in and sign out URLs** section, type the URI of the NGINX Plus instance including the port number, and ending in **/\_codexch**. Here we’re using **https://my-nginx-plus.example.com:443/_codexch**.

       **Notes:**

       - For production, we strongly recommend that you use SSL/TLS (port 443).
       - The port number is mandatory even when you're using the default port for HTTP (80) or HTTPS (443).

    3. In the **OAuth 2.0** section, select the **Authorization code grant** checkbox under **Allowed OAuth Flows** and the **email**, **openid**, and **profile** checkboxes under **Allowed OAuth Scopes**.
    4. Select the **Save changes** button.

    <img src="/nginx/images/cognito-sso-config-app-client-settings-tab.png" alt="" width="1024" height="1181" class="aligncenter size-full wp-image-63783" />

    <span id="cognito-domain-name"></span>
13. Select **Domain name** in the left navigation column. In the tab that opens, type a domain prefix in the **Domain prefix** field under **Amazon Cognito domain** (in this guide, **my-nginx-plus**). Select the **Save changes** button.

    <img src="/nginx/images/cognito-sso-config-domain-name-tab.png" alt="" width="1024" height="817" class="aligncenter size-full wp-image-63782" />

<span id="nginx-plus"></span>
<span id="configuring-nginx-plus"></span>

## Set up NGINX Plus

Configure NGINX Plus as the OpenID Connect relying party:

1. Create a clone of the [**nginx-openid-connect**](https://github.com/nginxinc/nginx-openid-connect) GitHub repository.

   ```shell
   git clone https://github.com/nginxinc/nginx-openid-connect
   ```

2. Copy these files from the clone to **/etc/nginx/conf.d**:

   - **frontend.conf**
   - **openid_connect.js**
   - **openid_connect.server\_conf**

   <span id="nginx-plus-variables"></span>
3. In your preferred text editor, open **/etc/nginx/conf.d/frontend.conf**. Change the second parameter of each of the following [set](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#set) directives to the specified value.

   The `<MY_COGNITO_DOMAIN_NAME>` variable is the full value in the **Domain prefix** field in [step 13 of _Set up Amazon Cognito_](#cognito-domain-name). In this guide it is **https://my-nginx-plus.auth.us-east-2.amazoncognito.com**.

   - `set $oidc_authz_endpoint` – `<MY_COGNITO_DOMAIN_NAME>/oauth2/authorize`
   - `set $oidc_token_endpoint` – `<MY_COGNITO_DOMAIN_NAME>/oauth2/token`
   - `set $oidc_client` – Value in the **App client id** field from [step 11 of _Set up Amazon Cognito_](#cognito-app-client-id-secret) (in this guide, `2or4cs8bjo1lkbq6143tqp6ist`)
   - `set $oidc_client_secret` – Value in the **App client secret** field from [step 11 of _Set up Amazon Cognito_](#cognito-app-client-id-secret) (in this guide, `1k63m3nrcnu...`)
   - `set $oidc_hmac_key` – A unique, long, and secure phrase

4. Configure the JWK file. The file's URL is

   **https://cognito-idp.**_region_**.amazonaws.com/**_User-Pool-ID_**/.well-known/jwks.json**

   where

   - _region_ is the same AWS region name as in the `<MY_COGNITO_DOMAIN_NAME>` variable used in [step 3](#nginx-plus-variables)
   - _User-Pool-ID_ is the value in the **Pool Id** field in [step 8 of _Set up Amazon Cognito_](#cognito-pool-id)

   In this guide, the URL is

   **https://cognito-idp.us-east-2.amazonaws.com/us-east-2_mLoGHJpOs/.well-known/jwks.json**.

   The method for configuring the JWK file depends on which version of NGINX Plus you are using:

   - In NGINX Plus R17 and later, NGINX Plus can read the JWK file directly. Change **/etc/nginx/conf.d/frontend.conf** as follows:

      1. Comment out (or remove) the [auth_jwt_key_file](https://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html#auth_jwt_key_file) directive.
      2. Uncomment the [auth_jwt_key_request](https://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html#auth_jwt_key_request) directive. (Its parameter, `/_jwks_uri`, refers to the value of the `$oidc_jwt_keyfile` variable, which you set in the next step.)
      3. Change the second parameter of the `set $oidc_jwt_keyfile` directive to the URL of the JWK file (`https://cognito-idp.../.well-known/jwks.json`).

   - In NGINX Plus R16 and earlier, the JWK file must be on the local disk. (You can also use this method with NGINX Plus R17 and later if you wish.)

      1. Copy the JSON contents from the JWK file (**<https://cognito-idp.../.well-known/jwks.json>**) to a local file (for example, **/etc/nginx/my\_cognito\_jwk.json**).
      2. In **/etc/nginx/conf.d/frontend.conf**, change the second parameter of the `set $oidc_jwt_keyfile` directive to the local file path.

5. At the time of publication, Cognito does not support the OpenID  **offline_access** scope. Open **/etc/nginx/conf.d/openid\_connect.server\_conf** in a text editor and remove `+offline_access` from the list of scopes on line 10, so that it looks like this:

   ```nginx
   return 302 "$oidc_authz_endpoint?response_type=code&scope=openid+profile+email&client_id=$oidc_clientaws...;
   ```

6. Confirm that the user named by the [user](https://nginx.org/en/docs/ngx_core_module.html#user) directive in the NGINX Plus configuration (in **/etc/nginx/nginx.conf** by convention) has read permission on the JWK file.

<span id="testing"></span>
<span id="testing"></span>

## Test your setup

In a browser, enter the address of your NGINX Plus instance and try to log in using the credentials of a user assigned to the application (see [step 9 in _Set up Amazon Cognito_](#cognito-users)). The NGINX logo that appears in the screenshot was added on Cognito's **UI customization** tab (not shown in this guide).

<img src="/nginx/images/cognito-sso-login.png" alt="" width="734" height="864" class="aligncenter size-full wp-image-63793" />

<span id="troubleshooting"></span>

## Troubleshooting

See the [**Troubleshooting**](https://github.com/nginxinc/nginx-openid-connect#troubleshooting) section at the **nginx-openid-connect** repository on GitHub.

## References

For more information, see:

- [The nginx-openid-connect project on GitHub](https://github.com/nginxinc/nginx-openid-connect)
- [Single Sign-On with Amazon Cognito with the native OIDC module]({{< ref "/nginx/deployment-guides/single-sign-on/cognito.md" >}})

---

## Revision history

- Version 1 (March 2020) – Initial version (NGINX Plus Release 20)
