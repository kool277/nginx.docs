---
description: Extend an F5 NGINX Plus keepalived pair with a passive third node, or with a virtual IP address for each node so that all nodes handle traffic.
title: Configuring Active-Active High Availability and Additional Passive Nodes with keepalived
toc: true
weight: 200
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "high availability, active-active, passive node, keepalived, VRRP, virtual IP address, VIP, nginx-ha-keepalived, priority"
f5-summary: >
  This guide shows how to extend an active-passive F5 NGINX Plus pair that uses keepalived.
  You can add a passive node that takes over when the other nodes fail, give each node its own virtual IP address so that all nodes handle traffic, or combine both.
  It also shows how to configure NGINX Plus for virtual IP addresses that move between nodes.
f5-audience: operator
---

<span id="intro"></span><span id="introduction"></span>

## Overview

F5 NGINX Plus uses [keepalived](https://www.keepalived.org/) for high availability (HA). The basic setup is an active-passive pair, as described in [High Availability Support for NGINX Plus in On-Premises Deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}). This guide extends that pair:

- **An additional passive node** takes over the virtual IP address (VIP) when both other nodes are down.
- **Active-active HA** gives each node its own VIP, so that all nodes handle traffic at the same time.

You can combine both methods, and extend them to more nodes.

{{< call-out class="note" >}} In a public cloud, use a Layer 4 (TCP) load balancer of the cloud provider to spread traffic across the NGINX Plus nodes, instead of keepalived. {{< /call-out >}}

<span id="why_passive"></span><span id="why-add-a-passive-node"></span>

### When to add a passive node

A two-node pair might not meet your requirements for redundancy. A third node that takes over when both other nodes are down adds redundancy, and keeps the configuration simple. It also lets you take one node down for maintenance without losing redundancy.

<span id="why_active"></span><span id="why-configure-active-active-ha"></span>

### When to use active-active HA

In an active-active setup, two or more nodes handle traffic at the same time. Each node holds at least one VIP. To spread clients across the VIPs, use one of these methods:

- Give each service a DNS name that resolves to one of the VIPs.
- Use round-robin DNS to map one DNS name to all the VIPs.
- Put a Layer 3 load balancer, such as an edge load balancer of the data center, in front of the VIPs.

Active-active HA adds capacity, but if one node of a pair fails, the capacity drops by half. Plan enough capacity for the remaining nodes.

The NGINX Plus nodes don't share their configuration. They also keep their own runtime state, such as the sessions of `sticky learn`, unless you set up [zone synchronization]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}}). For session persistence, use methods that work the same on every node: [`sticky cookie`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky_cookie), [`sticky route`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky_route), or [`ip_hash`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash). To share the sessions of [`sticky learn`](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#sticky_learn), see [Sticky learn zone synchronization]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md#sticky-learn-zone-synchronization" >}}).

---

## Before you begin

Before you begin, make sure you have:

- **An active-passive pair**: Two NGINX Plus nodes with the **nginx-ha-keepalived** package, set up as described in [High Availability Support for NGINX Plus in On-Premises Deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}}).
- **For a passive node**: A third node with NGINX Plus, in the same subnet as the pair.
- **For active-active HA**: A free IP address in the subnet for each additional VIP.
- **VRRP allowed between all nodes**: VRRP uses IP protocol 112.
- **The same NGINX Plus configuration on all nodes**: All nodes need the same NGINX Plus configuration and TLS certificates. To keep them the same, see [Synchronizing NGINX Configuration in a Cluster]({{< ref "/nginx/admin-guide/high-availability/configuration-sharing.md" >}}).

In this guide, the nodes are 192.168.10.10 (primary), 192.168.10.11 (backup), and 192.168.10.12 (new node). The first VIP is 192.168.10.100.

The examples show only the `chk_nginx_service` script. The files that `nginx-ha-setup` creates also have the `chk_manual_failover` script and other settings. Keep them, and keep the script paths of your files, which depend on the operating system. For the meaning of each directive, see [The keepalived configuration]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md#the-keepalived-configuration" >}}).

---

<span id="conf_passive"></span><span id="configuring-keepalived-for-an-additional-passive-node"></span>

## Add a passive node

1. On the new node, install the **nginx-plus** and **nginx-ha-keepalived** packages.

1. Copy **/etc/keepalived/keepalived.conf** from the backup node to the same location on the new node.

1. On the new node, edit **keepalived.conf**:

    - Set `priority` in each `vrrp_instance` block lower than on every other node, for example `99`.
    - Set `unicast_src_ip` to the address of the new node.
    - List the addresses of both other nodes in the `unicast_peer` block.

    The file on the new node is now:

    ```none
    vrrp_script chk_nginx_service {
        script   "/usr/lib/keepalived/nginx-ha-check"
        interval 3
        weight   50
    }

    vrrp_instance VI_1 {
        interface         eth0
        state             BACKUP
        priority          99
        virtual_router_id 51
        advert_int        1
        accept
        unicast_src_ip    192.168.10.12

        unicast_peer {
            192.168.10.10
            192.168.10.11
        }

        virtual_ipaddress {
            192.168.10.100
        }

        track_script {
            chk_nginx_service
        }

        notify "/usr/lib/keepalived/nginx-ha-notify"
    }
    ```

1. On the other two nodes, add the address of the new node to the `unicast_peer` block. On the primary node, the block is now:

    ```none
    unicast_peer {
        192.168.10.11
        192.168.10.12
    }
    ```

1. Restart keepalived on all nodes:

    ```shell
    sudo systemctl restart keepalived
    ```

1. Test the new node. Stop NGINX Plus on the first two nodes:

    ```shell
    sudo systemctl stop nginx
    ```

    Within a few seconds, the new node holds the VIP:

    ```shell
    ip -4 -brief address show eth0
    ```

    ```none
    eth0             UP             192.168.10.12/24 192.168.10.100/32
    ```

    Then start NGINX Plus again on the first two nodes, with `sudo systemctl start nginx`. The primary node takes the VIP back.

---

<span id="conf_active"></span><span id="configuring-keepalived-for-active-active-ha"></span>

## Set up active-active HA

To send traffic to both nodes of a pair at the same time, add a second VIP, 192.168.10.101 in this guide. The backup node becomes the primary node for the new VIP, so that each node holds one VIP:

{{< table >}}

| VRRP instance | VIP | Priority on 192.168.10.10 | Priority on 192.168.10.11 |
|---|---|---|---|
| `VI_1` | 192.168.10.100 | 101 | 100 |
| `VI_2` | 192.168.10.101 | 99 | 100 |

{{< /table >}}

Each node adds the weight of its scripts while NGINX Plus is up. If NGINX Plus fails on one node, the other node holds both VIPs.

1. On the backup node (192.168.10.11), edit **keepalived.conf**:

    - Copy the whole `vrrp_instance VI_1` block, and paste it below the existing block.
    - In the copy, change the name to `VI_2`, and the `virtual_router_id` to `61` or another unused value.
    - In the copy, change the address in `virtual_ipaddress` to the new VIP.
    - In the copy, keep `priority` at `100`.

    The file on the backup node is now:

    ```none
    vrrp_script chk_nginx_service {
        script   "/usr/lib/keepalived/nginx-ha-check"
        interval 3
        weight   50
    }

    vrrp_instance VI_1 {
        interface         eth0
        state             BACKUP
        priority          100
        virtual_router_id 51
        advert_int        1
        accept
        unicast_src_ip    192.168.10.11

        unicast_peer {
            192.168.10.10
        }

        virtual_ipaddress {
            192.168.10.100
        }

        track_script {
            chk_nginx_service
        }

        notify "/usr/lib/keepalived/nginx-ha-notify"
    }

    vrrp_instance VI_2 {
        interface         eth0
        state             BACKUP
        priority          100
        virtual_router_id 61
        advert_int        1
        accept
        unicast_src_ip    192.168.10.11

        unicast_peer {
            192.168.10.10
        }

        virtual_ipaddress {
            192.168.10.101
        }

        track_script {
            chk_nginx_service
        }

        notify "/usr/lib/keepalived/nginx-ha-notify"
    }
    ```

1. On the primary node (192.168.10.10), make the same changes, but set `priority` in `VI_2` to `99`, lower than on the backup node.

1. Restart keepalived on both nodes:

    ```shell
    sudo systemctl restart keepalived
    ```

1. Check that each node holds one VIP:

    ```shell
    ip -4 -brief address show eth0
    ```

    The node 192.168.10.10 holds 192.168.10.100, and the node 192.168.10.11 holds 192.168.10.101.

---

<span id="nginx_for_active"></span><span id="configuring-nginx-plus-for-active-active-ha"></span>

## Configure NGINX Plus for active-active HA

When both nodes have VIPs, configure NGINX Plus for them. Choose one of these options:

- Each node handles at least one application, with its own VIP.
- Each node handles all applications on all VIPs.

<span id="configuring-nginx-plus-for-different-applications-on-each-node"></span>

### Serve different applications on each node

Each `server` block [listens](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) on the VIP of its application. In this example, application 1 is on 192.168.10.100, which node 192.168.10.10 holds, and application 2 is on 192.168.10.101, which node 192.168.10.11 holds. After a failover, the other node holds both VIPs and serves both applications.

```nginx
server {
    listen 80 default_server;
    return 444;
}

server {
    listen 192.168.10.100:80;

    location / {
        root /application1;
    }
}

server {
    listen 192.168.10.101:80;

    location / {
        root /application2;
    }
}
```

The first `server` block listens on all addresses of port 80. With it, NGINX binds only `*:80`, and chooses the `server` block by the address that each connection arrives on. Without it, NGINX binds each VIP. On a node that doesn't hold one of the VIPs, `nginx -t` and reloads then fail with `bind() to 192.168.10.101:80 failed (99: Cannot assign requested address)`. The default server answers requests to other addresses of the node by closing the connection.

<span id="configuring-nginx-plus-for-all-applications-on-all-nodes"></span>

### Serve all applications on all nodes

Each node listens on all addresses and serves every application. A failover moves a VIP to another node, so the DNS records don't need to change. Use round-robin DNS: add an `A` record for each VIP under the same fully qualified domain name (FQDN). Each response of the DNS server then lists all VIPs, in a changing order. For the steps, see the documentation of your DNS server.

```nginx
server {
    listen *:80;

    location /app1/ {
        alias /application1/;
    }

    location /app2/ {
        alias /application2/;
    }
}
```

Requests for **/app1/** get the files in **/application1**, and requests for **/app2/** get the files in **/application2**.

---

<span id="combine_methods"></span><span id="combining-and-expanding-methods"></span>

## Combine and extend the methods

You can combine an additional passive node with active-active HA, or extend active-active HA to any number of nodes.

<span id="configuring-all-active-ha-on-three-or-more-nodes"></span>

### Set up all-active HA on three or more nodes

Repeat the [steps for active-active HA](#conf_active) for each node. Each node is the primary node for one VIP, the second choice for another VIP, and the third choice for the third VIP:

{{< table >}}

| VRRP instance | VIP | Priority on 192.168.10.10 | Priority on 192.168.10.11 | Priority on 192.168.10.12 |
|---|---|---|---|---|
| `VI_1` | 192.168.10.100 | 101 | 99 | 100 |
| `VI_2` | 192.168.10.101 | 100 | 101 | 99 |
| `VI_3` | 192.168.10.102 | 99 | 100 | 101 |

{{< /table >}}

For example, if NGINX Plus fails on node 192.168.10.10, its VIP 192.168.10.100 moves to node 192.168.10.12, which has the next priority for `VI_1`. This is the file of node 192.168.10.10:

```none
vrrp_script chk_nginx_service {
    script   "/usr/lib/keepalived/nginx-ha-check"
    interval 3
    weight   50
}

vrrp_instance VI_1 {
    interface         eth0
    state             BACKUP
    priority          101
    virtual_router_id 51
    advert_int        1
    accept
    unicast_src_ip    192.168.10.10

    unicast_peer {
        192.168.10.11
        192.168.10.12
    }

    virtual_ipaddress {
        192.168.10.100
    }

    track_script {
        chk_nginx_service
    }

    notify "/usr/lib/keepalived/nginx-ha-notify"
}

vrrp_instance VI_2 {
    interface         eth0
    state             BACKUP
    priority          100
    virtual_router_id 61
    advert_int        1
    accept
    unicast_src_ip    192.168.10.10

    unicast_peer {
        192.168.10.11
        192.168.10.12
    }

    virtual_ipaddress {
        192.168.10.101
    }

    track_script {
        chk_nginx_service
    }

    notify "/usr/lib/keepalived/nginx-ha-notify"
}

vrrp_instance VI_3 {
    interface         eth0
    state             BACKUP
    priority          99
    virtual_router_id 71
    advert_int        1
    accept
    unicast_src_ip    192.168.10.10

    unicast_peer {
        192.168.10.11
        192.168.10.12
    }

    virtual_ipaddress {
        192.168.10.102
    }

    track_script {
        chk_nginx_service
    }

    notify "/usr/lib/keepalived/nginx-ha-notify"
}
```

The files of the other two nodes have their own address in `unicast_src_ip`, the other two addresses in `unicast_peer`, and their priorities from the table.

<span id="configuring-active-active-passive-ha"></span>

### Set up active-active-passive HA

This setup combines [an additional passive node](#conf_passive) with [active-active HA](#conf_active). Nodes 192.168.10.10 and 192.168.10.11 are the active pair from the previous sections, and node 192.168.10.12 is passive. This is the file of the passive node:

```none
vrrp_script chk_nginx_service {
    script   "/usr/lib/keepalived/nginx-ha-check"
    interval 3
    weight   50
}

vrrp_instance VI_1 {
    interface         eth0
    state             BACKUP
    priority          98
    virtual_router_id 51
    advert_int        1
    accept
    unicast_src_ip    192.168.10.12

    unicast_peer {
        192.168.10.10
        192.168.10.11
    }

    virtual_ipaddress {
        192.168.10.100
    }

    track_script {
        chk_nginx_service
    }

    notify "/usr/lib/keepalived/nginx-ha-notify"
}

vrrp_instance VI_2 {
    interface         eth0
    state             BACKUP
    priority          98
    virtual_router_id 61
    advert_int        1
    accept
    unicast_src_ip    192.168.10.12

    unicast_peer {
        192.168.10.10
        192.168.10.11
    }

    virtual_ipaddress {
        192.168.10.101
    }

    track_script {
        chk_nginx_service
    }

    notify "/usr/lib/keepalived/nginx-ha-notify"
}
```

The passive node has priority `98` for both VIPs, lower than every other node. If one active node fails, the other active node takes its VIP. The passive node takes over only when both active nodes are down.

On both active nodes, also add the address of the passive node to the `unicast_peer` block of each `vrrp_instance`. On node 192.168.10.10, both blocks are then:

```none
unicast_peer {
    192.168.10.11
    192.168.10.12
}
```

Without this step, the active nodes don't send advertisements to the passive node. The passive node then takes both VIPs, while the active nodes hold them too. Restart keepalived on all nodes after the changes.

---

## Troubleshooting

### nginx -t reports that it can't assign the requested address

**Symptom**: `nginx -t` or a reload fails with a message such as `nginx: [emerg] bind() to 192.168.10.101:80 failed (99: Cannot assign requested address)`.

**Cause**: The node doesn't hold that VIP, and no `server` block listens on all addresses of the port. So NGINX tries to bind the VIP itself.

**Fix**: Add a `server` block that listens on all addresses of the port, as described in [Serve different applications on each node](#configuring-nginx-plus-for-different-applications-on-each-node).

---

### The passive node takes a VIP while an active node is up

**Symptom**: When one active node fails, its VIP moves to the passive node instead of the other active node.

**Cause**: The passive node has the same priority for that VIP as the other active node. With equal priorities, VRRP gives the VIP to the node with the higher IP address ([RFC 5798, section 6.4.3](https://datatracker.ietf.org/doc/html/rfc5798#section-6.4.3)).

**Fix**: For each VIP, give the passive node a lower priority than every active node, and restart keepalived on the passive node.

---

## References

For more information, see:

- [High Availability Support for NGINX Plus in On-Premises Deployments]({{< ref "/nginx/admin-guide/high-availability/ha-keepalived.md" >}})
- [Synchronizing NGINX Configuration in a Cluster]({{< ref "/nginx/admin-guide/high-availability/configuration-sharing.md" >}})
- [Runtime State Sharing in a Cluster]({{< ref "/nginx/admin-guide/high-availability/zone_sync.md" >}})
- [The `listen` directive](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)
- [RFC 5798: Virtual Router Redundancy Protocol (VRRP) Version 3](https://datatracker.ietf.org/doc/html/rfc5798)
