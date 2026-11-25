---
title: OpenTelemetry
description: Install the OpenTelemetry dynamic module in F5 NGINX Plus to send distributed traces of requests to an OTLP endpoint.
toc: true
weight: 100
f5-content-type: installation-guide
f5-product: F5 NGINX Plus
f5-keywords: "OpenTelemetry, OTel, distributed tracing, otel_exporter, otel_trace, OTLP, nginx-plus-module-otel, dynamic module"
f5-summary: >
  This guide shows how to install the OpenTelemetry dynamic module in F5 NGINX Plus and turn on tracing.
  The module sends a trace span for each request to an OTLP endpoint over gRPC, such as an OpenTelemetry Collector.
  It replaces the OpenTracing module, which F5 removed in NGINX Plus R34.
f5-audience: operator
---

## Overview

[OpenTelemetry](https://opentelemetry.io/) (OTel) is an observability framework for tracing, monitoring, and troubleshooting applications. The OpenTelemetry dynamic module adds distributed tracing to F5 NGINX Plus. It sends spans of the requests that NGINX Plus handles to an OpenTelemetry Protocol (OTLP) endpoint over gRPC. For the trace context, it uses W3C `traceparent` and `tracestate` headers.

The module has these features:

- **Low overhead**: Its [project page](https://github.com/nginx/nginx-otel) says that it adds about 10 to 15% to request processing time. Other OpenTelemetry implementations can add up to 50%.
- **Configuration in NGINX**: You set it up with NGINX directives.
- **Per-request control**: Variables, such as cookies and tokens, can turn tracing on or off for each request. For an example, see [Trace a share of requests](#trace-a-share-of-requests).
- **Changes at runtime**: You can change sampling with the [NGINX Plus API]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md#using-the-rest-api" >}}) and the [key-value store]({{< ref "/nginx/admin-guide/security-controls/denylisting-ip-addresses.md" >}}), without a reload.

This is an [NGINX-authored module]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#nginx-authored-dynamic-modules" >}}). The `nginx-plus-module-otel` package in the NGINX Plus repository contains it, since NGINX Plus R29. Its reference is the [ngx_otel_module documentation](https://nginx.org/en/docs/ngx_otel_module.html). It replaces the [OpenTracing]({{< ref "/nginx/admin-guide/dynamic-modules/opentracing.md" >}}) module, which F5 removed in [NGINX Plus R34]({{< ref "/nginx/releases.md#r34" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus R29 or later**: NGINX Plus installed from the NGINX Plus repository. To check your release, run `nginx -v`. For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).
- **A supported operating system**: The module package isn't available for SLES 15 SP5 and later. For the operating systems of NGINX Plus, see the [technical specifications]({{< ref "/nginx/technical-specs.md" >}}).
- **An OTLP endpoint**: A service that receives traces over OTLP and gRPC, such as the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/). It forwards the traces to your tracing back end.
- **Root access**: You need `sudo` to install packages and to change the configuration.

---

<span id="installation"></span>

## Install the module package

Install the `nginx-plus-module-otel` package from the NGINX Plus repository:

- RHEL and its derivatives, and Amazon Linux 2023:

    ```shell
    sudo dnf install nginx-plus-module-otel
    ```

- Debian and Ubuntu:

    ```shell
    sudo apt update
    sudo apt install nginx-plus-module-otel
    ```

- Alpine Linux:

    ```shell
    sudo apk add nginx-plus-module-otel
    ```

- FreeBSD:

    ```shell
    sudo pkg install nginx-plus-module-otel
    ```

---

## Load the module

1. Open the NGINX Plus configuration file: `/etc/nginx/nginx.conf` on Linux, or `/usr/local/etc/nginx/nginx.conf` on FreeBSD.

1. At the top of the file, before the `events` block, add the [`load_module`](https://nginx.org/en/docs/ngx_core_module.html#load_module) directive:

    ```nginx
    load_module modules/ngx_otel_module.so;
    ```

---

<span id="configuration"></span><span id="usage-examples"></span>

## Configure the module

1. In the `http` block, set where NGINX Plus sends the traces, and the name of the service in the traces:

    ```nginx
    otel_exporter {
        endpoint localhost:4317;
    }

    otel_service_name nginx-plus;
    ```

    Replace `localhost:4317` with the host and port of your OTLP endpoint. To connect to it over TLS, start the value with `https://`. Without `otel_service_name`, traces show the service name `unknown_service:nginx`.

1. Turn on tracing with [`otel_trace`](https://nginx.org/en/docs/ngx_otel_module.html#otel_trace), and choose how to handle the trace context of requests with [`otel_trace_context`](https://nginx.org/en/docs/ngx_otel_module.html#otel_trace_context). The next sections show three common setups.

For all directives, variables, and default span attributes, see the [ngx_otel_module documentation](https://nginx.org/en/docs/ngx_otel_module.html).

<span id="simple-tracing"></span>

### Trace every request

This setup traces each request, even when no other service sends trace context. `otel_trace_context inject` adds a new trace context to the proxied request:

```nginx
location / {
    otel_trace         on;
    otel_trace_context inject;
    proxy_pass         http://backend;
}
```

<span id="parent-based-tracing"></span>

### Trace the requests that the caller traces

This setup traces a request only if the caller traced it. The [`$otel_parent_sampled`](https://nginx.org/en/docs/ngx_otel_module.html#var_otel_parent_sampled) variable holds the sampled flag of the incoming trace context. `otel_trace_context propagate` keeps that context and passes it on to the proxied server:

```nginx
location / {
    otel_trace         $otel_parent_sampled;
    otel_trace_context propagate;
    proxy_pass         http://backend;
}
```

<span id="ratio-based-tracing"></span>

### Trace a share of requests

This setup traces 10% of requests. The [`split_clients`](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html) block sets the `$ratio_sampler` variable to `on` for 10% of the trace IDs. To trace 10% of user sessions instead, use a session cookie, such as `$cookie_sessionid`, in place of `$otel_trace_id`.

```nginx
split_clients $otel_trace_id $ratio_sampler {
    10%     on;
    *       off;
}

server {
    location / {
        otel_trace         $ratio_sampler;
        otel_trace_context inject;
        proxy_pass         http://backend;
    }
}
```

### Complete example

This configuration traces every request to the proxied servers in the `backend` group:

```nginx
load_module modules/ngx_otel_module.so;

events {}

http {
    otel_exporter {
        endpoint localhost:4317;
    }

    otel_service_name nginx-plus;

    upstream backend {
        server 127.0.0.1:8080;
    }

    server {
        listen 80;

        location / {
            otel_trace         on;
            otel_trace_context inject;
            proxy_pass         http://backend;
        }
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

1. Send a request to a location with tracing. Replace `localhost` with your server name:

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' http://localhost/
    ```

1. Find the trace of the request in your tracing back end. Its service name is the value of `otel_service_name`.

---

## Troubleshooting

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "otel_exporter"`.

**Cause**: The configuration uses a directive of the module, but no `load_module` directive loads the module.

**Fix**: Add the `load_module` directive, as described in [Load the module](#load-the-module).

---

### nginx -t reports that the otel_exporter block is missing

**Symptom**: `nginx -t` fails with the message `nginx: [emerg] "otel_exporter" block is missing`.

**Cause**: The configuration turns on tracing with `otel_trace`, but it doesn't say where to send the traces.

**Fix**: Add an `otel_exporter` block with an `endpoint` to the `http` block, as described in [Configure the module](#configure-the-module).

---

### Traces don't appear in the tracing back end

**Symptom**: Requests succeed, but your tracing back end shows no traces from NGINX Plus.

**Cause**: NGINX Plus can't reach the OTLP endpoint, or the endpoint doesn't forward the traces to the back end.

**Fix**: Check the `endpoint` value and the NGINX Plus error log. Then check that the endpoint accepts OTLP over gRPC on that port.

---

<span id="more-info"></span>

## References

For more information, see:

- [Module ngx_otel_module](https://nginx.org/en/docs/ngx_otel_module.html)
- [The nginx-otel project on GitHub](https://github.com/nginx/nginx-otel)
- [OpenTracing]({{< ref "/nginx/admin-guide/dynamic-modules/opentracing.md" >}}), for moving from the OpenTracing module
- [Dynamic Modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}})
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
