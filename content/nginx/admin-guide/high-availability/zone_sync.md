---
description: Share the runtime state of F5 NGINX Plus nodes in a cluster, such as sticky learn sessions, rate limits, and key-value pairs, with zone synchronization.
title: Runtime State Sharing in a Cluster
toc: true
weight: 400
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "zone synchronization, zone_sync, cluster, shared memory zone, sticky learn, limit_req_zone, keyval_zone, sync, NGINX Plus API"
f5-summary: >
  This guide shows how to share runtime state between the F5 NGINX Plus nodes of a cluster with the zone_sync module.
  The nodes can share the sessions of sticky learn, the state of rate limits, and the key-value store.
  It also covers TLS for the cluster traffic, adding and removing nodes, and checking the state of the cluster with the NGINX Plus API.
f5-audience: operator
---

<span id="intro"></span><span id="introduction"></span>

## Overview

The F5 NGINX Plus nodes of a cluster can share the contents of these shared memory zones:

- The sessions of [sticky learn]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md#sticky" >}}) session persistence
- The state of [request rate limits]({{< ref "/nginx/admin-guide/security-controls/controlling-access-proxied-http.md#limit_req" >}})
- The [key-value store](https://nginx.org/en/docs/http/ngx_http_keyval_module.html#keyval_zone)

The [`zone_sync`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html) module sends the changes of these zones to the other nodes, over TCP connections in the `stream` context. Each node exchanges state with all other nodes, for each zone that has the same name on all of them. For how the synchronization works, see [How NGINX Plus Performs Zone Synchronization]({{< ref "/nginx/admin-guide/high-availability/zone_sync_details.md" >}}).

The shared state is eventually consistent. Latency, low bandwidth, and packet loss make it less consistent. Use data center networks for the cluster traffic. Don't stretch a cluster over the internet, or across regions or availability zones.

---

<span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- **NGINX Plus R15 or later**, to share sticky learn sessions. To share rate limits and the key-value store, you need NGINX Plus R16 or later. For details, see the [release notes]({{< ref "/nginx/releases.md#r16" >}}).
- **A port for the cluster traffic**: Every node must reach the other nodes on it. This guide uses port 9000.
- **The same zone names on all nodes**: A zone is shared only with the zones of the same name on the other nodes.

---

<span id="config"></span><span id="configuring-zone-synchronization"></span>

## Set up zone synchronization

Repeat these steps on every node of the cluster. Use the same configuration on all nodes.

1. In the top-level [`stream`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#stream) block, add a [`server`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#server) that [listens](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen) on the cluster port, and turn on synchronization with the [`zone_sync`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync) directive. Then list the nodes of the cluster with [`zone_sync_server`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_server) directives, one for each node:

    ```nginx
    stream {
        server {
            listen 9000;

            zone_sync;
            zone_sync_server nginx-node1.example.com:9000;
            zone_sync_server nginx-node2.example.com:9000;
            zone_sync_server nginx-node3.example.com:9000;
        }
    }
    ```

    Instead, you can find the nodes with DNS. Use one `zone_sync_server` directive with the `resolve` parameter, and a [`resolver`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#resolver) in the `stream` block. NGINX Plus then follows the addresses of the DNS name, so you can add and remove nodes without a reload:

    ```nginx
    stream {
        resolver 10.0.0.53 valid=20s;

        server {
            listen 9000;

            zone_sync;
            zone_sync_server nginx-cluster.example.com:9000 resolve;
        }
    }
    ```

1. Test the configuration, and reload NGINX Plus:

    ```shell
    sudo nginx -t && sudo nginx -s reload
    ```

1. Share the zones that you need, as described in [Share zones](#sync).

### Protect the cluster traffic with TLS

To encrypt the cluster traffic, and to check the certificates of the nodes, add these directives to the `server` block of each node:

- On the listening side, the `ssl` parameter of [`listen`](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#listen), and the certificate and key of the node in PEM format, with [`ssl_certificate`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate) and [`ssl_certificate_key`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_certificate_key).
- On the connecting side, [`zone_sync_ssl`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl) to use TLS for the connections to other nodes, and [`zone_sync_ssl_verify`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl_verify) with [`zone_sync_ssl_trusted_certificate`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl_trusted_certificate) to check their certificates.
- Optionally, certificate-based authentication between the nodes: [`zone_sync_ssl_certificate`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl_certificate) and [`zone_sync_ssl_certificate_key`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl_certificate_key) send a client certificate on outgoing connections. [`ssl_verify_client`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_verify_client) and [`ssl_trusted_certificate`](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html#ssl_trusted_certificate) require a client certificate on incoming connections.

With all of them, the `server` block of the first node is:

```nginx
stream {
    resolver 10.0.0.53 valid=20s;

    server {
        listen 10.0.0.1:9000 ssl;

        ssl_certificate     /etc/ssl/nginx-1.example.com.server_cert.pem;
        ssl_certificate_key /etc/ssl/nginx-1.example.com.key.pem;

        zone_sync;
        zone_sync_server nginx-cluster.example.com:9000 resolve;

        zone_sync_ssl                     on;
        zone_sync_ssl_verify              on;
        zone_sync_ssl_trusted_certificate /etc/ssl/server_ca.pem;
        zone_sync_ssl_name                nginx-cluster.example.com;

        zone_sync_ssl_certificate     /etc/ssl/nginx-1.example.com.client_cert.pem;
        zone_sync_ssl_certificate_key /etc/ssl/nginx-1.example.com.client_key.pem;

        ssl_verify_client       on;
        ssl_trusted_certificate /etc/ssl/client_ca.pem;
    }
}
```

With `zone_sync_ssl_verify`, each node checks the certificates of the other nodes against a name. By default, this is the host part of the `zone_sync_server` address. With the `resolve` parameter, it's the resolved IP address of each node. So the example sets [`zone_sync_ssl_name`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_ssl_name) to the cluster name. Each node certificate must then contain that name, for example as a subject alternative name.

<span id="config_finetune"></span><span id="fine-tuning-synchronization"></span>

### Fine-tune the synchronization

You usually don't need to change these settings. They go in the same `server` block:

{{< table >}}

| Directive | Default | Description |
|---|---|---|
| [`zone_sync_buffers`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_buffers) | `8 4k` or `8 8k` | The number and size of the buffers of each zone for pushing its contents. The size is one memory page by default. One buffer must be large enough for any entry of each zone. |
| [`zone_sync_connect_retry_interval`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_connect_retry_interval) | `1s` | The interval between connection attempts to another node. |
| [`zone_sync_connect_timeout`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_connect_timeout) | `5s` | The timeout for establishing a connection with another node. |
| [`zone_sync_interval`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_interval) | `1s` | The interval for polling updates in a shared memory zone. A higher value can make the nodes less consistent. A lower value uses more CPU and memory. |
| [`zone_sync_recv_buffer_size`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_recv_buffer_size) | The size of one `zone_sync_buffers` buffer times their number | The size of the receive buffer of each connection, for parsing the incoming messages. It must be at least as large as one buffer of `zone_sync_buffers`. |
| [`zone_sync_timeout`](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html#zone_sync_timeout) | `5s` | The timeout between two successive reads or writes on a connection to another node. If no data moves within this time, NGINX Plus closes the connection. |

{{< /table >}}

---

<span id="manage"></span><span id="managing-cluster-nodes"></span>

## Manage the nodes of the cluster

<span id="manage_start"></span><span id="starting-a-node"></span>

### Add a node

- With DNS, add the IP address of the new node to the DNS record of the cluster. Then start NGINX Plus on the new node.
- With a static list, add the address of the new node to the `zone_sync_server` directives on all nodes. Then reload NGINX Plus on the other nodes.

The new node finds the other nodes from DNS or its configuration, and starts to send them updates. The other nodes find the new node through DNS, or after the reload, and push updates to it.

<span id="manage_stop"></span><span id="stopping-a-node"></span>

### Stop a node

To stop a node, send NGINX Plus the `QUIT` signal:

```shell
sudo nginx -s quit
```

The node finishes the zone synchronization, and closes its connections gracefully.

<span id="manage_remove"></span><span id="removing-a-node"></span>

### Remove a node

- With DNS, remove the IP address of the node from the DNS record of the cluster.
- With a static list, remove the address of the node from the `zone_sync_server` directives on all nodes. Then reload NGINX Plus on each node.

The other nodes close their connections to the removed node, and no longer connect to it. Then you can [stop the node](#manage_stop).

---

<span id="sync"></span><span id="using-synchronization-in-a-cluster"></span>

## Share zones

To share a zone, add the `sync` parameter to the directive that defines it, on every node. The zone must have the same name on all nodes.

<span id="sync_sticky_learn"></span><span id="sticky-learn-zone-synchronization"></span>

### Share sticky learn sessions

Add the [`sync`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky_learn_sync) parameter to the `sticky learn` directive:

```nginx
resolver 10.0.0.53 valid=20s;

upstream my_backend {
    zone my_backend 64k;

    server backends.example.com resolve;

    sticky learn zone=sessions:1m
                 create=$upstream_cookie_session
                 lookup=$cookie_session
                 sync;
}

server {
    listen 80;

    location / {
        proxy_pass http://my_backend;
    }
}
```

The `resolve` parameter of `server` needs a [`resolver`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver). For how sticky learn works, see [Enabling Session Persistence]({{< ref "/nginx/admin-guide/load-balancer/http-load-balancer.md#sticky" >}}).

<span id="sync_limit_req"></span><span id="request-limits-zone-synchronization"></span>

### Share rate limits

Add the [`sync`](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html#limit_req_zone) parameter to the `limit_req_zone` directive. The nodes then share the state of the zone. This example reuses the `my_backend` upstream group of the previous example:

```nginx
limit_req_zone $remote_addr zone=req:1m rate=100r/s sync;

server {
    listen 80;

    location / {
        limit_req zone=req;

        proxy_pass http://my_backend;
    }
}
```

For how rate limiting works, see [Limiting the Request Rate]({{< ref "/nginx/admin-guide/security-controls/controlling-access-proxied-http.md#limit_req" >}}).

<span id="sync_keyval"></span><span id="key-value-storage-zone-synchronization"></span>

### Share the key-value store

Add the `sync` parameter to the [`keyval_zone`](https://nginx.org/en/docs/http/ngx_http_keyval_module.html#keyval_zone) directive. Synchronization needs the `timeout` parameter of the zone too:

```nginx
keyval_zone zone=one:32k state=/var/lib/nginx/state/one.keyval timeout=1h sync;
keyval      $arg_text $text zone=one;

server {
    # ...
    location / {
        return 200 $text;
    }

    location /api {
        api write=on;
    }
}
```

With synchronization, deleting a key-value pair through the API removes it only on the node that receives the request. The other nodes remove the pair when its timeout expires. For how to manage the key-value store, see [Dynamic Denylisting of IP Addresses]({{< ref "/nginx/admin-guide/security-controls/denylisting-ip-addresses.md" >}}).

---

<span id="monitor"></span><span id="monitoring-cluster-state"></span>

## Check the state of the cluster

The [NGINX Plus API](https://nginx.org/en/docs/http/ngx_http_api_module.html#stream_zone_sync_) shows the state of the synchronization on each node:

- The names of the shared zones
- The number of records of each zone on the node
- The number of records that the node still needs to send
- The number of nodes that the node is connected to

<span id="monitor_api_config"></span><span id="configuring-the-api"></span>

### Turn on the NGINX Plus API

Add a `location` with the [`api`](https://nginx.org/en/docs/http/ngx_http_api_module.html#api) directive. [Restrict access]({{< ref "/nginx/admin-guide/security-controls/controlling-access-proxied-http.md" >}}) to it. This example allows only the local host, and asks for a password for methods other than `GET`, such as `PATCH`, `POST`, and `DELETE`:

```nginx
server {
    listen 80;
    server_name www.example.com;

    location /api {
        limit_except GET {
            auth_basic           "NGINX Plus API";
            auth_basic_user_file /etc/nginx/api.htpasswd;
        }

        api write=on;

        allow 127.0.0.1;
        deny  all;
    }
}
```

For more about the API, see [Using the API for Dynamic Configuration]({{< ref "/nginx/admin-guide/load-balancer/dynamic-configuration-api.md#api_use" >}}).

<span id="monitor_api_use"></span><span id="polling-sync-status-with-the-api"></span>

### Check the synchronization status

On a node, request the status of the synchronization from the API:

```shell
curl -s http://127.0.0.1/api/9/stream/zone_sync | jq
```

The output is similar to:

```json
{
  "zones" : {
    "zone1" : {
      "records_pending" : 2061,
      "records_total" : 260575
    },
    "zone2" : {
      "records_pending" : 0,
      "records_total" : 14749
    }
  },
  "status" : {
    "bytes_in" : 1364923761,
    "msgs_in" : 337236,
    "msgs_out" : 346717,
    "bytes_out" : 1402765472,
    "nodes_online" : 15
  }
}
```

[`records_total`](https://nginx.org/en/docs/http/ngx_http_api_module.html#def_nginx_stream_zone_sync_zone) is the number of records in a zone. `records_pending` is the number of records that the node still needs to send to the cluster. `nodes_online` is the number of other nodes that the node is connected to. The cluster is healthy when all nodes have about the same number of records, and few pending records.

---

## Troubleshooting

### nginx -t reports that no listen is defined

**Symptom**: `nginx -t` fails with `nginx: [emerg] no "listen" is defined for server`.

**Cause**: The `stream` `server` block with `zone_sync` has no `listen` directive.

**Fix**: Add `listen` with the cluster port, as described in [Set up zone synchronization](#config).

---

### nginx -t reports that no resolver is defined

**Symptom**: `nginx -t` fails with a message such as `nginx: [emerg] no resolver defined to resolve names at run time in upstream "my_backend"`.

**Cause**: A `server` directive in an `upstream` block has the `resolve` parameter, but no `resolver` is defined.

**Fix**: Add a [`resolver`](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) directive to the `http` block or the `upstream` block. For `zone_sync_server ... resolve`, add it to the `stream` block.

---

### nodes_online is lower than expected

**Symptom**: `nodes_online` in the API output is lower than the number of other nodes, and their records differ.

**Cause**: The node can't connect to some nodes. For example, a firewall blocks the cluster port, an address in `zone_sync_server` is wrong, or a TLS certificate check fails.

**Fix**: Check the error log of the node for failed connections to the other nodes. Then check the port, the addresses, and the certificates.

---

## References

For more information, see:

- [How NGINX Plus Performs Zone Synchronization]({{< ref "/nginx/admin-guide/high-availability/zone_sync_details.md" >}})
- [Module ngx_stream_zone_sync_module](https://nginx.org/en/docs/stream/ngx_stream_zone_sync_module.html)
- [High Availability Support for NGINX Plus in On-Premises Deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}})
