---
title: Prometheus-njs
description: Install the Prometheus-njs module in F5 NGINX Plus to export the metrics of the NGINX Plus API in the Prometheus format.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "Prometheus, prometheus-njs, metrics, NGINX Plus API, js_import, nginx-plus-module-prometheus, njs"
f5-summary: >
  This guide shows how to install the Prometheus-njs module in F5 NGINX Plus and expose a metrics endpoint for Prometheus.
  The module converts the status metrics of the NGINX Plus API into the Prometheus format.
  It runs in njs, so the package manager installs the njs dynamic module with it, and you load that module.
f5-audience: operator
---

## Overview

The Prometheus-njs module is written in [njs](https://nginx.org/en/docs/njs/). It reads the status metrics of the [NGINX Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html), and returns them in the Prometheus format. Prometheus can then collect them from NGINX Plus. The `nginx-plus-module-prometheus` package in the NGINX Plus repository contains it.

Some setups route requests to upstream groups with generic names, and a key-value store maps the real names to them. In that case, the module can report the statistics under the real names. For details, see [Report upstream groups under their real names](#prom_keyval).

### Exported metrics {#data}

The module exports these NGINX Plus status metrics:

- [nginx](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_object): `/nginx`
- [Processes](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_processes): `/processes`
- [Resolvers](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_resolver_zone): `/resolvers/`
- [Connections](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_connections): `/connections`
- [SSL](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_ssl_object): `/ssl`
- [Slabs](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_slab_zone): `/slabs/`
- [Workers](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_worker): `/workers/`
- [HTTP](https://nginx.org/en/docs/http/ngx_http_api_module.html#http_): `/http/`
- [HTTP Requests](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_requests): `/http/requests`
- [HTTP Caches](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_cache): `/http/caches/`
- [HTTP Server Zones](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_server_zone): `/http/server_zones/`
- [HTTP Location Zones](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_location_zone): `/http/location_zones/`
- [HTTP Upstreams](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream): `/http/upstreams/`
- [HTTP Limit Conn](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_limit_conn_zone): `/http/limit_conns/`
- [HTTP Limit Req](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_limit_req_zone): `/http/limit_reqs/`
- [Stream Server Zones](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_server_zone): `/stream/server_zones/`
- [Stream Zone Sync](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_zone_sync): `/stream/zone_sync/`
- [Stream Upstreams](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_upstream): `/stream/upstreams/`
- [Stream Limit Conn](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_limit_conn_zone): `/stream/limit_conns/`

The module converts the `state` values of servers in [`/http/upstreams/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_http_upstream) and [`/stream/upstreams/`](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_upstream) to numbers:

{{< table >}}

| NGINX Plus state | Prometheus value |
|---|---|
| `up` | `1` |
| `draining` | `2` |
| `down` | `3` |
| `unavail` | `4` |
| `checking` | `5` |
| `unhealthy` | `6` |

{{< /table >}}

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus**: NGINX Plus installed from the NGINX Plus repository. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: For details, see [Dynamic modules]({{< ref "/nginx/technical-specs.md#dynamic-modules" >}}) in the technical specifications.
- **Prometheus**: A Prometheus server that can reach NGINX Plus over HTTP.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

## Install the module package {#install}

Install the `nginx-plus-module-prometheus` package from the NGINX Plus repository. The package manager also installs the [njs module]({{< ref "/nginx/admin-guide/dynamic-modules/nginscript.md" >}}) package, `nginx-plus-module-njs`.

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-prometheus
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-prometheus
    ```

- SLES:

    ```shell
    sudo zypper install nginx-plus-module-prometheus
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-prometheus
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-prometheus
    ```

---

## Load the njs module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive for the njs module:

    ```nginx
    load_module modules/ngx_http_js_module.so;
    ```

    If another directive already loads this module, for example for njs, don't load it again.

---

## Configure the module {#conf}

1. In the `http` block, import the script of the module with [`js_import`](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_import):

    ```nginx
    js_import /usr/share/nginx-plus-module-prometheus/prometheus.js;
    ```

1. In a `server` block, turn on the [NGINX Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html#api), and limit who can reach it:

    ```nginx
    location /api {
        api;
        allow 127.0.0.1;
        deny  all;
    }
    ```

    The module reads the API with subrequests to `/api`, so keep that location name. NGINX Plus doesn't apply `allow` and `deny` to subrequests, so the module still reaches the API.

1. In the same `server` block, add a location for the metrics, for example `/metrics`:

    ```nginx
    location = /metrics {
        js_content prometheus.metrics;
    }
    ```

1. Optional: If the error log shows `too big subrequest response`, make the buffer for subrequest responses bigger with [`subrequest_output_buffer_size`](https://nginx.org/en/docs/http/ngx_http_core_module.html#subrequest_output_buffer_size) in the `http` block:

    ```nginx
    subrequest_output_buffer_size 32k;
    ```

1. Add NGINX Plus to a [`scrape_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config) section of the Prometheus configuration. Replace `nginx-plus.example.com:80` with the address of NGINX Plus. Prometheus reads `/metrics` by default.

    ```yaml
    scrape_configs:
      - job_name: nginx-plus
        static_configs:
          - targets: ['nginx-plus.example.com:80']
    ```

This is a complete NGINX Plus configuration:

```nginx
load_module modules/ngx_http_js_module.so;

events {}

http {
    js_import /usr/share/nginx-plus-module-prometheus/prometheus.js;
    subrequest_output_buffer_size 32k;

    upstream backend {
        zone   backend 64k;
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        listen 80;
        status_zone backend_zone;

        location / {
            proxy_pass http://backend;
        }

        location /api {
            api;
            allow 127.0.0.1;
            deny  all;
        }

        location = /metrics {
            js_content prometheus.metrics;
        }
    }
}
```

---

## Set the module variables {#vars}

Set these variables with the [`set`](https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#set) directive in the metrics location:

- [`$prom_keyval`](#prom_keyval)
- [`$prom_keyval_stream`](#prom_keyval_stream)
- [`$prom_metrics_disabled`](#prom_metrics_disabled)

### Report upstream groups under their real names {#prom_keyval}

Upstream groups can have generic names, with a [key-value store](https://nginx.org/en/docs/http/ngx_http_keyval_module.html) that maps the real names to them. Then set `$prom_keyval` to the name of that store. The module then reports the statistics under the real names. In this example, the store is `http_upstream_keyval`:

```nginx
http {
    keyval_zone zone=http_upstream_keyval:64k;
    keyval      $host $upstream zone=http_upstream_keyval;

    upstream 0 {
        zone   http_backend 64k;
        server backend1.example.com;
        server backend2.example.com;
    }

    server {
        location / {
            proxy_pass http://$upstream;
        }

        location /api {
            api;
        }

        location = /metrics {
            set        $prom_keyval "http_upstream_keyval";
            js_content prometheus.metrics;
        }
    }
}
```

### Report stream upstream groups under their real names {#prom_keyval_stream}

For upstream groups in the [`stream`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#stream) context, define a key-value store in the `stream` block, and set `$prom_keyval_stream` to its name:

```nginx
http {
    # The same http block as in the previous example,
    # with this metrics location:
    # ...
    server {
        # ...
        location = /metrics {
            set $prom_keyval        "http_upstream_keyval";
            set $prom_keyval_stream "stream_upstream_keyval";
            js_content              prometheus.metrics;
        }
    }
}

stream {
    keyval_zone zone=stream_upstream_keyval:32k;

    upstream stream_backend {
        zone   stream_backend 64k;
        server backend1.example.com:12345;
        server backend2.example.com:12345;
    }

    server {
        listen      12345;
        proxy_pass  stream_backend;
        status_zone backend_stream_zone;
    }
}
```

### Leave out metrics {#prom_metrics_disabled}

To save storage in Prometheus, leave out some NGINX Plus metrics. Set `$prom_metrics_disabled` to a comma-separated list of API endpoints:

```nginx
location = /metrics {
    set $prom_metrics_disabled "stream/upstreams, resolvers";
    js_content                 prometheus.metrics;
}
```

You can leave out these endpoints:

- `nginx`
- `processes`
- `resolvers`
- `connections`
- `ssl`
- `slabs`
- `workers`
- `http/requests`
- `http/caches`
- `http/upstreams`
- `http/server_zones`
- `http/location_zones`
- `http/limit_conns`
- `http/limit_reqs`
- `stream/upstreams`
- `stream/server_zones`
- `stream/zone_sync`
- `stream/limit_conns`

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

1. Request the metrics:

    ```shell
    curl -s http://localhost/metrics | head
    ```

    The output shows metrics in the Prometheus text format.

---

## Troubleshooting

### The error log shows "too big subrequest response"

**Symptom**: The metrics are missing or incomplete, and the error log shows `too big subrequest response`.

**Cause**: An API response is bigger than the buffer for subrequest responses.

**Fix**: Set `subrequest_output_buffer_size` to a bigger value, for example `32k`, as described in [Configure the module](#conf).

---

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] unknown directive "js_import"`.

**Cause**: No `load_module` directive loads the njs module.

**Fix**: Add the `load_module` directive, as described in [Load the njs module](#load-the-njs-module).

---

### The configuration uses js_include

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] unknown directive "js_include"`.

**Cause**: The configuration was written for Prometheus-njs before version 1.3.1. It uses `js_include`, which njs deprecated in version 0.4.0 and removed in version 0.7.1.

**Fix**: Replace `js_include` with `js_import`, and refer to the function with the module name, `prometheus.metrics`, as shown in [Configure the module](#conf).

---

## References {#info}

For more information, see:

- [Module ngx_http_api_module](https://nginx.org/en/docs/http/ngx_http_api_module.html)
- [njs scripting language](https://nginx.org/en/docs/njs/)
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
