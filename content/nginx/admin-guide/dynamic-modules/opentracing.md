---
title: OpenTracing
description: Replace the removed OpenTracing dynamic module in F5 NGINX Plus with the OpenTelemetry dynamic module for distributed tracing.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "opentracing, opentelemetry, otel, distributed tracing, nginx-plus-module-opentracing, nginx-plus-module-otel, migration"
f5-summary: >
  This guide shows how to replace the OpenTracing dynamic module with the OpenTelemetry dynamic module.
  F5 removed the OpenTracing module package from the NGINX Plus repository in NGINX Plus R34.
  It maps the OpenTracing directives to their OpenTelemetry equivalents.
f5-audience: operator
---

## Overview

The OpenTracing dynamic module sent traces from F5 NGINX Plus to distributed tracing services, such as Jaeger, Zipkin, and Datadog. It used a tracer plugin for each service. F5 deprecated it in [NGINX Plus R31]({{< ref "/nginx/releases.md#r31" >}}) and removed the `nginx-plus-module-opentracing` package in [NGINX Plus R34]({{< ref "/nginx/releases.md#r34" >}}).

The [OpenTelemetry]({{< ref "/nginx/admin-guide/dynamic-modules/opentelemetry.md" >}}) dynamic module replaces it. It exports traces with the OpenTelemetry protocol (OTLP) over gRPC, and it uses W3C `traceparent` and `tracestate` headers for the trace context. When you finish this guide, NGINX Plus sends its traces through the OpenTelemetry module.

---

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus R29 or later**: The OpenTelemetry module package, `nginx-plus-module-otel`, is available since NGINX Plus R29.
- **An OTLP endpoint**: A service that receives traces over OTLP and gRPC, such as the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/). It forwards the traces to your tracing back end.
- **Root access**: You need `sudo` to change the configuration and to install packages.

---

## Replace the OpenTracing directives

1. Find the OpenTracing directives in the configuration:

    ```shell
    sudo grep -rn opentracing /etc/nginx/
    ```

    The output shows the file and line number of each directive.

1. Install and load the OpenTelemetry module, as described in [OpenTelemetry]({{< ref "/nginx/admin-guide/dynamic-modules/opentelemetry.md" >}}).

1. Replace each OpenTracing directive with its OpenTelemetry equivalent:

    {{< table >}}

   | OpenTracing directive | OpenTelemetry directive |
   |---|---|
   | `opentracing on;` | `otel_trace on;` |
   | `opentracing_load_tracer`, with a tracer plugin and its configuration file | An `otel_exporter` block in the `http` context, with the `endpoint` of your OTLP service |
   | The service name in the configuration file of the tracer | `otel_service_name`, in the `http` context |
   | `opentracing_operation_name` | `otel_span_name` |
   | `opentracing_tag` | `otel_span_attr` |
   | No `opentracing_propagate_context` directive, so only `opentracing_trust_incoming_span on;`, the default | `otel_trace_context extract;` |
   | `opentracing_propagate_context;`, with the default `opentracing_trust_incoming_span on;` | `otel_trace_context propagate;` |
   | `opentracing_propagate_context;` with `opentracing_trust_incoming_span off;` | `otel_trace_context inject;` |

    {{< /table >}}

    For the other directives, see the [OpenTelemetry module documentation](https://nginx.org/en/docs/ngx_otel_module.html).

1. Delete the `load_module` directive of the module: `load_module modules/ngx_http_opentracing_module.so;`

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

1. If the `nginx-plus-module-opentracing` package is still installed, remove it as described in [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}}).

---

## Verify the traces

1. Send a request to a location with `otel_trace on`. Replace `localhost` with your server name:

    ```shell
    curl -s -o /dev/null -w '%{http_code}\n' http://localhost/
    ```

1. Find the trace of the request in your tracing back end. Its service name is the value of `otel_service_name`.

---

## Troubleshooting

### nginx -t reports an unknown directive

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] unknown directive "opentracing"`.

**Cause**: NGINX Plus no longer loads the OpenTracing module, but the configuration still uses its directives.

**Fix**: Replace the directives, as described in [Replace the OpenTracing directives](#replace-the-opentracing-directives).

---

### Traces lose their parent span

**Symptom**: The traces of NGINX Plus and of your other services are no longer connected.

**Cause**: The OpenTelemetry module reads and writes only the W3C `traceparent` and `tracestate` headers. Tracer plugins for OpenTracing used their own headers, such as `uber-trace-id` for Jaeger.

**Fix**: Set up your other services to use W3C trace context.

---

## References

For more information, see:

- [OpenTelemetry]({{< ref "/nginx/admin-guide/dynamic-modules/opentelemetry.md" >}})
- [Module ngx_otel_module](https://nginx.org/en/docs/ngx_otel_module.html)
- [The nginx-opentracing project on GitHub](https://github.com/opentracing-contrib/nginx-opentracing)
- [Uninstall a dynamic module]({{< ref "/nginx/admin-guide/dynamic-modules/uninstall.md" >}})
