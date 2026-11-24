---
description: The retired NGINX plug-in for New Relic, version 2, and how to monitor NGINX Open Source and F5 NGINX Plus with New Relic today.
title: Monitoring NGINX and NGINX Plus with the New Relic Plug-In
toc: true
weight: 500
f5-content-type: reference
f5-product: F5 NGINX Plus
f5-keywords: "New Relic, plug-in, nginx-nr-agent, monitoring, stub_status, NGINX Plus API, infrastructure agent"
f5-summary: >
  This page describes version 2 of the NGINX plug-in for New Relic, which sent metrics of NGINX Open Source and F5 NGINX Plus to New Relic.
  New Relic retired its plug-ins on June 16, 2021, so the plug-in no longer works.
  To monitor NGINX with New Relic today, use the NGINX integration of the New Relic infrastructure agent.
f5-audience: operator
---

<span id="installation"></span><span id="whats-next"></span>

## Overview

The NGINX plug-in for New Relic, `nginx-nr-agent`, sent the metrics of NGINX Open Source and F5 NGINX Plus to New Relic. Version 2.0 of the plug-in was written in Python, and came as packages for RHEL-based and Debian-based systems.

{{< call-out class="caution" >}} New Relic retired its plug-ins on June 16, 2021, so this plug-in no longer sends data to New Relic. To monitor NGINX with New Relic today, use the [NGINX integration](https://docs.newrelic.com/install/nginx/) of the New Relic infrastructure agent. It reads the metrics of NGINX Open Source from [`stub_status`](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html#stub_status), and the metrics of NGINX Plus from the [NGINX Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html). {{< /call-out >}}

The rest of this page describes the retired plug-in, for reference.

---

<span id="whats-new"></span>

## What version 2 of the plug-in added

- The plug-in was rewritten in Python, so it no longer needed Ruby.
- It came as packages for RHEL-based and Debian-based systems, with an init script that could start the plug-in at boot.
- For NGINX Plus, it had two new tabs in the New Relic UI:

    - **Servers**: Summary counters for the virtual servers whose `server` block has the [`status_zone`](https://nginx.org/en/docs/http/ngx_http_api_module.html#status_zone) directive.
    - **Cache**: Cumulative statistics for all configured caches.

- It logged verbose messages by default.

---

<span id="configuring-the-plugin"></span>

## Configure the plug-in

The configuration file of the plug-in is **/etc/nginx-nr-agent/nginx-nr-agent.ini**. The minimal configuration has:

- Your New Relic license key, in the `newrelic_license_key` setting of the `global` section.
- At least one `source` section. Its name appears only in log entries, and can be almost any string. It needs two settings:

    - `name`: The name of the NGINX instance in the New Relic UI.
    - `url`: The full URL of the status data of the instance. The plug-in reads the output of the [`stub_status`](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html#stub_status) directive of NGINX Open Source and NGINX Plus. It also reads the JSON output of the [`api`](https://nginx.org/en/docs/http/ngx_http_api_module.html#api) directive of NGINX Plus.

If the location of the status data has HTTP basic authentication, set its credentials with the optional `http_user` and `http_pass` settings. For basic authentication in NGINX, see the [`auth_basic`](https://nginx.org/en/docs/http/ngx_http_auth_basic_module.html#auth_basic) directive.

The default log file is **/var/log/nginx-nr-agent.log**.

---

<span id="running-the-plugin"></span>

## Run the plug-in

The init script manages the plug-in. Run `service nginx-nr-agent` with `start`, `stop`, or `status` to start the plug-in, stop it, or show its status. When the init script starts the plug-in, its daemon runs as the `nobody` user.

To start the plug-in directly, run `/usr/bin/nginx-nr-agent.py`, with these optional parameters:

- `-c`, `--config`: The path of the configuration file.
- `-p`, `--pidfile`: The path of the PID file.
- `-f`, `--foreground`: Don't detach from the terminal, which helps with debugging.

When the plug-in worked, the NGINX instance appeared in the New Relic UI:

<img src="/nginx/images/0s-instances.png" alt="An NGINX instance in the list of instances of the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="dashboard-examples"></span>

## Dashboard tabs

The plug-in showed the metrics of NGINX in these tabs of the New Relic UI.

### Overview

The **Overview** tab showed the basic metrics of the instance: the numbers of active and idle connections, and the request rate.

<img src="/nginx/images/1s-overview.png" alt="The Overview tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

### Connections

The **Connections** tab showed metrics of client connections.

<img src="/nginx/images/2s-connections.png" alt="The Connections tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

### Requests

The **Requests** tab showed the number of requests in progress, and the request rate.

<img src="/nginx/images/3s-requests.png" alt="The Requests tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="upstreams-nginxplus-only"></span>

### Upstreams (NGINX Plus only)

The **Upstreams** tab showed detailed statistics of all configured upstream groups.

<img src="/nginx/images/4s-upstreams.png" alt="The Upstreams tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="servers-nginxplus-only"></span>

### Servers (NGINX Plus only)

The **Servers** tab showed detailed statistics of all configured virtual server zones.

<img src="/nginx/images/5s-servers.png" alt="The Servers tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

<span id="cache-nginxplus-only"></span>

### Cache (NGINX Plus only)

The **Cache** tab showed summary statistics of all configured caches.

<img src="/nginx/images/6s-cache.png" alt="The Cache tab of the NGINX plug-in in the New Relic UI" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

## References

For more information, see:

- [The NGINX integration of New Relic](https://docs.newrelic.com/install/nginx/)
- [Live Activity Monitoring]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}})
- [Module ngx_http_stub_status_module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html)
- [Module ngx_http_api_module](https://nginx.org/en/docs/http/ngx_http_api_module.html)
