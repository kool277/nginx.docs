---
title: Migrating Load Balancer Configuration from F5 BIG-IP LTM to F5 NGINX Plus
description: Convert F5 BIG-IP LTM virtual servers, pools, TLS profiles, persistence, OneConnect, and monitors into equivalent F5 NGINX Plus configuration.
toc: true
weight: 100
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "migration, F5 BIG-IP LTM, TMSH, virtual server, pool, SSL offload, session persistence, OneConnect, health check, NGINX Plus"
f5-summary: >
  This guide shows how to convert the load balancing configuration of F5 BIG-IP LTM into F5 NGINX Plus configuration.
  For each BIG-IP LTM feature, it shows the TMSH commands and the equivalent NGINX Plus directives.
  It ends with a complete NGINX Plus configuration for a basic BIG-IP LTM setup.
f5-audience: operator
---

<span id="about-nginx"></span>

## Overview

F5 NGINX Plus is a software replacement for hardware [application delivery controllers](https://www.f5.com/glossary/application-delivery-controller) (ADCs). It runs on bare metal, virtual machines, or containers, on premises or in the cloud. This guide helps you move the load balancing configuration of an F5 BIG-IP Local Traffic Manager (LTM) to NGINX Plus. For each feature, it shows the BIG-IP LTM commands and the NGINX Plus configuration that does the same.

[NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) is a software load balancer, reverse proxy, and web server. It adds features to [NGINX Open Source](https://nginx.org/en/), such as active health checks, the [NGINX Plus API]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}), and [high availability]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}).

<span id="scenarios"></span><span id="deployment-scenarios"></span>

### Deployment scenarios

A hardware application delivery controller (ADC) usually sits at the edge of the network, and every request passes through it. NGINX Plus can run in two ways next to or instead of it:

- **Behind the hardware ADC**: The ADC keeps handling the ingress traffic at the edge. NGINX Plus instances handle the application traffic of each application or team.

    <img src="/nginx/images/nginx-plus-behind-hardware-adc.png" alt="NGINX Plus runs behind a hardware ADC and handles the application traffic" style="border:2px solid #666666; padding:2px; margin:2px;" />

- **Instead of the hardware ADC**: NGINX Plus replaces the appliance at the edge and handles all traffic that enters the network.

    <img src="/nginx/images/nginx-plus-without-hardware-adc.png" alt="NGINX Plus replaces the hardware ADC and handles all traffic that enters the network" style="border:2px solid #666666; padding:2px; margin:2px;" />

---

<span id="prereqs"></span>

## Before you begin

Before you begin, make sure you have:

- **BIG-IP LTM knowledge**: You know its concepts and its Traffic Management Shell (TMSH) commands.
- **NGINX knowledge**: You can read NGINX configuration. The guide links to the documentation of each directive.
- **NGINX Plus**: For the steps, see [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}}).

All IP addresses and object names in this guide are examples. Replace them with the values of your BIG-IP LTM configuration.

---

<span id="mapping"></span><span id="networking"></span>

## Map the BIG-IP LTM networking concepts

<span id="mapping-network"></span>

### Network architecture

Don't translate BIG-IP LTM commands one to one. The two products handle networking differently.

BIG-IP LTM separates the management network (management plane) from the application traffic network (data plane). The management network is usually isolated, while the traffic network faces clients and servers. BIG-IP LTM can also use a single NIC for the client and server networks, which is common in cloud environments.

NGINX Plus uses the networking of its Linux host. It can bind several IP subnets or VLANs to one NIC, or use several NICs for separate client and server networks.

<span id="mapping-concepts"></span>

### Networking concepts

{{< table >}}

| BIG-IP LTM concept | NGINX Plus equivalent |
|---|---|
| **Self IP address**: the address on a VLAN where BIG-IP LTM receives data plane traffic | The host interface that NGINX Plus listens on. NGINX Plus uses the networking of the operating system, so it needs no separate self IP address. |
| **Management IP address and port**: the addresses for the GUI and SSH | The IP address of the Linux host. A separate management address is possible, but not needed. |
| **Virtual server**: the IP address and port that clients connect to | A [`server`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) block. Its [`listen`](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) directive sets the IP address and port. |
| **Pool and node list**: the back-end servers of an application, and the servers themselves | An [`upstream`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) block. It lists the servers, and it also sets the load balancing and session persistence methods. |
| **iRules**: Tcl scripts for content switching and traffic changes | `server` and `location` blocks for most content switching and header changes. For more complex logic, [njs](https://nginx.org/en/docs/njs/) or Lua. |
| **High availability**: a floating virtual IP address (VIP) for an active-passive pair, with built-in failover | The `nginx-ha-keepalived` package, which moves a VIP with the VRRP protocol, for [on-premises deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}), and [active-active pairs]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived-nodes.md" >}}). For cloud environments, see the AWS ([with a Network Load Balancer]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-network-load-balancer.md" >}}) or [with Elastic IP addresses]({{< ref "/nginx/deployment-guides/amazon-web-services/high-availability-keepalived.md" >}})), [Google Cloud]({{< ref "/nginx/deployment-guides/google-cloud-platform/high-availability-all-active.md" >}}), and [Azure]({{< ref "/nginx/deployment-guides/microsoft-azure/high-availability-standard-load-balancer.md" >}}) guides. |

{{< /table >}}

---

<span id="converting"></span><span id="convert-configuration"></span>

## Convert the load balancer configuration

BIG-IP LTM has three ways to change its configuration: the GUI, TMSH, and the iControl API. All of them become TMSH commands, so this guide shows TMSH. The commands assume that you're in the `(tmos.ltm)` module, so they leave out `ltm`.

NGINX Plus keeps its configuration in text files. You can edit them directly, or manage them with tools such as the [Ansible role for NGINX](https://github.com/nginx/ansible-role-nginx).

The examples identify virtual servers by IP address only. NGINX Plus can also choose a `server` block by the `Host` header: add a [`server_name`](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) directive with host names, wildcards, or regular expressions. For details, see [Server names](https://nginx.org/en/docs/http/server_names.html).

<span id="virtual-servers"></span>

### Virtual servers

A virtual server at 192.168.10.10 listens on port 80, and load balances requests to the two servers of the `test_pool` group.

BIG-IP LTM:

```none
# create pool test_pool members add { 10.10.10.10:80 10.10.10.20:80 }
# create virtual test_virtual { destination 192.168.10.10:80 pool test_pool source-address-translation { type automap } ip-protocol tcp profiles add { http } }
# save sys config
```

NGINX Plus:

```nginx
http {
    upstream test_pool {
        server 10.10.10.10:80;
        server 10.10.10.20:80;
    }

    server {
        listen 192.168.10.10:80;

        location / {
            proxy_pass http://test_pool;
        }
    }
}
```

The `listen` address must exist on the NGINX Plus host. Otherwise, `nginx -t` fails with `bind() to 192.168.10.10:80 failed (99: Cannot assign requested address)`. NGINX Plus connects to the servers from its own address, like the BIG-IP LTM `automap` source address translation.

<span id="ssl-offload"></span><span id="ssl"></span>

### SSL/TLS offload

BIG-IP LTM attaches a profile for each certificate and key to a virtual server. A client profile encrypts the traffic to clients, and a server profile the traffic to the servers. NGINX Plus sets the certificate and key in the `server` block. It uses the OpenSSL library of the operating system.

A load balancer handles TLS traffic in one of two ways:

- **Termination**: The load balancer decrypts the client traffic, and sends it to the servers unencrypted. Use it when the network to the servers is secure.
- **Proxy**: The load balancer decrypts the client traffic, and encrypts it again for the servers. Use it when the network to the servers isn't secure.

BIG-IP LTM, the virtual server and the pool for TLS:

```none
# create pool ssl_test_pool members add { 10.10.10.10:443 10.10.10.20:443 }
# create virtual test_ssl_virtual { destination 192.168.10.10:443 pool ssl_test_pool source-address-translation { type automap } ip-protocol tcp profiles add { http } }
# save /sys config
```

BIG-IP LTM, termination with a client SSL profile:

```none
# create profile client-ssl test_ssl_client_profile cert test.crt key test.key
# modify virtual test_ssl_virtual profiles add { test_ssl_client_profile }
# save /sys config
```

BIG-IP LTM, proxy with a server SSL profile:

```none
# create profile server-ssl test_ssl_server_profile cert test.crt key test.key
# modify virtual test_ssl_virtual profiles add { test_ssl_server_profile }
# save /sys config
```

NGINX Plus, termination:

```nginx
upstream ssl_test_pool {
    server 10.10.10.10:443;
    server 10.10.10.20:443;
}

server {
    listen 192.168.10.10:443 ssl;
    ssl_certificate     /etc/nginx/ssl/test.crt;
    ssl_certificate_key /etc/nginx/ssl/test.key;

    location / {
        proxy_pass http://ssl_test_pool;
    }
}
```

NGINX Plus, proxy, with verification of the server certificates:

```nginx
upstream ssl_test_pool {
    server 10.10.10.10:443;
}

server {
    listen 192.168.10.10:443 ssl;
    ssl_certificate     /etc/nginx/ssl/test.crt;
    ssl_certificate_key /etc/nginx/ssl/test.key;

    location / {
        proxy_pass https://ssl_test_pool;
        proxy_ssl_certificate         /etc/nginx/ssl/client.pem;
        proxy_ssl_certificate_key     /etc/nginx/ssl/client.key;
        proxy_ssl_protocols           TLSv1.2 TLSv1.3;
        proxy_ssl_ciphers             HIGH:!aNULL:!MD5;
        proxy_ssl_trusted_certificate /etc/nginx/ssl/trusted_ca_cert.crt;
        proxy_ssl_verify              on;
        proxy_ssl_verify_depth        2;
        proxy_ssl_name                app.example.com;
        proxy_ssl_server_name         on;
    }
}
```

[`proxy_ssl_name`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_name) sets the name that the server certificates must contain. Replace `app.example.com` with that name. By default, NGINX Plus checks them against the name in `proxy_pass`. Here, that's the `upstream` name `ssl_test_pool`, so the check fails. [`proxy_ssl_server_name`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_server_name) sends the same name to the servers with SNI.

<span id="session-persistence"></span>

### Session persistence

BIG-IP LTM and NGINX Plus both set session persistence, also called affinity, on the server group. That's the BIG-IP LTM pool, or the NGINX Plus `upstream` block. Stateful applications need it. The [`sticky`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky) directive is also in NGINX Open Source since version 1.29.6.

#### Cookie persistence

The `sticky cookie` method works like the cookie insert method of BIG-IP LTM. The load balancer adds a cookie that names the server, and the client sends it back with later requests. If that server fails, the load balancer chooses another one.

BIG-IP LTM, for HTTP and for HTTPS:

```none
# create persistence cookie test_bigip_cookie cookie-name BIGIP_COOKIE_PERSIST expiration 1:0:0
# modify virtual test_virtual { persist replace-all-with { test_bigip_cookie } }
# save /sys config
```

```none
# create persistence cookie test_bigip_cookie cookie-name BIGIP_COOKIE_PERSIST expiration 1:0:0
# modify virtual test_ssl_virtual { persist replace-all-with { test_bigip_cookie } }
# save /sys config
```

NGINX Plus, for HTTP and for HTTPS:

```nginx
upstream test_pool {
    server 10.10.10.10:80;
    server 10.10.10.20:80;
    sticky cookie mysession expires=1h;
}
```

```nginx
upstream ssl_test_pool {
    server 10.10.10.10:443;
    server 10.10.10.20:443;
    sticky cookie mysession expires=1h;
}
```

#### Source address persistence

The load balancer computes a hash of the client IP address, and sends each request to the server for that hash. The [`ip_hash`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) method of NGINX uses the first three octets of an IPv4 address, or the whole IPv6 address. So all clients of a `/24` IPv4 network go to the same server.

BIG-IP LTM:

```none
# modify virtual test_virtual { persist replace-all-with {source_addr} }
# save /sys config
```

NGINX Plus:

```nginx
upstream test_pool {
    ip_hash;
    server 10.10.10.10:80;
    server 10.10.10.20:80;
}
```

#### Token persistence

The `sticky learn` method uses a cookie or another token that the server creates, such as `jsessionid`. NGINX Plus keeps a table in shared memory that maps each token value to a server. BIG-IP LTM needs an iRule for this, which is outside the scope of this guide.

NGINX Plus:

```nginx
upstream test_pool {
    server 10.10.10.10:80;
    server 10.10.10.20:80;
    sticky learn create=$upstream_cookie_jsessionid
                 lookup=$cookie_jsessionid
                 zone=client_sessions:1m;
}
```

<span id="keepalive-connections"></span>

### Keepalive connections

Opening a new connection to the server for each request costs time and resources. BIG-IP LTM reuses server connections with a OneConnect profile. NGINX Plus keeps a cache of idle connections to each server group with the [`keepalive`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive) directive.

BIG-IP LTM:

```none
# modify virtual test_virtual profiles add { oneconnect }
# modify virtual test_ssl_virtual profiles add { oneconnect }
# save /sys config
```

NGINX Plus:

```nginx
upstream test_pool {
    server 10.10.10.10:80;
    server 10.10.10.20:80;
    keepalive 32;
}

server {
    listen 192.168.10.10:80;

    location / {
        proxy_pass http://test_pool;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

Before NGINX 1.29.7, the proxy needs HTTP/1.1 and an empty `Connection` header to keep connections open, as in this example. Since 1.29.7, these are the defaults. An `upstream` block without a `keepalive` directive then also keeps up to 32 idle connections in each worker process. It reuses them only for the location that opened them, as with `keepalive 32 local;`. To turn that off, use `keepalive 0;`.

<span id="health-checks"></span>

### Monitors and health checks

BIG-IP LTM calls them monitors, and NGINX Plus calls them [health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}}). BIG-IP LTM attaches a monitor to a pool, and it checks each node. NGINX Plus sets the [`health_check`](https://nginx.org/en/docs/http/ngx_http_upstream_hc_module.html#health_check) directive in the `location` that proxies to the group, and checks each server of the group. Active health checks are an NGINX Plus feature.

In this example, both products check the servers every 5 seconds, the default of NGINX Plus. NGINX Plus has no separate timeout for the check. With `fails=2`, it marks a server as unhealthy after two failed checks.

BIG-IP LTM, for HTTP and HTTPS:

```none
# create monitor http test_monitor defaults-from http send "GET /index.html HTTP/1.0\r\n\r\n" interval 5 timeout 20
# modify pool test_pool monitor test_monitor
# create monitor https test_ssl_monitor defaults-from https send "GET /index.html HTTP/1.0\r\n\r\n" interval 5 timeout 20
# modify pool ssl_test_pool monitor test_ssl_monitor
# save /sys config
```

NGINX Plus, for HTTP and HTTPS:

```nginx
upstream test_pool {
    # ...
    zone test_pool_zone 64k;
}

server {
    # ...
    location / {
        proxy_pass   http://test_pool;
        health_check interval=5 fails=2;
    }
}
```

The `zone` directive is needed: health checks keep the state of the servers in shared memory.

---

<span id="summary"></span><span id="configuration-summary"></span>

## Review the complete configuration

These are the BIG-IP LTM commands for a basic environment, and the equivalent NGINX Plus configuration.

BIG-IP LTM:

```none
# create pool test_pool members add { 10.10.10.10:80 10.10.10.20:80 }
# create virtual test_virtual { destination 192.168.10.10:80 pool test_pool source-address-translation { type automap } ip-protocol tcp profiles add { http } }
# create pool ssl_test_pool members add { 10.10.10.10:443 10.10.10.20:443 }
# create virtual test_ssl_virtual { destination 192.168.10.10:443 pool ssl_test_pool source-address-translation { type automap } ip-protocol tcp profiles add { http } }
# create profile client-ssl test_ssl_client_profile cert test.crt key test.key
# modify virtual test_ssl_virtual profiles add { test_ssl_client_profile }
# create profile server-ssl test_ssl_server_profile cert test.crt key test.key
# modify virtual test_ssl_virtual profiles add { test_ssl_server_profile }
# create persistence cookie test_bigip_cookie cookie-name BIGIP_COOKIE_PERSIST expiration 1:0:0
# modify virtual test_virtual { persist replace-all-with { test_bigip_cookie } }
# modify virtual test_ssl_virtual { persist replace-all-with { test_bigip_cookie } }
# modify virtual test_virtual profiles add { oneconnect }
# modify virtual test_ssl_virtual profiles add { oneconnect }
# create monitor http test_monitor defaults-from http send "GET /index.html HTTP/1.0\r\n\r\n" interval 5 timeout 20
# modify pool test_pool monitor test_monitor
# create monitor https test_ssl_monitor defaults-from https send "GET /index.html HTTP/1.0\r\n\r\n" interval 5 timeout 20
# modify pool ssl_test_pool monitor test_ssl_monitor
# save /sys config
```

NGINX Plus:

```nginx
upstream test_pool {
    zone test_pool_zone 64k;
    server 10.10.10.10:80;
    server 10.10.10.20:80;
    sticky cookie mysession expires=1h;
    keepalive 32;
}

upstream ssl_test_pool {
    zone ssl_test_pool_zone 64k;
    server 10.10.10.10:443;
    server 10.10.10.20:443;
    sticky cookie mysession expires=1h;
    keepalive 32;
}

server {
    listen 192.168.10.10:80 default_server;

    location / {
        proxy_pass         http://test_pool;
        health_check;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   Connection "";
    }

    location ~ /favicon.ico {
        root /usr/share/nginx/images;
    }
}

server {
    listen 192.168.10.10:443 ssl default_server;
    ssl_certificate     /etc/nginx/ssl/test.crt;
    ssl_certificate_key /etc/nginx/ssl/test.key;

    location / {
        proxy_pass         https://ssl_test_pool;
        proxy_http_version 1.1;
        proxy_set_header   Host $host;
        proxy_set_header   Connection "";
        health_check;
    }

    location ~ /favicon.ico {
        root /usr/share/nginx/images;
    }
}

server {
    listen 8080;
    status_zone status-page;
    root /usr/share/nginx/html;

    location /api {
        api write=on;
        # directives that control access, such as 'allow' and 'deny'
    }

    location = /dashboard.html {
        root /usr/share/nginx/html;
    }

    # Redirect requests for the old (pre-R14) dashboard
    location = /status.html {
        return 301 /dashboard.html;
    }

    location ~ /favicon.ico {
        root /usr/share/nginx/images;
    }
}
```

Each `location /` block sets both `proxy_set_header` directives. A `location` that sets any `proxy_set_header` directive doesn't inherit the ones from its `server` block.

The `server` block on port 8080 turns on the NGINX Plus API and the [dashboard]({{< ref "/nginx/admin-guide/monitoring/live-activity-monitoring.md" >}}). Allow only trusted clients to reach it, because `write=on` lets the API change the configuration.

---

## Troubleshooting

### nginx -t can't bind to the virtual server address

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] bind() to 192.168.10.10:80 failed (99: Cannot assign requested address)`.

**Cause**: The `listen` directive uses an IP address that the NGINX Plus host doesn't have.

**Fix**: Add the address to an interface of the host, or change the `listen` directive to an address of the host.

---

### Proxied TLS requests return 502 Bad Gateway

**Symptom**: Requests to the TLS virtual server return `502 Bad Gateway`. The error log shows a message such as `upstream SSL certificate does not match "ssl_test_pool" while SSL handshaking to upstream`.

**Cause**: `proxy_ssl_verify on` checks the server certificates against the name in `proxy_pass`, the `upstream` name, which the certificates don't contain.

**Fix**: Set `proxy_ssl_name` to the name in the server certificates.

---

## References

For more information, see:

- [HTTP load balancing]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md" >}})
- [Securing HTTP traffic to upstream servers]({{< ref "/nginx/admin-guide/security-controls/securing-http-traffic-upstream.md" >}})
- [HTTP health checks]({{< ref "/nginx/admin-guide/load-balancer/http-health-check.md" >}})
- [Migrating load balancer configuration from Citrix ADC to NGINX Plus]({{< ref "/nginx/deployment-guides/migrate-hardware-adc/citrix-adc-configuration.md" >}})

---

## Revision history

- Version 3 (November 2026): Rewritten to the how-to template. Adds `proxy_ssl_name` to the TLS proxy example, the `ip_hash` key, and the keepalive defaults of NGINX 1.29.7.
- Version 2 (April 2018): Updated information about high availability and the NGINX Plus API (NGINX Plus R13, NGINX Open Source 1.13.4)
- Version 1 (February 2017): Initial version (NGINX Plus R11, NGINX Open Source 1.11.5)
