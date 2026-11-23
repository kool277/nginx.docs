---
title: Migrating Load Balancer Configuration from Citrix ADC to NGINX Plus
description: Convert Citrix ADC virtual servers, service groups, TLS, persistence, and monitors into equivalent F5 NGINX Plus configuration.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "migration, Citrix ADC, NetScaler, virtual server, service group, SSL termination, session persistence, monitor, health check, NGINX Plus"
f5-summary: >
  This guide shows how to convert the load balancing configuration of Citrix ADC into F5 NGINX Plus configuration.
  For each Citrix ADC feature, it shows the CLI commands and the equivalent NGINX Plus directives.
  It ends with a complete NGINX Plus configuration, and notes on high availability and logging.
f5-audience: operator
---

<span id="about-nginx"></span>

## Overview

F5 NGINX Plus is a software replacement for hardware [application delivery controllers](https://www.f5.com/glossary/application-delivery-controller) (ADCs). It runs on bare metal, virtual machines, or containers, on premises or in the cloud. This guide shows how to move the load balancing configuration of Citrix ADC, formerly Citrix NetScaler, to NGINX Plus.

NGINX Plus and Citrix ADC are both full reverse proxies and load balancers. The client sees the load balancer as the application, and the back-end servers see the load balancer as the client. That gives both products fine-grained control over the traffic.

[NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) adds features to [NGINX Open Source](https://nginx.org/en/), such as active health checks, the [NGINX Plus API]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}), and [high availability]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}).

---

<span id="prereqs"></span>

## Before you begin

Before you begin, make sure you have:

- **Citrix ADC knowledge**: You know its concepts and its CLI commands.
- **NGINX knowledge**: You can read NGINX configuration. The guide links to the documentation of each directive.
- **NGINX Plus**: For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

All IP addresses and object names in this guide are examples. Replace them with the values of your Citrix ADC configuration.

---

<span id="networking"></span>

## Map the Citrix ADC networking concepts

Citrix ADC defines three types of IP addresses:

{{< table >}}

| Citrix ADC address | NGINX Plus equivalent |
|---|---|
| **Citrix ADC IP address (NSIP)**: the management address of an appliance | The IP address of the NGINX Plus host. |
| **Subnet IP address (SNIP)**: the source address that the back-end servers see | By default, also the IP address of the NGINX Plus host. Citrix ADC manages its routing table in its CLI or GUI. For NGINX Plus, you change the routing of the Linux or FreeBSD host. To choose the source address yourself, use [`proxy_bind`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_bind). |
| **Virtual IP address (VIP)**: the address that clients connect to | The same concept. If the primary device fails, the VIP moves to the backup. See [Set up high availability](#ha). |

{{< /table >}}

---

<span id="convert-configuration"></span>

## Convert the load balancer configuration

Citrix ADC stores its configuration as CLI commands, even for changes that you make in the GUI. So this guide shows CLI commands. NGINX Plus keeps its configuration in text files with directives.

<span id="virtual-servers"></span>

### Virtual servers

Citrix ADC chooses a virtual server only by IP address and port. To use the `Host` header too, you need AppExpert policies or a content switching virtual server.

An NGINX Plus virtual server is a [`server`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) block. Its [`listen`](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) directives set the IP addresses and ports, and its [`server_name`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) directives match the `Host` header. Server names can be host names, wildcards, or regular expressions. For details, see [Server names](https://nginx.org/en/docs/http/server_names.html).

This example is a virtual server for host names that end in `.example.com`.

Citrix ADC:

```none
add lb vserver myvserver HTTP 10.0.0.99 80
```

NGINX Plus:

```nginx
server {
    listen 10.0.0.99:80;
    server_name .example.com;
    #...
}
```

The `listen` address must exist on the NGINX Plus host. Otherwise, `nginx -t` fails with `bind() to 10.0.0.99:80 failed (99: Cannot assign requested address)`.

<span id="ssl"></span>

### SSL/TLS termination

In these examples, the load balancer terminates TLS, so the back-end servers don't need to. NGINX Plus uses the OpenSSL library of the operating system. It reads the certificate and key from the [`ssl_certificate` and `ssl_certificate_key`](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate) directives.

Citrix ADC:

```none
add ssl certKey test.crt -cert test.crt -key test.key
add lb vserver mysslvserver SSL 10.0.0.98 443
bind ssl vserver mysslvserver -certkeyName test.crt
```

NGINX Plus:

```nginx
server {
    listen 10.0.0.98:443 ssl;
    ssl_certificate     test.crt;
    ssl_certificate_key test.key;
    #...
}
```

Relative file names start in the configuration directory, for example `/etc/nginx`.

<span id="service-group"></span>

### Service group entities

Citrix ADC uses three entity types for back-end servers:

- **Server**: the IP address or host name of a back-end server.
- **Service**: a server with a port and a monitor.
- **Service group**: a group of servers and ports with a monitor.

NGINX Plus describes a group of back-end servers with an [`upstream`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) block. It has a [`server`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) directive for each server. These examples define the group `myapp` with three servers.

Citrix ADC:

```none
add serviceGroup myapp HTTP
bind serviceGroup myapp 10.0.0.100 80
bind serviceGroup myapp 10.0.0.101 80
bind serviceGroup myapp 10.0.0.102 80
```

NGINX Plus:

```nginx
upstream myapp {
    server 10.0.0.100:80;
    server 10.0.0.101:80;
    server 10.0.0.102:80;
}
```

<span id="session-persistence"></span>

### Session persistence

Stateful applications need session persistence, and it also helps with continuous delivery. Citrix ADC sets session persistence on the virtual server. NGINX Plus sets it on the group of back-end servers, the `upstream` block, with the [`sticky`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky) directive. That directive is also in NGINX Open Source since version 1.29.6.

#### Cookie persistence

The `sticky cookie` method works like the cookie insert method of Citrix ADC. The load balancer adds a cookie that names the server, and the client sends it back with later requests. It's simple to set up, and if that server fails, the load balancer chooses another one.

Citrix ADC:

```none
add lb vserver mysslvserver SSL 10.0.0.91 443 -persistenceType COOKIEINSERT -timeout 60 -cookieName mysession
```

NGINX Plus:

```nginx
upstream myapp {
    server 10.0.0.100:80;
    server 10.0.0.101:80;
    server 10.0.0.102:80;
    sticky cookie mysession expires=1h;
}
```

#### JSESSIONID persistence

Both products can also use a token that the server creates, such as the `JSESSIONID` cookie. NGINX Plus calls this method `sticky learn`.

Citrix ADC:

```none
set lb vserver mysslvserver -persistencetype RULE -rule 'HTTP.REQ.COOKIE.VALUE("jsessionid")' -resRule 'HTTP.RES.SET_COOKIE.COOKIE("jsessionid")'
```

NGINX Plus:

```nginx
upstream myapp {
    server 10.0.0.100:80;
    server 10.0.0.101:80;
    server 10.0.0.102:80;
    sticky learn create=$upstream_cookie_jsessionid
                 lookup=$cookie_jsessionid
                 zone=client_sessions:1m;
}
```

A third method, `sticky route`, is comparable to the [custom server ID persistence](https://docs.netscaler.com/us/en/citrix-adc/current-release/load-balancing/load-balancing-persistence/custom-server-id-persistence.html) of Citrix ADC.

<span id="health-checks"></span>

### Monitors and health checks

Citrix ADC calls them monitors, and NGINX Plus calls them [health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}}). You attach a Citrix ADC monitor to a service or service group. An NGINX Plus health check goes in the `location` that proxies to the group. Active health checks are an NGINX Plus feature.

#### Default health check

In these examples, the load balancer requests `/` from the servers. Citrix ADC sends a `HEAD` request and accepts only the status `200`. NGINX Plus sends a `GET` request, and by default accepts any `2xx` or `3xx` status.

Citrix ADC:

```none
add lb monitor httphealth HTTP -respCode 200 -httpRequest "HEAD /"
bind serviceGroup myapp -monitorName httphealth
```

NGINX Plus:

```nginx
location / {
    proxy_pass http://myapp;
    health_check;
}
```

#### Custom health check

Both products can also check the response in detail. These examples mark a server as healthy if it returns the status `200` and a body that contains `Welcome to nginx!`.

Citrix ADC:

```none
add lb monitor httphealth-ecv HTTP-ECV -send "GET /" -recv "Welcome to nginx!"
bind serviceGroup myapp -monitorName httphealth-ecv
```

NGINX Plus:

```nginx
match server_ok {
    status 200;
    body ~ "Welcome to nginx!";
}

server {
    #...
    location / {
        proxy_pass http://myapp;
        health_check match=server_ok;
    }
}
```

<span id="configuration-summary"></span>

### Review the complete configuration

These examples combine the commands and directives of the previous sections. The NGINX Plus configuration adds three directives:

- [`zone`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#zone) keeps the state of the group in shared memory, which health checks need.
- [`proxy_set_header`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) passes the `Host` header of the client request to the servers.
- [`proxy_http_version`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version) uses HTTP/1.1 for the connections to the servers. Since NGINX 1.29.7, HTTP/1.1 is the default.

Citrix ADC:

```none
add ssl certKey test.crt -cert test.crt -key test.key
add lb vserver mysslvserver SSL 10.0.0.98 443 -persistenceType COOKIEINSERT -timeout 60 -cookieName mysession
bind ssl vserver mysslvserver -certkeyName test.crt
add serviceGroup myapp HTTP
bind serviceGroup myapp 10.0.0.100 80
bind serviceGroup myapp 10.0.0.101 80
bind serviceGroup myapp 10.0.0.102 80
add lb monitor httphealth HTTP -respCode 200 -httpRequest "HEAD /"
bind serviceGroup myapp -monitorName httphealth
bind lb vserver mysslvserver myapp
```

NGINX Plus:

```nginx
upstream myapp {
    zone myapp 64k;
    server 10.0.0.100:80;
    server 10.0.0.101:80;
    server 10.0.0.102:80;
    sticky cookie mysession expires=1h;
}

server {
    listen 10.0.0.98:443 ssl default_server;
    ssl_certificate     test.crt;
    ssl_certificate_key test.key;
    proxy_set_header    Host $host;

    location / {
        proxy_pass         http://myapp;
        health_check;
        proxy_http_version 1.1;
    }
}
```

---

<span id="ha"></span>

## Set up high availability

NGINX Plus and Citrix ADC handle high availability (HA) in similar ways. Citrix ADC monitors the instances and moves the VIP in a proprietary way.

For [on-premises deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}), NGINX Plus uses the `nginx-ha-keepalived` package. It moves the VIP between an active-passive pair of NGINX Plus instances with the VRRP protocol. For pairs where both instances handle traffic, see [active-active HA with keepalived]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived-nodes.md" >}}). For cloud environments, see these guides:

- [Active-Active HA for NGINX Plus on AWS Using AWS Network Load Balancer]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-network-load-balancer.md" >}})
- [Active-Passive HA for NGINX Plus on AWS Using Elastic IP Addresses]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-keepalived.md" >}})
- [All-Active HA for NGINX Plus on the Google Cloud Platform]({{< ref "/nginx/deployment-guides/google-cloud-platform/high-availability-all-active.md" >}})
- [Active-Active HA for NGINX Plus on Microsoft Azure Using the Azure Standard Load Balancer]({{< ref "/nginx/deployment-guides/microsoft-azure/high-availability-standard-load-balancer.md" >}})

---

<span id="logging"></span>

## Logging and monitoring

Citrix ADC logs errors in its event log, and NGINX Plus in its [error log](https://nginx.org/en/docs/ngx_core_module.html#error_log). By default, Citrix ADC doesn't log individual requests; it needs a separate web logging client for that. NGINX Plus logs each request in its [access log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) by default.

The [NGINX Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html) collects many statistics. You can read them through the API, show them on the [dashboard]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}), or pass them to other monitoring tools.

---

## Troubleshooting

### nginx -t can't bind to the virtual server address

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] bind() to 10.0.0.98:443 failed (99: Cannot assign requested address)`.

**Cause**: The `listen` directive uses an IP address that the NGINX Plus host doesn't have.

**Fix**: Add the address to an interface of the host, or change the `listen` directive to an address of the host.

---

## References

For more information, see:

- [HTTP load balancing]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md" >}})
- [HTTP health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}})
- [Migrating load balancer configuration from F5 BIG-IP LTM to F5 NGINX Plus]({{< ref "/nginx/deployment-guides/migrate-hardware-adc/f5-big-ip-configuration.md" >}})

---

## Revision history

- Version 4 (November 2026): Rewritten to the how-to template. Fixes the name of the virtual server in the summary, and says that stateful applications need session persistence.
- Version 3 (April 2019): Product name changed to Citrix ADC
- Version 2 (April 2018): Updated information about high availability and the NGINX Plus API (NGINX Plus R13, NGINX Open Source 1.13.4)
- Version 1 (November 2016): Initial version (NGINX Plus R11, NGINX 1.11.5)
