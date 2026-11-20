---
description: Track the performance of F5 NGINX Plus and your apps in real time, on
  the built-in live activity monitoring dashboard or by feeding the JSON to other
  tools.
title: Live Activity Monitoring
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "NGINX Plus API, dashboard, status_zone, zone, api directive, statistics, upstream management, OpenAPI, Swagger UI"
f5-summary: >
  This page explains how to collect statistics in NGINX Plus, turn on the NGINX Plus REST API and the live activity monitoring dashboard, and use them.
  The dashboard and the API show the load and health of your servers in real time, and let you change upstream servers without a reload.
  The API, the dashboard, and the directives that collect the statistics need NGINX Plus.
f5-audience: operator
---

<span id="live-activity-monitoring"></span>

## Overview {#overview}

F5 NGINX Plus has two tools for real-time monitoring:

- **The dashboard**, available since NGINX Plus [Release 9]({{< ref "nginx/releases.md#r9" >}}). It's a web page that shows the main load and performance metrics of your servers, upstream groups, caches, and more.
- **The NGINX Plus REST API**, available since NGINX Plus [Release 14]({{< ref "nginx/releases.md#r14" >}}). It returns the same statistics in JSON, so that you can feed them to other monitoring tools or your own dashboard. It can also reset statistics and change upstream servers and key-value stores without a reload. The dashboard gets its data from the API.

Before NGINX Plus R14, the [`status`](https://nginx.org/en/docs/http/ngx_http_status_module.html#status) and [`upstream_conf`](https://nginx.org/en/docs/http/ngx_http_upstream_conf_module.html) modules provided the statistics and upstream management. The [`api`](https://nginx.org/en/docs/http/ngx_http_api_module.html) module replaced them, and they were removed in R16.

[![The NGINX Plus dashboard overview, with requests, connections, server zones, upstreams, and caches](/nginx/images/nginx-plus-dashboard-r35-overview.png)](https://demo.nginx.com/dashboard.html "Live status metrics from NGINX Plus")

## Before you begin {#prereq}

- NGINX Plus R14 or later. The examples on this page use API version `10`, which needs NGINX Plus PLS.37.1.0.1 or later. For an earlier release, replace `10` in the URLs with the API version of your release from the [OpenAPI versions](#yaml-version) table.
- `sudo` access to the NGINX Plus configuration.
- To make changes through the API or the dashboard, decide which clients may do it. Write access to the API can remove servers from your upstream groups.

## Collect data for statistics {#status_data}

<span id="gathering-data-to-appear-in-statistics"></span>

NGINX Plus collects statistics only for the objects that have a shared memory zone. Worker processes share their counters in that zone. Add a zone to each object that you want to monitor:

- **Virtual servers and locations.** Add the [`status_zone`](https://nginx.org/en/docs/http/ngx_http_api_module.html#status_zone) directive to [HTTP]({{< ref "nginx/admin-guide/load-balancer/http-load-balancer.md" >}}) and [TCP/UDP]({{< ref "nginx/admin-guide/load-balancer/tcp-udp-load-balancer.md" >}}) `server` blocks. Several `server` blocks can share one zone name, and their counters add up. Since [R19]({{< ref "nginx/releases.md#r19" >}}), you can also add `status_zone` to `location` blocks. The dashboard then shows servers and locations separately:

    ```nginx
    server {
        # ...
        status_zone status_page;

        location / {
            proxy_pass  http://backend;
            status_zone location_zone;
        }
    }
    ```

- **Upstream groups.** Add the [`zone`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#zone) directive to each [`upstream`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) block:

    ```nginx
    upstream backend {
        zone   backend 64k;
        server backend1.example.com;
        server backend2.example.com;
    }
    ```

- **Caches.** Each cache already has a shared memory zone: the `keys_zone` parameter of the [`proxy_cache_path`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path), [`fastcgi_cache_path`](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_cache_path), [`scgi_cache_path`](https://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_cache_path), or [`uwsgi_cache_path`](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_cache_path) directive. See [NGINX Content Caching]({{< ref "nginx/admin-guide/content-cache/content-caching.md" >}}):

    ```nginx
    http {
        # ...
        proxy_cache_path /data/nginx/cache keys_zone=one:10m;
    }
    ```

- **Health checks.** Turn on health checks with the [`health_check`](https://nginx.org/en/docs/http/ngx_http_upstream_hc_module.html#health_check) directive, for an upstream group that has a `zone`. See [HTTP Health Checks]({{< ref "nginx/admin-guide/load-balancer/http-health-check.md" >}}) and [TCP Health Checks]({{< ref "nginx/admin-guide/load-balancer/tcp-health-check.md" >}}):

    ```nginx
    server {
        # ...
        status_zone status_page;

        location / {
            proxy_pass http://backend;
            health_check;
        }
    }
    ```

- **Cluster.** Organize the NGINX Plus instances in a cluster, and turn on zone synchronization on each one. See [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}}).

- **DNS resolvers.** Add the [`status_zone`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver_status_zone) parameter to the [`resolver`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) directive:

    ```nginx
    resolver 192.168.33.70 status_zone=resolver-zone1;

    server {
        # ...
    }
    ```

{{< call-out class="note" title="Important" >}} Make sure each shared memory zone is large enough. When a new API version [adds metrics](#yaml-version), you might need to make existing zones larger. {{< /call-out >}}

When you're done, test the configuration and reload NGINX Plus:

```shell
sudo nginx -t && sudo nginx -s reload
```

## Configure the API {#json_configure}

<span id="configuring-the-api"></span>

The API works in a `location` with the [`api`](https://nginx.org/en/docs/http/ngx_http_api_module.html#api) directive. Put it in a separate `server` block that only trusted clients can reach.

1. In the [`http`](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) context, add a [`server`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) block for the API, and a [`location`](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) with the `api` directive:

    ```nginx
    http {
        # ...
        server {
            listen 192.168.1.23;
            # ...

            location /api {
                api;
            }
        }
    }
    ```

    By default, the API is read-only.

2. To make changes through the API, add the `write=on` parameter. You need it to [reset statistics](#json_delete), to change [upstream servers]({{< ref "nginx/admin-guide/load-balancer/dynamic-configuration-api.md" >}}) or a [key-value store]({{< ref "/nginx/admin-guide/security-controls/denylisting-ip-addresses.md" >}}), and to [manage upstream servers in the dashboard](#dashboard_upstream):

    ```nginx
    location /api {
        api write=on;
    }
    ```

3. Allow access to the API only from trusted networks, with the [`allow`](https://nginx.org/en/docs/http/ngx_http_access_module.html#allow) and [`deny`](https://nginx.org/en/docs/http/ngx_http_access_module.html#deny) directives:

    ```nginx
    location /api {
        api write=on;
        allow 192.168.1.0/24;
        deny  all;
    }
    ```

4. Allow the `PATCH`, `POST`, and `DELETE` methods only to some users, with [HTTP basic authentication](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html) inside a [`limit_except`](https://nginx.org/en/docs/http/ngx_http_core_module.html#limit_except) block. See [Restricting Access with HTTP Basic Authentication]({{< ref "/nginx/admin-guide/security-controls/configuring-http-basic-authentication.md" >}}) to create the password file:

    ```nginx
    location /api {
        limit_except GET {
            auth_basic           "NGINX Plus API";
            auth_basic_user_file /etc/nginx/.htpasswd;
        }
        api   write=on;
        allow 192.168.1.0/24;
        deny  all;
    }
    ```

5. Test the configuration and reload NGINX Plus:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

## Turn on the dashboard {#dashboard_enable}

The dashboard is the `dashboard.html` file. The NGINX Plus package installs it in the default root directory, for example `/usr/share/nginx/html`. The dashboard requests its data from `/api/` on the same server, so serve it from the `server` block of the API, and put the `api` directive in `location /api`:

```nginx
location = /dashboard.html {
    root /usr/share/nginx/html;
}
```

## Use the dashboard {#dashboard}

<span id="using-the-dashboard"></span>

### Open the dashboard {#dashboard_access}

In a browser, open the address of the dashboard, in this example `http://192.168.1.23/dashboard.html`.

A live demo is available at [demo.nginx.com/dashboard.html](https://demo.nginx.com/dashboard.html).

### Dashboard tabs {#dashboard_tabs}

The dashboard shows its information in tabs.

![The row of tabs at the top of the NGINX Plus dashboard, which lead to server zones, upstream groups, caches, and more](/nginx/images/dashboard-tabs.png)

- **HTTP Zones** shows statistics for each [`server`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) and [`location`](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) with a [`status_zone`](https://nginx.org/en/docs/http/ngx_http_api_module.html#status_zone), and for each [`limit_req`](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html) zone in the `http` context. Charts for `limit_req` need the [`limit_req_zone`](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_zone) directive.

    ![The HTTP Zones tab of the NGINX Plus dashboard, with statistics of the interaction with clients](/nginx/images/dashboard-tab-http-zones.png)

- **TCP/UDP Zones** shows the TCP and UDP status zones, with charts for connection limiting ([`limit_conn`](https://nginx.org/en/docs/stream/ngx_stream_limit_conn_module.html)).

    ![The TCP/UDP Zones tab of the NGINX Plus dashboard](/nginx/images/dashboard-tab-tcp-zones.png)

- **HTTP Upstreams** shows each upstream group for HTTP and HTTPS traffic, and **TCP/UDP Upstreams** shows the TCP and UDP upstream groups. An upstream group appears only if its [`upstream`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) block has a [`zone`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#zone) directive.

    ![The HTTP Upstreams tab of the NGINX Plus dashboard, with the servers of each upstream group](/nginx/images/dashboard-tab-http-upstreams.png)

- **Caches** shows the state, fullness, and hit ratio of each [cache]({{< ref "nginx/admin-guide/content-cache/content-caching.md" >}}).

    ![The Caches tab of the NGINX Plus dashboard, with the readiness, fullness, and hit ratio of each cache](/nginx/images/dashboard-tab-caches.png)

- **Shared Zones** shows the memory use of each shared memory zone, including cache zones, the SSL session cache, upstream zones, keyval zones, the session log, sticky sessions, and `limit_conn` and `limit_req` zones.

    ![The Shared Zones tab of the NGINX Plus dashboard, with the memory use of each shared memory zone](/nginx/images/dashboard-tab-shared-zones.png)

- **Cluster** shows the synchronization state of shared memory zones across the cluster nodes. See [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}}).

    ![The Cluster tab of the NGINX Plus dashboard, with the synchronization state of shared memory zones on each cluster node](/nginx/images/dashboard-tab-cluster.png)

- **Resolvers** shows the DNS requests and responses of each resolver with a [`status_zone`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver_status_zone) parameter.

    ![The Resolvers tab of the NGINX Plus dashboard, with request and response counts of each DNS resolver zone](/nginx/images/dashboard-tab-resolvers.png)

- **Workers** shows each worker process and its connection statistics.

    ![The Workers tab of the NGINX Plus dashboard, with the worker processes and their connections](/nginx/images/dashboard-tab-workers.png)

### Manage upstream servers in the dashboard {#dashboard_upstream}

You can add, change, and remove upstream servers in the dashboard. This needs the API in [read-write mode](#json_configure).

1. On the **HTTP Upstreams** or **TCP/UDP Upstreams** tab, select the pencil icon next to the upstream group name.

    ![The editing mode of the Upstreams tab in the NGINX Plus dashboard, where you can add, remove, or change servers](/nginx/images/dashboard-tab-upstreams-edit.png)

2. To add a server, select **Add server**:

    ![The Add server form for adding a server to an upstream group in the NGINX Plus dashboard](/nginx/images/dashboard-add-server.png)

3. To change or remove servers, select the checkbox next to each server name, then select **Edit selected**:

    ![The Edit selected form for changing or removing servers of an upstream group in the NGINX Plus dashboard](/nginx/images/dashboard-r7-edit-server-interface.png)

4. Select **Save**.

### Dashboard options {#dashboard_options}

To set the update interval and the thresholds for warnings and alerts, select the gear icon in the tab menu:

![The dashboard settings form of the NGINX Plus dashboard](/nginx/images/dashboard-options.png)

- **Update every N sec**: how often the dashboard reloads its data. The default is `1` second.
- **4xx warnings threshold**: the ratio between `4xx` responses and all requests for **HTTP Upstreams** and **HTTP Zones**. The default is `30%`.
- **Calculate hit ratio for the past N sec**: the period over which **Caches** counts cache hits. The default is `300` seconds.
- **Non synced data threshold**: the ratio between `Pending` and `Total` records for **Cluster**. The default is `70%`.
- **Resolver errors threshold**: the ratio between resolver errors and `Requests` in each update interval for **Resolvers**. The default is `3%`.

## Use the REST API {#json}

<span id="using-the-rest-api"></span>

The API uses standard HTTP methods: `GET` returns statistics, `DELETE` resets statistics and removes upstream servers, `POST` and `PATCH` add and change upstream servers, and `PUT` adds a license token in API version 10. Request and response bodies are JSON. See [Dynamic Configuration of Upstreams with the NGINX Plus API]({{< ref "nginx/admin-guide/load-balancer/dynamic-configuration-api.md" >}}) for upstream management.

### Get statistics {#json_get}

The URL of a resource is a slash-separated path. For example:

[`https://demo.nginx.com/api/10/http/caches/http_cache?fields=expired,bypass`](https://demo.nginx.com/api/10/http/caches/http_cache?fields=expired,bypass)

- `/api` is the `location` with the `api` directive.
- `/10` is the API version. The current version is `10`.
- `/http/caches/http_cache` is the path to the resource.
- `?fields=expired,bypass` is optional. It lists the fields to return.

To get the list of top-level endpoints, send a `GET` request. These examples use `json_pp`, which comes with Perl, to format the JSON:

```shell
curl -s 'https://demo.nginx.com/api/10/' | json_pp
```

The output looks like this:

```json
[
   "nginx",
   "processes",
   "connections",
   "slabs",
   "http",
   "stream",
   "resolvers",
   "ssl",
   "license",
   "workers"
]
```

To get general information about NGINX Plus, request the `/nginx` endpoint:

```shell
curl -s 'https://demo.nginx.com/api/10/nginx' | json_pp
```

The output looks like this:

```json
{
   "version" : "1.31.8",
   "build" : "nginx-plus-r37.3.0",
   "address" : "18.193.151.235",
   "generation" : 1,
   "load_timestamp" : "2026-11-16T20:02:35.527Z",
   "timestamp" : "2026-11-19T06:06:03.023Z",
   "pid" : 766,
   "ppid" : 765
}
```

To return only some fields, list them in the `fields` argument:

```shell
curl -s 'https://demo.nginx.com/api/10/nginx?fields=version,build' | json_pp
```

```json
{
   "version" : "1.31.8",
   "build" : "nginx-plus-r37.3.0"
}
```

For all endpoints and methods, see the [API module reference](https://nginx.org/en/docs/http/ngx_http_api_module.html), also available in [Redoc format]({{< ref "nginx/admin-guide/monitoring/nginx-plus-api-reference.md" >}}).

### Reset statistics {#json_delete}

To reset statistics, send a `DELETE` request to the endpoint. The API must be in [read-write mode](#json_configure). If you protect write methods with basic authentication, as in [Configure the API](#json_configure), add your credentials with the `-u` option of `curl`. Replace `<USER>` and `<PASSWORD>` with a user name and password from the password file.

You can reset:

- the counts of abnormally terminated and respawned child processes
- the counts of accepted and dropped client connections
- the counts of SSL handshakes and session reuses
- the `reqs` and `fails` metrics of each memory slot
- the total count of client HTTP requests
- the counts of accepted and discarded requests, responses, and received and sent bytes in an HTTP server zone
- the counts of cache hits and misses in a cache zone
- the statistics of an HTTP or stream upstream server

For example, to reset the counts of abnormally terminated and respawned child processes:

```shell
curl -X DELETE -i -u '<USER>:<PASSWORD>' 'http://192.168.1.23/api/10/processes'
```

To reset the counts of accepted and dropped client connections:

```shell
curl -X DELETE -i -u '<USER>:<PASSWORD>' 'http://192.168.1.23/api/10/connections'
```

### Manage upstream servers with the API {#api_use}

The API can add a server to an upstream group with `POST`, change its parameters with `PATCH`, and remove it with `DELETE`. The upstream group must have a [`zone`](#status_data); otherwise, the API returns `400`. These examples use `curl`, but any HTTP client works. They include the credentials for basic authentication.

The URL has these parts:

- the address of the NGINX Plus instance, `192.168.1.23` in these examples
- the `location` with the `api` directive, `/api`
- the API version, `10`
- the path of the upstream group, `http/upstreams/appservers`

To add a server to the `appservers` upstream group:

```shell
curl -X POST -s -u '<USER>:<PASSWORD>' 'http://192.168.1.23/api/10/http/upstreams/appservers/servers' -d '{
    "server": "10.0.0.1:8089",
    "weight": 4,
    "max_conns": 0,
    "max_fails": 0,
    "fail_timeout": "10s",
    "slow_start": "10s",
    "backup": true,
    "down": true
}'
```

The JSON body is in single quotes, so it can span several lines without backslashes. A backslash at the end of a line inside single quotes stays in the body and makes the JSON invalid.

To mark the server with ID `0` as down:

```shell
curl -X PATCH -s -u '<USER>:<PASSWORD>' 'http://192.168.1.23/api/10/http/upstreams/appservers/servers/0' -d '{ "down": true }'
```

To remove the server with ID `0`:

```shell
curl -X DELETE -s -u '<USER>:<PASSWORD>' 'http://192.168.1.23/api/10/http/upstreams/appservers/servers/0'
```

## OpenAPI specification {#swagger}

The NGINX Plus API specification is available:

- as [reference documentation](https://nginx.org/en/docs/http/ngx_http_api_module.html)
- in [Redoc format]({{< ref "nginx/admin-guide/monitoring/nginx-plus-api-reference.md" >}})
- as an OpenAPI v2 [YAML file](#yaml-version)

You can open the YAML file in Swagger UI or in another tool that supports OpenAPI v2, to explore the API and send requests from a graphical interface. You can [set up Swagger UI](#swagger_enable) in your environment.

{{< call-out class="warning" >}}Swagger UI and the OpenAPI YAML files are for documentation and exploring the API only. For security reasons, don't use them in a production environment.{{< /call-out >}}

Up to NGINX Plus [Release 25]({{< ref "nginx/releases.md#r25" >}}), the NGINX Plus packages included Swagger UI. Since [Release 26]({{< ref "nginx/releases.md#r26" >}}), F5 publishes the YAML files separately, as listed in the next section, and you install Swagger UI yourself.

### OpenAPI versions {#yaml-version}

This table matches each OpenAPI YAML file and API version to the [NGINX Plus releases]({{< ref "nginx/releases.md" >}}) that use it.

{{<table>}}

|OpenAPI YAML File/API Version | NGINX Plus Version | Changes |
| ---| --- | --- |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v10/nginx_api.yaml) for API version 10 | NGINX Plus Release [PLS.37.1.0.1 CR]({{< ref "/nginx/releases.md#pls.37.1.0" >}}) | [Deferred autorenewal](https://nginx.org/en/docs/ngx_mgmt_module.html#license_pending_token) of JWT license tokens: the `PUT` method for adding a JWT license token, and the `pending_renewal` field were added to the [`/license`](https://nginx.org/en/docs/http/ngx_http_api_module.html#license) data.|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v9/nginx_api.yaml) for API version 9 | NGINX Plus Release [PLS37.0.0 LTS]({{< ref "/nginx/releases.md#r37.0" >}}) | The `response_time_hist` data for each HTTP [upstream](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream) were added|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v9/nginx_api.yaml) for API version 9 | NGINX Plus Releases [33]({{< ref "/nginx/releases.md#r33" >}}), [34]({{< ref "nginx/releases.md#r34" >}}), [35]({{< ref "nginx/releases.md#r35" >}}), [36]({{< ref "nginx/releases.md#r36" >}}) | The [`/license`](https://nginx.org/en/docs/http/ngx_http_api_module.html#license) data were added|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v9/nginx_api.yaml) for API version 9 | NGINX Plus Releases [30]({{< ref "nginx/releases.md#r30" >}}), [31]({{< ref "nginx/releases.md#r31" >}}), [32]({{< ref "nginx/releases.md#r32" >}}) | The [`/workers/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#workers_) data were added|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v8/nginx_api.yaml) for API version 8 | NGINX Plus Releases [27]({{< ref "nginx/releases.md#r27" >}}), [28]({{< ref "nginx/releases.md#r28" >}}), [29]({{< ref "nginx/releases.md#r29" >}}) | SSL statistics for each HTTP [upstream](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream) and stream [upstream](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_upstream), SSL statistics for each HTTP [server zone](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_server_zone) and stream [server zone](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_server_zone), extended statistics for [SSL](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_ssl_object) endpoint|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v7/nginx_api.yaml) for API version 7 | NGINX Plus Releases [25]({{< ref "nginx/releases.md#r25" >}}), [26]({{< ref "nginx/releases.md#r26" >}}) | The `codes` data in `responses` for each HTTP [upstream](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream), [server zone](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_server_zone), and [location zone](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_location_zone) were added|
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v6/nginx_api.yaml) for API version 6 | NGINX Plus Releases [20]({{< ref "nginx/releases.md#r20" >}}), [21]({{< ref "nginx/releases.md#r21" >}}), [22]({{< ref "nginx/releases.md#r22" >}}), [23]({{< ref "nginx/releases.md#r23" >}}), [24]({{< ref "nginx/releases.md#r24" >}}) | The [`/stream/limit_conns/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#stream_limit_conns_),  [`/http/limit_conns/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#http_limit_conns_), and  [`/http/limit_reqs/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#http_limit_reqs_) data were added |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v5/nginx_api.yaml) for API version 5 | NGINX Plus Release [19]({{< ref "nginx/releases.md#r19" >}}) | The `expire` parameter of a [key-value](https://nginx.org/en/docs/http/ngx_http_keyval_module.html) pair can be [set](https://nginx.org/en/docs/http/ngx_http_api_module.html#postHttpKeyvalZoneData) or [changed](https://nginx.org/en/docs/http/ngx_http_api_module.html#patchHttpKeyvalZoneKeyValue), the [`/resolvers/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#resolvers_) and  [`/http/location_zones/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#http_location_zones_) data were added |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v4/nginx_api.yaml) for API version 4 | NGINX Plus Release [18]({{< ref "nginx/releases.md#r18" >}}) | The `path` and `method` fields of [nginx error object](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_error) were removed. These fields continue to exist in earlier api versions, but show an empty value |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v3/nginx_api.yaml) for API version 3 | NGINX Plus Releases [15]({{< ref "nginx/releases.md#r15" >}}), [16]({{< ref "nginx/releases.md#r16" >}}), [17]({{< ref "nginx/releases.md#r17" >}}) | The [`/stream/zone_sync/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#stream_zone_sync_) data were added |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v2/nginx_api.yaml) for API version 2 | NGINX Plus Release [14]({{< ref "nginx/releases.md#r14" >}}) | The [`drain`](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream_conf_server) parameter was added |
|[{{<icon "download">}}OpenAPI v2](/nginx/admin-guide/yaml/v1/nginx_api.yaml) for API version 1 | NGINX Plus Release [13]({{< ref "nginx/releases.md#r13" >}})| The [`/stream/keyvals/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#stream_keyvals_) data were added |

{{</table >}}

### Set up Swagger UI {#swagger_enable}

<span id="the-swagger-ui"></span>

1. Download and install Swagger UI. See the [Swagger UI page](https://swagger.io/tools/swagger-ui/download/) for the package and the instructions.

2. Download the [OpenAPI YAML file](#yaml-version) that matches your NGINX Plus release, and put it in the directory with the Swagger UI files.

3. Point Swagger UI to the YAML file. Swagger UI loads an example specification by default. Set the `url` option in its `swagger-initializer.js` file, for example to `"/swagger-ui/nginx_api.yaml"`. Keep the setting in this external file: the Content Security Policy header in a later step blocks inline scripts. The YAML file sets the API path to `/api/<VERSION>` on the same host, so Swagger UI works when the API is at `/api`.

4. Add a `location` for Swagger UI, for example `/swagger-ui`, and set the path to its files with the [`root`](https://nginx.org/en/docs/http/ngx_http_core_module.html#root) directive. With `root /usr/share/nginx/html;`, the files of `/swagger-ui` are in `/usr/share/nginx/html/swagger-ui`:

    ```nginx
    location /swagger-ui {
        root /usr/share/nginx/html;
    }
    ```

    In NGINX Plus Release 25 and earlier, Swagger UI is installed in the default root directory, for example `/usr/share/nginx/html`.

5. Allow access only from a local network, with the [`allow`](https://nginx.org/en/docs/http/ngx_http_access_module.html#allow) and [`deny`](https://nginx.org/en/docs/http/ngx_http_access_module.html#deny) directives:

    ```nginx
    location /swagger-ui {
        root  /usr/share/nginx/html;
        allow 192.168.1.0/24;
        deny  all;
    }
    ```

6. Add a Content Security Policy header that allows loading resources only from the origin of Swagger UI, with the [`add_header`](https://nginx.org/en/docs/http/ngx_http_headers_module.html#add_header) directive:

    ```nginx
    location /swagger-ui {
        add_header Content-Security-Policy "default-src 'self'";
        root       /usr/share/nginx/html;
        allow      192.168.1.0/24;
        deny       all;
    }
    ```

7. Test the configuration and reload NGINX Plus:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

### Block access to Swagger UI {#swagger_disable}

In NGINX Plus Release 25 and earlier, the NGINX Plus package installs Swagger UI by default. For [security reasons](https://support.f5.com/csp/article/K73710094), you might want to block access to it. One way is to [return](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return) the `404` status code for the `/swagger-ui` location:

```nginx
location /swagger-ui {
    return 404;
}
```

### Use Swagger UI {#swagger_access}

1. In a browser, open the address of Swagger UI, in this example `http://192.168.1.23/swagger-ui/`.

    ![The Swagger UI page with the NGINX Plus API operations](/nginx/images/swagger-ui.png)

2. If Swagger UI is served over HTTPS, select **HTTPS** in the **Schemes** menu.
3. Select the operation you want to run.
4. Select **Try it out**.
5. Fill in the required fields. Usually, it's the name of a shared memory zone.
6. Optional: to return only some fields, list them separated by commas in the **Fields** box. If it's empty, all fields are returned.
7. Select **Execute**. The response and its HTTP status code appear under **Execute**.

### Live examples of the API and Swagger UI {#json_example}

The demo site has a live NGINX Plus API at [`https://demo.nginx.com/api/10/`](https://demo.nginx.com/api/10/). Send requests to it with `curl` or a browser:

```shell
curl -s 'https://demo.nginx.com/api/10/'
curl -s 'https://demo.nginx.com/api/10/nginx?fields=version,build'
curl -s 'https://demo.nginx.com/api/10/http/caches/http_cache'
curl -s 'https://demo.nginx.com/api/10/http/upstreams/'
curl -s 'https://demo.nginx.com/api/10/http/upstreams/demo-backend'
curl -s 'https://demo.nginx.com/api/10/http/upstreams/demo-backend/servers/0'
```

The Swagger UI demo is at <https://demo.nginx.com/swagger-ui/>.

[![The Swagger UI demo page with the NGINX Plus API operations](/nginx/images/swagger-ui.png)](https://demo.nginx.com/swagger-ui/)

The demo API is read-only: you can't reset statistics with `DELETE`, or add and change upstream servers with `POST` and `PATCH`. The demo site uses HTTPS and redirects HTTP requests to HTTPS, so select **HTTPS** in the **Schemes** menu of the Swagger UI demo.

## Example

This configuration collects statistics for a virtual server, a location, an upstream group, and a cache. It serves the API in read-write mode, with basic authentication for changes, and the dashboard on a separate address that only the local network can reach:

```nginx
http {
    proxy_cache_path /data/nginx/cache keys_zone=one:10m;

    upstream backend {
        zone   backend 64k;
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen      80;
        status_zone public;

        location / {
            proxy_pass  http://backend;
            proxy_cache one;
            status_zone root;
        }
    }

    server {
        listen 192.168.1.23:80;

        location /api {
            limit_except GET {
                auth_basic           "NGINX Plus API";
                auth_basic_user_file /etc/nginx/.htpasswd;
            }
            api   write=on;
            allow 192.168.1.0/24;
            deny  all;
        }

        location = /dashboard.html {
            root  /usr/share/nginx/html;
            allow 192.168.1.0/24;
            deny  all;
        }
    }
}
```

## Verify the configuration

1. Test the configuration and reload NGINX Plus:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

2. From the local network, request the list of HTTP server zones:

    ```shell
    curl -s 'http://192.168.1.23/api/10/http/server_zones/' | json_pp
    ```

    The output has a `public` object with request and response counters.

3. Open `http://192.168.1.23/dashboard.html` in a browser. The **HTTP Zones** tab shows the `public` server zone and the `root` location zone, and the **HTTP Upstreams** tab shows the `backend` group.

## Troubleshooting

### An object doesn't appear in the dashboard or the API

**Cause:** The object has no shared memory zone.

**Fix:** Add a `status_zone` to the `server` or `location`, or a `zone` to the `upstream` block, as described in [Collect data for statistics](#status_data).

### A `DELETE`, `POST`, or `PATCH` request fails with status `405`

**Cause:** The API is read-only.

**Fix:** Add `write=on` to the `api` directive, as described in [Configure the API](#json_configure).

### A `POST` or `PATCH` request fails with status `415`

**Cause:** The request body isn't valid JSON, for example because of backslashes at the line ends inside single quotes.

**Fix:** Check the body, for example with `echo '<BODY>' | json_pp`. See [Manage upstream servers with the API](#api_use).

### A request to an upstream group fails with status `400`

**Cause:** The upstream group has no `zone`, or a parameter in the body is invalid.

**Fix:** Add a [`zone`](#status_data) to the `upstream` block, and check the `error` object in the response.

### A request fails with status `401` or `403`

**Cause:** The API requires basic authentication for this method (`401`), or the client address isn't allowed (`403`).

**Fix:** Add `-u '<USER>:<PASSWORD>'` to the `curl` command, and send the request from an allowed network.

## References

- [`ngx_http_api_module`](https://nginx.org/en/docs/http/ngx_http_api_module.html) reference
- [NGINX Plus API reference in Redoc format]({{< ref "nginx/admin-guide/monitoring/nginx-plus-api-reference.md" >}})
- [Dynamic Configuration of Upstreams with the NGINX Plus API]({{< ref "nginx/admin-guide/load-balancer/dynamic-configuration-api.md" >}})
- [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}})
